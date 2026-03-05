# Raw OpenTelemetry (OTLP)

Send traces directly to Latitude using any language's OpenTelemetry library. This approach gives full observability (token counts, cost, latency) but requires manually creating spans and setting attributes.

For the full list of required span attributes, see [span-attributes.md](span-attributes.md).

## How it works

1. Configure an OTLP exporter pointing at `https://gateway.latitude.so/api/v4/traces` with Bearer auth
2. Create a **parent span** (`capture-{prompt_path}`) with `latitude.*` attributes
3. Create a **child span** (`chat {model}`) with `gen_ai.*` attributes for the LLM call
4. After the LLM responds, set response attributes (output messages, token usage, finish reason)
5. End both spans and shut down the tracer provider

## OTLP Exporter Setup

All languages follow the same pattern:

- **Endpoint**: `https://gateway.latitude.so/api/v4/traces`
- **Auth header**: `Authorization: Bearer {LATITUDE_API_KEY}`
- **Format**: Protobuf (preferred) or JSON
- **Service name**: any descriptive string (e.g. `latitude-telemetry-{language}`)
- **Span processor**: `SimpleSpanProcessor` (sends spans immediately)

---

## PHP Example

### Dependencies (composer.json)

```json
{
  "require": {
    "open-telemetry/sdk": "^1.0",
    "open-telemetry/exporter-otlp": "^1.0",
    "php-http/guzzle7-adapter": "^1.0"
  }
}
```

Plus the LLM provider package (e.g. `openai-php/client`, `guzzlehttp/guzzle`).

### Exporter setup

```php
use OpenTelemetry\Contrib\Otlp\ContentTypes;
use OpenTelemetry\Contrib\Otlp\OtlpHttpTransportFactory;
use OpenTelemetry\Contrib\Otlp\SpanExporter;
use OpenTelemetry\SDK\Common\Attribute\Attributes;
use OpenTelemetry\SDK\Resource\ResourceInfo;
use OpenTelemetry\SDK\Trace\SpanProcessor\SimpleSpanProcessor;
use OpenTelemetry\SDK\Trace\TracerProvider;

$endpoint = 'https://gateway.latitude.so/api/v4/traces';

$transport = (new OtlpHttpTransportFactory())->create(
    $endpoint,
    ContentTypes::PROTOBUF,
    ['Authorization' => 'Bearer ' . $latitudeApiKey],
);

$exporter = new SpanExporter($transport);

$resource = ResourceInfo::create(Attributes::create([
    'service.name' => 'latitude-telemetry-php',
]));

$tracerProvider = TracerProvider::builder()
    ->addSpanProcessor(new SimpleSpanProcessor($exporter))
    ->setResource($resource)
    ->build();

$tracer = $tracerProvider->getTracer('latitude.manual');
```

### Creating spans and calling the LLM

