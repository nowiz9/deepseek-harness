# Module 3: Tooling & The Retrieval Mechanism

Welcome to Module 3! We now know DSH uses Node.js, TypeScript, and the Cordis Plugin architecture. But how does an AI Agent actually read a codebase with 70,000 files?

---

## 1. The Context Window Problem

### The Bottleneck
Large Language Models (LLMs) like GPT-4 or DeepSeek have a "Context Window". This is the maximum amount of text they can "hold in their head" at one time. Even if a model has a 128,000 token context window, a 70,000 file codebase contains *millions* of tokens.

**You cannot just copy-paste the codebase into the prompt.**

### The Solution: Tool-Augmented Retrieval
Instead of giving the LLM the files, we give the LLM **Tools** (like a steering wheel and pedals). The LLM is forced to *drive* through the codebase, finding exactly what it needs.

---

## 2. The Toolbox: How does DSH search?

To search 70,000 files, you need speed. You cannot use pure JavaScript to search massive disks—it's too slow. DeepSeek Harness solves this by reaching outside of Node.js to use extremely optimized system binaries.

### A. Ripgrep (`rg`) - The Engine of `grep`
DeepSeek Harness uses a tool called `dsh-tool-fs-search`. Inside this plugin, it packages a binary called **Ripgrep**.
*   **What is it?** Ripgrep is a line-oriented search tool written in Rust. It is famous for being the fastest text search tool in the world. It respects `.gitignore` files automatically.
*   **How DSH uses it:** The AI doesn't know what Rust is. The AI just calls a tool named `grep` with a parameter `"find function login"`. DSH translates this into a system command (using a service called `ctx.subprocess`), executes the fast Rust binary, captures the output, and hands it back to the AI.

### B. Glob Patterns (`glob`)
When you want to find files by name (e.g., "Find all `.ts` files in the `src` folder"), DSH uses `glob`.
*   A glob pattern looks like this: `src/**/*.ts`.
*   The LLM uses the `glob` tool. DSH runs Ripgrep under the hood with specific flags (`rg --files --glob`) to instantly list files.

### C. LSP (Language Server Protocol)
*(While not explicitly detailed in the basic search, LSP is the next evolution).*
*   **What is it?** Created by Microsoft, LSP is the engine that powers VSCode's "Go to Definition" and "Find All References".
*   **Why it matters:** Instead of "dumb" text searching, an AI can use LSP tools to say, "Find exactly where the function `calculateTax` is defined, ignoring comments." DSH's plugin architecture allows an LSP plugin to snap right onto the pegboard.

---

## 3. The Execution Pipeline (The Guardrails)

You cannot blindly let an LLM run shell commands on your computer. What if it accidentally hallucinates `rm -rf /` (delete everything)? DSH uses an incredibly strict pipeline to protect the system.

Here is what happens when the LLM says: *"I want to run `grep` to find 'password'."*

1.  **`tools/pre-execute` (The Guard):** The Cordis waterfall begins. A security plugin checks if the Agent is allowed to search files.
2.  **`ctx.subprocess` (The Sandboxed Execution):** DSH deliberately does *not* give the LLM access to a real Bash shell. It uses `ctx.subprocess` to launch *only* the Ripgrep binary (`rg`), explicitly injecting the arguments. There is no shell interpreter, which means shell injection attacks (like `grep "a" && rm -rf /`) are mathematically impossible.
3.  **Output Bounding (The Cap):** What if Ripgrep finds 500,000 matches? The AI's context window would explode! DSH strictly limits the output. It uses rules like `grepMaxMatches: 250`.
4.  **The Spill Backend:** If there are 1,000 matches, DSH gives the AI the first 250, and saves the rest to a "Spill Artifact" on disk. It tells the AI: *"I truncated this. If you need more, look at the spill file."*

---

## 4. Visualizing the Retrieval Flow

```text
======================================================================
               THE 70K FILE SEARCH PIPELINE
======================================================================

 [1. THE AI AGENT]
    |
    | "I need to find 'login' in the code. I will call the 'grep' tool."
    v
 [2. CORDIS TOOL REGISTRY]
    | Checks the schema. "Yes, 'grep' is a valid tool."
    v
 [3. tools/pre-execute WATERFALL]
    | Security Plugin: "Agent has permission. APPROVED."
    v
 [4. dsh-tool-fs-search (The Plugin)]
    | Formats the command safely: rg --json "login"
    v
 [5. ctx.subprocess (The System Execution)]
    |   <---- (Spawns actual Ripgrep binary written in Rust)
    v
 [6. THE 70,000 FILES ON DISK]
    |   ----> (Ripgrep searches millions of lines in milliseconds)
    v
 [7. OUTPUT BOUNDING]
    | Found 1,000 matches.
    | Truncating to 250 matches to protect the AI's Context Window.
    | Saving 750 matches to a Spill File.
    v
 [8. THE RESULT]
    | Returns structured JSON to the AI Agent.

```

## Summary
How does DSH handle 70,000 files?
1. It **never** loads them into the prompt.
2. It gives the AI **Tools** (`grep`, `glob`, `read_file`).
3. It relies on **Ultra-fast binaries** (like Ripgrep) under the hood.
4. It strictly **Caps and Bounds** the output so the AI doesn't choke on too much information.

---
*Now that our AI can search 70,000 files, how does it keep track of its thoughts? How does it ask another AI for help? Proceed to the final chapter: [Module 4](../module-04-subagents/README.md).*