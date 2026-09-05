# Module 5: The Lifecycle of a Query (From Input to UI)

Welcome to Module 5! In this module, we will track a single user query from the moment it is typed in the Web UI to the moment the AI's response is rendered back on the screen.

If your goal is to build your own Agentic Harness from scratch, this module is your ultimate blueprint. We will map out the exact algorithms, CS terminologies, and Cordis pipeline steps used by DeepSeek Harness (DSH).

---

## 1. The Trigger: Waking the Agent Loop

### Analogy: The Sleepy Assistant
Imagine you have an extremely smart assistant who is asleep at their desk. When you place a sticky note on their desk (a user message), they wake up, read the note, open their filing cabinet, think, take action, and finally talk back to you. When they have nothing left to do, they go back to sleep.

In DSH, this is the **Turn Flow**.

### Step-by-Step Breakdown
1.  **The Inbox (Queueing Theory):** The user types `"Find the bug in math.ts"` and hits Enter. This message doesn't go straight to the LLM. It enters the Agent's **Inbox**.
2.  **`turn/start`:** The `core/agent-loop` plugin notices there is a message in the Inbox. It wakes up. A "Turn" has begun.
3.  **Claiming Input:** The loop claims the next message from the Inbox.
4.  **`step/start`:** The agent is now actively preparing to make a single request to the LLM (one "Step").

---

## 2. Context Projection (Assembling the Memory)

The LLM does not inherently know what happened 5 minutes ago. DSH must build the memory for it.

### CS Terminology: Event Sourcing & Projection
*   **Event Sourcing:** Instead of saving the *current state* of a chat, DSH saves every single action that ever happened into an append-only array (The `SessionEvent` Log).
*   **Projection:** DSH runs a function (Algorithm) called `deriveMessages()` over the event log. It iterates through the massive array of raw events and *projects* (transforms) them into the strict `role: 'user' | 'assistant'` array that an LLM API requires.

### The Code Reality
DSH fires the `agent/pre-step` waterfall event. Here, plugins can look at the projected messages.
*   If a plugin notices the messages are 200,000 tokens long, it can rewrite the array to compress it.
*   If the plugin rejects the message outright, the loop closes the turn and goes back to sleep.

---

## 3. Prompt Assembly (Building the Brain)

Before we talk to the LLM, we need to inject its instructions (System Prompt) and its capabilities (Tool Schemas).

### CS Terminology: Dynamic Composition
Instead of hardcoding a massive prompt, DSH asks the Cordis pegboard to assemble it dynamically.
1.  **`ctx.systemPrompt`:** DSH asks all plugins: *"Does anyone have rules for the AI?"* The Filesystem plugin might inject: *"You are on a Linux machine."* The Subagent plugin might inject: *"You can spawn helpers."* DSH glues these together.
2.  **`ctx.tools.schemas()`:** DSH asks the tool registry to provide the JSON Schemas for every tool the agent is allowed to use. (See Module 3 for what this JSON looks like).

---

## 4. The LLM Handshake (The Request)

We have our Projected History, our System Prompt, and our Tool Schemas. It is time to make the network call.

### The Algorithm: Streaming & Serialization
1.  **`agent/request`:** DSH triggers this event. The `llm/llm` adapter plugin catches it.
2.  **The API Call:** The adapter translates the DSH internal data into the exact format OpenAI, DeepSeek, or Anthropic requires, and opens a network connection.
3.  **`llm/stream` (SSE - Server-Sent Events):** The LLM does not return the whole answer at once. It returns data chunk by chunk (tokens) as it thinks. DSH catches these chunks (`assistant/chunk`) and immediately broadcasts them. This is how the UI gets that fast "typing" effect!

---

## 5. Tool Dispatch (Taking Action)

The LLM stops streaming text. Instead, it streams a special chunk that says: *"I want to call the tool `grep` with argument `math.ts`."*

