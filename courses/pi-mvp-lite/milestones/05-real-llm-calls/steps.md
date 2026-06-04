# Steps — Real LLM Calls

[:material-arrow-left: Back to Milestone Overview](README.md){: .md-button }

---

## Step 5.1: Implement message conversion for OpenAI

**Goal:** Create a function that converts your internal `AgentMessage[]` into the format OpenAI's API expects (`ChatCompletionMessageParam[]`).

### Why This Step Exists

Your agent uses custom message types (`UserMessage`, `AssistantMessage`, `ToolResultMessage`, `SystemMessage`) designed for your codebase. OpenAI's API uses its own format with different field names and structure. You need an **adapter function** that translates between the two.

This adapter is critical because it keeps your internal types decoupled from any specific provider. If you later add Anthropic support, you write a new converter — your core agent code never changes. This is the **Adapter Pattern**: a thin layer that translates between incompatible interfaces.

### Requirements

Create `src/providers/openai.ts`:

```typescript
import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
import type { AgentMessage, AssistantMessage } from "../types.js";

/**
 * Convert internal AgentMessage[] to OpenAI's ChatCompletionMessageParam[].
 *
 * Maps:
 *   SystemMessage  → skipped (sent as 'system' parameter separately)
 *   UserMessage    → { role: "user", content: string }
 *   AssistantMessage → { role: "assistant", content: string, tool_calls?: [...] }
 *   ToolResultMessage → { role: "tool", tool_call_id: "...", content: string }
 */
export function convertToOpenAIMessages(
  messages: AgentMessage[],
): ChatCompletionMessageParam[] {
  return messages.flatMap((msg) => {
    // System messages are sent separately as the 'system' parameter
    if (msg.role === "system") return [];

    if (msg.role === "user") {
      return [
        {
          role: "user" as const,
          content: msg.content.map((c) => c.text).join(""),
        },
      ];
    }

    if (msg.role === "assistant") {
      // Extract text content
      const textBlocks = msg.content.filter(
        (c): c is Extract<typeof c, { type: "text" }> => c.type === "text",
      );
      const text = textBlocks.map((c) => c.text).join("");

      // Extract tool calls (if any)
      const toolCallBlocks = msg.content.filter(
        (c): c is Extract<typeof c, { type: "toolCall" }> => c.type === "toolCall",
      );

      const openaiMsg: ChatCompletionMessageParam = {
        role: "assistant" as const,
        content: text || null,
      };

      // Add tool_calls if the assistant called any tools
      if (toolCallBlocks.length > 0) {
        (openaiMsg as any).tool_calls = toolCallBlocks.map((tc) => ({
          id: tc.id,
          type: "function" as const,
          function: {
            name: tc.name,
            arguments: JSON.stringify(tc.arguments),
          },
        }));
      }

      return [openaiMsg];
    }

    if (msg.role === "toolResult") {
      return [
        {
          role: "tool" as const,
          tool_call_id: msg.toolCallId,
          content: msg.content.map((c) => c.text).join(""),
        },
      ];
    }

    return [];
  });
}
```

**Why `flatMap` instead of `map`?** System messages are skipped (return `[]`), so we need to flatten the result. `map` would give us `[[], {...}, {...}]` — `flatMap` removes the empty arrays automatically.

**Why system messages are handled separately?** OpenAI's API accepts system instructions as the first message in the array with `role: "system"`, but it's cleaner to send them explicitly in the API call options. This keeps the converter focused on user/assistant/tool messages.

**Why `JSON.stringify(tc.arguments)` for tool calls?** OpenAI expects tool call arguments as a JSON string (not an object). Your internal types store them as `Record<string, unknown>` — stringify converts to the format OpenAI needs.

### ??? warning "Type Assertion Note"

The `(openaiMsg as any).tool_calls` cast is necessary because OpenAI's TypeScript types distinguish between messages with and without tool calls at the type level. In practice, OpenAI accepts both shapes. The full course uses proper type unions to avoid `as any`.

### ✅ Quick Checkpoint

- [ ] `src/providers/openai.ts` exists with `convertToOpenAIMessages` exported
- [ ] It handles all four message roles (skips system, converts the rest)
- [ ] Running `npm run build` compiles without errors

---

