# Steps — Project Setup & Hello Agent

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 1.1: Initialize the project

**Goal:** Create a `package.json` and install the libraries your agent depends on.

### Why This Step Exists

Every Node.js project starts with a `package.json` — it's the manifest that describes what your project is, what it depends on, and how to run it. Without it, `npm` doesn't know what packages to install or what scripts to execute. The `"type": "module"` field is especially critical: without it, Node.js assumes the old CommonJS module system, which will clash with the TypeScript settings in the next step.

### Requirements

1. **Create a project directory and initialize it:**
   ```bash
   mkdir my-agent && cd my-agent
   npm init -y
   ```
   The `-y` flag accepts all defaults (name, version, description, etc.) so you get a `package.json` instantly. You can edit these fields later.

2. **Enable ES modules** by adding `"type": "module"` to your `package.json`. It goes inside the top-level JSON object, next to `"name"` and `"version"`:
   ```json
   {
     "name": "my-agent",
     "version": "1.0.0",
     "type": "module",
     ...
   }
   ```

3. **Add build and start scripts** to the `"scripts"` section:
   ```json
   {
     "scripts": {
       "build": "tsc",
       "start": "node dist/cli.js"
     }
   }
   ```
   - `npm run build` will invoke TypeScript's compiler (`tsc`) to translate `.ts` → `.js`
   - `npm start` runs the compiled JavaScript from `dist/cli.js`

4. **Install runtime dependencies:**
   ```bash
   npm install openai zod commander
   ```
   These are libraries your agent needs to *run* (not just to compile):

   | Package | What It Does | When You'll Use It |
   |---------|-------------|-------------------|
   | `openai` | Official OpenAI SDK — sends requests to GPT models | Milestone 5 (real LLM calls) |
   | `zod` | Runtime validation library — checks that data matches expected shapes | Milestone 4 (validating tool arguments) |
   | `commander` | CLI argument parsing — lets users type commands like `my-agent run` | Milestone 3+ (CLI commands) |

5. **Install development dependencies:**
   ```bash
   npm install -D typescript @types/node
   ```
   The `-D` flag marks these as *devDependencies* — they're only needed during development (compilation), not when someone runs your agent later:

   | Package | What It Does |
   |---------|-------------|
   | `typescript` | The TypeScript compiler (`tsc`) that converts `.ts` → `.js` and checks types |
   | `@types/node` | Type definitions for Node.js built-in APIs like `fs`, `path`, `process.env` |

### ??? tip "Hints"

- If you get `"npm: command not found"`, Node.js isn't installed or isn't in your PATH. Run `node -v` to check.
- The difference between `npm install pkg` and `npm install -D pkg`: the former adds to `dependencies` (needed at runtime), the latter to `devDependencies` (only needed during development). Your compiled agent doesn't need TypeScript itself — only the tool that produces the JavaScript does.

### ??? example "What your package.json should look like"

```json
{
  "name": "my-agent",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/cli.js"
  },
  "dependencies": {
    "commander": "^12.0.0",
    "openai": "^4.0.0",
    "zod": "^3.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0"
  }
}
```

The exact version numbers may differ — that's fine. What matters is the structure: `"type": "module"` is present, the scripts are set, and all five packages are installed.

### ??? warning "Common Mistake"

**Forgetting `"type": "module"`** — This is the most common error in this step. If you skip it, Node.js treats your `.js` files as CommonJS. When TypeScript generates ES module `import` statements, Node.js throws `"SyntaxError: Cannot use import statement outside a module."` If you see this error later, come back and check this field first.

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `package.json` exists and contains `"type": "module"`
- [ ] Running `npx tsc --version` prints a version number (confirms TypeScript installed)
- [ ] A `node_modules/` directory was created by `npm install`

---

## Step 1.2: Configure TypeScript

**Goal:** Create a `tsconfig.json` that tells the TypeScript compiler how to transform your code for Node.js ES modules.

### Why This Step Exists

TypeScript is a *strict superset* of JavaScript — any valid JavaScript is also valid TypeScript. But TypeScript adds type annotations, interfaces, and strict mode checks that don't exist in plain JavaScript. The compiler (`tsc`) has two jobs:

1. **Type checking** — find errors in your code (wrong types, missing fields, etc.) before it runs
2. **Code generation** — strip away the TypeScript-only syntax and produce plain `.js` files

The settings in `tsconfig.json` control *how* the compiler does both jobs. Get them wrong, and you'll get runtime errors or broken type checking.

### Key Settings Explained

