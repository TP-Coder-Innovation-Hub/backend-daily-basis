# LLM Connectivity — Connecting Spring Boot to AI APIs

## The Goal

Your Spring Boot service needs to call OpenAI, Anthropic, or any LLM API. This covers API calls, prompt templates, streaming responses, and error handling.

## Step 1: Basic API Call with RestClient

```java
@Configuration
public class LlmConfig {
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
public class LlmService {
    private final RestClient openAiRestClient;

    public String complete(String prompt, String systemMessage) {
        var body = Map.of(
            "model", "gpt-4o",
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

public record OpenAiResponse(
    List<Choice> choices,
    Usage usage
) {}

public record Choice(Message message) {}
public record Message(String role, String content) {}
public record Usage(int prompt_tokens, int completion_tokens, int total_tokens) {}
```

## Step 2: Streaming Responses (SSE)

Streaming delivers tokens as they arrive instead of waiting for the full response.

```java
@Service
@RequiredArgsConstructor
public class StreamingLlmService {
    private final WebClient webClient;

    public Flux<String> streamComplete(String prompt) {
        var body = Map.of(
            "model", "gpt-4o",
            "messages", List.of(
                Map.of("role", "user", "content", prompt)
            ),
            "stream", true
        );
        return webClient.post()
            .uri("/chat/completions")
            .bodyValue(body)
            .retrieve()
            .bodyToFlux(String.class)
            .filter(line -> !line.equals("[DONE]"))
            .map(this::extractContent)
            .filter(content -> !content.isEmpty());
    }

    private String extractContent(String json) {
        try {
            var node = new ObjectMapper().readTree(json);
            return node.at("/choices/0/delta/content").asText("");
        } catch (Exception e) {
            return "";
        }
    }
}
```

## Step 3: Streaming Controller (SSE Endpoint)

```java
@RestController
@RequestMapping("/api/llm")
@RequiredArgsConstructor
public class LlmController {
    private final StreamingLlmService streamingService;
    private final LlmService llmService;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(@RequestParam String prompt) {
        return streamingService.streamComplete(prompt);
    }

    @PostMapping("/complete")
    public ResponseEntity<LlmResult> complete(@RequestBody PromptRequest request) {
        var result = llmService.complete(request.prompt(), request.system());
        return ResponseEntity.ok(new LlmResult(result));
    }
}

public record PromptRequest(String prompt, String system) {}
public record LlmResult(String content) {}
```

## Step 4: Prompt Templates

```java
@Component
public class PromptTemplates {
    private static final String SUMMARIZE_TEMPLATE = """
        You are a helpful assistant. Summarize the following text concisely.

        Text:
        %s

        Summary:
        """;

    private static final String CLASSIFY_TEMPLATE = """
        Classify the following text into one of these categories:
        %s

        Text: %s

        Respond with only the category name.
        """;

    public String formatSummarize(String text) {
        return String.format(SUMMARIZE_TEMPLATE, text);
    }

    public String formatClassify(String categories, String text) {
        return String.format(CLASSIFY_TEMPLATE, categories, text);
    }
}
```

## Step 5: Error Handling for Rate Limits

```java
@Service
@RequiredArgsConstructor
public class ResilientLlmService {
    private final RestClient openAiRestClient;

    @Retryable(
        retryFor = {HttpServerErrorException.class, ResourceAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String completeWithRetry(String prompt) {
        try {
            return callLlm(prompt);
        } catch (HttpClientErrorException.TooManyRequests e) {
            String retryAfter = e.getResponseHeaders()
                .getFirst("Retry-After");
            long waitSeconds = retryAfter != null
                ? Long.parseLong(retryAfter) : 5;
            throw new RateLimitExceededException(
                "Rate limited. Retry after " + waitSeconds + "s");
        }
    }

    private String callLlm(String prompt) {
        var body = Map.of(
            "model", "gpt-4o",
            "messages", List.of(
                Map.of("role", "user", "content", prompt)
            ),
            "max_tokens", 200
        );
        var response = openAiRestClient.post()
            .uri("/chat/completions")
            .body(body)
            .retrieve()
            .body(OpenAiResponse.class);
        return response.choices().getFirst().message().content();
    }
}
```

## Token Counting

Track tokens to stay within model limits and control costs:

```java
public int estimateTokens(String text) {
    return (int) (text.length() / 4.0); // rough estimate: ~4 chars per token
}
```

For accurate counting, use OpenAI's tokenizer library (`tiktoken` via JNI or a Java port).
