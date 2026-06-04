# Context for Milestone 4: Tool Execution

## Background

Tools are the bridge between the agent's reasoning and the real world. When an LLM decides it needs to read a file, run a command, or query a database, it does so by emitting a tool call — a structured request that says "execute function X with these arguments." The agent harness is responsible for:

1. **Registering** tools (making them discoverable)
2. **Detecting** tool calls in LLM responses
3. **Validating** arguments
4. **Executing** the tool and capturing results
5. **Formatting** results back into a message the LLM can understand

## How This Fits in the Bigger Picture

This milestone extends your loop from Milestone 3. The loop structure doesn't change much — you just add a step between "get response" and "check if done": "if there are tool calls, execute them and feed results back." The key insight: after executing tools, you go back to the LLM with the tool results appended to context. This is what makes agents iterative — they reason → act → observe → reason again.

## Key Decisions You'll Make

- **Sequential vs. parallel execution:** For this milestone, execute tools one at a time. Parallel execution is more complex and comes later.
- **Error handling:** Should tools throw exceptions or return error results? We'll use the latter — it lets the loop continue even if one tool fails.
- **Argument validation:** How strict should validation be? We'll use simple checks (is it a string? is it present?) rather than a full schema library.

## What "Good" Looks Like

By the end of this milestone:
- You can register tools with the agent
- The loop detects tool calls in LLM responses
- Arguments are validated before execution
- Tool results are appended to context and the loop continues
- You have at least one working tool (like `read_file`)