| Setting | Value | Why It Matters |
|---------|-------|---------------|
| `target` | `"ES2022"` | The JavaScript version to compile *to*. ES2022 supports modern syntax like `?.` (optional chaining) and top-level `await`. If you set this too low (like `ES5`), TypeScript will add polyfills that bloat your output. |
| `module` | `"NodeNext"` | Tells TypeScript to generate ES module syntax (`import`/`export`) that's compatible with Node.js's native ESM loader. This must match `"type": "module"` in `package.json`. |
| `moduleResolution` | `"NodeNext"` | How TypeScript finds imported files. With `"NodeNext"`, it follows Node.js's resolution rules: `.js` extension is required, and `node_modules` is searched automatically. |
| `strict` | `true` | Enables *all* type-checking features at once (null checks, implicit `any` detection, etc.). This catches bugs early. You can relax it later for specific files if needed. |
| `outDir` | `"./dist"` | Where compiled `.js` files go. Your source stays in `src/`, output lands in `dist/`. This keeps generated files separate from your hand-written code. |
| `rootDir` | `"./src"` | Where TypeScript looks for source files. Together with `outDir`, this means `src/cli.ts` → `dist/cli.js` (preserving the relative path). |
| `esModuleInterop` | `true` | Allows importing CommonJS packages (like many npm libraries) using ES module syntax. Without it, imports from older packages fail. |
| `sourceMap` | `true` | Generates `.js.map` files that let your debugger map back to the original TypeScript. Essential for meaningful stack traces. |
| `declaration` | `true` | Generates `.d.ts` type definition files alongside your compiled JavaScript. Useful if other projects import your code. |
| `skipLibCheck` | `true` | Skips type-checking of `node_modules`. Speeds up compilation and avoids errors from third-party packages with loose types. |
| `forceConsistentCasingInFileNames` | `true` | Prevents bugs where `import "./File"` works on macOS but fails on Linux (case-sensitive filesystems). Catches these mismatches at compile time. |

### Requirements

1. **Create `tsconfig.json`** in your project root (same level as `package.json`) with the settings shown above.

2. **Verify it compiles:** Create a temporary file `src/temp.ts` with:
   ```typescript
   console.log("test");
   ```
   Then run `npm run build`. You should see `dist/temp.js` appear. Delete both files after verifying — you'll create the real files in the next step.

### ??? tip "Hints"

- The `include: ["src/**/*"]` line tells TypeScript which files to compile. Only files inside `src/` are processed.
- The `exclude: ["node_modules", "dist"]` line prevents TypeScript from trying to compile `node_modules` (too many files) or `dist/` (already compiled).
- If you see an error about `"Cannot find module 'fs' or its corresponding type declarations"`, make sure `@types/node` is installed.

### ??? example "Complete tsconfig.json"

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

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `tsconfig.json` exists in your project root
- [ ] Running `npm run build` produces files in `dist/` (no errors)
- [ ] The compiled output uses ES module syntax (`import`/`export`, not `require`)

---

## Step 1.3: Create the project structure

**Goal:** Set up the source directory with five files, each responsible for one piece of your agent.

### Why This Step Exists

Even small projects benefit from separating concerns. When every piece of logic lives in one file, it becomes impossible to find, test, or modify individual parts. Each file here has a single responsibility:

| File | Responsibility | Analogy |
|------|---------------|---------|
| `cli.ts` | **Entry point** — parses arguments, loads config, starts the agent | The front door of your house |
| `config.ts` | **Settings** — knows how to read `.agentrc.json`, environment variables, and defaults | The thermostat settings |
| `types.ts` | **Shared types** — defines the shapes of messages, tools, and other data | The blueprint legend |
| `agent.ts` | **Core logic** — the Agent class that manages conversation state | The brain |
| `loop.ts` | **Control flow** — the agent loop that coordinates LLM calls and tool execution | The heartbeat |

This structure mirrors how pi.dev organizes its code. As your agent grows, you might split files further (e.g., `tools/` directory, `providers/` directory), but these five files are the minimum viable structure.

### Requirements

1. **Create the `src/` directory and five empty TypeScript files:**
   ```bash
   mkdir src
   touch src/cli.ts src/config.ts src/types.ts src/agent.ts src/loop.ts
   ```
   Each file starts empty — you'll fill them in during this milestone and the next ones.

