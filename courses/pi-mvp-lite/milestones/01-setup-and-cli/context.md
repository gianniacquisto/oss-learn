# Context for Milestone 1: Project Setup & Hello Agent

## Background

Before writing any agent logic, you need a working development environment. This means three things:

1. **A package manager** (`npm`) that tracks your dependencies and provides scripts to run the project
2. **A compiler** (TypeScript's `tsc`) that translates your `.ts` source files into `.js` files Node.js can execute
3. **A configuration system** that loads settings at startup without hardcoding them

This milestone walks you through all three, step by step.

### How pi.dev Does It

The real pi.dev agent is configured through environment variables (for secrets like API keys) and config files (for settings like model selection). We're building a simplified version of that pattern so you understand the *why* behind each piece. In production systems, this same approach lets you ship one binary that works in dozens of environments without code changes.

## The Module System Problem (And Why It Matters)

Node.js has two competing systems for loading code:

| | CommonJS | ES Modules |
|---|---|---|
| Import syntax | `const x = require('./x')` | `import x from './x.js'` |
| File extension in imports | `.js` optional | `.js` **required** |
| Default in Node.js | Yes (unless `"type": "module"`) | No (opt-in via `"type": "module"`) |
| Used by pi.dev | ❌ | ✅ |

If you set `module: "NodeNext"` in TypeScript but forget `"type": "module"` in `package.json`, Node.js will try to run ES module code as CommonJS and crash with errors like `"SyntaxError: Unexpected token 'export'"`. This is the #1 gotcha when setting up a new TypeScript project.

**Rule of thumb:** `"type": "module"` in `package.json` and `module: "NodeNext"` in `tsconfig.json` always go together. They tell Node.js and TypeScript to use the same module system.

## The Configuration Pattern

Your agent needs certain information at startup:

- **API key** — to authenticate with OpenAI's servers (a secret, never committed to git)
- **Model name** — which LLM to use (e.g., `"gpt-4o"`)
- **Provider** — which service to call (`"openai"` or `"anthropic"`)
- **System prompt** — the instructions that shape how the agent behaves

A good config system loads these from multiple sources in priority order:

1. **Config file** (`.agentrc.json`) — convenient for local development
2. **Environment variables** — portable, works in CI/CD and deployment
3. **Sensible defaults** — so the project runs out of the box for testing

If none of these provide an API key, the config system should fail fast with a clear error message. This is better than silently using a wrong value and wasting tokens on failed API calls.

## How This Fits in the Bigger Picture

```text
Milestone 1 (this one)       Milestone 2              Milestone 3
┌─────────────────┐          ┌──────────────────┐     ┌──────────────┐
│ package.json     │          │ Message types     │     │ Agent loop    │
│ tsconfig.json    │────────▶ │ Agent class       │────▶│ run() method  │
│ config.ts        │          │ State management  │     │ Inner loop    │
│ cli.ts           │          │                   │     └──────────────┘
└─────────────────┘          └──────────────────┘
```

Everything you build later lives inside the structure created here. The `config.ts` file loads settings that the `Agent` class (Milestone 2) consumes. The `cli.ts` entry point orchestrates everything.

## What "Good" Looks Like

By the end of this milestone:

- ✅ `npm install` sets up all dependencies
- ✅ `npm run build` compiles your TypeScript to `dist/` with zero errors
- ✅ `npm start` prints `"Hello agent"` and exits with code 0
- ✅ Missing API key produces a clear error: `"No API key found. Set AGENT_API_KEY or create .agentrc.json"`
- ✅ You can override settings via environment variables without changing any code

Nothing fancy — just a foundation that works reliably and is easy to extend.
