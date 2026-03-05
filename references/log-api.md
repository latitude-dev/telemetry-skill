# Create Log API

The simplest way to send LLM conversations to Latitude. After the LLM call completes, make a single HTTP POST with the full conversation. Works in any language with an HTTP client -- no OpenTelemetry dependencies needed.

**Trade-off**: Latitude only receives the conversation as a log entry. There is no token count, cost data, or latency breakdown. If you need those, use the [Raw OpenTelemetry approach](raw-opentelemetry.md) instead.

## API Endpoint

```
POST https://gateway.latitude.so/api/v3/projects/{projectId}/versions/{versionUuid}/documents/logs
```

### Headers

```
Authorization: Bearer {LATITUDE_API_KEY}
Content-Type: application/json
```

### Request Body

```json
{
  "path": "your-prompt-path",
  "messages": [
    {
      "role": "system",
      "content": [{ "type": "text", "text": "Your system prompt" }]
    },
    {
      "role": "user",
      "content": [{ "type": "text", "text": "User input" }]
    }
  ],
  "response": "The full assistant response text"
}
```

### Message Format (PromptL)

Messages must use PromptL format. Each message has:
- `role`: `"system"`, `"user"`, or `"assistant"`
- `content`: an array of content blocks, each with `type` and `text`

Convert standard chat messages to PromptL format:

```
Standard:  { "role": "user", "content": "Hello" }
PromptL:   { "role": "user", "content": [{ "type": "text", "text": "Hello" }] }
```

---

## PHP Example

### HTTP client

```php
use GuzzleHttp\Client;

$http = new Client([
    'base_uri' => 'https://gateway.latitude.so',
    'headers' => [
        'Authorization' => 'Bearer ' . $latitudeApiKey,
        'Accept' => 'application/json',
    ],
]);
```

### Full flow: call LLM, then send log

```php
// 1. Call the LLM as usual
$messages = [
    ['role' => 'system', 'content' => 'Your system prompt'],
    ['role' => 'user', 'content' => $userInput],
];

$openai = \OpenAI::client($openaiApiKey);
$response = $openai->chat()->create([
    'model' => $model,
    'messages' => $messages,
]);

$fullContent = $response->choices[0]->message->content;

// 2. Convert messages to PromptL format
$promptLMessages = array_map(function (array $msg): array {
    return [
        'role' => $msg['role'],
        'content' => [['type' => 'text', 'text' => $msg['content']]],
    ];
}, $messages);

// 3. Send the log to Latitude
$url = sprintf(
    '/api/v3/projects/%d/versions/%s/documents/logs',
    $projectId,
    $versionUuid,  // "live" by default
);

$http->post($url, [
    'json' => [
        'path' => $promptPath,
        'messages' => $promptLMessages,
        'response' => $fullContent,
    ],
]);
```

---

## Ruby Example

### HTTP client

```ruby
require 'faraday'
require 'json'

http = Faraday.new(url: 'https://gateway.latitude.so') do |f|
  f.headers['Authorization'] = "Bearer #{latitude_api_key}"
  f.headers['Content-Type'] = 'application/json'
  f.headers['Accept'] = 'application/json'
  f.adapter Faraday.default_adapter
end
```

### Full flow: call LLM, then send log

```ruby
# 1. Call the LLM as usual
messages = [
  { role: 'system', content: 'Your system prompt' },
  { role: 'user', content: user_input }
]

client = OpenAI::Client.new(access_token: openai_api_key)
full_content = ''

client.chat(parameters: {
  model: model,
  messages: messages.map { |m| m.transform_keys(&:to_s) },
  stream: proc { |chunk, _bytesize|
    delta = chunk.dig('choices', 0, 'delta', 'content')
    full_content += delta if delta
  }
})

# 2. Convert messages to PromptL format
promptl_messages = messages.map do |msg|
  {
    role: msg[:role],
    content: [{ type: 'text', text: msg[:content] }]
  }
end

# 3. Send the log to Latitude
url = "/api/v3/projects/#{project_id}/versions/#{version_uuid}/documents/logs"

http.post(url) do |req|
  req.body = JSON.generate(
    path: prompt_path,
    messages: promptl_messages,
    response: full_content
  )
end
```

---

## Adapting to other languages

This is a simple HTTP POST. In any language:

1. Call your LLM provider as you normally would
2. Collect the full assistant response
3. Convert your messages to PromptL format (wrap each message's content string in `[{ "type": "text", "text": "..." }]`)
4. POST to `https://gateway.latitude.so/api/v3/projects/{projectId}/versions/{versionUuid}/documents/logs` with the JSON body containing `path`, `messages`, and `response`

No special libraries required beyond an HTTP client.
