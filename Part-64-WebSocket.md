# Part 64: WebSocket & Real-time Communication
## ขั้นตอนที่ 4381-4450: Live Updates, Chat, Push Notifications

---

## 64.1 WebSocket vs REST

```
REST:
  Client → Request → Server → Response → Done
  ไม่มี connection ค้างอยู่
  เหมาะกับ: CRUD, stateless APIs

WebSocket:
  Client ↔ persistent bidirectional connection ↔ Server
  ส่งข้อมูลได้ทั้งสองทาง ตลอดเวลา
  เหมาะกับ: live chat, real-time dashboard, multiplayer game

Server-Sent Events (SSE):
  Server → push → Client (one-way)
  เหมาะกับ: live feed, notification, AI streaming response

Long Polling:
  Client polls every N seconds (fallback เมื่อ WebSocket ไม่รองรับ)
  เหมาะกับ: environment ที่ไม่รองรับ WebSocket
```

---

## 64.2 Spring WebSocket Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```java
import org.springframework.context.annotation.*;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.config.*;
import org.springframework.web.socket.config.annotation.*;

// Configuration: WebSocket + STOMP
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // In-memory broker for topics and queues
        config.enableSimpleBroker("/topic", "/queue");
        // Prefix for @MessageMapping methods
        config.setApplicationDestinationPrefixes("/app");
        // Prefix for user-specific messages
        config.setUserDestinationPrefix("/user");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();  // fallback for browsers that don't support WebSocket
    }
}
```

---

## 64.3 Live Chat Implementation

```java
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.*;
import org.springframework.stereotype.*;

// Message types
record ChatMessage(
    String messageId,
    String roomId,
    String senderId,
    String senderName,
    String content,
    java.time.Instant timestamp,
    MessageType type
) {}

enum MessageType { CHAT, JOIN, LEAVE, TYPING }

// Chat Controller
@Controller
class ChatController {
    
    private final SimpMessageSendingOperations messagingTemplate;
    private final ChatMessageRepository messageRepository;
    
    ChatController(SimpMessageSendingOperations messagingTemplate,
                   ChatMessageRepository messageRepository) {
        this.messagingTemplate = messagingTemplate;
        this.messageRepository = messageRepository;
    }
    
    // Handle message sent to /app/chat.send
    @MessageMapping("/chat.send")
    public void sendMessage(@Payload ChatMessage message) {
        // Save to DB
        messageRepository.save(message);
        
        // Broadcast to all subscribers of /topic/chat/{roomId}
        messagingTemplate.convertAndSend(
            "/topic/chat/" + message.roomId(), message);
    }
    
    // Handle join event
    @MessageMapping("/chat.join")
    public void joinRoom(@Payload ChatMessage message,
                         org.springframework.messaging.simp.SimpMessageHeaderAccessor headerAccessor) {
        // Store username in session
        headerAccessor.getSessionAttributes().put("username", message.senderName());
        headerAccessor.getSessionAttributes().put("roomId", message.roomId());
        
        // Notify room of new member
        messagingTemplate.convertAndSend(
            "/topic/chat/" + message.roomId(),
            new ChatMessage(
                java.util.UUID.randomUUID().toString(),
                message.roomId(),
                message.senderId(),
                message.senderName(),
                message.senderName() + " เข้าร่วมห้องสนทนา",
                java.time.Instant.now(),
                MessageType.JOIN
            )
        );
    }
    
    // Send to specific user: /user/{userId}/queue/private
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(@Payload PrivateMessage message) {
        messagingTemplate.convertAndSendToUser(
            message.recipientId(),
            "/queue/private",
            message
        );
    }
    
    // REST endpoint to get chat history
    @org.springframework.web.bind.annotation.GetMapping("/api/chat/{roomId}/history")
    @org.springframework.web.bind.annotation.ResponseBody
    java.util.List<ChatMessage> getChatHistory(
            @org.springframework.web.bind.annotation.PathVariable String roomId,
            @org.springframework.web.bind.annotation.RequestParam(defaultValue = "50") int limit) {
        return messageRepository.findByRoomIdOrderByTimestampDesc(roomId)
            .stream().limit(limit).toList();
    }
    
    record PrivateMessage(String senderId, String recipientId, String content) {}
}
```

---

## 64.4 Real-time Dashboard (Live Metrics)

```java
import org.springframework.scheduling.annotation.*;
import org.springframework.messaging.simp.*;

@Service
@EnableScheduling
class DashboardBroadcastService {
    
    private final SimpMessageSendingOperations messagingTemplate;
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    
    DashboardBroadcastService(SimpMessageSendingOperations messagingTemplate,
                               OrderRepository orderRepository,
                               UserRepository userRepository) {
        this.messagingTemplate = messagingTemplate;
        this.orderRepository = orderRepository;
        this.userRepository = userRepository;
    }
    
    // Broadcast dashboard metrics every 5 seconds
    @Scheduled(fixedRate = 5000)
    public void broadcastMetrics() {
        var metrics = new DashboardMetrics(
            orderRepository.countTodayOrders(),
            orderRepository.sumTodayRevenue(),
            userRepository.countActiveUsers(),
            orderRepository.countPendingOrders(),
            java.time.Instant.now()
        );
        
        messagingTemplate.convertAndSend("/topic/dashboard/metrics", metrics);
    }
    
    // Broadcast when new order placed
    public void notifyNewOrder(Order order) {
        messagingTemplate.convertAndSend("/topic/dashboard/new-order",
            new OrderEvent(
                order.getId(),
                order.getUserId(),
                order.getTotal(),
                order.getStatus(),
                java.time.Instant.now()
            )
        );
    }
    
