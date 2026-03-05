# Gateway Mode

Latitude acts as the LLM gateway -- the app sends the prompt path and parameters, Latitude calls the configured provider, and streams back the response. Telemetry is handled automatically by Latitude in this mode, so no separate telemetry setup is needed.

Requires a prompt saved in Latitude (the prompt path must point to an existing prompt in the project).

## Option A: SDK (Python, TypeScript)

### Python

#### Dependencies

```
latitude-sdk
```

#### Setup and usage

```python
import asyncio
from latitude_sdk import Latitude, LatitudeOptions, RunPromptOptions

latitude = Latitude(
    LATITUDE_API_KEY,
    LatitudeOptions(
        project_id=int(LATITUDE_PROJECT_ID),
        version_uuid=LATITUDE_PROMPT_VERSION_UUID,  # defaults to "live"
    ),
)

async def run_prompt_stream(user_input: str):
    queue = asyncio.Queue()
    sentinel = object()

    async def on_event(event):
        if isinstance(event, dict) and event["type"] == "text-delta":
            queue.put_nowait(event["textDelta"])

    async def run():
        try:
            await latitude.prompts.run(
                LATITUDE_PROMPT_PATH,
                RunPromptOptions(
                    parameters={"user_input": user_input},
                    stream=True,
                    on_event=on_event,
                ),
            )
        finally:
            queue.put_nowait(sentinel)

    task = asyncio.create_task(run())
    while True:
        chunk = await queue.get()
        if chunk is sentinel:
            break
        yield chunk
```

### TypeScript

#### Dependencies

```
@latitude-data/sdk
```

#### Setup and usage

```typescript
import { Latitude } from "@latitude-data/sdk";

const latitude = new Latitude(LATITUDE_API_KEY, {
  projectId: Number(LATITUDE_PROJECT_ID),
  versionUuid: LATITUDE_PROMPT_VERSION_UUID, // defaults to "live"
});

latitude.prompts.run(LATITUDE_PROMPT_PATH, {
  parameters: { user_input: input },
  stream: true,
  onEvent: ({ data }) => {
    if (data?.type === "text-delta" && data.textDelta != null) {
      // process text chunk
    }
  },
  onFinished: () => {
    // stream complete
  },
  onError: (err) => {
    // handle error
  },
});
```

---

## Option B: HTTP API (Any language)

For languages without a Latitude SDK, use the "Run a Prompt" HTTP endpoint directly. This works in any language with an HTTP client.

### Endpoint

```
POST https://gateway.latitude.so/api/v3/projects/{projectId}/versions/{versionUuid}/documents/run
```

### Headers

```
Authorization: Bearer {LATITUDE_API_KEY}
Content-Type: application/json
```

### Request body

```json
{
  "path": "your-prompt-path",
  "parameters": {
    "user_input": "Hello world"
  },
  "stream": true
}
```

- `path`: The prompt path in the Latitude project (required)
- `parameters`: Key-value pairs matching the prompt's template variables (optional)
- `stream`: Set to `true` for Server-Sent Events streaming, `false` for a single JSON response (optional, defaults to `false`)

### Non-streaming response

When `stream` is `false`, the response is a single JSON object:

```json
{
  "uuid": "conversation-uuid",
  "conversation": [...],
  "response": {
    "streamType": "text",
    "usage": {
      "promptTokens": 10,
      "completionTokens": 15,
      "totalTokens": 25
    },
    "text": "The generated response text",
    "toolCalls": [],
    "cost": 0.001
  }
}
```

### Streaming response

When `stream` is `true`, the response is a stream of Server-Sent Events (SSE). Listen for `text-delta` events to get text chunks as they arrive.

### curl example

```bash
curl -X POST "https://gateway.latitude.so/api/v3/projects/30403/versions/live/documents/run" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "wikipedia-article-generator",
    "parameters": { "user_input": "Artificial Intelligence" },
    "stream": false
  }'
```

### Key points

- Gateway mode means Latitude handles the LLM inference and telemetry automatically.
- The prompt must exist in the Latitude project at the given path.
- The `versionUuid` can be `live` to use the published version.
- Full API docs: https://docs.latitude.so/guides/api/api-access#7-run-a-prompt