2. **Create a `.gitignore`** to keep generated and secret files out of version control:
   ```
   node_modules/
   dist/
   .env
   ```

   - `node_modules/` — contains all installed packages. These are regenerated by `npm install`, so there's no need to track them.
   - `dist/` — contains compiled JavaScript. It's always generated from `src/`, so tracking it would create redundant files.
   - `.env` — may contain secrets like API keys. **Never commit this file.**

### ??? warning "Why .gitignore Matters"

If you accidentally commit `node_modules/` (it can be hundreds of megabytes), your repository becomes bloated and slow. If you commit `.env` with a real API key, that key is public on GitHub — and others could use your OpenAI quota. The `.gitignore` file prevents both problems.

### ??? example "Expected project structure"

```
my-agent/
├── package.json      # Project manifest + scripts
├── tsconfig.json     # TypeScript compiler settings
├── .gitignore        # Files to exclude from git
├── src/              # Your TypeScript source code
│   ├── cli.ts        # (empty for now — Step 1.5 fills this)
│   ├── config.ts     # (empty for now — Step 1.4 fills this)
│   ├── types.ts      # (empty for now — Milestone 2)
│   ├── agent.ts      # (empty for now — Milestone 2)
│   └── loop.ts       # (empty for now — Milestone 3)
├── node_modules/     # (created by npm install, ignored by git)
└── dist/             # (created by npm run build, ignored by git)
```

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `src/` directory contains exactly 5 `.ts` files
- [ ] `.gitignore` exists and lists `node_modules/`, `dist/`, and `.env`
- [ ] Running `npm run build` still works (empty files compile fine)

---

## Step 1.4: Implement config loading

**Goal:** Write a configuration system in `src/config.ts` that loads settings from a JSON file or environment variables, with sensible defaults and clear error messages.

### Why This Step Exists

Your agent needs certain information before it can do anything:

- **Which LLM model to use?** (`gpt-4o`, etc.)
- **How to authenticate?** (API key)
- **Which provider?** (OpenAI, Anthropic)
- **What instructions to follow?** (system prompt)

Hardcoding these values means changing and recompiling your code every time you want to switch models or API keys. A proper config system reads them from external sources so the same code works everywhere.

### The Loading Strategy (Priority Order)

```text
┌─────────────────────────────────────────┐
│ 1. Check for .agentrc.json in current dir │ ← Convenient for local dev
│    If found, read all fields from it     │
├─────────────────────────────────────────┤
│ 2. Check environment variables           │ ← Portable across environments
│    AGENT_PROVIDER, AGENT_API_KEY,       │
│    AGENT_MODEL                           │
├─────────────────────────────────────────┤
│ 3. Apply hardcoded defaults              │ ← So the project runs out of the box
│    provider: "openai"                    │
│    model: "gpt-4o"                      │
└─────────────────────────────────────────┘
       ↓
  If apiKey is empty → throw error
```

The key insight: **each source fills in what the previous sources didn't provide**. A config file might set the model but not the API key (you don't want secrets in files). Environment variables might set the API key but not the model. Defaults catch everything else.

### Requirements

#### Part A: Define the `AgentConfig` interface

In `src/config.ts`, define an interface that describes what a valid configuration looks like:

```typescript
export interface AgentConfig {
  model: string;          // e.g., "gpt-4o"
  apiKey: string;         // API key — never empty
  provider: "openai" | "anthropic";  // which LLM provider
  systemPrompt?: string;  // optional override for the default prompt
}
```

**Why use an interface?** An interface is a TypeScript contract — it says "any `AgentConfig` object must have these fields with these types." This lets the compiler catch mistakes like misspelling `"provider"` or passing a number where a string is expected.

#### Part B: Implement `loadConfig()`

Write a function that returns `AgentConfig` by following the priority order above:

1. **Try reading `.agentrc.json`:**
   - Use `fs.readFileSync(".agentrc.json", "utf-8")` to read the file
   - Parse it with `JSON.parse(content)`
   - If the file doesn't exist (caught by try/catch), fall through to step 2
   - If the file has invalid JSON, throw a helpful error

2. **Check environment variables:**
   - `process.env.AGENT_PROVIDER` — the provider name
   - `process.env.AGENT_API_KEY` — the API key
   - `process.env.AGENT_MODEL` — the model name
   - Use the nullish coalescing operator (`??`) to provide defaults:
     ```typescript
     process.env.AGENT_MODEL ?? "gpt-4o"
     ```

3. **Apply defaults for any remaining missing fields:**
   - Default provider: `"openai"`
   - Default model: `"gpt-4o"`