## Step 5.2: Implement streaming with the OpenAI SDK

**Goal:** Create a streaming function that calls OpenAI's API and yields events as tokens arrive.

### Why This Step Exists

The OpenAI SDK supports two modes:
- **Non-streaming:** `openai.chat.completions.create(...)` — waits for the full response, returns it all at once
- **Streaming:** `openai.chat.completions.create(..., { stream: true })` — returns an async iterator that yields chunks as tokens are generated

Streaming gives you a better user experience (text appears in real time) and fits your event system perfectly. Each chunk fires a `message_update` event, which the CLI prints immediately.

The streaming function uses TypeScript's **async generator** syntax (`async function*`) to yield events one at a time. The caller iterates with `for await (const event of stream)`, processing each token without waiting for the complete response.

### Requirements

Create `src/providers/stream.ts`:

```typescript
import OpenAI from "openai";
import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
import type { AssistantMessage, Tool } from "../types.js";
import { convertToOpenAIMessages } from "./openai.js";
import type { AgentMessage } from "../types.js";

/** Events yielded by the streaming function */
export interface StreamEvent {
  type: "text_delta" | "tool_calls" | "done";
  text?: string;           // present for text_delta events
  message?: AssistantMessage; // present for done events
}

/**
 * Call OpenAI's API with streaming enabled.
 * Yields StreamEvent objects as tokens arrive.
 */
export async function* streamChat(
  model: string,
  apiKey: string,
  systemPrompt: string,
  messages: AgentMessage[],
  tools?: Tool[],
): AsyncIterable<StreamEvent> {
  const openai = new OpenAI({ apiKey });

  // Convert internal messages to OpenAI format
  const openaiMessages: ChatCompletionMessageParam[] = [
    { role: "system", content: systemPrompt },
    ...convertToOpenAIMessages(messages),
  ];

  // Convert tools to OpenAI format
  const openaiTools = tools?.length
    ? tools.map((tool) => ({
        type: "function" as const,
        function: {
          name: tool.name,
          description: tool.description,
          parameters: {
            type: "object",
            properties: Object.fromEntries(
              tool.requiredFields.map((field) => [
                field,
                { type: "string", description: `The ${field} parameter` },
              ]),
            ),
            required: tool.requiredFields,
          },
        },
      }))
    : undefined;

  // Create the streaming request
  const stream = await openai.chat.completions.create({
    model,
    messages: openaiMessages,
    tools: openaiTools,
    stream: true,
  });

  let accumulatedText = "";
  const toolCalls: { id: string; name: string; arguments: string }[] = [];

  // Iterate over chunks as they arrive
  for await (const chunk of stream) {
    for (const choice of chunk.choices) {
      const delta = choice.delta;

      // Accumulate text tokens
      if (delta.content) {
        accumulatedText += delta.content;
        yield { type: "text_delta", text: delta.content };
      }

      // Collect tool call information
      if (delta.tool_calls) {
        for (const tc of delta.tool_calls) {
          if (tc.id && !toolCalls.find((t) => t.id === tc.id)) {
            toolCalls.push({ id: tc.id, name: "", arguments: "" });
          }
          const existing = toolCalls.find((t) => t.id === tc?.id);
          if (existing) {
            if (tc.function?.name) existing.name += tc.function.name;
            if (tc.function?.arguments) existing.arguments += tc.function.arguments;
          }
        }
      }
    }
  }

  // Determine stop reason
  let stopReason: AssistantMessage["stopReason"] = "stop";
  if (toolCalls.length > 0) {
    stopReason = "toolCalls";
  }

  // Build the final AssistantMessage
  const content: AssistantMessage["content"] = [];
  if (accumulatedText) {
    content.push({ type: "text", text: accumulatedText });
  }
  for (const tc of toolCalls) {
    content.push({
      type: "toolCall",
      id: tc.id,
      name: tc.name,
      arguments: tc.arguments ? JSON.parse(tc.arguments) : {},
    } as any);
  }

  const assistantMessage: AssistantMessage = {
    role: "assistant",
    content,
    model,
    provider: "openai",
    stopReason,
    timestamp: Date.now(),
  };

  yield { type: "done", message: assistantMessage };
}
```

