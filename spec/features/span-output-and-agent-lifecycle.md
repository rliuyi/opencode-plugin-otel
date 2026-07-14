# Span output and agent lifecycle

## Status

Implemented.

## Goal

Keep LLM, primary run, and subagent run spans consistent when output arrives through multiple event sources, while preserving the task invocation that caused each subagent run.

## Span ownership

| Span | Created from | Completed from | Output source |
|------|--------------|----------------|---------------|
| Primary run (`opencode.session`) | User turn | Existing run lifecycle | Final assistant text |
| Subagent run (`opencode.session`) | Child `chat.message` after task correlation | Child session lifecycle | Final assistant text |
| LLM (`opencode.llm`) | Incomplete assistant `message.updated` | Completed assistant `message.updated`, using the final tool handoff time when tools run | AI SDK structured output, otherwise final assistant text |

OpenCode sessions do not create long-lived entity spans. Every primary or subagent user turn creates a run span. A subagent run is parented to the exact `opencode.tool.task` span that supplied its prompt.

## Task correlation

OpenCode creates a child session before it adds `sessionId` to the task part metadata. `session.created` therefore records subagent identity and metrics without creating a span. The later task part update stores:

```text
pendingSubagentRuns[childSessionID] = [
  {
    parentSessionID,
    taskCallID,
    taskSpanContext,
    background
  }
]
```

The child `chat.message` consumes the first queued entry and starts its run span with `taskSpanContext` as the parent. The same flow applies to synchronous tasks, background tasks, and `task_id` resumptions. Multiple background calls that append work to the same `task_id` retain one FIFO entry per invocation. A background task may end before the child run, but the retained span context preserves the same parent span ID.

Task part updates are idempotent. A second `running` update that adds metadata reuses the existing task span instead of starting another one.

When tool tracing is disabled or task context is unavailable, the subagent run falls back to the active run of `parentSessionID`.

## Text accumulation

Every assistant `message.part.updated` text part is appended to:

```text
messageOutputs[sessionID:messageID]
```

When the assistant message completes, `handleMessageUpdated` reads the accumulated string before clearing it. When text exists, it sets:

- `output.value` to the complete text on the owning run span
- `output.mime_type` to `text/plain` on the owning run span

This makes the final visible answer available at the agent-level span instead of only at the child LLM span.

## LLM output precedence

The LLM span has two possible output producers:

1. AI SDK telemetry writes cleaned structured output with JSON MIME type.
2. OpenCode text parts provide a text-only fallback.

The AI SDK path wins. `llmTelemetryOutputs` records that `onStepFinish` supplied structured output. The assistant completion handler checks this marker before applying fallback attributes.

At LLM span creation, `output.value` and `output.mime_type` are initialized to an empty string and `text/plain`. These reserved attribute keys can later be updated even after many flattened input attributes have been added.

## LLM end time

OpenCode marks an assistant message complete only after local tool execution settles. Using that timestamp directly makes the LLM span include tool runtime. When a tool enters `running`, the handler records its start time on the active assistant message. For multiple tool calls, the latest handoff time is retained.

The completed assistant event still supplies final tokens, cost, finish reason, and structured output. After applying those attributes, the handler ends the span with the recorded handoff time instead of the later assistant completion time. This preserves complete LLM attributes while keeping tool execution outside the LLM span duration. Messages without local tool calls continue to use the assistant completion time.

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
2. Propagates final text to the owning primary or subagent run span.
3. Applies text fallback to the LLM span only when structured AI SDK output is absent.
4. Ends and removes the LLM span using the final tool handoff time when present.
5. Removes accumulated text, active-span correlation, and the structured-output marker.

A session sweep also ends unfinished LLM spans with an error and clears all state associated with that session.

## Event ordering behavior

- Text parts before completion are accumulated normally.
- Completion without AI SDK output uses accumulated text.
- AI SDK output before completion is preserved when the span ends.
- Completion without text and without AI SDK output leaves the reserved empty output value.
- Tool execution time is excluded from the LLM span end time and `duration_ms` attribute.
- A session ending before message completion marks the open LLM span as an error.
- Synchronous and background subagent runs use the initiating task span as their direct parent.
- A resumed `task_id` starts a new subagent run without requiring another `session.created` event.

## Tests

`tests/handlers/spans.test.ts` covers:

- final text on the primary run span
- final text on a subagent run span
- synchronous and background task parentage
- `task_id` resumption without `session.created`
- parallel task correlation and repeated-running idempotency
- reserved LLM output attributes
- initial agent assignment from assistant mode
- replacement of an unknown session agent at message completion

`tests/ai-telemetry.test.ts` covers structured LLM output preservation and correlation-state cleanup.