    record DashboardMetrics(
        long todayOrders,
        double todayRevenue,
        long activeUsers,
        long pendingOrders,
        java.time.Instant updatedAt
    ) {}
    
    record OrderEvent(
        String orderId,
        String userId,
        double total,
        String status,
        java.time.Instant timestamp
    ) {}
}
```

---

## 64.5 Client-Side JavaScript

```javascript
// WebSocket client using STOMP.js + SockJS
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

class ChatClient {
    constructor() {
        this.client = new Client({
            webSocketFactory: () => new SockJS('/ws'),
            reconnectDelay: 5000,  // auto-reconnect
            onConnect: this.onConnect.bind(this),
            onDisconnect: this.onDisconnect.bind(this),
            onStompError: (frame) => console.error('STOMP error', frame)
        });
    }
    
    connect() {
        this.client.activate();
    }
    
    onConnect(frame) {
        console.log('Connected to WebSocket');
        
        // Subscribe to chat room
        this.client.subscribe('/topic/chat/room-1', (message) => {
            const chatMsg = JSON.parse(message.body);
            this.displayMessage(chatMsg);
        });
        
        // Subscribe to private messages
        this.client.subscribe('/user/queue/private', (message) => {
            const privateMsg = JSON.parse(message.body);
            this.displayPrivateMessage(privateMsg);
        });
        
        // Join room
        this.client.publish({
            destination: '/app/chat.join',
            body: JSON.stringify({
                roomId: 'room-1',
                senderId: 'user-123',
                senderName: 'สมชาย',
                type: 'JOIN'
            })
        });
    }
    
    sendMessage(roomId, content) {
        this.client.publish({
            destination: '/app/chat.send',
            body: JSON.stringify({
                messageId: crypto.randomUUID(),
                roomId,
                senderId: 'user-123',
                senderName: 'สมชาย',
                content,
                timestamp: new Date().toISOString(),
                type: 'CHAT'
            })
        });
    }
    
    onDisconnect() {
        console.log('Disconnected from WebSocket');
    }
    
    displayMessage(msg) {
        console.log(`[${msg.senderName}]: ${msg.content}`);
    }
}

// Dashboard subscription
const dashboardClient = new Client({
    webSocketFactory: () => new SockJS('/ws'),
});

dashboardClient.onConnect = () => {
    dashboardClient.subscribe('/topic/dashboard/metrics', (message) => {
        const metrics = JSON.parse(message.body);
        document.getElementById('today-orders').textContent = metrics.todayOrders;
        document.getElementById('today-revenue').textContent = 
            `฿${metrics.todayRevenue.toLocaleString()}`;
    });
    
    dashboardClient.subscribe('/topic/dashboard/new-order', (message) => {
        const order = JSON.parse(message.body);
        showNotification(`คำสั่งซื้อใหม่ #${order.orderId}: ฿${order.total}`);
    });
};
```

---

## 64.6 Server-Sent Events (SSE) Alternative

```java
import org.springframework.web.servlet.mvc.method.annotation.*;

@RestController
@RequestMapping("/api/events")
class EventStreamController {
    
    private final java.util.concurrent.ConcurrentHashMap<String, SseEmitter> emitters =
        new java.util.concurrent.ConcurrentHashMap<>();
    
    // Client connects to receive events
    @GetMapping(value = "/subscribe/{userId}", produces = "text/event-stream")
    SseEmitter subscribe(@PathVariable String userId) {
        var emitter = new SseEmitter(Long.MAX_VALUE);
        emitters.put(userId, emitter);
        
        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> emitters.remove(userId));
        emitter.onError(e -> emitters.remove(userId));
        
        // Send initial data
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("Connected! userId=" + userId));
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
        
        return emitter;
    }
    
    // Push event to specific user
    public void pushToUser(String userId, String eventName, Object data) {
        var emitter = emitters.get(userId);
        if (emitter != null) {
            try {
                emitter.send(SseEmitter.event()
                    .name(eventName)
                    .data(data));
            } catch (Exception e) {
                emitters.remove(userId);
            }
        }
    }
    
    // Push to all connected clients
    public void broadcast(String eventName, Object data) {
        emitters.forEach((userId, emitter) -> {
            try {
                emitter.send(SseEmitter.event().name(eventName).data(data));
            } catch (Exception e) {
                emitters.remove(userId);
            }
        });
    }
}
```

---

## สรุป Part 64

```
Real-time Communication Options:

WebSocket (STOMP):
  ✓ Bidirectional: server → client AND client → server
  ✓ Low latency (~1ms)
  ✓ Spring: @EnableWebSocketMessageBroker + @MessageMapping
  ✓ Use for: chat, collaborative tools, live games

Server-Sent Events (SSE):
  ✓ Server → Client only (one-way push)
  ✓ Auto-reconnect built into browser
  ✓ Simpler than WebSocket
  ✓ Use for: notifications, live feed, AI streaming responses

STOMP Topics:
  /topic/xxx = broadcast to all subscribers
  /user/{id}/queue/xxx = send to specific user
  /app/xxx = send to @MessageMapping handler

Scale-out (multiple instances):
  Simple broker = in-memory (single instance only)
  RabbitMQ/ActiveMQ broker = distributed (multi-instance OK)
  
  spring.rabbitmq.host=localhost
  config.enableStompBrokerRelay("/topic", "/queue")
    .setRelayHost("localhost")
    .setRelayPort(61613)
```

➡️ [Part 65: Multi-tenancy Patterns](./Part-65-Multitenancy.md)
