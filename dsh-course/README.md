# DeepSeek Harness (DSH) Masterclass: From Zero to Agentic Engineer

Welcome to the ultimate beginner-friendly course on building state-of-the-art Agentic Systems! This course is designed to take you from a complete beginner to understanding how AI systems can navigate 70,000+ code files, make decisions, and use subagents to solve complex problems.

We will use **DeepSeek Harness (dsh)** as our primary study material. It is an enterprise-grade open-source system designed with an innovative "everything-is-a-plugin" architecture.

## Course Philosophy

1. **Top-Down & Zoom-In:** We will look at the big picture first, then zoom into the atomic level.
2. **No Magic:** We break down every piece of jargon. If a tool or technology is used, we explain *why* it exists, *what* preceded it, and *why* it was chosen.
3. **Visual Learning:** Heavy use of ASCII architectural diagrams to build mental models.
4. **Hands-On Practice:** "Try it out" sections in every module to cement your knowledge.

## Course Syllabus

### [Module 1: Foundations & The Tech Stack](./module-01-foundations/README.md)
*   **The Pre-requisites:** What is Node.js and TypeScript? Why do they exist?
*   **The Philosophy of Plugins:** What actually is a plugin? Monolithic vs. Modular design.
*   **Why this stack?** Why use TypeScript for an AI Agent instead of Python?

### [Module 2: The Core Architecture (Cordis)](./module-02-architecture/README.md)
*   **The Problem:** How do you build software that is infinitely extensible?
*   **The Solution:** The Cordis Framework.
*   **The Holy Trinity:** Context (`ctx`), Services, and Events.
*   **Hands-on:** Writing your very first dummy Cordis plugin.

### [Module 3: Tooling & The Retrieval Mechanism (Navigating 70k Files)](./module-03-tooling-retrieval/README.md)
*   **The Context Window Problem:** Why we can't just copy-paste a codebase into an LLM.
*   **The Toolbox:** Ripgrep (`rg`), Glob, and LSP (Language Server Protocol).
*   **The Pipeline:** How `dsh` safely executes shell commands (`ctx.subprocess`) and returns filtered context.
*   **Visualizing the Search-to-LLM flow.**

### [Module 4: Subagents & Context Management](./module-04-subagents/README.md)
*   **The Brain:** What is the "Turn Flow" and the Agent Loop?
*   **Context is King:** The Append-Only Session Log (`SessionEvent`). "Model-visible means logged."
*   **Subagents:** How multiple agents talk to each other without losing context.
*   **The Final Polish:** How state is saved and restored.

---
*Let's get started! Proceed to [Module 1](./module-01-foundations/README.md).*
