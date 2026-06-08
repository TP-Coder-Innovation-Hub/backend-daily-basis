# Serverless — Spring Boot on AWS Lambda

## When Serverless

| Serverless | Container |
|------------|-----------|
| Event-driven, intermittent traffic | Constant, predictable traffic |
| Pay per invocation | Pay for running containers |
| Auto-scale to zero | Always-on baseline |
| Cold start latency concern | No cold starts |

## Step 1: Spring Boot Lambda Adapter

```xml
<dependency>
    <groupId>com.amazonaws.serverless</groupId>
    <artifactId>aws-serverless-java-container-springboot3</artifactId>
    <version>2.0.1</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-core</artifactId>
    <version>1.2.3</version>
</dependency>
```

## Step 2: Lambda Handler

```java
public class LambdaHandler
        implements RequestHandler<Map<String, Object>, ApiGatewayResponse> {
    private static final SpringLambdaContainerHandler handler;

    static {
        try {
            handler = SpringLambdaContainerHandler
                .getAwsProxyHandler(Application.class);
        } catch (ContainerInitializationException e) {
            throw new RuntimeException("Failed to initialize", e);
        }
    }

    @Override
    public ApiGatewayResponse handleRequest(
            Map<String, Object> input, Context context) {
        var proxyRequest = new AwsProxyRequestBuilder()
            .withHttpMethod((String) input.get("httpMethod"))
            .withPath((String) input.get("path"))
            .withBody((String) input.get("body"))
            .build();
        var response = handler.proxy(proxyRequest, context);
        return ApiGatewayResponse.builder()
            .setStatusCode(response.getStatusCode())
            .setBody(response.getBody())
            .build();
    }
}
```

## Step 3: Your Spring Boot Application

```java
@SpringBootApplication
@RestController
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @GetMapping("/api/process")
    public Map<String, Object> process(
            @RequestParam String eventType) {
        return Map.of(
            "processed", true,
            "eventType", eventType,
            "timestamp", Instant.now()
        );
    }
}
```

## Step 4: SnapStart for Cold Start Reduction

Cold starts are the main serverless concern. Java/Spring Boot cold starts can be 2-5 seconds. SnapStart snapshots the initialized JVM, reducing cold starts to under 200ms.

```terraform
resource "aws_lambda_function" "app" {
  function_name = "product-service"
  handler       = "com.example.LambdaHandler::handleRequest"
  runtime       = "java21"
  memory_size   = 1024
  timeout       = 30

  snap_start {
    apply_on = "PublishedVersions"
  }
}
```

## Step 5: Event-Driven Lambda (SQS Trigger)

```java
public class SqsEventHandler
        implements RequestHandler<SQSEvent, String> {
    private static final SpringLambdaContainerHandler handler;

    static {
        try {
            handler = SpringLambdaContainerHandler
                .getAwsProxyHandler(Application.class);
        } catch (ContainerInitializationException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public String handleRequest(SQSEvent event, Context context) {
        var springContext = handler.getSpringAppContext();
        var processor = springContext.getBean(MessageProcessor.class);

        for (var message : event.getRecords()) {
            processor.process(message.getBody());
        }
        return "Processed " + event.getRecords().size() + " messages";
    }
}

@Service
public class MessageProcessor {
    public void process(String payload) {
        // Handle the SQS message
    }
}
```

## Step 6: Azure Functions Alternative

```xml
<dependency>
    <groupId>com.microsoft.azure.functions</groupId>
    <artifactId>azure-functions-java-library</artifactId>
    <version>3.1.0</version>
</dependency>
```

```java
public class OrderFunction {
    @FunctionName("processOrder")
    public HttpResponseMessage run(
            @HttpTrigger(name = "req", methods = HttpMethod.POST,
                authLevel = AuthorizationLevel.FUNCTION)
            HttpRequestMessage<Optional<String>> request,
            final ExecutionContext context) {
        var orderData = request.getBody().orElseThrow();
        var result = processOrder(orderData);
        return request.createResponseBuilder(HttpStatus.OK)
            .body(result)
            .build();
    }
}
```

## Key Points

- Use serverless for event-driven, intermittent workloads — not for always-on services
- SnapStart dramatically reduces Java cold starts on AWS Lambda
- Minimum 1024 MB memory for Spring Boot Lambda functions
- Use SQS/EventBridge triggers for async processing, API Gateway for HTTP
- Container deployment is simpler for services that run continuously
