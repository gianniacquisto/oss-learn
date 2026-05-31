# Steps for Milestone 5: LLM Integration & Streaming

---

## Step 5.1: Implement `convertToLlm` for your chosen provider

**Goal:** Create a function that converts your internal `AgentMessage[]` into the format expected by your LLM provider.

**Requirements:**
Create in `src/providers/`:

- A `convertToOpenAIMessages(messages: AgentMessage[]): OpenAI.Chat.ChatCompletionMessageParam[]` (or Anthropic equivalent)
  - Converts `UserMessage` → `{ role: "user", content: [...] }`
  - Converts `AssistantMessage` → `{ role: "assistant", content: [...] }` (flatten text content into a single string or array of text blocks)
  - Converts `ToolResultMessage` → `{ role: "tool", tool_call_id: ..., content: ... }` (OpenAI format) or the Anthropic equivalent
  - Filters out `SystemMessage` from the message array and handles it separately as the system parameter

- A `convertToolsToProviderFormat(tools: Tool[]): ProviderTool[]` that transforms your internal `Tool` objects into the provider's tool definition format
  - Maps `tool.name`, `tool.description`, and `tool.parameters` to the provider's expected schema

**Hints:**
- For OpenAI, tools are defined as `{ type: "function", function: { name, description, parameters } }`. Your Zod schema needs to be convertible to JSON Schema (Zod has `.jsonSchema()` or you can use a library like `zod-to-json-schema`).
- For Anthropic, tools are `{ name, description, input_schema }` where `input_schema` is a JSON Schema object.
- System messages go in the `system` parameter of the API call, not in the messages array (for OpenAI).

**Reference pattern:**
```typescript
// providers/openai.ts — structural reference only

import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
import type { Tool } from "../types.js";

export function convertToOpenAIMessages(messages: AgentMessage[]): ChatCompletionMessageParam[] {
  return messages.flatMap((msg) => {
    if (msg.role === "system") return []; // handled separately
    if (msg.role === "user") {
      const content = msg.content.map(c => c.type === "text" ? { type: "text", text: c.text } : c);
      return [{ role: "user", content }];
    }
    if (msg.role === "assistant") {
      // Flatten content blocks — text as string, tool calls as structured objects
      const textContent = msg.content.filter(c => c.type === "text").map(c => c.text).join("\n");
      const toolCalls = msg.content
        .filter(c => c.type === "toolCall")
        .map(c => ({ id: c.id, type: "function", function: { name: c.name, arguments: JSON.stringify(c.arguments) } }));

      if (toolCalls.length > 0) {
        return [{ role: "assistant", tool_calls: toolCalls }];
      }
      return [{ role: "assistant", content: textContent || "" }];
    }
    if (msg.role === "toolResult") {
      const content = msg.content.map(c => c.text).join("\n");
      return [{ role: "tool", tool_call_id: msg.toolCallId, content }];
    }
    return [];
  });
}

export function convertToolsToOpenAIFormat(tools: Tool[]): OpenAI.ChatCompletionTool[] {
  return tools.map(tool => ({
    type: "function",
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.parameters as any, // Will be JSON Schema from Zod
    },
  }));
}
```

---

## Step 5.2: Implement streaming with the OpenAI (or Anthropic) SDK

**Goal:** Replace your mock LLM call with a real streaming API call.

**Requirements:**
Create in `src/providers/`:

- A `streamChat` function that:
  - Takes model, apiKey, systemPrompt, messages, and tools as parameters
  - Returns an async iterable of events (similar to your AgentEvent type but provider-specific)
  - Uses the OpenAI SDK's `.stream()` method (or Anthropic's equivalent)

- The event stream should emit:
  - `{ type: "text_delta", text: string }` — a new token of text
  - `{ type: "tool_call_start", id: string, name: string }` — start of a tool call
  - `{ type: "tool_call_delta", id: string, argumentsDelta: string }` — partial JSON for a tool call argument
  - `{ type: "done", message: AssistantMessage }` — the complete accumulated response

**Hints:**
- Accumulate partial data in a local object as events arrive. When `done` fires, assemble the final `AssistantMessage`.
- For tool calls, you'll receive JSON strings incrementally. Accumulate them and parse at the end (or use a streaming JSON parser if you want to be fancy).
- The OpenAI SDK's stream returns chunks — each chunk has partial data that needs to be merged with previous chunks.

**Reference pattern:**
```typescript
// providers/stream.ts — structural reference only

export interface StreamEvent {
  type: "text_delta" | "tool_call_start" | "tool_call_delta" | "done";
  text?: string;
  id?: string;
  name?: string;
  argumentsDelta?: string;
  message?: AssistantMessage;
}

export async function* streamChat(
  model: string,
  apiKey: string,
  systemPrompt: string,
  messages: ChatCompletionMessageParam[],
  tools?: OpenAI.ChatCompletionTool[],
): AsyncIterable<StreamEvent> {
  const openai = new OpenAI({ apiKey });

  const stream = await openai.chat.completions.create({
    model,
    messages: [{ role: "system", content: systemPrompt }, ...messages],
    tools,
    stream: true,
  });

  let accumulatedText = "";
  let toolCalls: Array<{ id: string; name: string; args: string }> = [];

  for await (const chunk of stream) {
    for (const delta of chunk.choices) {
      if (delta.delta.content) {
        yield { type: "text_delta", text: delta.delta.content };
        accumulatedText += delta.delta.content;
      }
      if (delta.delta.tool_calls?.[0]) {
        const tc = delta.delta.tool_calls[0];
        if (!toolCalls.find(t => t.id === tc.id)) {
          toolCalls.push({ id: tc.id, name: tc.function!.name!, args: tc.function!.arguments || "" });
          yield { type: "tool_call_start", id: tc.id, name: tc.function!.name! };
        } else if (tc.function?.arguments) {
          yield { type: "tool_call_delta", id: tc.id, argumentsDelta: tc.function.arguments };
        }
      }
    }
  }

  // Assemble final message
  const assistantMessage: AssistantMessage = {
    role: "assistant",
    content: accumulatedText ? [{ type: "text", text: accumulatedText }] : [],
    model,
    provider: "openai",
    stopReason: toolCalls.length > 0 ? "toolCalls" : "stop",
    timestamp: Date.now(),
  };

  if (toolCalls.length > 0) {
    assistantMessage.content = toolCalls.map(tc => ({
      type: "toolCall" as const,
      id: tc.id,
      name: tc.name,
      arguments: JSON.parse(tc.args),
    }));
  }

  yield { type: "done", message: assistantMessage };
}
```

