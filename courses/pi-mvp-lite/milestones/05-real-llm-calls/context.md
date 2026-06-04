# Context for Milestone 5: Real LLM Calls

## Background

Connecting to an LLM API seems simple — send a request, get a response. But doing it *well* requires handling several complexities that your mock happily ignores. This milestone replaces `mockLLMCall()` with a real implementation that streams tokens from OpenAI.

### Why Streaming?

Without streaming, you send a request and wait 3-10 seconds for the complete response. The user sees nothing during that time — it feels broken. With streaming, tokens arrive as they're generated (typically 10-50 tokens per second), and you display each one immediately. The user sees text appearing like someone is typing — a much better experience.

Streaming also fits your event system perfectly: each token fires a `message_update` event, which the CLI prints to stdout. The architecture you built in Milestone 3 was designed for this.

### The Three Layers of Integration

```
Layer 1: Message Conversion          Layer 2: API Streaming           Layer 3: Loop Integration
┌─────────────────────────┐         ┌─────────────────────────┐     ┌─────────────────────┐
│ AgentMessage[]          │ convert │ OpenAI SDK .stream()    │────▶│ for await (event)   │
│   UserMessage           │────────▶│ Creates a stream        │     │   emit message_update│
│   AssistantMessage      │         │ of token chunks         │     │   accumulate text   │
│   ToolResultMessage     │         │                         │     │   build final msg   │
│   SystemMessage         │         │ Yields:                 │     └─────────────────────┘
│                         │         │   text_delta (per token)│
│ Translates to:          │         │   done (final message)  │
│ ChatCompletionMsgParam[]│         └─────────────────────────┘
└─────────────────────────┘
```

**Layer 1 (Conversion):** Your internal types don't match OpenAI's API format. The converter is an adapter that translates between them. This keeps your code decoupled from OpenAI — if you switch providers later, you only change the converter.

**Layer 2 (Streaming):** The OpenAI SDK's `.stream()` method returns an async iterator that yields chunks as tokens arrive. Each chunk contains a small piece of text (often just one word or even one character). You accumulate these into a complete message.

**Layer 3 (Loop Integration):** The loop consumes the stream with `for await...of`, emitting events for each token and accumulating text. When the stream ends, it has a complete `AssistantMessage` ready for tool detection.

### Why a Separate Providers Directory?

In `src/providers/`, you'll create files that handle provider-specific logic (OpenAI API calls, message conversion). This keeps your core agent code (`agent.ts`, `loop.ts`) provider-agnostic — it doesn't know whether it's talking to OpenAI, Anthropic, or a mock. The providers are interchangeable implementations of the same contract.

### What About Tool Definitions for OpenAI?

When your agent has tools registered, OpenAI needs to know about them so it can decide when to call one. You send tool definitions as part of the API request:

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "read_file",
        "description": "Read the contents of a file...",
        "parameters": {
          "type": "object",
          "properties": {
            "path": { "type": "string", "description": "File path to read" }
          },
          "required": ["path"]
        }
      }
    }
  ]
}
```

For this milestone, we'll keep tool definitions simple. The full course uses Zod schemas to generate these automatically from your `Tool` interface.

## How This Fits in the Bigger Picture

```text
Before (Milestones 1-4):                    After (Milestone 5):
┌──────────────┐                             ┌──────────────┐
│   cli.ts     │                             │   cli.ts     │
│    ↓         │                             │    ↓         │
│  agent.ts    │                             │  agent.ts    │
│    ↓         │                             │    ↓         │
│   loop.ts    │                             │   loop.ts    │
│    ↓         │                             │    ↓         │
│ mockLLMCall()│     ───replaced by──▶       │ streamChat() │
│ (hardcoded)  │                             │    ↓         │
└──────────────┘                             │ OpenAI API   │
                                             └──────────────┘
```

The only thing that changes is how the LLM response is obtained. The loop structure, event emission, tool detection, and agent state management stay exactly the same. This is why building the mock first was the right choice — you validated all the infrastructure before connecting to the real API.

## Key Decisions You'll Make

- **Streaming vs. non-streaming fallback:** If streaming fails (network hiccup), should the loop retry with non-streaming? For this MVP, we just propagate the error — the CLI shows it and exits.
- **Tool definition format:** How do you convert your `Tool` interface to OpenAI's function calling format? A simple converter that maps `name → name`, `description → description`, `requiredFields → required` is enough for the MVP.
- **Error handling strategy:** API errors (rate limits, auth failures) are caught and converted into agent error events — not thrown as exceptions that crash the program.

## What "Good" Looks Like

By the end of this milestone:
- ✅ Real API calls to OpenAI with proper authentication using your API key
- ✅ Streaming text appears in real time (not all at once after a delay)
- ✅ Event stream emits `message_update` per token during streaming
- ✅ Tool calls from the real LLM trigger your tool execution pipeline
- ✅ Invalid API key produces a clear error message, not a cryptic stack trace
- ✅ Multi-turn conversations work (the agent remembers previous messages)
