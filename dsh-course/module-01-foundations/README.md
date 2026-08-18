# Module 1: Foundations & The Tech Stack

Welcome to Module 1! To truly understand an advanced system like DeepSeek Harness (DSH), we must first deconstruct the exact foundational technologies it stands on. We will explore Node.js, TypeScript, and the Philosophy of Plugin Architectures. We won't just learn *what* they are, but *why* they beat the alternatives for this specific use case.

---

## 1. Node.js: The Engine of Concurrency

### What came before? (The Threaded Model)
Historically, backend programming languages like Java or Python handled multiple tasks using a **Threaded Model**.
If a Python server receives 100 requests to read a file, it creates 100 "Threads" (virtual mini-CPUs). Each thread asks the hard drive for the file and then *goes to sleep* (blocks) until the hard drive answers.
*   **The Problem:** Threads are heavy. If an AI agent wants to scan 10,000 files simultaneously, a threaded model would spawn 10,000 threads. The computer's memory would instantly fill up, and the operating system would crash from "Context Switching" (trying to juggle 10,000 paused tasks).

### Why does Node.js exist? (The Event Loop)
Created in 2009 by Ryan Dahl, Node.js took the ultra-fast V8 JavaScript engine from Google Chrome and added a C++ library called `libuv`. This introduced the **Single-Threaded Event Loop with Non-Blocking I/O**.

Instead of 10,000 threads, Node.js uses **ONE** thread.
*   **How it works:** When Node.js wants to read 10,000 files, it fires off 10,000 requests to the Operating System in a fraction of a second, saying: *"Here is a callback function. Run this function when the file is ready."*
*   Node.js doesn't wait. It immediately loops back to see if any previous files are ready.
*   The Operating System handles the hard drive access in the background, and as files finish loading, it drops the results into Node.js's "Event Queue", where the single thread processes them one by one.

### Deep Dive Example: Node.js vs Python

**Python (Blocking / Synchronous)**
```python
# The program STOPS at line 2 for 5 seconds. Line 3 cannot run.
file1 = open("giant_file_1.txt").read()
file2 = open("giant_file_2.txt").read()
print("Finished!")
```

**Node.js (Non-Blocking / Asynchronous)**
```javascript
const fs = require('fs');

// The program fires off the request and immediately moves to line 5.
fs.readFile('giant_file_1.txt', (err, data) => {
    console.log("File 1 is finally ready!");
});

fs.readFile('giant_file_2.txt', (err, data) => {
    console.log("File 2 is finally ready!");
});

console.log("Finished firing requests! Now I wait for callbacks.");

// Output:
// Finished firing requests! Now I wait for callbacks.
// File 1 is finally ready!
// File 2 is finally ready!
```

### Why DSH chose Node.js
When DSH's AI Agent calls the `glob` tool to search 70,000 files, or makes an HTTP request to the LLM API, it is waiting. Node.js allows DSH to orchestrate thousands of parallel filesystem operations, subprocess spawns (like running `ripgrep`), and network requests without consuming massive amounts of RAM or locking up the agent's core loop.

---

## 2. TypeScript: The Mathematical Safety Net

### The Danger of Dynamic Typing
JavaScript is "dynamically typed". This means variables can change their shape at runtime.

```javascript
// A perfectly valid JavaScript function
function processTool(toolConfig) {
    // We expect toolConfig to be an object with a 'timeout' number.
    // What if someone passes a string "500" instead of the number 500?
    // What if someone passes undefined?
    return toolConfig.timeout * 2;
}
```
In a small script, this is fine. In DeepSeek Harness, a `toolConfig` might be a deeply nested 500-line JSON object representing an AI's capability. If a property is missing, the AI might hallucinate or crash the server.

### Why does TypeScript exist?
Created by Microsoft, TypeScript is a superset of JavaScript. It adds **Static Compilation and Structural Typing**. It does not run in the browser or on Node.js directly. It is a *compiler* that checks your code mathematically, and then strips out all the types, spitting out pure JavaScript.

