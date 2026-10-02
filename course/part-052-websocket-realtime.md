# Part 052 – WebSocket & Real-Time Communication with Spring Boot

## Dependencies (pom.xml)

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-messaging</artifactId>
    </dependency>
</dependencies>
```

---

## 1. WebSocket Configuration with STOMP and SockJS

```java
package com.example.websocket.config;

import com.example.websocket.security.JwtChannelInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final JwtChannelInterceptor jwtChannelInterceptor;

    public WebSocketConfig(JwtChannelInterceptor jwtChannelInterceptor) {
        this.jwtChannelInterceptor = jwtChannelInterceptor;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Enable a simple in-memory broker for /topic (broadcast) and /queue (user-specific)
        registry.enableSimpleBroker("/topic", "/queue");
        // Prefix for @MessageMapping methods
        registry.setApplicationDestinationPrefixes("/app");
        // Prefix for user-specific destinations
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS()
                .setHeartbeatTime(25000)
                .setDisconnectDelay(5000);

        // Raw WebSocket endpoint (without SockJS fallback)
        registry.addEndpoint("/ws-native")
                .setAllowedOriginPatterns("*");
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(jwtChannelInterceptor);
    }
}
```

---

## 2. JWT Channel Interceptor (Security over STOMP)

```java
package com.example.websocket.security;

import com.example.websocket.service.JwtService;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class JwtChannelInterceptor implements ChannelInterceptor {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtChannelInterceptor(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor =
                MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);

        if (accessor != null && StompCommand.CONNECT.equals(accessor.getCommand())) {
            List<String> authHeaders = accessor.getNativeHeader("Authorization");
            if (authHeaders != null && !authHeaders.isEmpty()) {
                String authHeader = authHeaders.get(0);
                if (authHeader.startsWith("Bearer ")) {
                    String token = authHeader.substring(7);
                    String username = jwtService.extractUsername(token);

                    if (username != null) {
                        UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                        if (jwtService.isTokenValid(token, userDetails)) {
                            UsernamePasswordAuthenticationToken authentication =
                                    new UsernamePasswordAuthenticationToken(
                                            userDetails, null, userDetails.getAuthorities());
                            accessor.setUser(authentication);
                        }
                    }
                }
            }
        }
        return message;
    }
}
```

---

## 3. JWT Service

```java
package com.example.websocket.service;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import java.security.Key;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {

    @Value("${jwt.secret:404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970}")
    private String secretKey;

    @Value("${jwt.expiration:86400000}")
    private long jwtExpiration;

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return Jwts.builder()
                .setClaims(extraClaims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + jwtExpiration))
                .signWith(getSignInKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSignInKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    private Key getSignInKey() {
        byte[] keyBytes = hexStringToByteArray(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    private byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                    + Character.digit(s.charAt(i + 1), 16));
        }
        return data;
    }
}
```

---

## 4. Chat Message Model

```java
package com.example.websocket.model;

import java.time.LocalDateTime;

public class ChatMessage {

    public enum MessageType {
        CHAT, JOIN, LEAVE, ERROR
    }

    private MessageType type;
    private String content;
    private String sender;
    private String recipient;
    private String roomId;
    private LocalDateTime timestamp;

    public ChatMessage() {
        this.timestamp = LocalDateTime.now();
    }

    public ChatMessage(MessageType type, String content, String sender) {
        this.type = type;
        this.content = content;
        this.sender = sender;
        this.timestamp = LocalDateTime.now();
    }

