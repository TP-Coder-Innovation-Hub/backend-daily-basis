# WebSocket — Real-Time Bidirectional Communication

## Why WebSocket

HTTP is request-response: the client asks, the server answers. WebSocket opens a persistent, bidirectional connection. Both sides send messages anytime. Use it for real-time updates, live notifications, and collaborative features.

```
HTTP:        Client → Server (request) → Client (response)
WebSocket:   Client ↔ Server (persistent, bidirectional)
```

## WebSocket vs SSE vs Long Polling

| Approach | Direction | Use Case |
|----------|-----------|----------|
| WebSocket | Bidirectional | Chat, live collaboration |
| SSE | Server → Client only | Live scores, notifications |
| Long polling | Client → Server (hacky) | Legacy fallback |

## Step 1: Add WebSocket Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

## Step 2: WebSocket Configuration

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();
    }
}
```

STOMP is a messaging protocol over WebSocket. It adds subscription semantics — clients subscribe to topics, the server broadcasts to subscribers. SockJS provides fallback transports when WebSocket is unavailable.

## Step 3: Message Models

```java
public record NotificationMessage(
    String type,
    String title,
    String body,
    Instant timestamp
) {}

public record ChatMessage(
    String sender,
    String content,
    String roomId,
    Instant timestamp
) {}
```

## Step 4: Controller

```java
@Controller
@RequiredArgsConstructor
public class NotificationController {
    private final SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/chat/{roomId}")
    public void handleChat(@DestinationVariable String roomId,
            ChatMessage message) {
        var enriched = new ChatMessage(
            message.sender(),
            message.content(),
            roomId,
            Instant.now());
        messagingTemplate.convertAndSend(
            "/topic/rooms/" + roomId, enriched);
    }

    public void sendNotification(String userId,
            NotificationMessage notification) {
        messagingTemplate.convertAndSendToUser(
            userId, "/queue/notifications", notification);
    }
}
```

## Step 5: Broadcasting from REST Endpoints

You can send WebSocket messages from anywhere — not just `@MessageMapping` handlers:

```java
@RestController
@RequiredArgsConstructor
public class OrderController {
    private final SimpMessagingTemplate messagingTemplate;
    private final OrderService orderService;

    @PostMapping("/api/orders")
    public ResponseEntity<OrderResponse> create(
            @Valid @RequestBody OrderRequest request) {
        var order = orderService.create(request);
        messagingTemplate.convertAndSend("/topic/orders",
            new NotificationMessage("ORDER_CREATED",
                "New order", "Order #" + order.id() + " created",
                Instant.now()));
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
}
```

## Step 6: JavaScript Client

```javascript
const socket = new SockJS('http://localhost:8080/ws');
const stompClient = Stomp.over(socket);

stompClient.connect({}, () => {
    stompClient.subscribe('/topic/rooms/general', (message) => {
        const chat = JSON.parse(message.body);
        console.log(chat.sender + ': ' + chat.content);
    });

    stompClient.subscribe('/user/queue/notifications', (message) => {
        const notification = JSON.parse(message.body);
        console.log(notification.title + ': ' + notification.body);
    });
});

function sendMessage(roomId, sender, content) {
    stompClient.send('/app/chat/' + roomId, {},
        JSON.stringify({ sender, content, roomId }));
}
```

## Key Points

- Use STOMP over raw WebSocket — it gives you topics, subscriptions, and routing
- `@MessageMapping` handles incoming messages; `SimpMessagingTemplate` sends outbound
- `/topic/` broadcasts to all subscribers; `/user/` sends to a specific user
- SockJS fallback ensures connectivity when WebSocket is blocked (corporate proxies)
- Keep messages small — WebSocket frames should be concise for real-time feel
