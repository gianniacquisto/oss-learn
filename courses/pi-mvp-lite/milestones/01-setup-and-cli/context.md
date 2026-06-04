# Context for Milestone 1: Project Setup & Hello Agent

## Background

Every TypeScript project needs a few things to work well: package configuration, TypeScript compiler settings, and a way to run the code. For an agent harness specifically, you also need to think about how it will be configured — what settings does the agent need at startup?

In pi.dev, the agent is configured through a combination of environment variables (for API keys) and config files (for model selection, thinking level, etc.). You'll build a simplified version that supports both.

## How This Fits in the Bigger Picture

This milestone sets up the project structure. Everything you build in later milestones will live inside this foundation. Think of it like setting up a workshop before you start building — you need the right tools and layout before you can do the actual work.

## Key Decisions You'll Make

- **ES modules vs CommonJS:** ES modules (`"type": "module"`) are the modern standard and what pi.dev uses. CommonJS is the older Node.js default. We'll use ES modules.
- **Config strategy:** will the agent read from `.env`, JSON files, or both? You'll support both for flexibility — environment variables for secrets (like API keys), JSON files for settings (like which model to use).

## What "Good" Looks Like

By the end of this milestone:
- `npm install` sets up all dependencies
- `npm run build` compiles your TypeScript code
- `npm start` prints "Hello agent" to the terminal
- You can set an API key via environment variable and the app uses it

Nothing fancy — just a solid foundation to build on.