4. **Validate that the API key is present:**
   - If `apiKey` is empty after all sources, throw:
     ```typescript
     throw new Error(
       "No API key found. Set AGENT_API_KEY or create .agentrc.json with an apiKey field."
     );
     ```

#### Part C: Implement `getDefaultSystemPrompt()`

Write a function that returns a default system prompt string. A good starting point:

```typescript
"You are a helpful coding assistant. You can use tools to read files, run commands, and help the user with tasks."
```

This prompt shapes how the LLM behaves — it's the "personality" of your agent. You'll pass this to the Agent class in Milestone 2.

### ??? tip "Hints"

- **Reading files in Node.js:**
  ```typescript
  import fs from "fs";
  
  const content = fs.readFileSync(".agentrc.json", "utf-8");
  const data = JSON.parse(content);
  ```
  `readFileSync` throws an error if the file doesn't exist. Wrap it in `try/catch` so a missing file falls through to environment variables instead of crashing.

- **The nullish coalescing operator (`??`):** Returns the left side unless it's `null` or `undefined`, in which case returns the right side. This is different from `||` which also treats empty strings and `0` as falsy:
  ```typescript
  process.env.AGENT_MODEL ?? "gpt-4o"  // Only uses default if env var is null/undefined
  process.env.AGENT_MODEL || "gpt-4o"  // Also uses default if env var is an empty string
  ```

- **Merging config sources:** A clean approach is to start with defaults, then overlay environment variables, then overlay the config file (or vice versa depending on priority):
  ```typescript
  const config: AgentConfig = {
    provider: "openai",       // default
    model: "gpt-4o",          // default
    apiKey: "",               // no default — must be provided
    systemPrompt: undefined,  // optional
  };
  // Then override from env vars, then from config file
  ```

### ??? example "Reference implementation for config.ts"

Here's a complete working example. Study it, understand each part, then write your own version:

```typescript
import fs from "fs";
import path from "path";

export interface AgentConfig {
  model: string;
  apiKey: string;
  provider: "openai" | "anthropic";
  systemPrompt?: string;
}

/**
 * Load configuration from multiple sources in priority order:
 * 1. .agentrc.json file (if it exists)
 * 2. Environment variables (AGENT_PROVIDER, AGENT_API_KEY, AGENT_MODEL)
 * 3. Hardcoded defaults
 *
 * Throws if no API key is found.
 */
export function loadConfig(): AgentConfig {
  // Start with defaults
  const config: AgentConfig = {
    provider: "openai",
    model: "gpt-4o",
    apiKey: "",
  };

  // Try to load from .agentrc.json
  try {
    const filePath = path.join(process.cwd(), ".agentrc.json");
    const fileContent = fs.readFileSync(filePath, "utf-8");
    const fileConfig = JSON.parse(fileContent) as Partial<AgentConfig>;

    // Override defaults with file values (only for fields that exist in the file)
    if (fileConfig.provider) config.provider = fileConfig.provider;
    if (fileConfig.model) config.model = fileConfig.model;
    if (fileConfig.apiKey) config.apiKey = fileConfig.apiKey;
    if (fileConfig.systemPrompt !== undefined) {
      config.systemPrompt = fileConfig.systemPrompt;
    }
  } catch {
    // File doesn't exist or can't be read — that's fine, fall through to env vars
  }

  // Override with environment variables (highest priority for secrets)
  const envProvider = process.env.AGENT_PROVIDER;
  if (envProvider === "openai" || envProvider === "anthropic") {
    config.provider = envProvider;
  }
  if (process.env.AGENT_MODEL) {
    config.model = process.env.AGENT_MODEL;
  }
  if (process.env.AGENT_API_KEY) {
    config.apiKey = process.env.AGENT_API_KEY;
  }

  // Validate: API key must be present from some source
  if (!config.apiKey) {
    throw new Error(
      "No API key found. Set AGENT_API_KEY or create .agentrc.json with an apiKey field."
    );
  }

  return config;
}

/**
 * Return a default system prompt for the agent.
 */
export function getDefaultSystemPrompt(): string {
  return "You are a helpful coding assistant. You can use tools to read files, run commands, and help the user with tasks.";
}
```

### ??? warning "Common Mistakes"

- **Using `JSON.parse` without try/catch:** If the config file has a typo (extra comma, missing quote), `JSON.parse` throws and your entire app crashes. Always wrap it.
- **Overwriting instead of merging:** Don't do `config = fileConfig` — this replaces everything including fields that weren't in the file. Instead, copy individual fields or use spread: `config = { ...defaults, ...fileConfig }`.
- **Forgetting the API key validation:** Without it, your agent starts with an empty API key and fails later with a confusing OpenAI error instead of a clear message at startup.

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `src/config.ts` exports `AgentConfig`, `loadConfig()`, and `getDefaultSystemPrompt()`
- [ ] Running `npm run build` compiles without errors
- [ ] Calling `loadConfig()` with no config file and no env vars throws a clear error about the missing API key