**Why `async function*` (generator)?** A regular async function returns one value. An async generator (`function*`) yields values one at a time as they become available. This is how streaming works: tokens arrive over time, and each token is yielded immediately rather than buffered.

**Why accumulate text across chunks?** Each chunk from OpenAI contains just a few characters (sometimes one). The `accumulatedText` variable builds up the complete response by concatenating each delta. When the stream ends, you have the full text.

**Why collect tool calls separately?** Tool call information comes in chunks too (the function name and arguments are streamed). You accumulate them alongside text, then assemble everything into the final `AssistantMessage`.

**Why convert tools to OpenAI format?** OpenAI needs to know what tools are available so it can decide whether to call one. The converter maps your `Tool` interface to OpenAI's function calling format. This is a simplified version — the full course uses Zod schemas for proper parameter validation.

### ??? warning "API Key Security"

The API key is passed as a plain string to `new OpenAI({ apiKey })`. This is fine because:
- The key comes from your config (loaded from `.agentrc.json` or `AGENT_API_KEY` env var)
- It's never logged or printed to the console
- It's only used to create the OpenAI client instance

**Never** hardcode an API key in source code. Always load it from environment variables or a config file that's in `.gitignore`.

### ✅ Quick Checkpoint

- [ ] `src/providers/stream.ts` exists with `streamChat` exported
- [ ] It creates an OpenAI client with the provided API key
- [ ] It yields `text_delta` events as tokens arrive
- [ ] It yields a `done` event with the complete `AssistantMessage` when finished
- [ ] Running `npm run build` compiles without errors

---

## Step 5.3: Update the loop to use streaming

**Goal:** Modify `runLoop` in `src/loop.ts` to call `streamChat` instead of `mockLLMCall`, consuming tokens as they arrive and emitting events.

### Why This Step Exists

The loop from Milestone 3 called `mockLLMCall()` which returned a complete `AssistantMessage` instantly. Now you need to consume a stream of token deltas, accumulate them into a message, and emit events for each token. The structure of the loop stays the same — only the "call LLM" step changes.

This is the beauty of your architecture: the loop doesn't care *how* it gets responses. Whether from a mock or a streaming API, the loop receives an `AssistantMessage` and processes it the same way.

### Requirements

Update `src/loop.ts`. Replace the `mockLLMCall` usage in `runLoop` with streaming:

```typescript
import { streamChat } from "./providers/stream.js";
// Remove or keep mockLLMCall for testing — you can conditionally use one or the other

// ... inside runLoop, replace the mock call section:

onEvent({ type: "turn_start" });

// Use streaming API call instead of mock
const assistantMessageResult = await callLLMWithStream(
  config.model,
  config.apiKey,
  config.systemPrompt,
  messages,
  config.tools,
  onEvent,
);

const assistantMessage = assistantMessageResult;
messages.push(assistantMessage);
lastMessage = assistantMessage;

onEvent({ type: "message_end", message: assistantMessage });
onEvent({ type: "turn_end", message: assistantMessage });
```

And add the helper function that consumes the stream:

```typescript
import type { StreamEvent } from "./providers/stream.js";

/**
 * Call the LLM via streaming and emit events as tokens arrive.
 * Returns the complete AssistantMessage when done.
 */
async function callLLMWithStream(
  model: string,
  apiKey: string,
  systemPrompt: string,
  messages: AgentMessage[],
  tools: Tool[],
  onEvent: OnEvent,
): Promise<AssistantMessage> {
  const stream = streamChat(model, apiKey, systemPrompt, messages, tools);

  let accumulatedText = "";
  let assistantMessage: AssistantMessage | null = null;

  onEvent({ type: "message_start", message: { role: "assistant", content: [], model, provider: "openai", stopReason: "stop", timestamp: Date.now() } as AssistantMessage });

  for await (const event of stream) {
    if (event.type === "text_delta") {
      accumulatedText += event.text!;

      // Emit update with current accumulated text
      assistantMessage = {
        role: "assistant",
        content: [{ type: "text", text: accumulatedText }],
        model,
        provider: "openai",
        stopReason: "stop",
        timestamp: Date.now(),
      };
      onEvent({ type: "message_update", message: assistantMessage });
    }

    if (event.type === "done") {
      assistantMessage = event.message!;
      break;
    }
  }

  return assistantMessage!;
}
```

