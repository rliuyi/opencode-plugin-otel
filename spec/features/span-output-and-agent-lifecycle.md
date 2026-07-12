# Span output and agent lifecycle

## Status

Implemented.

## Goal

Keep LLM, primary run, and subagent session spans consistent when output arrives through multiple event sources, while assigning the concrete OpenCode agent name instead of leaving `agent.name` as `unknown`.

## Span ownership

| Span | Created from | Completed from | Output source |
|------|--------------|----------------|---------------|
| Primary run (`opencode.session`) | User turn | Existing run lifecycle | Final assistant text |
| Subagent session (`opencode.session`) | `session.created` with a parent | Session lifecycle | Final assistant text |
| LLM (`opencode.llm`) | Incomplete assistant `message.updated` | Completed assistant `message.updated` | AI SDK structured output, otherwise final assistant text |

Primary OpenCode sessions do not create one long-lived root session span. Each user turn has a run span. Subagents have their own session span under the parent trace.

## Text accumulation

Every assistant `message.part.updated` text part is appended to:

```text
messageOutputs[sessionID:messageID]
```

When the assistant message completes, `handleMessageUpdated` reads the accumulated string before clearing it. When text exists, it sets:

- `output.value` to the complete text on the owning run span
- `output.mime_type` to `text/plain` on the owning run span
- the same two attributes on the subagent session span, when present

This makes the final visible answer available at the agent-level span instead of only at the child LLM span.

## LLM output precedence

The LLM span has two possible output producers:

1. AI SDK telemetry writes cleaned structured output with JSON MIME type.
2. OpenCode text parts provide a text-only fallback.

The AI SDK path wins. `llmTelemetryOutputs` records that `onStepFinish` supplied structured output. The assistant completion handler checks this marker before applying fallback attributes.

At LLM span creation, `output.value` and `output.mime_type` are initialized to an empty string and `text/plain`. These reserved attribute keys can later be updated even after many flattened input attributes have been added.

## Agent name resolution

An LLM span resolves the agent in this order:

1. Assistant message agent or `mode` supplied to `startMessageSpan`.
2. Session metadata from `getSessionAgentMeta`.
3. `unknown` when neither source is available.

The incomplete assistant event passes `info.mode` into `startMessageSpan`, so the initial span normally has the concrete value. At completion, `handleMessageUpdated` checks `assistant.agent` and then `assistant.mode` again and updates both the LLM span and `sessionTotals` when a concrete value becomes available.

Updating `sessionTotals.agent` also fixes later session-level logs and attributes that use session-scoped agent metadata.

## Completion and cleanup

On completed assistant messages, the handler:

1. Updates metrics, tokens, cost, finish reason, status, and agent attributes.
2. Propagates final text to the run and subagent session spans.
3. Applies text fallback to the LLM span only when structured AI SDK output is absent.
4. Ends and removes the LLM span.
5. Removes accumulated text, active-span correlation, and the structured-output marker.

A session sweep also ends unfinished LLM spans with an error and clears all state associated with that session.

## Event ordering behavior

- Text parts before completion are accumulated normally.
- Completion without AI SDK output uses accumulated text.
- AI SDK output before completion is preserved when the span ends.
- Completion without text and without AI SDK output leaves the reserved empty output value.
- A session ending before message completion marks the open LLM span as an error.

## Tests

`tests/handlers/spans.test.ts` covers:

- final text on the primary run span
- final text on a subagent session span
- reserved LLM output attributes
- initial agent assignment from assistant mode
- replacement of an unknown session agent at message completion

`tests/ai-telemetry.test.ts` covers structured LLM output preservation and correlation-state cleanup.