    // Getters and setters
    public MessageType getType() { return type; }
    public void setType(MessageType type) { this.type = type; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getSender() { return sender; }
    public void setSender(String sender) { this.sender = sender; }
    public String getRecipient() { return recipient; }
    public void setRecipient(String recipient) { this.recipient = recipient; }
    public String getRoomId() { return roomId; }
    public void setRoomId(String roomId) { this.roomId = roomId; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public void setTimestamp(LocalDateTime timestamp) { this.timestamp = timestamp; }
}
```

---

## 5. Real-Time Chat Controller

```java
package com.example.websocket.controller;

import com.example.websocket.model.ChatMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Controller
public class ChatController {

    private static final Logger log = LoggerFactory.getLogger(ChatController.class);
    private final SimpMessagingTemplate messagingTemplate;
    private final Map<String, String> activeUsers = new ConcurrentHashMap<>();

    public ChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // Broadcast message to a topic room
    @MessageMapping("/chat.room/{roomId}")
    @SendTo("/topic/room/{roomId}")
    public ChatMessage sendRoomMessage(
            @DestinationVariable String roomId,
            @Payload ChatMessage message,
            Principal principal) {
        message.setSender(principal.getName());
        message.setRoomId(roomId);
        log.info("Message from {} in room {}: {}", principal.getName(), roomId, message.getContent());
        return message;
    }

    // Private message to a specific user
    @MessageMapping("/chat.private")
    public void sendPrivateMessage(@Payload ChatMessage message, Principal principal) {
        message.setSender(principal.getName());
        // Send to recipient's queue
        messagingTemplate.convertAndSendToUser(
                message.getRecipient(),
                "/queue/private",
                message
        );
        // Also echo back to sender
        messagingTemplate.convertAndSendToUser(
                principal.getName(),
                "/queue/private",
                message
        );
    }

    // User joins a room
    @MessageMapping("/chat.join/{roomId}")
    @SendTo("/topic/room/{roomId}")
    public ChatMessage userJoin(
            @DestinationVariable String roomId,
            SimpMessageHeaderAccessor headerAccessor,
            Principal principal) {
        String username = principal.getName();
        activeUsers.put(headerAccessor.getSessionId(), username);
        headerAccessor.getSessionAttributes().put("username", username);
        headerAccessor.getSessionAttributes().put("roomId", roomId);

        ChatMessage joinMessage = new ChatMessage(
                ChatMessage.MessageType.JOIN,
                username + " joined the room",
                "System"
        );
        joinMessage.setRoomId(roomId);
        return joinMessage;
    }

    // Subscribe handler - sends initial data on subscription
    @SubscribeMapping("/chat.history/{roomId}")
    @SendToUser
    public ChatMessage getHistory(@DestinationVariable String roomId, Principal principal) {
        // In a real app, fetch from database
        ChatMessage welcome = new ChatMessage(
                ChatMessage.MessageType.CHAT,
                "Welcome to room " + roomId + "! History would be loaded here.",
                "System"
        );
        welcome.setRoomId(roomId);
        return welcome;
    }

    // Handle errors
    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public ChatMessage handleException(Throwable exception) {
        log.error("WebSocket error: {}", exception.getMessage());
        return new ChatMessage(ChatMessage.MessageType.ERROR, exception.getMessage(), "System");
    }
}
```

---

## 6. WebSocket Event Listener (Join/Leave Tracking)

```java
package com.example.websocket.listener;

import com.example.websocket.model.ChatMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.SessionConnectedEvent;
import org.springframework.web.socket.messaging.SessionDisconnectEvent;

@Component
public class WebSocketEventListener {

    private static final Logger log = LoggerFactory.getLogger(WebSocketEventListener.class);
    private final SimpMessagingTemplate messagingTemplate;

    public WebSocketEventListener(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @EventListener
    public void handleWebSocketConnectListener(SessionConnectedEvent event) {
        StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
        String sessionId = headerAccessor.getSessionId();
        log.info("New WebSocket connection: sessionId={}", sessionId);
    }

    @EventListener
    public void handleWebSocketDisconnectListener(SessionDisconnectEvent event) {
        StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());

        String username = (String) headerAccessor.getSessionAttributes().get("username");
        String roomId   = (String) headerAccessor.getSessionAttributes().get("roomId");

        if (username != null && roomId != null) {
            log.info("User {} disconnected from room {}", username, roomId);
            ChatMessage leaveMessage = new ChatMessage(
                    ChatMessage.MessageType.LEAVE,
                    username + " left the room",
                    "System"
            );
            leaveMessage.setRoomId(roomId);
            messagingTemplate.convertAndSend("/topic/room/" + roomId, leaveMessage);
        }
    }
}
```

---

## 7. Live Dashboard – Pub/Sub with Scheduled Updates

```java
package com.example.websocket.controller;

import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;
import java.util.concurrent.atomic.AtomicInteger;

@Component
@EnableScheduling
public class DashboardPublisher {

    private final SimpMessagingTemplate messagingTemplate;
    private final Random random = new Random();
    private final AtomicInteger activeConnections = new AtomicInteger(0);

    public DashboardPublisher(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @Scheduled(fixedRate = 2000)
    public void publishSystemMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("timestamp", LocalDateTime.now().toString());
        metrics.put("cpuUsage", 20 + random.nextDouble() * 60);
        metrics.put("memoryUsage", 40 + random.nextDouble() * 40);
        metrics.put("activeConnections", activeConnections.get());
        metrics.put("requestsPerSecond", random.nextInt(500) + 100);
        metrics.put("errorRate", random.nextDouble() * 2);

        messagingTemplate.convertAndSend("/topic/dashboard/metrics", metrics);
    }

    @Scheduled(fixedRate = 5000)
    public void publishOrderUpdates() {
        Map<String, Object> order = new HashMap<>();
        order.put("orderId", "ORD-" + (random.nextInt(9000) + 1000));
        order.put("status", randomStatus());
        order.put("amount", Math.round(random.nextDouble() * 500 * 100.0) / 100.0);
        order.put("timestamp", LocalDateTime.now().toString());

        messagingTemplate.convertAndSend("/topic/dashboard/orders", order);
    }

    @Scheduled(fixedRate = 10000)
    public void publishStockPrices() {
        String[] symbols = {"AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"};
        for (String symbol : symbols) {
            Map<String, Object> stock = new HashMap<>();
            stock.put("symbol", symbol);
            stock.put("price", 100 + random.nextDouble() * 400);
            stock.put("change", (random.nextDouble() - 0.5) * 10);
            stock.put("volume", random.nextInt(1000000));
            messagingTemplate.convertAndSend("/topic/dashboard/stocks/" + symbol, stock);
        }
    }

    public void incrementConnections() { activeConnections.incrementAndGet(); }
    public void decrementConnections() { activeConnections.decrementAndGet(); }

    private String randomStatus() {
        String[] statuses = {"PENDING", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"};
        return statuses[random.nextInt(statuses.length)];
    }
}
```

---

## 8. Dashboard REST + WebSocket Controller

```java
package com.example.websocket.controller;

import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.security.Principal;
import java.util.HashMap;
import java.util.Map;

@Controller
public class DashboardController {

    private final SimpMessagingTemplate messagingTemplate;
    private final DashboardPublisher dashboardPublisher;

    public DashboardController(SimpMessagingTemplate messagingTemplate,
                                DashboardPublisher dashboardPublisher) {
        this.messagingTemplate = messagingTemplate;
        this.dashboardPublisher = dashboardPublisher;
    }

    // Client subscribes → immediately receives current snapshot
    @SubscribeMapping("/dashboard/snapshot")
    public Map<String, Object> getDashboardSnapshot(Principal principal) {
        Map<String, Object> snapshot = new HashMap<>();
        snapshot.put("message", "Connected to live dashboard, " + principal.getName());
        snapshot.put("serverTime", java.time.LocalDateTime.now().toString());
        snapshot.put("version", "2.1.0");
        return snapshot;
    }

    // Client can request a manual refresh
    @MessageMapping("/dashboard.refresh")
    @SendTo("/topic/dashboard/metrics")
    public Map<String, Object> requestRefresh() {
        Map<String, Object> metrics = new HashMap<>();
        metrics.put("manual", true);
        metrics.put("timestamp", java.time.LocalDateTime.now().toString());
        metrics.put("cpuUsage", 35.5);
        metrics.put("memoryUsage", 58.2);
        return metrics;
    }

    // Notify a specific user (e.g., on alert threshold breach)
    public void sendUserAlert(String username, String message) {
        Map<String, Object> alert = new HashMap<>();
        alert.put("type", "ALERT");
        alert.put("message", message);
        alert.put("timestamp", java.time.LocalDateTime.now().toString());
        messagingTemplate.convertAndSendToUser(username, "/queue/alerts", alert);
    }
}
```

---

## 9. WebSocket Interceptor (Logging + Rate Limiting)

```java
package com.example.websocket.interceptor;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Component
public class WebSocketLoggingInterceptor implements ChannelInterceptor {

    private static final Logger log = LoggerFactory.getLogger(WebSocketLoggingInterceptor.class);
    private static final int MAX_MESSAGES_PER_SECOND = 10;

    private final Map<String, AtomicInteger> messageCounters = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStart = new ConcurrentHashMap<>();

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor =
                MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);

        if (accessor == null) return message;

        String sessionId = accessor.getSessionId();
        StompCommand command = accessor.getCommand();

        if (command != null) {
            log.debug("STOMP {} from session {}", command, sessionId);
        }

        // Rate limit SEND commands
        if (StompCommand.SEND.equals(command) && sessionId != null) {
            if (isRateLimited(sessionId)) {
                log.warn("Rate limit exceeded for session {}", sessionId);
                throw new RuntimeException("Rate limit exceeded. Max " + MAX_MESSAGES_PER_SECOND + " messages/second.");
            }
        }

        return message;
    }

    @Override
    public void afterSendCompletion(Message<?> message, MessageChannel channel,
                                    boolean sent, Exception ex) {
        if (ex != null) {
            StompHeaderAccessor accessor =
                    MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
            log.error("Error processing message from session {}: {}",
                    accessor != null ? accessor.getSessionId() : "unknown",
                    ex.getMessage());
        }
    }

    private boolean isRateLimited(String sessionId) {
        long now = System.currentTimeMillis();
        long start = windowStart.computeIfAbsent(sessionId, k -> now);

        if (now - start > 1000) {
            windowStart.put(sessionId, now);
            messageCounters.put(sessionId, new AtomicInteger(1));
            return false;
        }

        AtomicInteger count = messageCounters.computeIfAbsent(sessionId, k -> new AtomicInteger(0));
        return count.incrementAndGet() > MAX_MESSAGES_PER_SECOND;
    }
}
```

---

## 10. Register Multiple Interceptors in Config

```java
package com.example.websocket.config;

import com.example.websocket.interceptor.WebSocketLoggingInterceptor;
import com.example.websocket.security.JwtChannelInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketFullConfig implements WebSocketMessageBrokerConfigurer {

    private final JwtChannelInterceptor jwtInterceptor;
    private final WebSocketLoggingInterceptor loggingInterceptor;

    public WebSocketFullConfig(JwtChannelInterceptor jwtInterceptor,
                               WebSocketLoggingInterceptor loggingInterceptor) {
        this.jwtInterceptor = jwtInterceptor;
        this.loggingInterceptor = loggingInterceptor;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue")
                .setHeartbeatValue(new long[]{10000, 10000})
                .setTaskScheduler(taskScheduler());
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(jwtInterceptor, loggingInterceptor);
        registration.taskExecutor().corePoolSize(4).maxPoolSize(8);
    }

    @Override
    public void configureClientOutboundChannel(ChannelRegistration registration) {
        registration.taskExecutor().corePoolSize(4).maxPoolSize(8);
    }

    private org.springframework.scheduling.TaskScheduler taskScheduler() {
        org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler scheduler =
                new org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler();
        scheduler.setPoolSize(1);
        scheduler.setThreadNamePrefix("ws-heartbeat-");
        scheduler.initialize();
        return scheduler;
    }
}
```

---

## 11. Spring Security Config for WebSocket

```java
package com.example.websocket.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.socket.EnableWebSocketSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/ws/**", "/ws-native/**")
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/ws/**", "/ws-native/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

---

## 12. Application Properties

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: websocket-demo

server:
  port: 8080

jwt:
  secret: 404E635266556A586E3272357538782F413F4428472B4B6250645367566B5970
  expiration: 86400000

logging:
  level:
    com.example.websocket: DEBUG
    org.springframework.web.socket: DEBUG
    org.springframework.messaging: DEBUG
```

---

## 13. JavaScript Client – SockJS + STOMP

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <script src="https://cdn.jsdelivr.net/npm/sockjs-client/dist/sockjs.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs/bundles/stomp.umd.min.js"></script>
</head>
<body>
<div id="chat">
    <div id="messages"></div>
    <input id="message" type="text" placeholder="Type a message..."/>
    <button onclick="sendMessage()">Send</button>
</div>

<script>
const JWT_TOKEN = 'your-jwt-token-here';
const ROOM_ID = 'general';

let stompClient = null;

function connect() {
    const socket = new SockJS('http://localhost:8080/ws');
    stompClient = new StompJs.Client({
        webSocketFactory: () => socket,
        connectHeaders: {
            Authorization: 'Bearer ' + JWT_TOKEN
        },
        debug: (str) => console.log(str),
        reconnectDelay: 5000,
        onConnect: (frame) => {
            console.log('Connected: ' + frame);

            // Subscribe to room topic
            stompClient.subscribe('/topic/room/' + ROOM_ID, (message) => {
                showMessage(JSON.parse(message.body));
            });

            // Subscribe to private queue
            stompClient.subscribe('/user/queue/private', (message) => {
                showMessage(JSON.parse(message.body), true);
            });

            // Subscribe to errors
            stompClient.subscribe('/user/queue/errors', (message) => {
                console.error('Error:', JSON.parse(message.body));
            });

            // Join the room
            stompClient.publish({
                destination: '/app/chat.join/' + ROOM_ID,
                body: JSON.stringify({ type: 'JOIN' })
            });
        },
        onStompError: (frame) => {
            console.error('STOMP error:', frame);
        }
    });

    stompClient.activate();
}

function sendMessage() {
    const content = document.getElementById('message').value.trim();
    if (!content || !stompClient?.connected) return;

    stompClient.publish({
        destination: '/app/chat.room/' + ROOM_ID,
        body: JSON.stringify({
            type: 'CHAT',
            content: content
        })
    });

    document.getElementById('message').value = '';
}

function showMessage(msg, isPrivate = false) {
    const div = document.createElement('div');
    div.textContent = `[${isPrivate ? 'PRIVATE ' : ''}${msg.sender}]: ${msg.content}`;
    document.getElementById('messages').appendChild(div);
}

connect();
</script>
</body>
</html>
```

---

## 14. Testing WebSocket Endpoints with StompSession

```java
package com.example.websocket;

import com.example.websocket.model.ChatMessage;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.simp.stomp.*;
import org.springframework.web.socket.WebSocketHttpHeaders;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;
import org.springframework.web.socket.sockjs.client.SockJsClient;
import org.springframework.web.socket.sockjs.client.WebSocketTransport;

import java.lang.reflect.Type;
import java.util.List;
import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChatControllerTest {

    @LocalServerPort
    private int port;

    private WebSocketStompClient stompClient;

    @BeforeEach
    void setUp() {
        stompClient = new WebSocketStompClient(
                new SockJsClient(List.of(new WebSocketTransport(new StandardWebSocketClient())))
        );
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());
    }

    @Test
    void testSendAndReceiveChatMessage() throws Exception {
        BlockingQueue<ChatMessage> received = new LinkedBlockingQueue<>();
        CountDownLatch connected = new CountDownLatch(1);

        StompSession session = stompClient.connect(
                "ws://localhost:" + port + "/ws",
                new WebSocketHttpHeaders(),
                new StompSessionHandlerAdapter() {
                    @Override
                    public void afterConnected(StompSession s, StompHeaders headers) {
                        connected.countDown();
                    }
                }
        ).get(5, TimeUnit.SECONDS);

        connected.await(5, TimeUnit.SECONDS);

        session.subscribe("/topic/room/general", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) { return ChatMessage.class; }

            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                received.offer((ChatMessage) payload);
            }
        });

        ChatMessage msg = new ChatMessage(ChatMessage.MessageType.CHAT, "Hello, world!", "testUser");
        msg.setRoomId("general");
        session.send("/app/chat.room/general", msg);

        ChatMessage response = received.poll(5, TimeUnit.SECONDS);
        assertThat(response).isNotNull();
        assertThat(response.getContent()).isEqualTo("Hello, world!");

        session.disconnect();
    }

    @Test
    void testPrivateMessage() throws Exception {
        BlockingQueue<ChatMessage> userAMessages = new LinkedBlockingQueue<>();
        BlockingQueue<ChatMessage> userBMessages = new LinkedBlockingQueue<>();

        // Connect user A
        StompSession sessionA = stompClient.connect(
                "ws://localhost:" + port + "/ws",
                new WebSocketHttpHeaders(),
                new StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS);

        sessionA.subscribe("/user/queue/private", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) { return ChatMessage.class; }
            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                userAMessages.offer((ChatMessage) payload);
            }
        });

        // Connect user B
        StompSession sessionB = stompClient.connect(
                "ws://localhost:" + port + "/ws",
                new WebSocketHttpHeaders(),
                new StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS);

        sessionB.subscribe("/user/queue/private", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) { return ChatMessage.class; }
            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                userBMessages.offer((ChatMessage) payload);
            }
        });

        ChatMessage privateMsg = new ChatMessage(ChatMessage.MessageType.CHAT, "Secret!", "userA");
        privateMsg.setRecipient("userB");
        sessionA.send("/app/chat.private", privateMsg);

        // Both should receive (sender echoed, recipient received)
        ChatMessage received = userBMessages.poll(5, TimeUnit.SECONDS);
        assertThat(received).isNotNull();
        assertThat(received.getContent()).isEqualTo("Secret!");

        sessionA.disconnect();
        sessionB.disconnect();
    }

    @Test
    void testJoinLeave() throws Exception {
        BlockingQueue<ChatMessage> events = new LinkedBlockingQueue<>();

        StompSession session = stompClient.connect(
                "ws://localhost:" + port + "/ws",
                new WebSocketHttpHeaders(),
                new StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS);

        session.subscribe("/topic/room/lobby", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) { return ChatMessage.class; }
            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                events.offer((ChatMessage) payload);
            }
        });

        session.send("/app/chat.join/lobby", new ChatMessage(ChatMessage.MessageType.JOIN, "", "testUser"));

        ChatMessage joinEvent = events.poll(5, TimeUnit.SECONDS);
        assertThat(joinEvent).isNotNull();
        assertThat(joinEvent.getType()).isEqualTo(ChatMessage.MessageType.JOIN);

        session.disconnect();

        // After disconnect, LEAVE event should be published
        ChatMessage leaveEvent = events.poll(5, TimeUnit.SECONDS);
        assertThat(leaveEvent).isNotNull();
        assertThat(leaveEvent.getType()).isEqualTo(ChatMessage.MessageType.LEAVE);
    }
}
```

---

## 15. Dashboard WebSocket Test

```java
package com.example.websocket;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.messaging.simp.stomp.*;
import org.springframework.web.socket.WebSocketHttpHeaders;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;
import org.springframework.web.socket.sockjs.client.SockJsClient;
import org.springframework.web.socket.sockjs.client.WebSocketTransport;

import java.lang.reflect.Type;
import java.util.List;
import java.util.Map;
import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class DashboardWebSocketTest {

    @LocalServerPort
    private int port;

    private WebSocketStompClient stompClient;

    @BeforeEach
    void setUp() {
        stompClient = new WebSocketStompClient(
                new SockJsClient(List.of(new WebSocketTransport(new StandardWebSocketClient())))
        );
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());
    }

    @Test
    @SuppressWarnings("unchecked")
    void testDashboardMetricsStream() throws Exception {
        BlockingQueue<Map<String, Object>> metrics = new LinkedBlockingQueue<>();

        StompSession session = stompClient.connect(
                "ws://localhost:" + port + "/ws",
                new WebSocketHttpHeaders(),
                new StompSessionHandlerAdapter() {}
        ).get(5, TimeUnit.SECONDS);

        session.subscribe("/topic/dashboard/metrics", new StompFrameHandler() {
            @Override
            public Type getPayloadType(StompHeaders headers) { return Map.class; }
            @Override
            public void handleFrame(StompHeaders headers, Object payload) {
                metrics.offer((Map<String, Object>) payload);
            }
        });

        // Wait for at least one metric update (scheduler fires every 2s)
        Map<String, Object> metric = metrics.poll(5, TimeUnit.SECONDS);
        assertThat(metric).isNotNull();
        assertThat(metric).containsKeys("timestamp", "cpuUsage", "memoryUsage");
        assertThat((Double) metric.get("cpuUsage")).isBetween(0.0, 100.0);

        session.disconnect();
    }
}
```

---

## 16. Auth REST Controller for Token Generation

```java
package com.example.websocket.controller;

import com.example.websocket.service.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final JwtService jwtService;
    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;

    public AuthController(JwtService jwtService,
                          AuthenticationManager authenticationManager,
                          UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.authenticationManager = authenticationManager;
        this.userDetailsService = userDetailsService;
    }

    @PostMapping("/login")
    public Map<String, String> login(@RequestBody Map<String, String> request) {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.get("username"), request.get("password"))
        );
        UserDetails userDetails = userDetailsService.loadUserByUsername(request.get("username"));
        String token = jwtService.generateToken(userDetails);
        return Map.of(
                "token", token,
                "username", userDetails.getUsername()
        );
    }
}
```

---

## 17. User Details Service (In-Memory for Demo)

```java
package com.example.websocket.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;

@Configuration
public class UserConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        return new InMemoryUserDetailsManager(
                User.withUsername("alice")
                        .password(passwordEncoder().encode("password"))
                        .roles("USER")
                        .build(),
                User.withUsername("bob")
                        .password(passwordEncoder().encode("password"))
                        .roles("USER")
                        .build(),
                User.withUsername("admin")
                        .password(passwordEncoder().encode("admin"))
                        .roles("USER", "ADMIN")
                        .build()
        );
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationProvider authenticationProvider(UserDetailsService uds) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(uds);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## 18. Error Handling Configuration

```java
package com.example.websocket.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageDeliveryException;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.web.socket.messaging.StompSubProtocolErrorHandler;

import java.nio.charset.StandardCharsets;

@Configuration
public class WebSocketErrorHandler extends StompSubProtocolErrorHandler {

    @Override
    public Message<byte[]> handleClientMessageProcessingError(
            Message<byte[]> clientMessage, Throwable ex) {

        if (ex instanceof MessageDeliveryException) {
            return prepareErrorMessage(ex.getMessage());
        }
        return prepareErrorMessage("An unexpected error occurred: " + ex.getMessage());
    }

    private Message<byte[]> prepareErrorMessage(String errorMessage) {
        StompHeaderAccessor headers = StompHeaderAccessor.create(StompCommand.ERROR);
        headers.setMessage(errorMessage);
        headers.setLeaveMutable(true);
        return MessageBuilder.createMessage(
                errorMessage.getBytes(StandardCharsets.UTF_8),
                headers.getMessageHeaders()
        );
    }
}
```

```java
// Register error handler in WebSocket config
@Override
public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
    registration.setMessageSizeLimit(64 * 1024);   // 64KB max message size
    registration.setSendTimeLimit(20 * 1000);       // 20 sec send timeout
    registration.setSendBufferSizeLimit(512 * 1024); // 512KB send buffer
    registration.addDecoratorFactory(handler -> new WebSocketHandlerDecorator(handler) {
        @Override
        public void afterConnectionEstablished(WebSocketSession session) throws Exception {
            super.afterConnectionEstablished(session);
        }
    });
}
```

---

## Summary

| Feature | Annotation / Class |
|---|---|
| Broadcast message | `@SendTo("/topic/...")` |
| User-specific message | `@SendToUser` / `convertAndSendToUser` |
| Route variable | `@DestinationVariable` |
| Incoming payload | `@Payload` |
| Channel interceptor | `ChannelInterceptor.preSend()` |
| JWT in STOMP | `CONNECT` frame `Authorization` header |
| Error response | `@MessageExceptionHandler` |
| Heartbeat | `enableSimpleBroker().setHeartbeatValue()` |
| SockJS fallback | `.withSockJS()` on endpoint |
| Testing | `WebSocketStompClient` + `BlockingQueue` |