```php
use OpenTelemetry\API\Trace\SpanKind;
use OpenTelemetry\API\Trace\StatusCode;

// 1. Parent span
$parentSpan = $tracer->spanBuilder("capture-$promptPath")
    ->setSpanKind(SpanKind::KIND_CLIENT)
    ->startSpan();

$parentSpan->setAttribute('latitude.type', 'unresolved_external');
$parentSpan->setAttribute('latitude.prompt_path', $promptPath);
$parentSpan->setAttribute('latitude.project_id', $projectId);

$parentScope = $parentSpan->activate();

try {
    // 2. Child completion span
    $completionSpan = $tracer->spanBuilder("chat $model")
        ->setSpanKind(SpanKind::KIND_CLIENT)
        ->startSpan();

    $completionSpan->setAttribute('latitude.type', 'completion');
    $completionSpan->setAttribute('gen_ai.operation.name', 'completion');
    $completionSpan->setAttribute('gen_ai.system', 'openai'); // or the provider being used
    $completionSpan->setAttribute('gen_ai.request.model', $model);

    // Set input message attributes
    foreach ($messages as $i => $msg) {
        $completionSpan->setAttribute("gen_ai.prompt.$i.role", $msg['role']);
        $completionSpan->setAttribute("gen_ai.prompt.$i.content", $msg['content']);
    }
    $completionSpan->setAttribute('gen_ai.request.messages', json_encode($messages));
    $completionSpan->setAttribute('gen_ai.request.configuration', json_encode(['model' => $model]));

    $completionScope = $completionSpan->activate();

    // 3. Call the LLM (example with OpenAI)
    $openai = \OpenAI::client($openaiApiKey);
    $stream = $openai->chat()->createStreamed([
        'model' => $model,
        'messages' => $messages,
        'stream_options' => ['include_usage' => true],
    ]);

    $fullContent = '';
    $finishReason = null;
    $promptTokens = null;
    $outputTokens = null;
    $responseModel = null;

    foreach ($stream as $chunk) {
        $delta = $chunk->choices[0]->delta->content ?? null;
        if ($delta !== null) {
            $fullContent .= $delta;
        }
        $finishReason = $chunk->choices[0]->finishReason ?? $finishReason;
        $responseModel = $chunk->model ?? $responseModel;
        if (isset($chunk->usage)) {
            $promptTokens = $chunk->usage->promptTokens ?? $promptTokens;
            $outputTokens = $chunk->usage->completionTokens ?? $outputTokens;
        }
    }

    // 4. Set response attributes
    $completionSpan->setAttribute('gen_ai.completion.0.role', 'assistant');
    $completionSpan->setAttribute('gen_ai.completion.0.content', $fullContent);
    $completionSpan->setAttribute('gen_ai.response.messages',
        json_encode([['role' => 'assistant', 'content' => $fullContent]]));

    if ($responseModel !== null) {
        $completionSpan->setAttribute('gen_ai.response.model', $responseModel);
    }
    if ($finishReason !== null) {
        $completionSpan->setAttribute('gen_ai.response.finish_reasons', [$finishReason]);
    }
    if ($promptTokens !== null) {
        $completionSpan->setAttribute('gen_ai.usage.input_tokens', $promptTokens);
        $completionSpan->setAttribute('gen_ai.usage.prompt_tokens', $promptTokens);
    }
    if ($outputTokens !== null) {
        $completionSpan->setAttribute('gen_ai.usage.output_tokens', $outputTokens);
        $completionSpan->setAttribute('gen_ai.usage.completion_tokens', $outputTokens);
    }

    $completionSpan->setStatus(StatusCode::STATUS_OK);
    $completionScope->detach();
    $completionSpan->end();

    $parentSpan->setStatus(StatusCode::STATUS_OK);
} catch (\Throwable $e) {
    $parentSpan->setStatus(StatusCode::STATUS_ERROR, $e->getMessage());
    $parentSpan->recordException($e);
    throw $e;
} finally {
    $parentScope->detach();
    $parentSpan->end();
    $tracerProvider->shutdown();
}
```

---

## Ruby Example

### Dependencies (Gemfile)

```ruby
gem 'opentelemetry-sdk'
gem 'opentelemetry-exporter-otlp'
```

Plus the LLM provider gem (e.g. `ruby-openai`, `faraday`).

### Exporter setup

```ruby
require 'opentelemetry/sdk'
require 'opentelemetry/exporter/otlp'

exporter = OpenTelemetry::Exporter::OTLP::Exporter.new(
  endpoint: 'https://gateway.latitude.so/api/v4/traces',
  headers: { 'Authorization' => "Bearer #{latitude_api_key}" },
  compression: 'none'
)

resource = OpenTelemetry::SDK::Resources::Resource.create(
  'service.name' => 'latitude-telemetry-ruby'
)

tracer_provider = OpenTelemetry::SDK::Trace::TracerProvider.new(resource: resource)
tracer_provider.add_span_processor(
  OpenTelemetry::SDK::Trace::Export::SimpleSpanProcessor.new(exporter)
)

tracer = tracer_provider.tracer('latitude.manual')
```

### Creating spans and calling the LLM

