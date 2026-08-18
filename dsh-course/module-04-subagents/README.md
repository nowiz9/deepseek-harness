# Module 4: Subagents & Context Management

Welcome to the final module! You now understand the tech stack, the Cordis plugin architecture, and how tools like `ripgrep` allow an AI to safely search 70,000 files.

But what happens when a task is too big for one AI? What happens if the AI reads 50 files and forgets what it was doing in the first one? This module tackles **Context Management**, the Agent Loop, and **Subagents**.

---

## 1. Context is King: The Append-Only Log

When you chat with ChatGPT, it seems like it remembers your conversation. In reality, LLMs have no memory. They are stateless math equations. Every time you send a new message, the *entire history* of the conversation must be sent back to the LLM.

In DeepSeek Harness (DSH), this history is managed by `core/session` and is called the **Session Log**.

### "Model-Visible Means Logged"
DSH operates on a strict invariant: **If the LLM sees it, it must be reconstructable from the log.**
*   Every user prompt, every tool call, every tool result, and every error is appended to this log as a `SessionEvent`.
*   **Why is this brilliant?** Because the log is append-only and durable. If the power goes out, or the Node.js server crashes, you can restart DSH, load the `SessionEvent` array from disk, and the AI will perfectly resume its thought process exactly where it left off.

### Deep Dive: What does a SessionEvent look like?
Here is a conceptual look at what DSH actually saves to memory/disk:

```json
[
  {
    "type": "user/message",
    "content": "Find the bug in login.ts"
  },
  {
    "type": "assistant/message",
    "content": "I will search for the login function.",
    "toolCalls": [
      { "id": "call_123", "name": "grep", "args": {"pattern": "function login"} }
    ]
  },
  {
    "type": "tool/result",
    "callId": "call_123",
    "result": { "isError": false, "content": "Line 42: function login(user) {" }
  }
]
```

---

## 2. The Turn Flow (The Agent Loop)

An AI doesn't just "think" endlessly. It operates in structured cycles controlled by the `core/agent-loop` plugin. This cycle is called the **Turn Flow**.

*   **A Step:** The LLM makes a request -> It calls a tool (`grep`) -> The tool executes -> The tool returns a result. That entire sequence is *one* step.
*   **A Turn:** A turn consists of zero or more steps. A Turn opens when the user gives a prompt. The Turn stays open while the AI takes multiple Steps (searching, reading, evaluating in a loop). The Turn only closes when the AI says, "I am done, here is the final answer."

### The Context Compression Waterfall:
Before every Step, Cordis fires the `agent/pre-step` waterfall event.
This is where plugins can look at the massive Session Log array and rewrite it *before* it is sent to the LLM. If the log has 500 tool results, a plugin can intercept `agent/pre-step`, take the oldest 400 results, compress them into a summary like *"I previously searched 50 files and found nothing"*, and only send that summary to the LLM. This is how DSH dynamically manages context windows!

---

## 3. Subagents: A Swarm of Intelligence

Sometimes, context compression isn't enough. If the main AI is trying to build a web server, it might hit a complex SQL database bug. Instead of trying to fix it itself and polluting its clean context window with 50 SQL file reads, it spawns a **Subagent**.

### What is a Subagent?
In DSH, a Subagent is literally just a Tool. The `subagent` plugin registers a tool named `subagent` or `run_agent`.

1.  The Main Agent calls the tool: `{"name": "subagent", "arguments": {"goal": "Fix the SQL bug in database.ts"}}`.
2.  DSH creates a *brand new* Agent in a *brand new, empty* Session Log.
3.  The Subagent works on the problem in a loop.
4.  When it finishes, it returns a final summary string back to the Main Agent's tool result.

### Deep Dive Code: The Isolation Boundary
Here is conceptually how DSH isolates agents using Cordis Scopes:

```typescript
// The Main Agent scope
const mainScope = ctx.isolate(['tools', 'sessions']);
mainScope.plugin(MainAgentBrain);
mainScope.set('tools', [ReadFileTool, SubagentTool]);

// Inside SubagentTool.execute():
async function executeSubagent(goal: string) {
    // 1. We create a completely new, isolated Cordis Scope!
    const subScope = ctx.isolate(['tools', 'sessions']);

    // 2. We give the Subagent DIFFERENT tools!
    // The main agent might not have Terminal access, but we give it to the subagent.
    subScope.set('tools', [ReadFileTool, WriteFileTool, TerminalExecuteTool]);

    // 3. We start a NEW Session Log
    const subSession = new SessionLog();
    subSession.append({ type: 'user', content: goal });

    // 4. Run the Agent Loop until it finishes
    const result = await runAgentLoop(subScope, subSession);

    // 5. Throw away the subScope, returning ONLY the summary string to the Main Agent.
    return result.summary;
}
```

### Why is this revolutionary?
1.  **Context Isolation:** The Main Agent doesn't have to read the 50 SQL files the Subagent read. The Main Agent's context window stays clean, holding only the "big picture".
2.  **Specialized Tooling:** Because of Cordis's Spatial Isolation, you can give the Subagent *different tools*. This minimizes the risk of the Main Agent accidentally running a dangerous command while it's supposed to be writing documentation.

---

## 4. Visualizing Subagent Communication

```text
======================================================================
               THE SUBAGENT HIERARCHY
======================================================================

[ MAIN AGENT ]
|  (Session Log A - 1000 tokens)
|  Goal: "Deploy Web Server"
|
|--- Step 1: Read README.md
|--- Step 2: "I found a SQL bug. I will spawn a subagent."
|            Calls tool: subagent("Fix database.ts")
|
+===================================================+
|               [ SUBAGENT ]                        |
|               (Session Log B - 0 tokens)          |
|               Goal: "Fix database.ts"             |
|                                                   |
|--- Step 1: Read database.ts                       |
|--- Step 2: Grep for 'SELECT'                      |
|--- Step 3: Edit database.ts                       |
|--- Step N: ... (Log B grows to 50,000 tokens) ... |
|                                                   |
|   (Returns: "Bug fixed on line 42.")              |
+===================================================+
|
|--- Step 3 (Main Agent resumes): "Excellent."
|  (Session Log A is now 1050 tokens! The 50,000 tokens were thrown away!)
```

## Summary
To build an enterprise-grade AI system:
1.  **State must be immutable.** DSH uses an Append-Only `SessionEvent` log to ensure perfect replayability and crash-recovery.
2.  **Context is managed through Turns and Steps.** Plugins intercept the `agent/pre-step` waterfall to compress context on the fly.
3.  **Subagents solve Context limits.** By spawning isolated Agents with their own specific goals and limited tools, the system can scale infinitely without overflowing the context window of any single LLM.

---

## Congratulations!
You have completed the DeepSeek Harness Masterclass. You now understand Node.js event loops, TypeScript interfaces, Cordis Inversion of Control, Ripgrep secure execution pipelines, Append-only logs, and Subagent delegation scopes. You are ready to dive into the codebase and start building!