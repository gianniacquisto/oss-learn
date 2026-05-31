# Context for Milestone 1: Project Setup & Configuration

## Background

Every TypeScript project needs a few things to work well: package configuration, TypeScript compiler settings, and a way to run the code. For an agent harness specifically, you also need to think about how it will be configured — what settings does the agent need at startup?

In pi.dev, the agent is configured through a combination of environment variables (for API keys) and config files (for model selection, thinking level, etc.). You'll build a simplified version that supports both.

## How This Fits in the Bigger Picture

This milestone sets up the project structure. Everything you build in later milestones will live inside this foundation. The key decisions you make now:
- **Project layout:** single package vs. monorepo? For this MVP, a flat structure is fine — you'll have one `src/` directory with your agent code and one `bin/` or `cli.ts` entry point.
- **Config strategy:** will the agent read from `.env`, JSON files, or both? You'll support both for flexibility.

## Key Decisions You'll Make

- Should you use ES modules (`"type": "module"` in package.json) or CommonJS? ES modules are the modern standard and what pi.dev uses, but they require Node 18+ with `--experimental-vm-modules` for some features. For this MVP, ES modules are fine.
- How will config be loaded — eagerly at startup or lazily on first use? Eager loading is simpler; lazy loading defers cost until needed.