---

## Step 5.3: Update the loop to use streaming events

**Goal:** Modify `runLoop` to consume the stream and emit your agent's event types as tokens arrive.

**Requirements:**
Update `src/loop.ts`:

- Replace the mock LLM call with a call to `streamChat()` from your provider module
- Iterate over the stream events and:
  - For `text_delta`: accumulate text in a partial message, emit `message_update` events
  - For `tool_call_start` / `tool_call_delta`: accumulate tool call data in the partial message
  - For `done`: finalize the assistant message, emit `message_end`, and return it

- The key change: instead of getting one response at a time, you now get incremental updates. Your event emission needs to keep up with the stream without blocking it.

**Hints:**
- Use an async generator or a callback-based approach for emitting events during streaming. Each text delta should trigger a `message_update` event that consumers can render immediately.
- The partial message accumulates across deltas — each update modifies the same object. This is why your `AssistantMessage` content needs to be mutable during streaming.

**Reference pattern:**
```typescript
// loop.ts runLoop update — structural reference only (showing streaming integration)

const stream = await streamChat(
  config.model,
  config.apiKey,
  config.systemPrompt,
  llmMessages,
  tools ? convertToolsToOpenAIFormat(tools) : undefined,
);

let partialMessage: AssistantMessage | null = null;

for await (const event of stream) {
  switch (event.type) {
    case "text_delta":
      if (!partialMessage) {
        partialMessage = createPartialAssistantMessage(config.model, config.provider);
        context.messages.push(partialMessage);
        await emit({ type: "message_start", message: { ...partialMessage } });
      }
      // Append text to content
      const lastContent = partialMessage.content[partialMessage.content.length - 1];
      if (lastContent?.type === "text") {
        lastContent.text += event.text;
      } else {
        partialMessage.content.push({ type: "text", text: event.text });
      }
      await emit({ type: "message_update", message: { ...partialMessage } });
      break;

    case "done":
      if (!partialMessage) partialMessage = event.message;
      await emit({ type: "message_end", message: partialMessage });
      return partialMessage;
  }
}
```

---

## Step 5.4: Wire the CLI to display streaming output

**Goal:** Update your CLI to render streaming events in real time, showing text as it arrives and tool calls as they're detected.

**Requirements:**
Update `cli.ts`:

- Subscribe to agent events with a renderer that:
  - Prints `[agent_start]`, `[turn_start]` etc. for lifecycle events (prefixed with brackets)
  - For `message_update`: print the text delta immediately (no newline, just append to current line)
  - For `tool_call_start`: print `[TOOL: toolName(args)]` on a new line
  - For `message_end`: print a blank line after the message
  - For `agent_end`: print the final summary

- The output should look like a real conversation — text appearing as it's generated, tool calls shown in brackets.

**Hints:**
- Use `process.stdout.write()` instead of `console.log()` for streaming text (no automatic newline). Append to a buffer and flush on newlines or at the end.
- For tool call output, you might want to show the accumulated arguments as they arrive: `[TOOL: read_file(./README.md)]` — but only after all deltas for that tool call have been received.

**Reference pattern:**
```typescript
// cli.ts event renderer — structural reference only

agent.subscribe((event) => {
  switch (event.type) {
    case "agent_start":
      console.log("\n[Agent started]");
      break;
    case "turn_start":
      console.log("[Turn start]\n> ");
      break;
    case "message_update":
      // Print text deltas without newlines
      const text = event.message.content
        .filter(c => c.type === "text")
        .map(c => c.text)
        .join("");
      process.stdout.write(text);
      break;
    case "tool_call_start":
      console.log(`\n[TOOL: ${event.name}(${JSON.stringify(event.args)})]`);
      break;
    case "message_end":
      console.log("\n");
      break;
    case "agent_end":
      console.log("[Agent finished]");
      break;
  }
});
```

---

## Step 5.5: Test with a real conversation

**Goal:** Verify the full end-to-end flow works with a real LLM.

**Requirements:**
- Set your API key: `AGENT_API_KEY=sk-your-key-here` (or create `.agentrc.json`)
- Run the CLI and prompt it with something that would trigger a tool call, like "Read the file at ./README.md"
- Verify you see streaming text output followed by a tool call execution
- Try a follow-up: after the agent reads the file, ask "What does this file contain?" — verify it uses the tool result from context

**Hints:**
- If the model doesn't trigger tool calls naturally, try adding to your system prompt: "You have access to tools. Use them when appropriate."
- Check that the tool call arguments match what you expect (the path should be `./README.md`, not something else).
- If streaming looks garbled, check that you're accumulating text correctly across deltas — each delta is a *partial* string, not the full accumulated text.

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
