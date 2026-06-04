# Context for Milestone 4: Tool Execution

## Background

Tools are the bridge between the agent's reasoning and the real world. When an LLM decides it needs to read a file, run a command, or query a database, it doesn't *do* those things itself — it emits a structured request ("call `read_file` with path='README.md'"). Your code detects that request, runs the actual function, and feeds the result back.

This milestone adds the tool execution pipeline to your existing loop from Milestone 3. The loop structure gets one critical addition: between "get response" and "check if done," there's now a step that says "if there are tool calls, execute them and feed results back."

### The Tool Pipeline in Detail

```
LLM response contains: "I'll read the file" + tool call { name: "read_file", args: { path: "README.md" } }
     │
     ▼
┌─────────────────┐
│ 1. DETECT       │ Your code scans assistant message content for blocks with type: "toolCall"
└────────┬────────┘
         │ found tool calls?
         ▼
┌─────────────────┐
│ 2. FIND TOOL    │ Look up the registered tool by name (agent.state.tools.find(t => t.name === ...))
└────────┬────────┘
         │ tool exists?
         ▼
┌─────────────────┐
│ 3. VALIDATE     │ Check that required arguments are present (e.g., "path" must exist)
└────────┬────────┘
         │ valid?
         ▼
┌─────────────────┐
│ 4. EXECUTE      │ Call tool.execute(toolCallId, args) — runs the actual code
└────────┬────────┘
         │ result received
         ▼
┌─────────────────┐
│ 5. FEED BACK    │ Create a ToolResultMessage with the output, append to conversation
└────────┬────────┘
         │
         ▼
   Loop continues — call LLM again with tool results in context
```

### Why Tool Call IDs?

When the LLM calls multiple tools at once (e.g., "read file A" and "read file B"), each call gets a unique ID like `"call_abc123"`. Your code:
1. Runs tool A with ID `"call_abc123"` → gets result
2. Runs tool B with ID `"call_def456"` → gets result
3. Creates `ToolResultMessage` with `toolCallId: "call_abc123"` for result A
4. Creates `ToolResultMessage` with `toolCallId: "call_def456"` for result B

The LLM uses these IDs to match each result back to its original call. Without IDs, the model couldn't distinguish which result belongs to which tool — especially if both tools return similar output.

### Why Validate Before Execute?

Running a tool with missing arguments causes runtime errors (e.g., `fs.readFile(undefined)` throws). By validating required fields *before* execution:
- You get clear error messages ("Missing required field 'path'") instead of cryptic stack traces
- The LLM sees the validation error and can retry with correct arguments
- The agent loop never crashes from a bad tool call

### Sequential vs. Parallel Execution

This milestone executes tools **one at a time** (sequentially). This is simpler to implement and easier to debug. In production systems, tools are often executed in parallel (all independent calls run simultaneously) — but that requires managing concurrent async operations and handling partial failures. For an MVP, sequential is the right choice.

## How This Fits in the Bigger Picture

```text
                    Milestone 3 Loop (what you have)
                    ┌────────────────────────┐
                    │ while(true) {           │
                    │   response = call LLM() │
                    │   push response         │
                    │   emit events           │
                    │   if done → break       │◀── exits after 1 turn
                    │ }                       │
                    └────────────────────────┘

                    Milestone 4 Loop (what you're building)
                    ┌─────────────────────────────────┐
                    │ while(true) {                    │
                    │   response = call LLM()          │
                    │   push response                  │
                    │   emit events                    │
                    │                                  │
                    │   if response has tool calls:    │◀── NEW
                    │     for each tool call:          │
                    │       find tool by name          │
                    │       validate args              │
                    │       execute tool               │
                    │       push result                │
                    │     continue (call LLM again)    │◀── inner loop
                    │                                  │
                    │   if done → break                │
                    │ }                                 │
                    └─────────────────────────────────┘
```

The key change: after getting a response, you check for tool calls. If found, execute them and `continue` the loop (calling LLM again). If no tool calls, check the stop condition as before.

## Key Decisions You'll Make

- **Tool lookup by name vs. map:** We use `tools.find(t => t.name === ...)` (linear search) because you'll have few tools. The full course uses a `Map<string, Tool>` for O(1) lookups at scale.
- **Validation as simple field checks:** We check "is the required field present?" using `field in args`. The full course uses Zod schemas for type-level validation. Simple checks are enough for an MVP.
- **Error results vs. throws:** When a tool fails, it returns `{ isError: true }` instead of throwing. This lets the loop continue and the LLM decide how to handle the error.

## What "Good" Looks Like

By the end of this milestone:
- ✅ The `read_file` tool is registered with the agent
- ✅ A mock response with a tool call triggers execution of `read_file`
- ✅ The tool reads a real file and returns its content as a `ToolResultMessage`
- ✅ The loop continues after tool execution (calls LLM again with results)
- ✅ Missing arguments produce validation errors, not crashes
- ✅ Tool failures return error results that the LLM can see