**Why a separate `callLLMWithStream` helper?** It keeps the stream consumption logic isolated from the loop logic. The loop says "get me a response" and doesn't care if it comes from streaming, non-streaming, or a mock. This separation makes testing easier (you can test the stream consumer independently).

**Why emit `message_update` per token?** Each token fires an event that the CLI can print immediately. This is what creates the "text appearing in real time" effect. Without per-token events, you'd only see the complete response after the entire generation finishes.

**Why accumulate text in the loop (not just use the final message)?** The `message_update` events need to show the *current* state of the response (all tokens so far). The CLI uses this to display the growing text. The final `done` event has the complete message, but the updates let the UI stay current during generation.

### ??? tip "Keeping the Mock for Testing"

You might want to keep `mockLLMCall` available for quick testing without API costs. One approach: add a flag in config:

```typescript
const useMock = process.env.USE_MOCK === "true";

if (useMock) {
  assistantMessage = await mockLLMCall(messages);
} else {
  assistantMessage = await callLLMWithStream(...);
}
```

### ✅ Quick Checkpoint

- [ ] `runLoop` calls `callLLMWithStream` instead of `mockLLMCall`
- [ ] The stream consumer emits `message_start`, `message_update` (per token), and `message_end` events
- [ ] Running `npm run build` compiles without errors
- [ ] With a valid API key, the CLI shows streaming text output

---

## Step 5.4: Wire the CLI for streaming display

**Goal:** Update `src/cli.ts` to render streaming text in real time using `process.stdout.write()` instead of `console.log()`.

### Why This Step Exists

`console.log()` adds a newline after each call — fine for discrete events but terrible for streaming text (each token would appear on its own line). `process.stdout.write()` writes raw text without newlines, letting you build up the response on a single line.

For a clean display, you track how much text was already printed and only write the *new* portion on each update. This avoids re-printing the entire message on every token.

### Requirements

Update `src/cli.ts`:

```typescript
import { loadConfig } from "./config.js";
import { Agent } from "./agent.js";
import { readFileTool } from "./tools/read-file.js";
import type { AgentEvent } from "./types.js";

async function main() {
  try {
    const config = loadConfig();

    const agent = new Agent({
      systemPrompt: config.systemPrompt,
      model: config.model,
    });

    // Register tools
    agent.registerTool(readFileTool);

    // Track how much text we've already printed (for incremental display)
    let lastPrintedLength = 0;

    // Event logger with streaming support
    const onEvent = (event: AgentEvent) => {
      switch (event.type) {
        case "agent_start":
          console.log("\n🤖 Agent thinking...\n");
          break;

        case "turn_start":
          // Silently start the turn — we'll see output from message events
          break;

        case "message_start":
          process.stdout.write("Assistant: ");
          break;

        case "message_update": {
          if (event.message.role === "assistant") {
            const fullText = event.message.content
              .map((c) => (c.type === "text" ? c.text : ""))
              .join("");

            // Only print the NEW text since last update
            if (fullText.length > lastPrintedLength) {
              const newText = fullText.slice(lastPrintedLength);
              process.stdout.write(newText);
              lastPrintedLength = fullText.length;
            }
          } else if (event.message.role === "toolResult") {
            // Tool result — print on its own line
            const toolMsg = event.message as any;
            const status = toolMsg.isError ? "❌" : "✅";
            console.log(`\n  [${status} ${toolMsg.toolName}]`);
            const text = toolMsg.content?.map((c: any) => c.text).join("") || "";
            if (text.length > 80) {
              console.log(`    ${text.slice(0, 80)}...`);
            } else {
              console.log(`    ${text}`);
            }
          }
          break;
        }

        case "message_end": {
          if (event.message.role === "assistant") {
            console.log("\n"); // newline after the assistant's message
            lastPrintedLength = 0;
          }
          break;
        }

        case "turn_end":
          break;

        case "agent_end":
          console.log(`Done. (${event.messages.length} messages in conversation)\n`);
          break;
      }
    };

    // Test with a real prompt
    await agent.prompt("Explain what TypeScript interfaces are in 2 sentences.", onEvent);

    process.exit(0);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    console.error(`\nError: ${message}`);
    process.exit(1);
  }
}

main();
```