### The Guarded Pipeline (Middleware Pattern)
DSH does not blindly execute the tool. It pushes the tool call through the `dsh-tools` execution pipeline:

1.  **Parsing:** The JSON arguments are parsed. If the LLM made a typo, it fails here.
2.  **`tools/pre-execute` (The Bouncer):** A waterfall event fires. Does this agent have permission to run `grep`? If yes, it proceeds.
3.  **`tools/execute` (The Wrapper):** Another waterfall. A timeout policy wraps the execution. If the tool takes longer than 30 seconds, it is violently aborted (`AbortSignal`).
4.  **The Execution:** The `grep` function actually runs (spawning Ripgrep as we learned in Module 3).
5.  **`tools/post-execute` (The Inspector):** The tool returns a result. A plugin can look at the result and say, *"This result has API keys in it, block it!"*
6.  **`tool/result`:** The final, safe result is appended to the `SessionEvent` log.

Because the tool finished, DSH loops back to Step 1 (`step/end` -> new `step/start`) to tell the LLM the tool's result!

---

## 6. The UI Bridge (Rendering the Data)

How does a raw JSON tool result turn into a beautiful interface in the web browser? DSH uses a concept called **Render Intents**.

### The Problem: Protocol Coupling
If the backend hardcodes HTML or React components into the database, it becomes impossible to write a CLI (Command Line) version of the app later. The backend must be ignorant of the frontend.

### CS Terminology: The ViewModel / Presentation Layer
In DSH, the backend tools define how they *want* to be presented using a neutral JSON vocabulary.

*   When a tool is running (Pending), DSH calls `presentCall()`.
*   When a tool finishes, DSH calls `presentResult()`.

These functions return a `card`-tagged object (a Discriminated Union):
```json
{
  "card": "diff",
  "title": "Modified math.ts",
  "diffs": [{"path": "math.ts", "oldText": "x + 1", "newText": "x + 2"}]
}
```

The Web UI (running React/Vue) receives this JSON over a websocket. It looks at `"card": "diff"`, ignores the backend completely, and loads its own beautiful `DiffViewerComponent` to render the red/green lines. If you built a CLI, it would read the exact same JSON and print colored text to the terminal!

---

## 7. How to Build Your Own Harness (The Blueprint)

If you want to build your own Agentic Harness from scratch, follow this exact architectural blueprint:

1.  **The State Engine:** Build an Append-Only Event Log. Never mutate past history. Your state is a `List<Events>`.
2.  **The Plugin Bus:** Build a Middleware/Waterfall system (like Cordis). Core logic should be empty. Features should be injected by intercepting events.
3.  **The Tool Registry:** Build a registry that maps string names (`"grep"`) to safe, sandboxed execution functions. Implement strict JSON Schema validation before the function is allowed to run.
4.  **The Agent Loop (The Brain):**
    ```python
    # Pseudo-code for a Harness Brain
    while inbox.has_messages() or tools_are_pending():
        history = project_events_to_llm_format(event_log)
        prompt = assemble_system_prompts()
        schemas = get_tool_schemas()

        response = call_llm(history, prompt, schemas)

        if response.is_text:
            append_to_log(response.text)
            send_to_ui_stream(response.text)

        if response.wants_tool_call:
            if check_permissions(response.tool_name):
                result = execute_tool_safely(response.tool_name)
                append_to_log(result)
                # Loop restarts automatically to give result back to LLM!
    ```
5.  **The Presentation Layer:** Keep the backend dumb. The backend should only emit "Render Intents" (JSON objects describing data shapes like `diff` or `terminal`). Let the frontend decide how to paint it.

---
### Congratulations!
You now possess the complete architectural blueprint for DeepSeek Harness. You have traced a query from the user's keyboard, through the event-sourced log, into the LLM, through the secure tool execution pipeline, and back out to the React UI as a semantic View Model. You are officially an Agentic Engineer.