---

## Step 1.5: Create the CLI entry point

**Goal:** Write `src/cli.ts` — the script that runs when you type `npm start`. It loads configuration, verifies everything works, and prints `"Hello agent"`.

### Why This Step Exists

Every command-line tool needs an entry point — a single file that Node.js executes first. The `cli.ts` file is that file. In this milestone it does almost nothing (just loads config and prints a greeting), but its structure sets up the pattern you'll extend in every subsequent milestone.

The **`async main()` pattern** used here is the standard way to write CLI tools in TypeScript:

1. A `main()` function contains all your logic
2. It's `async` so it can use `await` for async operations (API calls, file reads)
3. It's called once at the bottom of the file
4. Errors are caught with try/catch and printed to stderr before exiting

This pattern separates *what* happens (inside `main`) from *when* it happens (the call at the bottom). It also makes testing easy — you can import `main` and call it directly in tests.

### Requirements

#### Part A: Import and load config

```typescript
import { loadConfig } from "./config.js";
```

**Important:** Notice the `.js` extension even though the file is `config.ts`. This is required when using `module: "NodeNext"` — TypeScript rewrites the import at compile time to point to the compiled JavaScript file. If you use `.ts`, the compiler will error with `"Cannot find module './config.ts'"`.

#### Part B: Write the main function

```typescript
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

**What each piece does:**

- `async function main()` — allows using `await` inside (you'll need this in Milestone 3 for LLM calls)
- `const config = loadConfig()` — loads configuration; throws if API key is missing
- `console.log("Hello agent")` — the success output (goes to stdout)
- `process.exit(0)` — exits with code 0 (success). Convention: 0 means "everything worked"
- `catch (error)` — catches any error from config loading or other operations
- `error instanceof Error ? error.message : String(error)` — safely extracts the error message. Not all thrown values are `Error` objects, so we handle both cases
- `console.error(...)` — prints to stderr (not stdout). This follows Unix convention: errors go to stderr, output goes to stdout
- `process.exit(1)` — exits with code 1 (failure). Any non-zero exit code signals an error to the shell

#### Part C: Verify it works

1. **Set a fake API key** (you don't need a real one yet):
   ```bash
   AGENT_API_KEY=fake npm run build && npm start
   ```
   Expected output: `Hello agent`

2. **Try without an API key** to verify the error handling:
   ```bash
   npm run build && npm start
   ```
   Expected output: `Error: No API key found. Set AGENT_API_KEY or create .agentrc.json with an apiKey field.`

3. **Or create a config file** instead of using env vars:
   ```json
   // .agentrc.json
   {
     "apiKey": "fake-key-for-testing",
     "model": "gpt-4o"
   }
   ```
   Then run `npm start` — it should find the key in the file and print `"Hello agent"`.

### ??? tip "Hints"

- **Why not just throw errors instead of catch/exit?** If an error is thrown at the top level without a catch, Node.js prints a long stack trace. By catching it, you control the message and exit code — giving the user a clean, readable error.
- **Why `async main` when nothing is async yet?** The pattern is set up now because Milestone 3 will need `await` for LLM API calls. Adding `async` then would require restructuring the error handling too. Better to get it right once at the start.

### ??? warning "Import Gotcha"

With `module: "NodeNext"`, all relative imports must use `.js` extensions:
- ✅ `import { loadConfig } from "./config.js";`
- ❌ `import { loadConfig } from "./config";` (will compile but crash at runtime)
- ❌ `import { loadConfig } from "./config.ts";` (TypeScript error)

TypeScript is unusual here: you write `.js` in your imports, and the compiler handles translating to the right file. This matches how Node.js resolves ES module imports at runtime.

### ✅ Quick Checkpoint

Before moving on, verify:
- [ ] `npm run build` compiles with zero errors
- [ ] `AGENT_API_KEY=fake npm start` prints `Hello agent`
- [ ] Running without an API key shows a clear error message and exits with code 1
- [ ] Creating `.agentrc.json` with an API key also works

---

## All Done? Check Your Work [:material-arrow-right:](done.md)

Before moving to Milestone 2, follow the verification steps in [done.md](./done.md).
