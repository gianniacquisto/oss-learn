# Steps for Milestone 4: Real LLM Calls

---

## Step 4.1: Implement message conversion for OpenAI

**Goal:** Create a function that converts your internal `AgentMessage[]` into the format expected by the OpenAI API.

**Requirements:**
Create in `src/providers/` (new directory):

- A `convertToOpenAIMessages` function that takes `AgentMessage[]` and returns `OpenAI.ChatCompletionMessageParam[]`:
  - `UserMessage` → `{ role: "user", content: [...] }`
  - `AssistantMessage` → `{ role: "assistant", content: [...] }`
  - `SystemMessage` → handled separately as the `system` parameter (not in the messages array)
  - `ToolResultMessage` → `{ role: "tool", tool_call_id: ..., content: ... }`

**Hints:**
- System messages go in the `system` parameter of the API call, not in the messages array.
- For assistant messages with text, flatten the content blocks into a single string.
- OpenAI's `ChatCompletionMessageParam` type is imported from the `openai` package.

??? example "Reference pattern"
    === "providers/openai.ts"
        ```typescript
        import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
        import type { AgentMessage } from "../types.js";

        export function convertToOpenAIMessages(
          messages: AgentMessage[],
        ): ChatCompletionMessageParam[] {
          return messages.flatMap((msg) => {
            if (msg.role === "system") return []; // handled separately
            if (msg.role === "user") {
              return [{ role: "user", content: msg.content.map(c => c.text).join("") }];
            }
            if (msg.role === "assistant") {
              return [{ role: "assistant", content: msg.content.map(c => c.text).join("") }];
            }
            if (msg.role === "toolResult") {
              return [{
                role: "tool",
                tool_call_id: msg.toolCallId,
                content: msg.content.map(c => c.text).join(""),
              }];
            }
            return [];
          });
        }
        ```

---

## Step 4.2: Implement streaming with the OpenAI SDK

**Goal:** Replace your mock LLM call with a real streaming API call.

**Requirements:**
In `src/providers/stream.ts`:

- A `streamChat` async generator function that:
  - Takes model, apiKey, systemPrompt, and messages as parameters
  - Uses the OpenAI SDK's `.stream()` method
  - Yields events as tokens arrive:
    - `{ type: "text_delta", text: string }` — a new token of text
    - `{ type: "done", message: AssistantMessage }` — the complete accumulated response

**Hints:**
- Use `for await (const chunk of stream)` to iterate over the OpenAI stream.
- Accumulate text in a local variable as deltas arrive.
- When the stream ends, assemble the final `AssistantMessage` from the accumulated text.
- Import `OpenAI` from the `openai` package.

??? example "Reference pattern"
    === "providers/stream.ts"
        ```typescript
        import OpenAI from "openai";
        import type { ChatCompletionMessageParam } from "openai/resources/chat/completions";
        import type { AssistantMessage } from "../types.js";

        export interface StreamEvent {
          type: "text_delta" | "done";
          text?: string;
          message?: AssistantMessage;
        }

        export async function* streamChat(
          model: string,
          apiKey: string,
          systemPrompt: string,
          messages: ChatCompletionMessageParam[],
        ): AsyncIterable<StreamEvent> {
          const openai = new OpenAI({ apiKey });

          const stream = await openai.chat.completions.create({
            model,
            messages: [{ role: "system", content: systemPrompt }, ...messages],
            stream: true,
          });

          let accumulatedText = "";

          for await (const chunk of stream) {
            for (const delta of chunk.choices) {
              if (delta.delta.content) {
                yield { type: "text_delta", text: delta.delta.content };
                accumulatedText += delta.delta.content;
              }
            }
          }

          // Assemble final message
          const assistantMessage: AssistantMessage = {
            role: "assistant",
            content: accumulatedText ? [{ type: "text", text: accumulatedText }] : [],
            model,
            provider: "openai",
            stopReason: "stop",
            timestamp: Date.now(),
          };

          yield { type: "done", message: assistantMessage };
        }
        ```

---

## Step 4.3: Update the loop to use streaming

