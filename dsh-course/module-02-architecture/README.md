# Module 2: The Core Architecture (Cordis)

Welcome to Module 2! In Module 1, we learned about Dependency Injection and Inversion of Control. In DeepSeek Harness (DSH), the master framework that implements this "pegboard" architecture is named **Cordis**.

Cordis is the heart of DSH. It implements a concept known as *Spatiotemporal Composability*. We will break down exactly what that means, and look at the real code patterns used in the DSH repository.

---

## 1. What is Cordis?

If you want to build an AI agent that can browse files, run Python code, and talk to an LLM, you have a problem: How do these completely different plugins share data safely? How do they load and unload without crashing the server?

**Cordis** acts as the central nervous system.
In DeepSeek Harness, **literally everything is a plugin**.
*   The LLM adapter (`dsh-llm`)? A plugin.
*   The filesystem reader (`dsh-fs`)? A plugin.
*   The agent tool registry (`dsh-tools`)? A plugin.

Cordis manages the lifecycle of these plugins. It ensures they start up correctly, share data securely, and shut down cleanly without leaving memory leaks.

---

## 2. The Holy Trinity of Cordis

Cordis is built on three atomic concepts: **Context (`ctx`)**, **Services**, and **Events/Waterfalls**.

### A. The Context (`ctx`)
Think of `ctx` as the physical Pegboard. Every plugin function receives the `ctx` object when it starts. The `ctx` is how a plugin reaches out to the rest of the world. It holds the plugin's configuration, its lifecycle methods, and access to the registry.

### B. Services (The "I provide / I consume" contract)
A Service is a capability that one plugin provides, which other plugins can use.
This is pure Dependency Injection.

**Example: The Filesystem Service (`ctx.fs`)**

1.  **Defining the Contract:** First, we define what the service *looks like*.
    ```typescript
    // We declare the shape of the service.
    interface FileSystem {
        readText(path: string): Promise<string>;
    }
    // We tell Cordis that 'ctx.fs' exists.
    declare module 'cordis' {
        interface Context {
            fs: FileSystem;
        }
    }
    ```

2.  **Providing the Service:** The `fs-local` plugin tells Cordis: *"I am the provider for `ctx.fs`!"*
    ```typescript
    export function LocalFsPlugin(ctx: Context) {
        // We inject the implementation into the pegboard
        ctx.set('fs', {
            readText: async (path) => {
                // Actual node.js disk reading code goes here
                return "hello world";
            }
        });
    }
    ```

3.  **Consuming the Service:** The `agent` plugin doesn't care *who* provided it, it just uses it.
    ```typescript
    export function AgentPlugin(ctx: Context) {
        // We wait for someone (anyone!) to provide the 'fs' service
        ctx.inject(['fs'], (ctx) => {
            ctx.fs.readText("/etc/password").then(text => console.log(text));
        });
    }
    ```

### C. Events & Waterfalls (The Interceptors)
Events are how plugins react to things happening in the system *without* directly importing or relying on each other. Cordis uses highly sophisticated "Waterfalls".

Imagine a piece of paper being passed down a line of 5 people.
1. The 1st person looks at the paper. They can alter it, approve it, or reject it.
2. They pass it to the 2nd person, who does the same.
3. This is a **Waterfall Event**.

In DSH, when an AI agent wants to use a tool, it triggers a `tools/pre-execute` waterfall event.
Security plugins can listen to this event. If a security plugin sees the AI trying to read a protected file, it can intercept the waterfall and say "DENIED".

**Deep Dive Code: Intercepting a Waterfall**
```typescript
// The Security Plugin
export function SecurityPlugin(ctx: Context) {

    // We listen to the waterfall event.
    // We receive the 'exec' (the tool call) and a 'next' function.
    ctx.on('tools/pre-execute', async (exec, next) => {

        if (exec.name === 'bash' && exec.arguments.includes('rm -rf')) {
            // We short-circuit the waterfall! We DO NOT call next().
            // The tool is immediately blocked.
            return { kind: 'deny', reason: 'Destructive commands are forbidden.' };
        }

        // If it's safe, we delegate to the next listener in the waterfall.
        return next();
    });
}
```
The AI Tool execution pipeline literally loops through these listeners. Because of this, the core tool registry code never has to contain hardcoded security checks.

---

## 3. Spatiotemporal Composability (Unpacking the Jargon)

Cordis is based on a paper called *"A Programming Paradigm for Spatiotemporal Composability"*. What do these big words mean in practice?

### Temporal (Time): Flawless Unloading via Disposers
Plugins can be loaded and unloaded *while the program is running*.
If you register an event listener in normal Node.js, and then delete the plugin, that event listener stays in memory forever (a Memory Leak).

In Cordis, every action returns a **Disposer** (a cleanup function). Cordis tracks these automatically.
```typescript
export function MyPlugin(ctx: Context) {
    // ctx.on returns a Disposer function
    const dispose = ctx.on('some/event', () => console.log("Event!"));

    // If Cordis decides to unload MyPlugin, it automatically calls dispose().
    // Time is rewound to exactly how it was before the plugin loaded!
}
```

### Spatial (Space): Scoped Isolation
Plugins can be isolated into "Realms" or "Scopes".
You can have multiple Agents running in the exact same Node.js process, but they are isolated in different spaces.
*   Agent A (The Admin) has access to the "Delete File" tool.
*   Agent B (The Web Scraper) does not.

Cordis handles this using `ctx.isolate()`.
```typescript
// Create a completely isolated bubble inside the main context
const subContext = ctx.isolate(['tools']);

// Any tools registered in subContext are ONLY visible to subContext!
subContext.plugin(AdminToolsPlugin);
```

---

## 4. Visualizing the Architecture

```text
=========================================================
            THE CORDIS CONTEXT (ctx)
=========================================================

  [Global Context]
       |
       |--- Provides: ctx.fs (Local Disk)
       |--- Provides: ctx.llm (OpenAI API)
       |
       |--- [Scope: Agent A (Admin)]
       |        | (Inherits ctx.fs and ctx.llm)
       |        |--- Provides: ctx.tools (Delete File)
       |
       |--- [Scope: Agent B (Scraper)]
                | (Inherits ctx.fs and ctx.llm)
                |--- Provides: ctx.tools (Read Webpage)
                |
                v
      (Agent B calls 'Read Webpage' Tool)
      triggers: "tools/pre-execute" (Waterfall)
                |
                v
       +--------------------------+
       | PLUGIN: Rate Limiter     |
       | (Checks if Agent B has   |
       |  made too many requests) |
       +--------------------------+

```

## Summary
Why does DeepSeek Harness use Cordis?
1.  **Zero hardcoding:** Everything is loosely coupled via Dependency Injection (Services).
2.  **Extensible pipelines:** The core loops are empty; behavior is injected via Waterfall Events.
3.  **Memory safety:** Plugins track their own lifecycles (Temporal).
4.  **Security boundaries:** Agents can run in the same process but possess completely different toolsets (Spatial).

---
*Next, we will tackle the biggest challenge: How does this architecture use tools to read 70,000 code files? Proceed to [Module 3](../module-03-tooling-retrieval/README.md).*