**Why `process.stdout.write()` instead of `console.log()`?** `console.log()` appends `\n` (newline) after every call. `process.stdout.write()` writes raw bytes — no newlines, no formatting. For streaming text, you want each token to appear next to the previous one on the same line.

**Why track `lastPrintedLength`?** Each `message_update` event contains the *full* accumulated text so far (not just the new token). If you printed the full text every time, you'd see duplicate output. By tracking how much was already printed, you only write the *new* portion:
```
Event 1: "Hello"        → print "Hello"       (lastPrintedLength = 5)
Event 2: "Hello world"  → print " world"      (lastPrintedLength = 11)
Event 3: "Hello world!" → print "!"           (lastPrintedLength = 12)
```

**Why reset `lastPrintedLength` on `message_end`?** When one message finishes and the next begins, you start counting from zero again. Otherwise, the next message would try to slice from the wrong position.

### ✅ Quick Checkpoint

- [ ] `npm run build` compiles without errors
- [ ] `AGENT_API_KEY=sk-... npm start` shows text appearing token by token
- [ ] Tool results display on their own lines with ✅/❌ status
- [ ] The conversation transcript count is printed at the end

---

## Step 5.5: Test with a real conversation

**Goal:** Verify the full end-to-end flow works with a real LLM — text streaming, multi-turn conversation, and tool usage.

### Why This Step Exists

Everything you've built has been verified in isolation (types compile, mock works, tools execute). Now you need to verify the complete system works with a *real* LLM:
- Does streaming actually show text in real time?
- Does the agent remember previous messages (multi-turn)?
- Does the LLM call tools when appropriate?
- Do errors produce clear messages instead of crashes?

### Requirements

#### Test 1: Basic conversation

```bash
AGENT_API_KEY=sk-your-key-here npm run build && npm start
```

Expected: Text appears token by token. The response should be a coherent explanation of TypeScript interfaces.

#### Test 2: Multi-turn memory

After the first response completes, modify your CLI temporarily to send a second prompt:

```typescript
await agent.prompt("Explain what TypeScript interfaces are in 2 sentences.", onEvent);
await agent.prompt("Now explain type aliases — how are they different?", onEvent);
```

Expected: The second response should reference or contrast with interfaces (proving the agent remembers the conversation).

#### Test 3: Tool usage

```bash
AGENT_API_KEY=sk-your-key-here npm run build && npm start
```

With the prompt `"Read the package.json file"`, expected flow:
1. LLM decides to call `read_file` tool
2. Your code executes the tool and reads `package.json`
3. Tool result is fed back to LLM
4. LLM generates a response about the file contents

#### Test 4: Error handling

Set an invalid API key:
```bash
AGENT_API_KEY=invalid-key npm run build && npm start
```

Expected: A clear error message like `"Error: Error code: 401 — Invalid API key"` instead of a raw stack trace.

### ??? tip "Debugging Tips"

- **Response seems wrong:** Check your system prompt. If it's too vague, the LLM might not behave as expected. Try: `"You are a helpful coding assistant. Be concise and accurate."`
- **Streaming looks garbled:** Check that you're accumulating text correctly. Each delta should be appended to `accumulatedText`, not replacing it.
- **Tool not being called:** The LLM decides when to call tools based on the system prompt and message context. If it doesn't call a tool, try a more explicit prompt: `"Use the read_file tool to read package.json and tell me what dependencies it has."`

### ??? example "Expected output for a basic conversation"

```
🤖 Agent thinking...

Assistant: TypeScript interfaces define the shape that objects must conform to — they specify required properties and their types. Unlike classes, interfaces are purely structural and disappear after compilation, making them ideal for defining contracts between components.

Done. (2 messages in conversation)
```

Notice the text appeared incrementally (not all at once), and the final message count shows the conversation has a user message + assistant response.

### ✅ Quick Checkpoint

- [ ] Real streaming text appears token by token (not all at once)
- [ ] Multi-turn conversation works (agent remembers previous messages)
- [ ] Tool calls are detected and executed when the LLM requests them
- [ ] Invalid API key produces a clear error message, not a crash

---

## All Done? Check Your Work [:material-arrow-right:](done.md)

This is the final milestone. Follow the verification steps in [done.md](./done.md).
