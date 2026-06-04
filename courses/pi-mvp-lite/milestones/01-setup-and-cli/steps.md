# Steps — Project Setup & Hello Agent

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 1.1: Initialize the project

**Goal:** Create a package.json and install dependencies.

**Requirements:**
- Run `npm init -y` to create a basic package.json
- Open `package.json` in your editor and add `"type": "module"` (this tells Node.js to use ES modules)
- Install these dependencies:
  ```bash
  npm install openai zod commander
  ```
- Install TypeScript and Node.js types as dev dependencies:
  ```bash
  npm install -D typescript @types/node
  ```

??? tip "Hints"
    - `"type": "module"` goes inside the top-level object in package.json, alongside `"name"`, `"version"`, etc.
    - `openai` is the OpenAI SDK — you'll use it to call GPT in Milestone 4.
    - `zod` is a validation library — you'll use it to check tool arguments in Milestone 3.
    - `commander` is a CLI library — you'll use it for parsing command-line arguments.
    - `@types/node` gives you TypeScript types for Node.js APIs like `process.env` and `fs`.

??? example "Reference pattern"
    === "package.json (key fields)"
        ```json
        {
          "name": "my-agent",
          "version": "0.1.0",
          "type": "module",
          "bin": {
            "my-agent": "./dist/cli.js"
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
- Create a file called `tsconfig.json` in your project root
- Set the following compiler options:
  - `target: "ES2022"` — the JavaScript version you're compiling to
  - `module: "NodeNext"` — the module system (ES modules for Node.js)
  - `moduleResolution: "NodeNext"` — how to resolve imports
  - `strict: true` — enable all type checking
  - `outDir: "./dist"` — where compiled JavaScript goes
  - `rootDir: "./src"` — where your TypeScript source lives
  - `esModuleInterop: true` — allows importing CommonJS modules
  - `sourceMap: true` — generates source maps for debugging

??? tip "Hints"
    - `module: "NodeNext"` is important because it lets you use `.js` extensions in imports (which TypeScript rewrites from `.ts`) while respecting Node's native ESM resolution.
    - The `outDir` and `rootDir` settings tell TypeScript: "read source from `src/`, write compiled JS to `dist/`."
    - `strict: true` enables all the good type-checking features. It might feel strict at first, but it catches bugs early.

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

Create these directories and files:

```
src/
├── cli.ts        # Entry point — runs the agent
├── config.ts     # Configuration loading
├── types.ts      # Type definitions
├── agent.ts      # The Agent class
└── loop.ts       # The agent loop
```

Also create a `.gitignore` file with:
```
node_modules/
dist/
.env
```

??? tip "Hints"
    - Each file starts empty for now — you'll fill them in during this milestone and later ones.
    - The naming is intentional: `cli.ts` is the command-line interface, `config.ts` handles configuration, `types.ts` defines types, `agent.ts` is the main Agent class, and `loop.ts` is the agent loop logic.
    - A `.gitignore` file tells Git which files to ignore (like `node_modules/` which is downloaded, not written by you).

??? example "Reference pattern"
    === "Project structure"
        ```text
        my-agent/
        ├── package.json
        ├── tsconfig.json
        ├── .gitignore
        ├── src/
        │   ├── cli.ts
        │   ├── config.ts
        │   ├── types.ts
        │   ├── agent.ts
        │   └── loop.ts
        └── node_modules/    (created by npm install)
        └── dist/            (created by npm run build)
        ```

---

## Step 1.4: Implement config loading

**Goal:** Create a configuration system that loads settings from environment variables and/or a JSON config file, with sensible defaults.

**Requirements:**
- In `src/config.ts`, define an `AgentConfig` interface with these fields:
  - `model: string` — the LLM model to use (e.g., `"gpt-4o"`)
  - `apiKey: string` — the API key for the provider
  - `provider: "openai" | "anthropic"` — which LLM provider to use
  - `systemPrompt?: string` — optional system prompt override

- Implement a `loadConfig()` function that:
  1. Checks for an optional `.agentrc.json` file in the current directory
  2. Falls back to environment variables (`AGENT_PROVIDER`, `AGENT_API_KEY`, `AGENT_MODEL`)
  3. Uses sensible defaults if neither is provided
  4. Throws a descriptive error if the API key is missing

- Implement a `getDefaultSystemPrompt()` function that returns a basic system prompt string

??? tip "Hints"
    - To read a file: `import fs from "fs"; const content = fs.readFileSync(".agentrc.json", "utf-8"); const config = JSON.parse(content);`
    - Wrap file reading in try/catch so a missing file falls through to environment variables.
    - For environment variables: `process.env.AGENT_API_KEY ?? ""` — the `??` is the nullish coalescing operator (returns the right side if the left is null or undefined).
    - A good default system prompt: `"You are a helpful assistant. You can use tools to help the user."`

??? example "Reference pattern"
    === "config.ts"
        ```typescript
        export interface AgentConfig {
          model: string;
          apiKey: string;
          provider: "openai" | "anthropic";
          systemPrompt?: string;
        }

        // Load config from .agentrc.json, env vars, or defaults
        export function loadConfig(): AgentConfig {
          // TODO: Read .agentrc.json if it exists
          // TODO: Fall back to environment variables
          // TODO: Apply defaults
          // TODO: Throw if apiKey is missing
          throw new Error("Not implemented yet");
        }

        export function getDefaultSystemPrompt(): string {
          // TODO: Return a basic system prompt
          throw new Error("Not implemented yet");
        }
        ```

---

## Step 1.5: Create the CLI entry point

**Goal:** A minimal `cli.ts` that loads config and prints "Hello agent" to verify everything compiles and runs.

**Requirements:**
- In `src/cli.ts`:
  - Import `loadConfig()` from `./config.js`
  - Load the configuration (don't use it yet, just call the function)
  - Print `"Hello agent"` to stdout
  - Handle errors gracefully: if config loading fails, print a helpful error message and exit with code 1

??? tip "Hints"
    - Import uses `.js` extension even though the file is `.ts` — TypeScript rewrites this automatically when using `module: "NodeNext"`.
    - Use `console.error()` for error output (not `console.log()`) — this follows Unix conventions.
    - Exit with `process.exit(0)` on success and `process.exit(1)` on failure.
    - Wrap the main logic in a try/catch block.

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
