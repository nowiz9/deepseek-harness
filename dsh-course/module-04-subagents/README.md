# Module 4: Subagents & Context Management

Welcome to the final module! You now understand the tech stack, the Cordis plugin architecture, and how tools like `ripgrep` allow an AI to safely search 70,000 files.

But what happens when a task is too big for one AI? What happens if the AI reads 50 files and forgets what it was doing in the first one? This module tackles **Context Management** and **Subagents**.

---

## 1. Context is King: The Append-Only Log

When you chat with ChatGPT, it seems like it remembers your conversation. In reality, LLMs have no memory. Every time you send a message, the *entire history* of the conversation must be sent back to the LLM.

In DeepSeek Harness (DSH), this history is called the **Session Log** (`ctx.sessions`).

### "Model-Visible Means Logged"
DSH has an ironclad rule: **If the LLM sees it, it must be in the log.**
*   Every prompt, every tool call, every tool result, every error is appended to this log as a `SessionEvent`.
*   **Why is this brilliant?** Because the log is append-only and durable. If the power goes out, or the Node.js server crashes, you can restart DSH, load the `SessionEvent` log, and the AI will perfectly resume its thought process exactly where it left off. It guarantees 100% replay fidelity.

---

## 2. The Turn Flow (The Agent Loop)

An AI doesn't just "think" endlessly. It operates in structured cycles called **Turns** and **Steps**.

*   **A Step:** The LLM makes a request -> It calls a tool (`grep`) -> The tool executes -> The tool returns a result. That is *one* step.
*   **A Turn:** A turn consists of zero or more steps. A Turn opens when the user gives a prompt ("Find a bug in login.ts"). The Turn stays open while the AI takes multiple Steps (searching, reading, evaluating). The Turn only closes when the AI says, "I am done, here is the answer."

### The Waterfall in Action:
Before every Step, the `agent/pre-step` waterfall event fires. This allows Cordis plugins to look at the Session Log and say: *"Wait, this log is getting too big. Let's summarize older messages before sending them to the LLM."* This is how DSH manages context windows dynamically!

---

## 3. Subagents: A Swarm of Intelligence

Sometimes, a task requires a specialist. If the main AI is trying to build a web server, it might hit a complex SQL database bug. Instead of trying to fix it itself, it spawns a **Subagent**.

### What is a Subagent?
In DSH, a Subagent is literally just a Tool. It is a tool called `subagent`.
*   The Main Agent calls the tool: `subagent(prompt: "Fix the SQL bug in database.ts")`.
*   DSH creates a *brand new* Agent in a *brand new* Session Log.
*   The Subagent is given tools. It works on the problem.
*   When it finishes, it returns a final summary back to the Main Agent.

### Why is this revolutionary?
1.  **Context Isolation:** The Main Agent doesn't have to read the 50 SQL files the Subagent read. The Main Agent's context window stays clean and focused on the "big picture".
2.  **Specialized Tooling:** You can spawn a Subagent and give it *different tools*. The Main Agent might only have "Read File" tools. The Subagent could be given "Terminal Execution" tools. This minimizes the risk of the Main Agent accidentally running a dangerous command.
3.  **Spatiotemporal Composability (Again!):** Because of Cordis, the Subagent is just a plugin. It can be running in the exact same Node.js process, or it could be delegated to an entirely different computer!

---

## 4. Visualizing Subagent Communication

```text
======================================================================
               THE SUBAGENT HIERARCHY
======================================================================

[ MAIN AGENT ]
|  (Session Log A)
|  Goal: "Deploy Web Server"
|  Context: Clean, high-level.
|
|--- Step 1: Read README.md
|--- Step 2: "I found a SQL bug. I will spawn a subagent."
|            Calls tool: subagent("Fix database.ts")
|
+===================================================+
|               [ SUBAGENT ]                        |
|               (Session Log B - Completely New)    |
|               Goal: "Fix database.ts"             |
|                                                   |
|--- Step 1: Read database.ts                       |
|--- Step 2: Grep for 'SELECT'                      |
|--- Step 3: Run SQL tests (Fails)                  |
|--- Step 4: Edit database.ts                       |
|--- Step 5: Run SQL tests (Passes)                 |
|                                                   |
|   (Returns: "Bug fixed on line 42.")              |
+===================================================+
|
|--- Step 3 (Main Agent resumes): "Excellent. Proceeding with deployment."

```

## Summary
To build an enterprise-grade AI system:
1.  **State must be immutable.** DSH uses an Append-Only `SessionEvent` log to ensure perfect replayability and crash-recovery.
2.  **Context is managed through Turns and Steps.** Plugins can intercept these steps to compress or modify context on the fly.
3.  **Subagents solve Context limits.** By spawning isolated Agents with their own specific goals and limited tools, the system can scale infinitely without overflowing the context window of any single LLM.

---

### Hands-on: Try it out! (Thought Experiment)
Imagine you are building an AI to write a Book.
*   If you use **One Agent**, by Chapter 10, the AI has read 50,000 words. Its context window is full. It forgets the name of the main character from Chapter 1.
*   If you use **Subagents**, you have a "Director Agent". The Director's only job is to hold the Outline. The Director spawns a Subagent for Chapter 1: *"Write Chapter 1 based on this paragraph."* The Subagent writes it, returns the final text, and *dies*. The Director saves the text to disk, and spawns a *new* Subagent for Chapter 2. The Director never runs out of memory!

---

## Congratulations!
You have completed the DeepSeek Harness Masterclass. You now understand Node.js, TypeScript, Plugin Architectures, Ripgrep search strategies, and Subagent delegation. You are ready to start building!