**Goal:** Modify `runLoop` to consume the stream and emit your agent's event types as tokens arrive.

**Requirements:**
Update `src/loop.ts`:

- Replace the mock LLM call with a call to `streamChat()` from your provider module
- Iterate over the stream events and:
  - For `text_delta`: accumulate text in a partial message, emit `message_update` events
  - For `done`: finalize the assistant message, emit `message_end`, and return it

**Hints:**
- An async generator (`for await...of`) lets you iterate over events as they arrive.
- The partial message accumulates across deltas — each update modifies the same object.
- You still need to handle the case where the loop has multiple turns (tool calls → results → more LLM calls).

??? example "Reference pattern"
    === "loop.ts — update runLoop (key changes)"
        ```typescript
        import { streamChat } from "./providers/stream.js";
        import { convertToOpenAIMessages } from "./providers/openai.js";

        // ... inside runLoop, replace the mock call with:

        const llmMessages = convertToOpenAIMessages(messages);

        const stream = await streamChat(
          config.model,
          config.apiKey,
          systemPrompt,
          llmMessages,
        );

        let accumulatedText = "";
        let assistantMessage: AssistantMessage | null = null;

        for await (const event of stream) {
          if (event.type === "text_delta") {
            accumulatedText += event.text!;
            assistantMessage = {
              role: "assistant",
              content: [{ type: "text", text: accumulatedText }],
              model: config.model,
              provider: "openai",
              stopReason: "stop",
              timestamp: Date.now(),
            };
            await onEvent({ type: "message_update", message: assistantMessage });
          }

          if (event.type === "done") {
            assistantMessage = event.message!;
            messages.push(assistantMessage);
            await onEvent({ type: "message_end", message: assistantMessage });
            break;
          }
        }
        ```

---

## Step 4.4: Wire the CLI to display streaming output

**Goal:** Update your CLI to render streaming events in real time, showing text as it arrives.

**Requirements:**
Update `cli.ts`:

- Update the event logger to handle streaming events:
  - For `message_update`: print the text delta immediately (no newline, just append to current line)
  - For `message_end`: print a blank line after the message
  - For other events: print them with brackets as before

**Hints:**
- Use `process.stdout.write()` instead of `console.log()` for streaming text — it doesn't add a newline automatically.
- For `message_update`, you want to print only the *new* text, not the entire accumulated message. But since our event includes the full message, you can track the last printed length and only print the difference.

??? example "Reference pattern"
    === "cli.ts — update the event logger"
        ```typescript
        let lastPrintedLength = 0;

        const onEvent = (event) => {
          switch (event.type) {
            case "agent_start":
              console.log("\n[Agent started]");
              break;
            case "turn_start":
              console.log("[Turn started]");
              break;
            case "message_start":
              console.log("[Message started]");
              break;
            case "message_update":
              // Print only the new text since last update
              const fullText = event.message.content.map(c => c.text).join("");
              const newText = fullText.slice(lastPrintedLength);
              if (newText) {
                process.stdout.write(newText);
                lastPrintedLength = fullText.length;
              }
              break;
            case "message_end":
              console.log("\n[Message ended]\n");
              lastPrintedLength = 0;
              break;
            case "turn_end":
              console.log("[Turn ended]");
              break;
            case "agent_end":
              console.log("[Agent finished]");
              break;
          }
        };
        ```

---

## Step 4.5: Test with a real conversation

**Goal:** Verify the full end-to-end flow works with a real LLM.

**Requirements:**
- Set your API key: `AGENT_API_KEY=sk-your-key-here` (or create `.agentrc.json`)
- Run the CLI and prompt it with something simple like "Hello"
- Verify you see streaming text output — text appearing as it's generated, not all at once
- Try a follow-up: ask a question that requires the agent to remember the previous message

**Hints:**
- If you get an authentication error, double-check that your API key is correct and has available credits.
- If the response seems wrong or unhelpful, check your system prompt — it might need more guidance.
- If streaming looks garbled, check that you're accumulating text correctly across deltas.

---

## Final Check for This Milestone

Before moving on, verify everything works by following the instructions in [done.md](./done.md).