### Deep Dive: Structural Typing in DSH
TypeScript allows DSH to define strict contracts. Look at how DSH might define a Tool Execution:

```typescript
// We define a strict interface. If data doesn't perfectly match this, the code WON'T COMPILE.
interface ToolExecution {
    readonly callId: string;
    readonly name: string;
    readonly arguments: unknown; // 'unknown' means we force the developer to manually validate it later!
    readonly isParallelSafe: boolean;
}

// We can also use "Discriminated Unions" - a very powerful TypeScript feature.
// A tool result is EITHER a Success OR a Failure. It cannot be both.
type ToolResult =
  | { kind: 'success', data: string, error?: never }
  | { kind: 'failure', error: string, data?: never };

function handleResult(res: ToolResult) {
    if (res.kind === 'success') {
        // TypeScript is smart! It KNOWS 'res.data' exists here, but 'res.error' does not.
        console.log(res.data);
    } else {
        // TypeScript KNOWS 'res.error' exists here.
        console.error(res.error);
    }
}
```

### Why DSH chose TypeScript
By using TypeScript, DSH guarantees that when a Subagent sends a message to a Main Agent, the message format is 100% correct. Entire categories of bugs (like "Cannot read property 'name' of undefined") are eliminated before the programmer even runs the code.

---

## 3. The Philosophy of Plugins (Inversion of Control)

### The Monolith (Hardcoded Dependencies)
In a traditional system, components are hardcoded together. If the AI Agent needs to read a file, the code looks like this:

```typescript
import { FileSystemReader } from './my-hardcoded-fs.ts';

class Agent {
    private fs = new FileSystemReader(); // HARDCODED!

    think() {
        this.fs.read("/etc/password");
    }
}
```
**The Problem:** What if we want to run this Agent in a secure Sandbox where it can't read the real hard drive, but instead reads a fake "virtual" hard drive? We would have to rewrite the `Agent` class!

### The Modern Way: Dependency Injection and Inversion of Control (IoC)
A Plugin Architecture relies on **Inversion of Control**. The Agent should not know *how* to read a file, nor should it create the File Reader. It should just ask the system for "a thing that reads files".

```typescript
// The Agent only knows about the INTERFACE, not the implementation.
interface FileSystem {
    read(path: string): string;
}

class Agent {
    // The FileSystem is INJECTED into the Agent from the outside.
    constructor(private fs: FileSystem) {}

    think() {
        this.fs.read("/etc/password");
    }
}
```

### Visualizing the Plugin Pegboard

```text
=========================================
        THE CORDIS PEGBOARD (IoC)
=========================================

+---------------------------------------+
|  CORDIS REGISTRY (The Middleman)      |
|  "I hold all registered services."    |
+---------------------------------------+
       ^                         ^
       |                         |
       | (Provides Service)      | (Requests Service)
       |                         |
[fs-local Plugin]          [Agent Plugin]
"I am a FileSystem!        "Hey Cordis, do you have
 I read real disks."        a FileSystem I can use?"
```

Now, if we want to secure the system, we just unplug `fs-local Plugin` and plug in `fs-sandbox Plugin`. The `Agent Plugin` doesn't change a single line of code! This is why DeepSeek Harness says: **"everything is a plugin"**.

---
### Hands-On: The IoC Thought Experiment
Imagine you are building a Car.
1. **Monolith:** You weld the tires directly to the axles. If a tire goes flat, you have to buy a new car.
2. **Plugin (IoC):** You standardise a 5-bolt hub. The car doesn't care if the tire is made by Michelin or Goodyear, or if it's a winter tire or a summer tire. As long as the tire has 5 bolts (the `interface`), the car runs.

In DSH, Cordis is the 5-bolt hub. Every tool, every LLM adapter, and every filesystem is just a tire.

---
*Ready to see how Cordis actually implements this magical pegboard in code? Proceed to [Module 2](../module-02-architecture/README.md).*
