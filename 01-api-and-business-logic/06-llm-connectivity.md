# LLM Connectivity — Connecting Spring Boot to AI APIs

> **Last verified:** June 2026

## Two Approaches: Raw HTTP vs Spring AI

| | Raw RestClient/WebClient | Spring AI |
|---|---|---|
| Best when | You need one provider, full control over the request shape | You want provider portability, prompt templates, tool calling |
| Tradeoff | More boilerplate, vendor-locked | Extra dependency, abstraction hides API quirks |

Use raw HTTP to understand what the API actually does. Use Spring AI when you want to swap OpenAI for Claude or Gemini by changing one property.

## Step 1: OpenAI with RestClient

```java
@Configuration
public class OpenAiConfig {
    @Bean
    public RestClient openAiRestClient(
            @Value("${openai.api-key}") String apiKey) {
        return RestClient.builder()
            .baseUrl("https://api.openai.com/v1")
            .defaultHeader("Authorization", "Bearer " + apiKey)
            .defaultHeader("Content-Type", "application/json")
            .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class OpenAiService {
    private final RestClient openAiRestClient;

    public String complete(String prompt, String systemMessage) {
        var body = Map.of(
            "model", "gpt-4.1-mini",
            "messages", List.of(
                Map.of("role", "system", "content", systemMessage),
                Map.of("role", "user", "content", prompt)
            ),
            "max_tokens", 500,
            "temperature", 0.3
        );
        var response = openAiRestClient.post()
            .uri("/chat/completions")
            .body(body)
            .retrieve()
            .body(OpenAiResponse.class);
        return response.choices().getFirst().message().content();
    }
}

public record OpenAiResponse(List<Choice> choices, Usage usage) {}
public record Choice(Message message) {}
public record Message(String role, String content) {}
public record Usage(int prompt_tokens, int completion_tokens, int total_tokens) {}
```

## Step 2: Anthropic (Claude) with RestClient

Anthropic uses a different endpoint shape and the `x-api-key` header:

```java
@Configuration
public class AnthropicConfig {
    @Bean
    public RestClient anthropicRestClient(
            @Value("${anthropic.api-key}") String apiKey) {
        return RestClient.builder()
            .baseUrl("https://api.anthropic.com/v1")
            .defaultHeader("x-api-key", apiKey)
            .defaultHeader("anthropic-version", "2023-06-01")
            .defaultHeader("Content-Type", "application/json")
            .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class AnthropicService {
    private final RestClient anthropicRestClient;

    public String complete(String prompt, String systemMessage) {
        var body = Map.of(
            "model", "claude-sonnet-4-20250514",
            "max_tokens", 500,
            "system", systemMessage,
            "messages", List.of(
                Map.of("role", "user", "content", prompt)
            )
        );
        var response = anthropicRestClient.post()
            .uri("/messages")
            .body(body)
            .retrieve()
            .body(AnthropicResponse.class);
        return response.content().getFirst().text();
    }
}

public record AnthropicResponse(List<ContentBlock> content, AnthropicUsage usage) {}
public record ContentBlock(String type, String text) {}
public record AnthropicUsage(int input_tokens, int output_tokens) {}
```

Key differences from OpenAI:
- Endpoint is `/messages` not `/chat/completions`
- `system` is a top-level field, not a message in the array
- Response wraps content in `ContentBlock` objects with `type` and `text`
- Auth header is `x-api-key` + `anthropic-version`

## Step 3: Spring AI — One API, Multiple Providers

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

For Claude, swap the dependency:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# application.yml — OpenAI
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4.1-mini
          temperature: 0.3
```

```yaml
# application.yml — Claude (just change the prefix)
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          temperature: 0.3
```

The service code stays the same regardless of provider:

```java
@Service
@RequiredArgsConstructor
public class ChatService {
    private final ChatClient chatClient;

    public String complete(String prompt, String systemMessage) {
        return chatClient.prompt()
            .system(systemMessage)
            .user(prompt)
            .call()
            .content();
    }

    public Flux<String> stream(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content();
    }
}
```

## Step 4: Streaming Controller (SSE Endpoint)

```java
@RestController
@RequestMapping("/api/llm")
@RequiredArgsConstructor
public class LlmController {
    private final ChatService chatService;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(@RequestParam String prompt) {
        return chatService.stream(prompt);
    }

    @PostMapping("/complete")
    public ResponseEntity<LlmResult> complete(@RequestBody PromptRequest request) {
        var result = chatService.complete(request.prompt(), request.system());
        return ResponseEntity.ok(new LlmResult(result));
    }
}

public record PromptRequest(String prompt, String system) {}
public record LlmResult(String content) {}
```

## Step 5: Prompt Templates

With Spring AI, templates use `{placeholders}` instead of `%s`:

```java
@Component
public class PromptTemplates {
    private static final String SUMMARIZE_TEMPLATE = """
        You are a helpful assistant. Summarize the following text concisely.

        Text:
        {text}

        Summary:
        """;

    private static final String CLASSIFY_TEMPLATE = """
        Classify the following text into one of these categories:
        {categories}

        Text: {text}

        Respond with only the category name.
        """;

    public Prompt summarizePrompt(String text) {
        return PromptTemplate.builder()
            .template(SUMMARIZE_TEMPLATE)
            .variables(Map.of("text", text))
            .build();
    }
}
```

Without Spring AI, use `String.format()` with `%s` — functionally identical but provider-specific code handles the call.

## Step 6: Error Handling for Rate Limits

```java
@Service
@RequiredArgsConstructor
public class ResilientChatService {
    private final ChatClient chatClient;

    @Retryable(
        retryFor = {HttpServerErrorException.class, ResourceAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String completeWithRetry(String prompt) {
        try {
            return chatClient.prompt()
                .user(prompt)
                .call()
                .content();
        } catch (HttpClientErrorException.TooManyRequests e) {
            String retryAfter = e.getResponseHeaders()
                .getFirst("Retry-After");
            long waitSeconds = retryAfter != null
                ? Long.parseLong(retryAfter) : 5;
            throw new RateLimitExceededException(
                "Rate limited. Retry after " + waitSeconds + "s");
        }
    }
}
```

## Token Counting

```java
public int estimateTokens(String text) {
    return (int) (text.length() / 4.0);
}
```

For accurate counting, Spring AI exposes token usage in the response metadata. For raw API calls, check the `usage` field in the response body — both OpenAI and Anthropic return actual token counts.
