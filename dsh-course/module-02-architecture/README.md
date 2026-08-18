# Module 2: The Core Architecture (Cordis)

Welcome to Module 2! In Module 1, we learned about the "Pegboard" concept—the Plugin Architecture. In DeepSeek Harness (DSH), that pegboard has a name: **Cordis**.

Cordis is a framework that implements *Spatiotemporal Composability*. Don't let the jargon scare you! We will break down exactly what that means.

---

## 1. What is Cordis?

If you want to build an AI agent that can browse files, run Python code, and talk to an LLM, you have a problem: How do these completely different pieces of code share data without getting tangled up in a messy "Monolith"?

**Cordis** solves this by acting as a central nervous system.
In DeepSeek Harness, **literally everything is a plugin**.
*   The LLM adapter? A plugin.
*   The filesystem reader? A plugin.
*   The system prompt? A plugin.
*   The agent itself? A plugin!

Cordis manages the lifecycle of these plugins. It ensures they start up correctly, share data safely, and most importantly, shut down cleanly without leaving memory leaks.

## 2. The Holy Trinity of Cordis

Cordis is built on three atomic concepts: **Context (`ctx`)**, **Services**, and **Events**.

### A. The Context (`ctx`)
Think of `ctx` as the physical Pegboard. It represents the *environment* the plugin is living in. Every plugin receives the `ctx` object when it starts. The `ctx` is how a plugin reaches out to the rest of the world.

### B. Services
A Service is a capability that one plugin provides, which other plugins can use.
*   *Example:* The Filesystem Plugin provides a `ctx.fs` service.
*   If the LLM Plugin needs to read a file, it doesn't talk to the hard drive directly. It asks the pegboard: "Hey `ctx`, do you have an `fs` service? Yes? Please read this file."

### C. Events (The Waterfall)
Events are how plugins react to things happening in the system without directly relying on each other. Cordis uses highly sophisticated "Waterfalls".

Imagine a piece of paper being passed down a line of 5 people.
1. The 1st person looks at the paper. They can alter it, approve it, or reject it.
2. They pass it to the 2nd person, who does the same.
3. This is a **Waterfall Event**.

In DSH, when an AI agent wants to use a tool (like `read_file`), it triggers a `tools/pre-execute` waterfall event.
Security plugins can listen to this event. If a security plugin sees the AI trying to read a password file, it can intercept the waterfall and say "DENIED", stopping the tool from running. The AI tool plugin never even knows the security plugin exists!

---

## 3. Visualizing Cordis

```text
=========================================================
            THE CORDIS CONTEXT (ctx)
=========================================================

  +------------------+          +-------------------+
  | PLUGIN: fs-local |          | PLUGIN: dsh-tools |
  | (Provides ctx.fs)|          | (Uses ctx.fs)     |
  +--------+---------+          +---------+---------+
           |                              ^
           v    (Registers Service)       | (Calls Service)
  ***************************************************
  *                ctx.fs (Service)                 *
  ***************************************************
                             |
                             v
                    (Triggers Event)
            "tools/pre-execute" (Waterfall)
                             |
                             v
                +--------------------------+
                | PLUGIN: Security Policy  |
                | (Listens to the event,   |
                |  can ALLOW or DENY)      |
                +--------------------------+

```

## 4. Spatiotemporal Composability (Unpacking the Jargon)

Cordis is based on a paper called *"A Programming Paradigm for Spatiotemporal Composability"*.
*   **Spatial (Space):** Plugins can be isolated. You can have multiple Agents running in the exact same Node.js process, but they are isolated in different "spaces" (Scopes). Agent A has access to the "Delete File" tool, but Agent B does not.
*   **Temporal (Time):** Plugins can be loaded and unloaded *while the program is running*. If you unload the "Security Policy" plugin, Cordis automatically un-registers its waterfall listeners. The system rewinds time to exactly how it was before the plugin loaded. No memory leaks!

---

## 5. Hands-on: Writing a Dummy Cordis Plugin

Let's look at what actual Cordis code looks like! Here is how you would write a simple plugin in DSH.

```typescript
// 1. We define our Plugin function. It takes the 'ctx' (the pegboard).
export function LoggerPlugin(ctx: Context) {

  // 2. We use 'ctx' to listen to an EVENT.
  // Whenever the 'agent/request' event happens (the AI is about to think)
  ctx.on('agent/request', (exec, next) => {

    // We log what the AI is trying to do
    console.log("The AI Agent is making a request!");

    // 3. We call next() to pass the waterfall down to the next plugin.
    // If we didn't call next(), the AI would literally freeze forever!
    return next();
  });

}
```

**Why is this revolutionary?**
If we didn't use Cordis, the developers would have to hard-code `console.log()` directly into the Core AI loop. Every new feature would bloat the core loop until it was 10,000 lines long. With Cordis, the core loop is completely empty, and features just "snap" onto it!

---
*Next, we will tackle the biggest challenge: How does this architecture handle 70,000 code files? Proceed to [Module 3](../module-03-tooling-retrieval/README.md).*
