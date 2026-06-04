---
comments: Milestone 4 of 5
---

# ④ Tool Execution

## What You'll Build

A complete tool registration, detection, validation, and execution system. Your agent will be able to detect when the LLM wants to use a tool, validate the arguments, run the tool code, and feed the results back into the conversation. By the end you will have:

- **Tool call types** — `ToolCallBlock` added to assistant messages so the LLM can request tool usage
- **Tool registration** — `Agent.registerTool()` method for adding tools at runtime
- **Tool detection helpers** — functions that find tool calls in assistant messages
- **Argument validation** — checking that required fields are present before execution
- **Tool execution** — running a tool's `execute()` function and capturing results (success or error)
- **Loop integration** — the agent loop detects tool calls, executes them, and continues looping until done
- **A working `read_file` tool** — reads files from disk, returns content to the agent

When you run the CLI with a tool call in the mock response, you'll see the full cycle: LLM requests tool → your code runs it → result feeds back → loop continues.

## Why This Matters

Tools are what make agents *agentic*. Without tools, your agent is just a text generator — it can talk about reading files but can't actually do it. With tools, it interacts with the real world: reading files, running commands, querying databases, making API calls.

The tool system you build here follows a five-stage pipeline:

```text
1. REGISTER → You define tools (name, description, execute function)
     ↓
2. DETECT   → The loop finds tool calls in the LLM's response
     ↓
3. VALIDATE → Arguments are checked before execution (prevents crashes)
     ↓
4. EXECUTE  → Your code runs the tool and captures output or errors
     ↓
5. FEEDBACK → Results become ToolResultMessages added to conversation
```

This pipeline exists in every production agent system — from OpenAI's function calling to Anthropic's tool use. Understanding it means understanding how agents actually *do* things.

### Key Concepts Introduced Here

??? example "The Inner Loop (Tool Iteration)"
    Milestone 3 built the *outer* loop (one LLM call → one response). This milestone adds the *inner* loop: when the LLM calls a tool, you run it, add the result to context, and call the LLM *again*. The LLM might call another tool, or it might give a final answer. This "act → observe → reason again" cycle is what makes agents iterative.

??? example "Tool Call IDs (Matching Requests to Results)"
    When the LLM calls multiple tools at once, each call gets a unique ID. Your code runs each tool and creates a result with that same ID. The LLM uses the ID to match each result back to its original call — without IDs, the model wouldn't know which result belongs to which tool.

??? example "Error Results (Tools That Fail Gracefully)"
    Tools can fail (file not found, network error, invalid input). Instead of crashing the entire agent, failed tools return `{ isError: true }` with an error message. The LLM sees this and can adapt — retrying with different arguments, trying a different tool, or explaining the error to the user.

## Prerequisites for This Milestone

- **Milestone 3 complete** — you have a working agent loop with event emission
- **Understanding of async error handling** — tools use `try/catch` to return errors gracefully
- **The files from previous milestones** — especially `loop.ts` which gets the biggest update

---

## Steps

| Step | Task | What You'll Learn |
|------|------|-------------------|
| 4.1 | Add tool call types | Extending discriminated unions, tool call structure |
| 4.2 | Update Tool interface with requiredFields | Contract design for tool validation |
| 4.3 | Add `registerTool()` to Agent class | Runtime extensibility of agent capabilities |
| 4.4 | Implement tool call detection helpers | Filtering and extracting from message content |
| 4.5 | Implement argument validation | Preventing crashes from missing or invalid inputs |
| 4.6 | Implement tool execution with error handling | Graceful failure — tools can't crash the agent |
| 4.7 | Wire tools into the loop | The inner loop: detect → execute → feed back → repeat |
| 4.8 | Create a `read_file` tool | Building a concrete, working tool end-to-end |
| 4.9 | Wire the CLI to register and test tools | Full integration: tool registered → called → result returned |

---

[:material-arrow-left: Back to Course](../../README.md){: .md-button }&nbsp;&nbsp;[:material-book-open-variant: Read Full Context](context.md){: .md-button }&nbsp;&nbsp;[:material-flag: Done Checklist](done.md){: .md-button }&nbsp;&nbsp;[:material-arrow-right: Start Building →](steps.md){: .md-button .md-button--primary }
