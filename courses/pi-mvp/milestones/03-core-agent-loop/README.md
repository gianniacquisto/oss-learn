# Milestone 3: Core Agent Loop

## What You'll Build

The heart of your agent harness: a function that runs the core loop — send messages to an LLM, receive a response, detect if it contains tool calls, and loop until completion. At this stage, responses will be non-streaming text-only (no tools yet), but the loop structure is in place so you can extend it in Milestone 4.

## Why This Matters

The agent loop is where all the pieces come together. It's a deceptively complex piece of code because it has to handle:
- **State transitions:** idle → processing → idle, with error handling at each step
- **Control flow:** when to stop looping (no tool calls? shouldStopAfterTurn?)
- **Error recovery:** what happens when the LLM returns an error vs. a normal response

In pi.dev, this loop is split into two functions: `runAgentLoop` (for new prompts) and `runAgentLoopContinue` (for continuing from existing context). You'll build both patterns.

## Prerequisites for This Milestone

- Complete Milestones 1–2
- Understand how the OpenAI or Anthropic chat completion API works (request shape, response shape)
