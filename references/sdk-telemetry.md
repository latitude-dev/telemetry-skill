# SDK Telemetry (Python & TypeScript)

The official Latitude SDK telemetry wrapper auto-instruments LLM calls. Wrap your LLM-calling function with a capture decorator/function and the SDK handles span creation, token tracking, and trace export.

## Python

### Dependencies

```
latitude-telemetry
```

Plus the LLM provider package (e.g. `openai`).

### Setup

```python
from latitude_telemetry import Telemetry, Instrumentors, TelemetryOptions

telemetry = Telemetry(
    LATITUDE_API_KEY,
    TelemetryOptions(instrumentors=[Instrumentors.OpenAI]),
)
```

The `Instrumentors` enum tells the SDK which LLM client library to auto-instrument. `Instrumentors.OpenAI` patches the OpenAI client.

### Wrapping an LLM call

Decorate the function that calls the LLM with `@telemetry.capture()`:

```python
from openai import OpenAI

@telemetry.capture(project_id=LATITUDE_PROJECT_ID, path=LATITUDE_PROMPT_PATH)
async def my_llm_call(user_input: str):
    client = OpenAI()
    stream = client.chat.completions.create(
        model="gpt-4.1",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": user_input},
        ],
        stream=True,
        stream_options={"include_usage": True},
    )
    for chunk in stream:
        if not chunk.choices:
            continue
        delta = chunk.choices[0].delta
        if delta and delta.content is not None:
            yield delta.content
```

### If using Latitude as a prompt manager

Add `latitude-sdk` and `promptl-ai` to dependencies. Fetch and render the prompt from Latitude instead of hardcoding messages:

```python
from latitude_sdk import Latitude, LatitudeOptions, RenderPromptOptions
from promptl_ai import Adapter

latitude = Latitude(
    LATITUDE_API_KEY,
    LatitudeOptions(
        project_id=int(LATITUDE_PROJECT_ID),
        version_uuid=LATITUDE_PROMPT_VERSION_UUID,  # defaults to "live"
    ),
)

@telemetry.capture(project_id=LATITUDE_PROJECT_ID, path=LATITUDE_PROMPT_PATH)
async def my_llm_call(user_input: str):
    prompt = await latitude.prompts.get(LATITUDE_PROMPT_PATH)
    rendered = await latitude.prompts.render(
        prompt.content,
        RenderPromptOptions(
            parameters={"user_input": user_input},
            adapter=Adapter.OpenAI,
        ),
    )
    client = OpenAI()
    stream = client.chat.completions.create(
        model=rendered.config["model"],
        messages=rendered.messages,
        stream=True,
        stream_options={"include_usage": True},
    )
    for chunk in stream:
        if not chunk.choices:
            continue
        delta = chunk.choices[0].delta
        if delta and delta.content is not None:
            yield delta.content
```

### Key points

- `stream_options={"include_usage": True}` ensures token counts are included in the streamed response so the SDK can capture them.
- The decorator sends traces to Latitude automatically on function completion.

---

## TypeScript

### Dependencies

```
@latitude-data/telemetry
```

Plus the LLM provider package (e.g. `openai`).

### Setup

```typescript
import { LatitudeTelemetry, Instrumentation } from "@latitude-data/telemetry";
import OpenAI from "openai";

const telemetry = new LatitudeTelemetry(LATITUDE_API_KEY, {
  instrumentations: {
    [Instrumentation.OpenAI]: OpenAI,
  },
});
```

Pass the OpenAI constructor (not an instance) to `instrumentations` so the SDK can patch it.

### Wrapping an LLM call

Use `telemetry.capture()` around the code that calls the LLM:

```typescript
await telemetry.capture(
  {
    projectId: Number(LATITUDE_PROJECT_ID),
    path: LATITUDE_PROMPT_PATH,
  },
  async () => {
    const openai = new OpenAI();
    const stream = await openai.chat.completions.create({
      model: "gpt-4.1",
      messages: [
        { role: "system", content: "You are a helpful assistant." },
        { role: "user", content: input },
      ],
      stream: true,
      stream_options: { include_usage: true },
    });

    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content;
      if (content) {
        // process content (e.g. write to response)
      }
    }
  },
);
```

### If using Latitude as a prompt manager

Add `@latitude-data/sdk` to dependencies. Fetch and render the prompt from Latitude instead of hardcoding messages:

```typescript
import { Latitude, Adapters } from "@latitude-data/sdk";

const latitude = new Latitude(LATITUDE_API_KEY, {
  projectId: Number(LATITUDE_PROJECT_ID),
  versionUuid: LATITUDE_PROMPT_VERSION_UUID, // defaults to "live"
});

await telemetry.capture(
  {
    projectId: Number(LATITUDE_PROJECT_ID),
    path: LATITUDE_PROMPT_PATH,
  },
  async () => {
    const prompt = await latitude.prompts.get(LATITUDE_PROMPT_PATH);
    const rendered = await latitude.prompts.render({
      prompt: { content: prompt.content },
      parameters: { user_input: input },
      adapter: Adapters.openai,
    });

    const model = (rendered.config as { model?: string }).model ?? "gpt-4o";
    const openai = new OpenAI();
    const stream = await openai.chat.completions.create({
      model,
      messages: rendered.messages,
      stream: true,
      stream_options: { include_usage: true },
    });

    for await (const chunk of stream) {
      const content = chunk.choices[0]?.delta?.content;
      if (content) {
        // process content
      }
    }
  },
);
```

### Key points

- `stream_options: { include_usage: true }` ensures token counts are captured.
- The `capture()` callback is async -- the SDK waits for it to complete before sending the trace.
- The second argument to `LatitudeTelemetry` constructor takes `instrumentations` which maps provider enum values to their constructor classes.
