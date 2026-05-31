# Milestone 4: Tool Execution System

## What You'll Build

A tool registration and execution system that lets the agent call external functions (file reading, command execution, etc.) when the LLM requests it. Tools are registered with a name, description, parameter schema, and execute function. The loop validates arguments against the schema before calling `execute()`, then feeds results back to the model for the next turn.

## Why This Matters

This is what transforms your agent from a chatbot into an *agent* — it can take actions in the world based on what it learns from the LLM. In pi.dev, tools include file reading/writing, shell command execution, git operations, and more. You'll build a minimal but extensible tool system that demonstrates the core pattern.

## Prerequisites for This Milestone

- Complete Milestones 1–3
- Comfortable with TypeScript generics (you used them in Milestone 2)
- Understand JSON Schema basics (types, required fields, descriptions)