```ruby
# 1. Parent span
parent_span = tracer.start_span("capture-#{prompt_path}", kind: :client)
parent_span.set_attribute('latitude.type', 'unresolved_external')
parent_span.set_attribute('latitude.prompt_path', prompt_path)
parent_span.set_attribute('latitude.project_id', project_id)

parent_ctx = OpenTelemetry::Trace.context_with_span(parent_span)
parent_token = OpenTelemetry::Context.attach(parent_ctx)

begin
  messages = [
    { role: 'system', content: 'Your system prompt here' },
    { role: 'user', content: user_input }
  ]

  # 2. Child completion span
  completion_span = tracer.start_span("chat #{model}", kind: :client)
  completion_span.set_attribute('latitude.type', 'completion')
  completion_span.set_attribute('gen_ai.operation.name', 'completion')
  completion_span.set_attribute('gen_ai.system', 'openai') # or the provider being used
  completion_span.set_attribute('gen_ai.request.model', model)

  messages.each_with_index do |msg, i|
    completion_span.set_attribute("gen_ai.prompt.#{i}.role", msg[:role])
    completion_span.set_attribute("gen_ai.prompt.#{i}.content", msg[:content])
  end
  completion_span.set_attribute('gen_ai.request.messages', JSON.generate(messages))
  completion_span.set_attribute('gen_ai.request.configuration', JSON.generate(model: model))

  completion_ctx = OpenTelemetry::Trace.context_with_span(completion_span)
  completion_token = OpenTelemetry::Context.attach(completion_ctx)

  begin
    # 3. Call the LLM (example with OpenAI)
    client = OpenAI::Client.new(access_token: openai_api_key)

    full_content = ''
    finish_reason = nil
    prompt_tokens = nil
    output_tokens = nil
    response_model = nil

    client.chat(parameters: {
      model: model,
      messages: messages.map { |m| m.transform_keys(&:to_s) },
      stream: proc { |chunk, _bytesize|
        delta = chunk.dig('choices', 0, 'delta', 'content')
        full_content += delta if delta

        finish_reason = chunk.dig('choices', 0, 'finish_reason') || finish_reason
        response_model = chunk['model'] || response_model
        if chunk['usage']
          prompt_tokens = chunk.dig('usage', 'prompt_tokens') || prompt_tokens
          output_tokens = chunk.dig('usage', 'completion_tokens') || output_tokens
        end
      },
      stream_options: { include_usage: true }
    })

    # 4. Set response attributes
    completion_span.set_attribute('gen_ai.completion.0.role', 'assistant')
    completion_span.set_attribute('gen_ai.completion.0.content', full_content)
    completion_span.set_attribute('gen_ai.response.messages',
      JSON.generate([{ 'role' => 'assistant', 'content' => full_content }]))

    completion_span.set_attribute('gen_ai.response.model', response_model) if response_model
    completion_span.set_attribute('gen_ai.response.finish_reasons', [finish_reason]) if finish_reason

    if prompt_tokens
      completion_span.set_attribute('gen_ai.usage.input_tokens', prompt_tokens)
      completion_span.set_attribute('gen_ai.usage.prompt_tokens', prompt_tokens)
    end
    if output_tokens
      completion_span.set_attribute('gen_ai.usage.output_tokens', output_tokens)
      completion_span.set_attribute('gen_ai.usage.completion_tokens', output_tokens)
    end

    completion_span.status = OpenTelemetry::Trace::Status.ok
  rescue StandardError => e
    completion_span.status = OpenTelemetry::Trace::Status.error(e.message)
    completion_span.record_exception(e)
    raise
  ensure
    OpenTelemetry::Context.detach(completion_token)
    completion_span.finish
  end

  parent_span.status = OpenTelemetry::Trace::Status.ok
rescue StandardError => e
  parent_span.status = OpenTelemetry::Trace::Status.error(e.message)
  parent_span.record_exception(e)
  raise
ensure
  OpenTelemetry::Context.detach(parent_token)
  parent_span.finish
  tracer_provider.shutdown
end
```

---

## Adapting to other languages

The pattern is identical for any language with an OpenTelemetry SDK (Go, Java, C#, Rust, etc.):

1. Install the OTel SDK and OTLP exporter for the language
2. Configure the exporter with the Latitude endpoint and Bearer auth header
3. Create the two-span hierarchy (parent + completion child) with the same attribute names
4. Set `gen_ai.usage.*` attributes from the LLM response for token/cost tracking
5. End spans and shut down the provider

The span attribute names and structure are the same regardless of language.
