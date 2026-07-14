# AI SDK LLM payload capture

## Status

Implemented.

## Goal

Attach the model-facing input and output of each OpenCode LLM call to the existing `opencode.llm` span without modifying OpenCode source or wrapping every provider's `fetch` implementation.

The implementation uses the AI SDK `TelemetryIntegration` API because OpenCode routes `session.llm` model calls through the AI SDK. It observes the normalized call lifecycle independently of the selected provider, so there is no provider allowlist or provider enumeration step.

## Architecture

`src/index.ts` creates the shared `HandlerContext` and registers an integration through `registerAiTelemetry(ctx)`. Plugin disposal removes that context's listener.

`src/ai-telemetry.ts` stores one process-global broker under `globalThis.__opencodePluginOtelAiTelemetry`. The broker registers exactly one AI SDK integration and fans callbacks out to the active plugin listeners. This avoids duplicate global integrations when the plugin is loaded more than once in the same process.

An AI SDK operation is accepted only when both conditions are true:

1. `functionId` is `session.llm`.
2. `metadata.sessionId` matches an entry in `HandlerContext.activeMessageSpans`.

`startMessageSpan` creates that active entry when OpenCode emits an incomplete assistant `message.updated` event. The entry associates the session with the current message ID, LLM span, and optional model-to-tool handoff time. The completed assistant event removes it.

```text
OpenCode message.updated (assistant, incomplete)
  -> startMessageSpan
  -> activeMessageSpans[sessionID] = { messageID, span }

AI SDK onStart
  -> llm.invocation_parameters

AI SDK onStepStart
  -> input.value
  -> llm.input_messages.*
  -> llm.tools.*

AI SDK onStepFinish
  -> output.value
  -> llm.output_messages.*
  -> llmTelemetryOutputs[sessionID:messageID] = true

OpenCode message.part.updated (tool, running)
  -> activeMessageSpans[sessionID].outputEndTime = latest tool start

OpenCode message.updated (assistant, completed)
  -> token/cost/status attributes
  -> preserve AI SDK output when marker is present
  -> end span at outputEndTime when present
  -> clear correlation state
```

Callbacks from other AI SDK functions are ignored. A callback also has no effect when its plugin context has no active LLM span for the supplied session.

## OpenInference attributes

The integration emits both a compact JSON representation and flattened OpenInference message attributes.

| Attribute | Value |
|-----------|-------|
| `input.value` | JSON array of cleaned input messages |
| `input.mime_type` | `application/json` |
| `llm.input_messages.<index>.*` | Flattened OpenInference input message attributes |
| `llm.invocation_parameters` | JSON object containing model invocation controls |
| `llm.tools.<index>.tool.json_schema` | JSON schema for each active function tool |
| `output.value` | JSON array of cleaned assistant messages |
| `output.mime_type` | `application/json` |
| `llm.output_messages.<index>.*` | Flattened OpenInference output message attributes |

Example input:

```json
[
  { "role": "system", "content": "You are OpenCode." },
  {
    "role": "user",
    "content": [{ "type": "text", "text": "Locate the failure." }]
  }
]
```

Example output:

```json
[
  {
    "role": "assistant",
    "content": [
      { "type": "reasoning", "text": "Inspect the latest span first." },
      { "type": "text", "text": "The output attribute was dropped." },
      {
        "type": "tool_use",
        "id": "call_1",
        "name": "bash",
        "arguments": { "command": "pwd" }
      }
    ]
  }
]
```

## Payload normalization

The exported payload intentionally omits AI SDK transport and bookkeeping objects that are not model message content:

- request and response bodies
- provider metadata
- usage and token accounting
- warnings
- step history wrappers
- retry and timeout controls
- functions, symbols, and unsupported content parts

The cleaner retains:

- `system`, `user`, `assistant`, and `tool` message roles
- text and reasoning parts
- image URLs and image data URLs
- tool call ID, name, and JSON arguments
- tool result text and `toolCallId`
- active tool name, description, and input JSON schema
- invocation controls such as token limit, sampling values, stop sequences, seed, tool choice, and provider options

Binary image values are converted to base64 data URLs when a media type is available. Serialization handles bigint values, circular references, and serialization errors without failing the telemetry callback.

## System message recovery

The normal source is `OnStepStartEvent.system`. Some OpenAI OAuth paths move instructions into `providerOptions.<provider>.instructions` and leave `event.system` undefined.

After normalizing the explicit system value and messages, the integration searches provider options for an `instructions` string. It prepends that value as a system message only when no system message is already present.

## Reasoning context

Reasoning parts are preserved when the AI SDK includes them in `event.messages` or `event.content`. The integration does not independently add reasoning from an earlier turn to a later request. Therefore, previous reasoning appears in `input.value` only when OpenCode and the provider adapter actually place it in the next model input.

## Multi-step calls

One OpenCode assistant message can produce multiple AI SDK steps, for example a tool-call step followed by a final generation step. Every `onStepStart` and `onStepFinish` updates the same LLM span attributes.

The latest step replaces earlier values for the same attribute names. The span represents the current or final model-facing call, not a cumulative `steps[]` envelope.

## Output precedence

`onStepFinish` sets `llmTelemetryOutputs[sessionID:messageID]`. When the completed OpenCode assistant event later ends the LLM span, `handleMessageUpdated` checks this marker and does not replace the structured JSON output with its text-only fallback.

If the AI SDK finish callback never arrives, the OpenCode text-part accumulator remains the fallback source for `output.value`.

## State and cleanup

| State | Key | Purpose | Removed |
|-------|-----|---------|---------|
| `activeMessageSpans` | `sessionID` | Resolve an AI SDK callback to the active LLM span | Assistant completion or session sweep |
| `llmTelemetryOutputs` | `sessionID:messageID` | Protect structured output from text fallback | Assistant completion or session sweep |

The maps use `setBoundedMap` where entries are added, limiting retained state if normal completion events are missed.

## Known limitations

- Correlation depends on AI SDK `functionId` and `metadata.sessionId` remaining available on `session.llm` calls.
- Only one active assistant LLM span is tracked per session. A newer message replaces the active entry.
- If two plugin instances simultaneously own an active span with the same session ID in one process, the global broker cannot distinguish them from callback metadata alone.
- The JSON values can contain prompts, reasoning, tool arguments, tool results, and image data. Export destinations must be treated as sensitive.

## Tests

`tests/ai-telemetry.test.ts` covers:

- cleaned OpenInference input, output, tools, and invocation parameters
- system instruction recovery from OpenAI provider options
- current-step replacement behavior
- structured output precedence at assistant completion
- isolation when another plugin context has no matching active span
- rejection of AI SDK operations outside `session.llm`
