# LLM header capture

## Status

Implemented without an allowlist or redaction policy.

## Goal

Record the request headers sent to the model provider and the response headers returned by the provider on the current LLM span.

The headers are captured from AI SDK telemetry callbacks rather than OTLP exporter configuration:

- request: `OnStepStartEvent.headers`
- response: `OnStepFinishEvent.response.headers`

These are provider request and response headers. They are unrelated to `OPENCODE_OTLP_HEADERS`, which authenticates telemetry exports to the collector.

## Attribute format

All request headers are collected into one custom span attribute, and all response headers are collected into another:

| Attribute | Encoding |
|-----------|----------|
| `http.request.headers` | JSON string containing a flat header object |
| `http.response.headers` | JSON string containing a flat header object |

Example:

```json
{
  "content-type": "application/json",
  "authorization": "Bearer token"
}
```

Header names are trimmed and lowercased. Entries with an empty normalized name or an `undefined` value are omitted. Header values remain strings rather than single-element lists.

The attributes are custom aggregate attributes. They deliberately do not use the OpenTelemetry per-header shape `http.request.header.<name>` or `http.response.header.<name>`, whose semantic convention represents values as string arrays.

## Duplicate headers

The AI SDK callback type exposes headers as `Record<string, string | undefined>`. By the time the integration receives them, repeated header lines have already been represented or coalesced as one string per key. The integration does not split comma-separated values and cannot reconstruct duplicate wire-level header lines.

## Lifecycle

Request headers are written during `onStepStart`; response headers are written during `onStepFinish`. In a multi-step generation, a later step replaces the previous JSON value on the same LLM span.

No header attribute is emitted when the callback has no headers or every entry is omitted during normalization.

## Security

There is currently no allowlist, denylist, hashing, or redaction. Sensitive values such as these can be exported in plaintext:

- `authorization`
- provider API keys
- `cookie`
- `set-cookie`
- organization or tenant identifiers

The OTLP backend and collector pipeline must enforce the required access and retention policy. A future redaction feature should be applied before JSON serialization in `headerAttribute`.

## Tests

`tests/ai-telemetry.test.ts` verifies that:

- names are lowercased
- undefined values are removed
- values are strings
- request and response maps are emitted as separate JSON attributes
- per-header attributes are not emitted
