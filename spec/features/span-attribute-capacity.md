# Span attribute capacity

## Status

Implemented and configurable.

## Problem

OpenTelemetry SDK span limits default to a relatively small attribute count. Flattened OpenInference messages consume one attribute for each role, content part, tool call field, and tool schema field. Long LLM requests can therefore reach the limit before output is attached.

When the limit is reached, later new attributes are dropped by the SDK. This produced traces where early LLM spans had output but later spans in the same conversation showed `output.value` as undefined, even though the AI SDK finish callback executed.

## Configuration

The plugin configures `BasicTracerProvider.spanLimits.attributeCountLimit`.

| Source | Name | Precedence |
|--------|------|------------|
| Plugin option | `spanAttributeCountLimit` | Highest |
| Environment | `OPENCODE_SPAN_ATTRIBUTE_COUNT_LIMIT` | Second |
| Built-in default | `4096` | Fallback |

Only positive integer values are accepted. Invalid, zero, or negative values fall through to the next configuration source or the default.

Example environment configuration:

```bash
export OPENCODE_SPAN_ATTRIBUTE_COUNT_LIMIT=8192
```

Example plugin option:

```json
[
  "@devtheops/opencode-plugin-otel",
  { "spanAttributeCountLimit": 8192 }
]
```

The resolved value is logged at plugin startup and passed from `src/index.ts` to `setupOtel` in `src/otel.ts`.

## Output reservation

Capacity is complemented by reserving `output.value` and `output.mime_type` when an LLM span starts. Subsequent AI SDK or fallback output updates those existing keys.

This protects the core output fields if a request later approaches the limit, but it does not protect arbitrary flattened fields beyond the configured capacity. Operators should increase the limit for unusually large prompts or tool catalogs.

## Trade-offs

A higher limit allows more complete LLM payloads but can increase span memory, serialization cost, network usage, and backend storage. The configured limit controls attribute count, not the byte length of individual attribute values.

This feature does not truncate prompt text, reasoning text, image data URLs, tool arguments, or header JSON values.

## Tests

- `tests/config.test.ts` verifies the default, environment value, invalid-value fallback, and plugin-option precedence.
- `tests/otel.test.ts` creates a span with 256 attributes and verifies that the default provider configuration retains the last attribute with no dropped attributes.
- `tests/handlers/spans.test.ts` verifies that LLM output keys exist when the span is created.
