---
comments: Milestone 1 of 5
---

# ① Project Setup & Hello Agent

## What You'll Build

A working TypeScript project with a minimal CLI entry point. By the end of this milestone you will have:

- A `package.json` that declares your project's dependencies and scripts
- A `tsconfig.json` that configures TypeScript to compile ES modules for Node.js
- A clean source directory (`src/`) with five files, each with a single responsibility
- A configuration loader that reads from a JSON file or environment variables
- A CLI script that you run with `npm start` and it prints `"Hello agent"`

None of this is "agent" logic yet. But every line sets up a pattern you'll rely on in later milestones.

## Why This Matters

Every real project starts with setup — and getting it right the first time saves hours of debugging later. The choices you make here (modules vs CommonJS, how config is loaded, how the project is structured) cascade through everything that follows. In this milestone you'll learn **why** these choices exist and **how** they connect.

Specifically:

- **ES modules vs CommonJS:** Node.js has two systems for loading code from files. Using the modern system (ES modules) means your code can use `import`/`export` syntax, works with browser tooling, and matches how pi.dev is written. Getting this wrong leads to cryptic "Cannot use import statement" errors.
- **TypeScript configuration:** TypeScript doesn't just catch type errors — its compiler settings control *how* your code runs. The `module: "NodeNext"` setting alone affects whether your imports resolve correctly.
- **Config loading:** An agent needs API keys, model names, and other settings at startup. Loading them from both a config file (for local development) and environment variables (for deployment) is the standard pattern you'll see in every real project.
- **Separation of concerns:** Splitting `cli.ts` (entry point), `config.ts` (settings), and `agent.ts` (core logic) into separate files isn't just "clean code" advice — it means each piece can be tested, modified, or replaced independently.

## Prerequisites for This Milestone

- **Node.js 18+** installed (`node -v` should show `v18.x` or higher)
- **npm** available (`npm -v` should work — it ships with Node.js)
- **A text editor** — VS Code is recommended but any editor works
- **Basic familiarity** with creating files and running terminal commands

If you haven't used TypeScript before, don't worry. This milestone teaches you what you need as we go.

## Concepts You'll Learn

??? example "ES Modules in Node.js"
    Node.js historically used `require()` (CommonJS). Modern JavaScript uses `import`/`export` (ES modules). Setting `"type": "module"` in `package.json` tells Node.js to treat `.js` files as ES modules. This is required because TypeScript with `module: "NodeNext"` generates ES module output, and the two must match.

??? example "TypeScript Compilation"
    TypeScript is a *superset* of JavaScript — you write `.ts` files but Node.js can only run `.js` files. The TypeScript compiler (`tsc`) translates your code to plain JavaScript while checking types along the way. The `outDir` setting in `tsconfig.json` controls where the output goes, and `rootDir` tells it where your source lives.

??? example "Environment Variables"
    Environment variables are key-value pairs set outside your program (in the shell or a `.env` file). They're the standard way to pass secrets like API keys into a program without hardcoding them. In Node.js, you access them via `process.env.VARIABLE_NAME`.

---

## Steps

| Step | Task | What You'll Learn |
|------|------|-------------------|
| 1.1 | Initialize the project | How `npm` manages packages and what `"type": "module"` does |
| 1.2 | Configure TypeScript | How compiler options affect your code's output and runtime behavior |
| 1.3 | Create the project structure | Why separate files for config, types, agent, and CLI matters |
| 1.4 | Implement config loading | Reading files, parsing JSON, environment variables, and error handling |
| 1.5 | Create the CLI entry point | The `async main()` pattern, error propagation, and process exit codes |

---

[:material-arrow-left: Back to Course](../../README.md){: .md-button }&nbsp;&nbsp;[:material-book-open-variant: Read Full Context](context.md){: .md-button }&nbsp;&nbsp;[:material-flag: Done Checklist](done.md){: .md-button }&nbsp;&nbsp;[:material-arrow-right: Start Building →](steps.md){: .md-button .md-button--primary }
