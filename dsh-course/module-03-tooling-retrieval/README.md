# Module 3: Tooling & The Retrieval Mechanism

Welcome to Module 3! We now know DSH uses Node.js, TypeScript, and the Cordis Plugin architecture. But how does an AI Agent actually read a massive, 70,000-file enterprise codebase?

---

## 1. The Context Window Bottleneck

### The Mathematical Problem
Large Language Models (LLMs) like GPT-4 or Claude have a "Context Window" (e.g., 128,000 tokens).
A typical enterprise codebase with 70,000 files contains *millions* of tokens.

**You cannot simply copy-paste the codebase into the LLM prompt.** If you try, the LLM will either crash with a "Context Window Exceeded" error, or it will suffer from "Lost in the Middle" syndrome (forgetting things placed in the middle of a massive prompt).

### The Solution: Tool-Augmented Retrieval
Instead of feeding the LLM the files, DSH gives the LLM **Tools** (like a steering wheel and pedals). The LLM is forced to *drive* through the codebase, actively querying and finding exactly what it needs.

---

## 2. The Search Engine: Ripgrep (`rg`)

To search 70,000 files in milliseconds, you cannot use pure JavaScript reading files one by one. DeepSeek Harness solves this by reaching outside of Node.js to use extremely optimized system binaries.

DeepSeek Harness packages a binary called **Ripgrep** (`@vscode/ripgrep`) inside the `dsh-tool-fs-search` plugin.

*   **What is it?** Ripgrep is a line-oriented search tool written in Rust. It is the fastest text search tool in the world. It automatically skips hidden files and files listed in `.gitignore`.
*   **How DSH uses it:** The AI doesn't know what Rust or Ripgrep is. The AI just calls a Cordis Tool named `grep`.

### Deep Dive: The Tool Schema
When DSH registers the `grep` tool, it sends a JSON Schema to the LLM that looks like this:

```json
{
  "name": "grep",
  "description": "Search file contents by regex pattern.",
  "parameters": {
    "type": "object",
    "properties": {
      "pattern": { "type": "string", "description": "The ripgrep regex." },
      "path": { "type": "string", "description": "Optional directory to search." }
    },
    "required": ["pattern"]
  }
}
```

When the LLM replies with `{"name": "grep", "arguments": {"pattern": "function login"}}`, DSH intercepts this.

---

## 3. Safe Execution (The `ctx.subprocess` Seam)

You cannot blindly let an LLM run shell commands on your computer. What if it hallucinates or is maliciously prompted to execute `rm -rf /` (delete everything)?

DSH uses an incredibly strict pipeline to protect the system. It uses a Cordis service called `ctx.subprocess`.

**Crucial Security Design:**
DSH deliberately does *not* give the LLM access to a real Bash shell (`/bin/sh`) for search. It uses `ctx.subprocess` to spawn the Ripgrep binary *directly*, explicitly injecting the arguments as an array.

```typescript
// SECURE: Spawning the binary directly with strict arguments.
// The OS treats the pattern purely as text, NOT as a shell command.
ctx.subprocess.spawn('rg', ['--json', '--files', pattern]);

// VULNERABLE (What DSH DOES NOT DO):
// If pattern was `"a" && rm -rf /`, the shell would execute the deletion!
exec(`rg --json --files ${pattern}`);
```
Because there is no shell interpreter, shell injection attacks are mathematically impossible.

---

## 4. Output Bounding & The Spill Store

What if Ripgrep finds 50,000 matches for the word "const"? If DSH returned all 50,000 matches to the LLM, the context window would instantly explode.

DSH implements strict **Output Bounding**.
If the configuration sets `grepMaxMatches: 250`:
1. Ripgrep returns 1,000 matches.
2. DSH takes the first 250 matches and formats them nicely for the LLM prompt.
3. **The Spill Backend:** What happens to the other 750 matches? DSH saves the *complete* 1,000 matches into an artifact file on the hard drive (e.g., `/tmp/dsh-spill-123.txt`).
4. DSH appends a note to the LLM's prompt: *"Showing 250 of 1000 matches. The complete result could not be saved inline. Read /tmp/dsh-spill-123.txt for the rest."*

This allows the AI to decide if it wants to use a `read_file` tool to inspect the spill artifact!

---

## 5. The Next Evolution: LSP (Language Server Protocol)

While Ripgrep is essentially "dumb" text searching (matching strings to strings), DSH is built to support Semantic searching using **LSP**.

*   **What is it?** Created by Microsoft, LSP is the engine that powers VSCode's "Go to Definition" and "Find All References".
*   **How it works:** Instead of searching for the word "tax", LSP builds an Abstract Syntax Tree (AST) of the code. It *understands* the code.
*   **The Protocol:** The Agent sends a JSON-RPC request to an LSP Server:
    ```json
    {
      "jsonrpc": "2.0",
      "method": "textDocument/definition",
      "params": {
        "textDocument": { "uri": "file:///src/billing.ts" },
        "position": { "line": 10, "character": 15 }
      }
    }
    ```
    The LSP Server responds with the exact file and line number where that function is defined.
*   Because of Cordis, DSH can simply plug an "LSP Service" onto the pegboard. The AI is given a `go_to_definition` tool, allowing it to navigate 70,000 files semantically, exactly like a human software engineer using an IDE.

---

## 6. Visualizing the Retrieval Flow

```text
======================================================================
               THE 70K FILE SEARCH PIPELINE
======================================================================

 [1. THE AI AGENT]
    |
    | "I need to find 'login'. I will call the 'grep' tool."
    | -> {"name": "grep", "arguments": {"pattern": "login"}}
    v
 [2. CORDIS TOOL REGISTRY]
    | Validates JSON schema.
    v
 [3. tools/pre-execute WATERFALL]
    | Security Plugin: "Agent has permission. APPROVED."
    v
 [4. dsh-tool-fs-search (The Plugin)]
    | Formats the array safely: ['rg', '--json', 'login']
    v
 [5. ctx.subprocess (The System Execution)]
    |   <---- (Spawns actual Ripgrep binary natively. NO SHELL INJECTION)
    v
 [6. THE 70,000 FILES ON DISK]
    |   ----> (Ripgrep searches millions of lines in milliseconds)
    v
 [7. OUTPUT BOUNDING]
    | Found 1,000 matches.
    | Truncating to 250 matches to protect the AI's Context Window.
    | ctx.spillStore -> Saves all 1,000 to /tmp/spill.txt
    v
 [8. THE RESULT]
    | Returns 250 formatted matches + Spill notification to the AI.

```

## Summary
How does DSH handle 70,000 files?
1. It **never** loads them into the prompt.
2. It translates LLM Tool calls into ultra-fast binary executions (Ripgrep) or semantic queries (LSP).
3. It uses array-based subprocess spawning to guarantee security against shell injections.
4. It relies on a "Spill Store" to strictly cap token usage without hiding data from the AI.

---
*Now that our AI can search 70,000 files, how does it keep track of its thoughts? How does it ask another AI for help? Proceed to the final chapter: [Module 4](../module-04-subagents/README.md).*