# Feature implementation specifications

This directory documents behavior that is implemented in the current source tree. The documents describe runtime ownership, state transitions, exported attributes, fallback behavior, and test coverage.

| Feature | Document | Primary implementation |
|---------|----------|------------------------|
| AI SDK LLM payload capture and OpenInference normalization | [AI SDK LLM payload capture](./ai-sdk-llm-payload-capture.md) | `src/ai-telemetry.ts` |
| LLM request and response header capture | [LLM header capture](./llm-header-capture.md) | `src/ai-telemetry.ts` |
| LLM, run, and session output lifecycle plus agent attribution | [Span output and agent lifecycle](./span-output-and-agent-lifecycle.md) | `src/handlers/message.ts`, `src/handlers/session.ts` |
| Increased and configurable span attribute capacity | [Span attribute capacity](./span-attribute-capacity.md) | `src/config.ts`, `src/otel.ts` |

The features are connected. AI SDK callbacks enrich an LLM span created from OpenCode events, the OpenCode message lifecycle owns completion and fallback output, and the increased span attribute capacity prevents flattened OpenInference fields from being dropped in long requests.
