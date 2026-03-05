# Span Attributes Reference

Required OpenTelemetry span attributes for the Raw OTLP approach. Both spans use `SpanKind::CLIENT`.

## Parent Span

Name: `capture-{prompt_path}` (e.g. `capture-wikipedia-article-generator`)

| Attribute              | Value                   | Description                                      |
| ---------------------- | ----------------------- | ------------------------------------------------ |
| `latitude.type`        | `"unresolved_external"` | Tells Latitude this is an externally-traced call |
| `latitude.prompt_path` | `LATITUDE_PROMPT_PATH`  | Path or tag for grouping in the Latitude project |
| `latitude.project_id`  | `LATITUDE_PROJECT_ID`   | Latitude project ID (integer)                    |

## Child Span (Completion)

Name: `chat {model}` (e.g. `chat gpt-4.1`)

Must be a child of the parent span (activated parent context).

### Request attributes (set before the LLM call)

| Attribute                      | Value          | Description                                                 |
| ------------------------------ | -------------- | ----------------------------------------------------------- |
| `latitude.type`                | `"completion"` | Marks this as an LLM completion span                        |
| `gen_ai.operation.name`        | `"completion"` | Operation type                                              |
| `gen_ai.system`                | `"openai"`     | LLM provider identifier (use the actual provider)           |
| `gen_ai.request.model`         | model name     | Requested model (e.g. `"gpt-4.1"`)                          |
| `gen_ai.prompt.{i}.role`       | message role   | Role of the i-th input message (`"system"`, `"user"`, etc.) |
| `gen_ai.prompt.{i}.content`    | message text   | Content of the i-th input message                           |
| `gen_ai.request.messages`      | JSON string    | Full input messages array as JSON                           |
| `gen_ai.request.configuration` | JSON string    | Request config as JSON (e.g. `{"model": "gpt-4.1"}`)        |

### Response attributes (set after the LLM call completes)

| Attribute                        | Value         | Description                                               |
| -------------------------------- | ------------- | --------------------------------------------------------- |
| `gen_ai.completion.0.role`       | `"assistant"` | Role of the output message                                |
| `gen_ai.completion.0.content`    | response text | Full assistant response content                           |
| `gen_ai.response.messages`       | JSON string   | Output messages array as JSON                             |
| `gen_ai.response.model`          | model name    | Actual model used in response (may differ from requested) |
| `gen_ai.response.finish_reasons` | array         | Finish reasons (e.g. `["stop"]`)                          |

### Usage attributes (for token/cost tracking)

| Attribute                        | Value   | Description                        |
| -------------------------------- | ------- | ---------------------------------- |
| `gen_ai.usage.input_tokens`      | integer | Number of prompt/input tokens      |
| `gen_ai.usage.prompt_tokens`     | integer | Same as input_tokens (alias)       |
| `gen_ai.usage.output_tokens`     | integer | Number of completion/output tokens |
| `gen_ai.usage.completion_tokens` | integer | Same as output_tokens (alias)      |

To get token counts from streaming responses, pass `stream_options: { include_usage: true }` (or equivalent) in the LLM request. The usage data typically arrives in the final streamed chunk.

## Error handling

On success, set span status to OK. On error:

- Set span status to ERROR with the error message
- Record the exception on the span (`recordException` / `record_exception`)
- Both spans should be ended in a `finally` / `ensure` block
- Always shut down the tracer provider in the outermost `finally` / `ensure`
