# Steps — Project Setup & Configuration

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 1.1: Initialize the project

**Goal:** Create a package.json and install dependencies.

**Requirements:**
- Use `npm init -y` to create a basic package.json
- Set `"type": "module"` in package.json (ES modules)
- Install these dependencies:
  - `@anthropic-ai/sdk` — for Anthropic API integration
  - `openai` — for OpenAI API integration
  - `zod` — for runtime validation of tool arguments and config
  - `commander` — for CLI argument parsing
- Install TypeScript as a dev dependency: `npm install -D typescript @types/node`

??? tip "Hints"
    - You'll need `@types/node` for Node.js type definitions (fs, path, process.env, etc.)
    - Keep the dependency list minimal right now. You can add more later if needed.

??? example "Reference pattern"
    === "package.json"
        ```json
        {
          "name": "pi-mvp",
          "version": "0.1.0",
          "type": "module",
          "bin": {
            "pi-mvp": "./dist/cli.js"
          },
          "scripts": {
            "build": "tsc",
            "start": "node dist/cli.js"
          }
        }
        ```

---

## Step 1.2: Configure TypeScript

**Goal:** Create a tsconfig.json that compiles your project correctly for Node.js ES modules.

**Requirements:**
- Set `module` to `NodeNext` (or `ESNext`)
- Set `target` to `ES2022` or later
- Enable `strict: true` for full type checking
- Set `outDir` to `dist/` and `rootDir` to `src/`
- Enable `esModuleInterop` and `moduleResolution: "NodeNext"`

??? tip "Hints"
    - `module: "NodeNext"` is important because it lets you use `.js` extensions in imports (which TypeScript rewrites from `.ts`) while respecting Node's native ESM resolution. This is what pi.dev uses.
    - You'll want source maps for debugging: `"sourceMap": true`.

??? example "Reference pattern"
    === "tsconfig.json"
        ```json
        {
          "compilerOptions": {
            "target": "ES2022",
            "module": "NodeNext",
            "moduleResolution": "NodeNext",
            "strict": true,
            "esModuleInterop": true,
            "outDir": "./dist",
            "rootDir": "./src",
            "sourceMap": true,
            "declaration": true,
            "skipLibCheck": true,
            "forceConsistentCasingInFileNames": true
          },
          "include": ["src/**/*"],
          "exclude": ["node_modules", "dist"]
        }
        ```

---

## Step 1.3: Create the project structure

**Goal:** Set up your source directory with the files you'll need going forward.

**Requirements:**

Create these directories and files under `src/`:
- `src/cli.ts` — CLI entry point (run with `npm start`)
- `src/config.ts` — Configuration loading and types
- `src/types.ts` — Core type definitions for the agent
- `src/agent.ts` — The Agent class (you'll flesh this out in later milestones)
- `src/loop.ts` — The agent loop logic (Milestone 3)

??? tip "Hints"
    - For now, each file can have minimal content. You're setting up the skeleton.
    - Create a `.gitignore` that excludes `node_modules/`, `dist/`, and `.env`.

??? example "Project structure"
    ```text
    src/
    ├── cli.ts      # Entry point — parses args, loads config, runs agent
    ├── config.ts   # Config types + loading logic
    ├── types.ts    # Core type definitions (AgentMessage, AgentTool, etc.)
    ├── agent.ts    # Agent class — stateful wrapper around the loop
    └── loop.ts     # The core agent loop — prompt → LLM → tools → repeat
    ```

---

## Step 1.4: Implement config loading

**Goal:** Create a configuration system that loads settings from environment variables and/or a JSON config file, with sensible defaults.

**Requirements:**
- Define a `AgentConfig` interface with these fields:
  - `model: string` — the LLM model to use (e.g., `"claude-3-5-sonnet"`, `"gpt-4o"`)
  - `apiKey: string` — the API key for the provider
  - `provider: "openai" | "anthropic"` — which LLM provider to use
  - `systemPrompt?: string` — optional system prompt override
  - `thinkingLevel?: "off" | "low" | "medium" | "high"` — reasoning level
- Implement a `loadConfig()` function that:
  1. Checks for an optional `.agentrc.json` file in the current directory
  2. Falls back to environment variables (`AGENT_PROVIDER`, `AGENT_API_KEY`, `AGENT_MODEL`)
  3. Uses sensible defaults if neither is provided
  4. Throws a descriptive error if the API key is missing

??? tip "Hints"
    - Use `fs.readFileSync` with `JSON.parse` for config file loading. Wrap in try/catch so a missing or invalid file falls through to env vars.
    - For environment variables, use `process.env.VAR_NAME ?? "default"`.
    - Consider creating a `getDefaultSystemPrompt()` function that returns a reasonable default system prompt string.

??? example "Reference pattern"
    === "config.ts"
        ```typescript
        export interface AgentConfig {
          model: string;
          apiKey: string;
          provider: "openai" | "anthropic";
          systemPrompt?: string;
          thinkingLevel: ThinkingLevel;
        }

        export type ThinkingLevel = "off" | "low" | "medium" | "high";

        export function loadConfig(): AgentConfig {
          // 1. Try loading from .agentrc.json
          // 2. Fall back to environment variables
          // 3. Apply defaults for any missing fields
          // 4. Validate that apiKey is present (throw if not)
        }

        export function getDefaultSystemPrompt(): string {
          return "You are a helpful assistant...";
        }
        ```

---

## Step 1.5: Create the CLI entry point

**Goal:** A minimal `cli.ts` that loads config and prints "Hello agent" to verify everything compiles and runs.

**Requirements:**
- Import `loadConfig()` from `./config.js`
- Load the configuration (don't use it yet, just call the function)
- Print `"Hello agent"` to stdout
- Handle errors gracefully: if config loading fails, print a helpful error message and exit with code 1

??? tip "Hints"
    - Use `console.error()` for error output (not `console.log()`) — this follows Unix conventions.
    - Exit with `process.exit(0)` on success and `process.exit(1)` on failure.
    - Keep the CLI minimal here — you'll add real command parsing in later milestones.

??? example "Reference pattern"
    === "cli.ts"
        ```typescript
        import { loadConfig } from "./config.js";

        async function main() {
          try {
            const config = loadConfig();
            console.log("Hello agent");
            process.exit(0);
          } catch (error) {
            const message = error instanceof Error ? error.message : String(error);
            console.error(`Error: ${message}`);
            process.exit(1);
          }
        }

        main();
        ```

---

## Done? Check your work [:material-arrow-right:](done.md)

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
