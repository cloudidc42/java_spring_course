# Part 052: WebSocket and Real-time Features

## Table of Contents
1. [WebSocket Protocol Overview](#websocket-overview)
2. [STOMP Protocol](#stomp)
3. [Spring Boot WebSocket with STOMP](#spring-websocket)
4. [@MessageMapping, @SendTo, @SendToUser](#message-mapping)
5. [SimpMessagingTemplate for Server Push](#server-push)
6. [WebSocket Security](#websocket-security)
7. [WebSocket with JWT Authentication](#jwt-auth)
8. [Scaling with Redis Pub/Sub](#redis-scaling)
9. [Connection Lifecycle Management](#lifecycle)
10. [Reconnection and Heartbeat](#reconnect-heartbeat)
11. [WebSocket vs SSE](#websocket-vs-sse)
12. [Real Example: Real-time Collaborative Document Editing](#real-example)
13. [Summary](#summary)

---

## 1. WebSocket Protocol Overview {#websocket-overview}

WebSocket provides a full-duplex communication channel over a single TCP connection.

```
HTTP Request (Upgrade to WebSocket):
GET /ws HTTP/1.1
Host: api.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

[Connection is now bidirectional]
Client ←──────────────────────── Server
Client ──────────────────────────► Server
```

### When to Use WebSocket vs HTTP

| Scenario | Technology | Reason |
|---------|-----------|--------|
| Chat application | WebSocket | Bidirectional, low latency |
| Live notifications | WebSocket or SSE | Server-to-client push |
| Collaborative editing | WebSocket | Bidirectional, real-time |
| Live dashboard | SSE | Server-to-client only |
| File upload progress | SSE | One-way progress updates |
| REST API calls | HTTP | Request-response, cacheable |
| Order status updates | SSE or polling | Infrequent, server-to-client |

---

## 2. STOMP Protocol {#stomp}

STOMP (Simple Text Oriented Messaging Protocol) runs on top of WebSocket and provides:
- Message framing
- Destinations (topics/queues)
- Subscriptions
- Receipts and error handling

```
STOMP Frame:
┌─────────────────────────────────┐
│ COMMAND                         │
│ header1:value1                  │
│ header2:value2                  │
│                                 │
│ body                            │
│ ↑ null byte (^@) terminates     │
└─────────────────────────────────┘

Client Commands: CONNECT, SEND, SUBSCRIBE, UNSUBSCRIBE, DISCONNECT
Server Commands: CONNECTED, MESSAGE, RECEIPT, ERROR
```

### STOMP Message Flow

```
Client                    Spring Server
  │                            │
  │──CONNECT──────────────────►│
  │◄─CONNECTED─────────────────│
  │                            │
  │──SUBSCRIBE /topic/chat────►│ (register interest)
  │                            │
  │──SEND /app/chat────────────►│ (send to controller)
  │                            │
  │         handleMessage()    │
  │                            │
  │◄─MESSAGE /topic/chat───────│ (broadcast to subscribers)
  │                            │
  │◄─MESSAGE /queue/private────│ (to specific user only)
```

---

## 3. Spring Boot WebSocket with STOMP {#spring-websocket}

### Maven Dependencies

```xml
<!-- pom.xml -->
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
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- JWT validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Jackson for JSON messages -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

### WebSocket Configuration

```java
package com.example.ws.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.ChannelRegistration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Enable simple in-memory broker for topics and queues
        // For production: replace with Redis broker relay (see section 8)
        registry.enableSimpleBroker(
                "/topic",   // Broadcast topics (1:many)
                "/queue"    // User-specific queues (1:1)
        );

        // Prefix for client-to-server @MessageMapping methods
        registry.setApplicationDestinationPrefixes("/app");

        // Prefix for user-specific destinations
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")  // Restrict in production
                .withSockJS();  // Fallback for browsers without WebSocket support
    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        // Add authentication interceptor (see JWT section)
        registration.interceptors(jwtChannelInterceptor());
    }

    private com.example.ws.security.JwtChannelInterceptor jwtChannelInterceptor() {
        return new com.example.ws.security.JwtChannelInterceptor();
    }
}
```

---

## 4. @MessageMapping, @SendTo, @SendToUser {#message-mapping}

### Message Models

```java
package com.example.ws.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ChatMessage {
    private String id;
    private String content;
    private String sender;
    private String room;
    private MessageType type;
    private Instant timestamp;

    public enum MessageType { CHAT, JOIN, LEAVE, TYPING }
}
```

```java
package com.example.ws.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DocumentEdit {
    private String documentId;
    private String userId;
    private String userName;
    private int position;        // Character position in document
    private int length;          // Length of replaced text
    private String insertedText; // New text being inserted
    private long version;        // Operational transform version
    private Instant timestamp;
}
```

```java
package com.example.ws.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.Instant;
import java.util.Set;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PresenceUpdate {
    private String documentId;
    private Set<UserPresence> activeUsers;
    private Instant timestamp;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UserPresence {
        private String userId;
        private String userName;
        private String color;    // Cursor color for visual distinction
        private Integer cursorPosition;
    }
}
```

### WebSocket Controller

```java
package com.example.ws.controller;

import com.example.ws.dto.ChatMessage;
import com.example.ws.dto.DocumentEdit;
import com.example.ws.service.ChatService;
import com.example.ws.service.PresenceService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Controller
@RequiredArgsConstructor
public class WebSocketController {

    private final ChatService chatService;
    private final PresenceService presenceService;

    /**
     * @MessageMapping: handles messages sent to /app/chat/{room}
     * @SendTo: broadcasts result to all subscribers of /topic/chat/{room}
     */
    @MessageMapping("/chat/{room}")
    @SendTo("/topic/chat/{room}")
    public ChatMessage handleChatMessage(
            @DestinationVariable String room,
            @Payload ChatMessage message,
            Principal principal) {

        log.info("Chat message: room={}, from={}, length={}",
                room, principal.getName(), message.getContent().length());

        ChatMessage enriched = ChatMessage.builder()
                .id(UUID.randomUUID().toString())
                .content(message.getContent())
                .sender(principal.getName())
                .room(room)
                .type(ChatMessage.MessageType.CHAT)
                .timestamp(Instant.now())
                .build();

        chatService.saveMessage(enriched);
        return enriched;
    }

    /**
     * Handle user joining a room
     * @SendTo broadcasts to all subscribers of the topic
     */
    @MessageMapping("/chat/{room}/join")
    @SendTo("/topic/chat/{room}")
    public ChatMessage handleJoin(
            @DestinationVariable String room,
            SimpMessageHeaderAccessor headerAccessor,
            Principal principal) {

        String username = principal.getName();
        headerAccessor.getSessionAttributes().put("currentRoom", room);

        presenceService.userJoinedRoom(username, room);

        return ChatMessage.builder()
                .id(UUID.randomUUID().toString())
                .sender(username)
                .room(room)
                .type(ChatMessage.MessageType.JOIN)
                .content(username + " joined the room")
                .timestamp(Instant.now())
                .build();
    }

    /**
     * @SendToUser: sends to /user/{username}/queue/errors (only the sender sees it)
     * Useful for validation errors, private notifications
     */
    @MessageMapping("/chat/private/{targetUser}")
    @SendToUser("/queue/private-messages")
    public ChatMessage handlePrivateMessage(
            @DestinationVariable String targetUser,
            @Payload ChatMessage message,
            Principal principal) {

        // Validate message
        if (message.getContent() == null || message.getContent().isBlank()) {
            throw new IllegalArgumentException("Message cannot be empty");
        }

        return ChatMessage.builder()
                .id(UUID.randomUUID().toString())
                .content(message.getContent())
                .sender(principal.getName())
                .room("private:" + targetUser)
                .type(ChatMessage.MessageType.CHAT)
                .timestamp(Instant.now())
                .build();
    }

    /**
     * @SubscribeMapping: handles SUBSCRIBE frame directly
     * Returns data immediately to the subscriber (like an HTTP GET)
     * Client receives this when first subscribing
     */
    @SubscribeMapping("/chat/{room}/history")
    public java.util.List<ChatMessage> getChatHistory(
            @DestinationVariable String room,
            Principal principal) {

        log.debug("History request: room={}, user={}", room, principal.getName());
        return chatService.getRecentMessages(room, 50);
    }

    /**
     * Handle document edits
     * Broadcasts to all collaborators except the sender
     */
    @MessageMapping("/document/{docId}/edit")
    @SendTo("/topic/document/{docId}")
    public DocumentEdit handleDocumentEdit(
            @DestinationVariable String docId,
            @Payload DocumentEdit edit,
            Principal principal) {

        edit.setUserId(principal.getName());
        edit.setTimestamp(Instant.now());

        log.debug("Document edit: docId={}, user={}, pos={}", docId, principal.getName(), edit.getPosition());
        return edit;
    }

    /**
     * Handle exceptions in message handlers
     * Sends error back to the originating user
     */
    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public String handleException(Exception ex, Principal principal) {
        log.warn("WebSocket error for user {}: {}", principal.getName(), ex.getMessage());
        return "Error: " + ex.getMessage();
    }
}
```

---

## 5. SimpMessagingTemplate for Server Push {#server-push}

```java
package com.example.ws.service;

import com.example.ws.dto.ChatMessage;
import com.example.ws.dto.DocumentEdit;
import com.example.ws.dto.PresenceUpdate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.SimpMessageType;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class PushNotificationService {

    private final SimpMessagingTemplate messagingTemplate;

    /**
     * Broadcast to all subscribers of a topic
     * Called from REST endpoints or background jobs
     */
    public void broadcastToRoom(String room, ChatMessage message) {
        String destination = "/topic/chat/" + room;
        messagingTemplate.convertAndSend(destination, message);
        log.debug("Broadcast to {}: {}", destination, message.getSender());
    }

    /**
     * Send to a specific user
     * Routes to /user/{userId}/queue/notifications
     */
    public void sendToUser(String userId, String notification) {
        messagingTemplate.convertAndSendToUser(
                userId,
                "/queue/notifications",
                Map.of("message", notification, "timestamp", Instant.now())
        );
        log.debug("Sent notification to user: {}", userId);
    }

    /**
     * Send to user with custom headers
     */
    public void sendSystemAlertToUser(String userId, String alertType, Object payload) {
        SimpMessageHeaderAccessor headerAccessor =
                SimpMessageHeaderAccessor.create(SimpMessageType.MESSAGE);
        headerAccessor.setHeader("alertType", alertType);
        headerAccessor.setLeaveMutable(true);

        messagingTemplate.convertAndSendToUser(
                userId,
                "/queue/alerts",
                payload,
                headerAccessor.getMessageHeaders()
        );
    }

    /**
     * Broadcast document presence update to all collaborators
     */
    public void broadcastPresenceUpdate(String documentId, PresenceUpdate update) {
        messagingTemplate.convertAndSend(
                "/topic/document/" + documentId + "/presence",
                update
        );
    }

    /**
     * Async notification - doesn't block the calling thread
     */
    @Async
    public void sendAsyncNotification(String userId, Object payload) {
        try {
            messagingTemplate.convertAndSendToUser(userId, "/queue/async", payload);
        } catch (Exception e) {
            log.error("Failed to send async notification to user: {}", userId, e);
        }
    }

    /**
     * Server-initiated broadcast to all connected clients
     * Useful for system announcements
     */
    public void broadcastSystemMessage(String message) {
        messagingTemplate.convertAndSend("/topic/system",
                Map.of("message", message, "type", "SYSTEM", "timestamp", Instant.now()));
        log.info("System broadcast: {}", message);
    }
}
```

---

## 6. WebSocket Security {#websocket-security}

```java
package com.example.ws.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.socket.EnableWebSocketSecurity;
import org.springframework.security.messaging.access.intercept.MessageMatcherDelegatingAuthorizationManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableWebSocketSecurity
public class WebSocketSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/ws/**").permitAll()  // WebSocket upgrade
                        .anyRequest().authenticated()
                )
                .csrf(csrf -> csrf
                        .ignoringRequestMatchers("/ws/**")
                )
                .build();
    }

    /**
     * Authorization rules for WebSocket messages
     */
    @Bean
    public MessageMatcherDelegatingAuthorizationManager.Builder messageSecurityConfig(
            MessageMatcherDelegatingAuthorizationManager.Builder messages) {

        messages
            // All users can subscribe to public topics
            .simpSubscribeDestMatchers("/topic/chat/**").permitAll()
            .simpSubscribeDestMatchers("/topic/system").permitAll()

            // Only authenticated users can send messages
            .simpMessageDestMatchers("/app/**").authenticated()

            // Users can only subscribe to their own queue
            .simpSubscribeDestMatchers("/user/queue/**").authenticated()

            // Admin-only destinations
            .simpMessageDestMatchers("/app/admin/**").hasRole("ADMIN")
            .simpSubscribeDestMatchers("/topic/admin/**").hasRole("ADMIN")

            // Everything else requires authentication
            .anyMessage().authenticated();

        return messages;
    }
}
```

---

## 7. WebSocket with JWT Authentication {#jwt-auth}

### JWT Channel Interceptor

```java
package com.example.ws.security;

import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.support.MessageHeaderAccessor;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Component
public class JwtChannelInterceptor implements ChannelInterceptor {

    private final JwtDecoder jwtDecoder;

    public JwtChannelInterceptor(JwtDecoder jwtDecoder) {
        this.jwtDecoder = jwtDecoder;
    }

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(
                message, StompHeaderAccessor.class);

        if (accessor == null) return message;

        // Only authenticate on CONNECT command
        if (StompCommand.CONNECT.equals(accessor.getCommand())) {
            String authHeader = accessor.getFirstNativeHeader("Authorization");

            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                log.warn("WebSocket CONNECT without JWT");
                throw new org.springframework.security.authentication
                        .AuthenticationCredentialsNotFoundException(
                                "Missing JWT token");
            }

            String token = authHeader.substring(7);

            try {
                Jwt jwt = jwtDecoder.decode(token);

                // Extract user info from JWT
                String userId = jwt.getSubject();
                List<String> roles = jwt.getClaimAsStringList("roles");
                if (roles == null) roles = List.of();

                List<SimpleGrantedAuthority> authorities = roles.stream()
                        .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                        .collect(Collectors.toList());

                UsernamePasswordAuthenticationToken auth =
                        new UsernamePasswordAuthenticationToken(userId, null, authorities);

                accessor.setUser(auth);

                log.debug("WebSocket authenticated: userId={}", userId);

            } catch (Exception e) {
                log.warn("WebSocket JWT validation failed: {}", e.getMessage());
                throw new org.springframework.security.authentication
                        .BadCredentialsException("Invalid JWT token");
            }
        }

        return message;
    }
}
```

### Frontend JavaScript (STOMP with JWT)

```javascript
// Frontend WebSocket client with JWT auth
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

class WebSocketService {
    constructor(jwtToken) {
        this.jwtToken = jwtToken;
        this.client = null;
        this.subscriptions = new Map();
    }

    connect(onConnected, onDisconnected) {
        this.client = new Client({
            webSocketFactory: () => new SockJS('/ws'),

            connectHeaders: {
                Authorization: `Bearer ${this.jwtToken}`
            },

            // Reconnection settings
            reconnectDelay: 5000,
            heartbeatIncoming: 10000,
            heartbeatOutgoing: 10000,

            onConnect: (frame) => {
                console.log('Connected:', frame);
                onConnected(frame);
            },

            onDisconnect: () => {
                console.log('Disconnected');
                onDisconnected();
            },

            onStompError: (frame) => {
                console.error('STOMP error:', frame.headers.message);
            }
        });

        this.client.activate();
    }

    subscribe(destination, callback) {
        if (!this.client?.connected) {
            throw new Error('Not connected');
        }

        const subscription = this.client.subscribe(destination, (message) => {
            callback(JSON.parse(message.body));
        });

        this.subscriptions.set(destination, subscription);
        return subscription;
    }

    send(destination, body) {
        this.client?.publish({
            destination,
            body: JSON.stringify(body),
            headers: { 'content-type': 'application/json' }
        });
    }

    unsubscribe(destination) {
        const sub = this.subscriptions.get(destination);
        if (sub) {
            sub.unsubscribe();
            this.subscriptions.delete(destination);
        }
    }

    disconnect() {
        this.client?.deactivate();
    }
}

// Usage example
const ws = new WebSocketService(localStorage.getItem('jwt_token'));

ws.connect(
    () => {
        // Subscribe to chat room
        ws.subscribe('/topic/chat/general', (message) => {
            console.log('Chat message:', message);
        });

        // Subscribe to personal notifications
        ws.subscribe('/user/queue/notifications', (notification) => {
            console.log('Notification:', notification);
        });

        // Send a message
        ws.send('/app/chat/general', {
            content: 'Hello world!',
            type: 'CHAT'
        });
    },
    () => console.log('Connection closed')
);
```

---

## 8. Scaling WebSocket with Redis Pub/Sub {#redis-scaling}

With multiple application instances, a user connected to instance A won't receive messages sent via instance B. Redis pub/sub solves this.

```
Instance A                    Instance B
┌─────────┐                  ┌─────────┐
│ User 1  │                  │ User 2  │
│ User 2  │                  │ User 3  │
└────┬────┘                  └────┬────┘
     │                            │
     ▼                            ▼
┌──────────────────────────────────────┐
│         Redis Message Broker         │
│  (all instances subscribe to Redis)  │
└──────────────────────────────────────┘
```

### Redis Message Broker Configuration

```java
package com.example.ws.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class RedisWebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Value("${spring.redis.host:localhost}")
    private String redisHost;

    @Value("${spring.redis.port:6379}")
    private int redisPort;

    @Value("${spring.redis.password:}")
    private String redisPassword;

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Use Redis as external broker relay
        // This enables message sharing across all application instances
        registry.enableStompBrokerRelay(
                        "/topic",
                        "/queue"
                )
                .setRelayHost(redisHost)
                .setRelayPort(redisPort)
                .setClientLogin(redisPassword.isEmpty() ? "guest" : "default")
                .setClientPasscode(redisPassword.isEmpty() ? "guest" : redisPassword)
                .setSystemLogin(redisPassword.isEmpty() ? "guest" : "default")
                .setSystemPasscode(redisPassword.isEmpty() ? "guest" : redisPassword)
                .setSystemHeartbeatSendInterval(5000)
                .setSystemHeartbeatReceiveInterval(4000)
                .setVirtualHost("/");

        // Note: Use RabbitMQ with STOMP plugin for production multi-instance
        // redis-pubsub alone doesn't speak STOMP natively
        // RabbitMQ configuration:
        // registry.enableStompBrokerRelay("/topic", "/queue")
        //         .setRelayHost("rabbitmq-host")
        //         .setRelayPort(61613)  // RabbitMQ STOMP port
        //         .setClientLogin("guest")
        //         .setClientPasscode("guest");

        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

### Custom Redis Pub/Sub for Non-STOMP Broadcasts

```java
package com.example.ws.redis;

import com.example.ws.dto.DocumentEdit;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.connection.Message;
import org.springframework.data.redis.connection.MessageListener;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.listener.ChannelTopic;
import org.springframework.data.redis.listener.RedisMessageListenerContainer;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class RedisWebSocketBridge {

    private final RedisTemplate<String, Object> redisTemplate;
    private final SimpMessagingTemplate messagingTemplate;
    private final RedisMessageListenerContainer listenerContainer;
    private final ObjectMapper objectMapper;

    /**
     * Publish edit to Redis (from any instance)
     */
    public void publishEdit(String documentId, DocumentEdit edit) {
        String channel = "doc:edit:" + documentId;
        redisTemplate.convertAndSend(channel, edit);
    }

    /**
     * Subscribe to document edits from Redis and forward to WebSocket
     */
    public void subscribeToDocument(String documentId) {
        String channel = "doc:edit:" + documentId;

        listenerContainer.addMessageListener(
                new DocumentEditListener(documentId),
                new ChannelTopic(channel)
        );
    }

    private class DocumentEditListener implements MessageListener {
        private final String documentId;

        DocumentEditListener(String documentId) {
            this.documentId = documentId;
        }

        @Override
        public void onMessage(Message message, byte[] pattern) {
            try {
                DocumentEdit edit = objectMapper.readValue(
                        message.getBody(), DocumentEdit.class);

                // Forward to all WebSocket subscribers of this document
                messagingTemplate.convertAndSend(
                        "/topic/document/" + documentId, edit);

            } catch (Exception e) {
                log.error("Failed to process Redis message for doc: {}", documentId, e);
            }
        }
    }
}
```

---

## 9. Connection Lifecycle Management {#lifecycle}

```java
package com.example.ws.listener;

import com.example.ws.dto.ChatMessage;
import com.example.ws.service.PresenceService;
import com.example.ws.service.PushNotificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.event.EventListener;
import org.springframework.messaging.simp.SimpMessageHeaderAccessor;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.messaging.*;

import java.security.Principal;
import java.time.Instant;

@Slf4j
@Component
@RequiredArgsConstructor
public class WebSocketEventListener {

    private final PushNotificationService pushNotificationService;
    private final PresenceService presenceService;

    /**
     * Fired when a client connects (after STOMP CONNECT)
     */
    @EventListener
    public void handleSessionConnected(SessionConnectedEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        Principal user = accessor.getUser();

        if (user != null) {
            String userId = user.getName();
            presenceService.userConnected(userId, accessor.getSessionId());
            log.info("WebSocket connected: userId={}, sessionId={}",
                    userId, accessor.getSessionId());
        }
    }

    /**
     * Fired when a client sends a SUBSCRIBE frame
     */
    @EventListener
    public void handleSubscribe(SessionSubscribeEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        Principal user = accessor.getUser();
        String destination = accessor.getDestination();

        if (user != null && destination != null) {
            log.debug("Subscribe: userId={}, destination={}", user.getName(), destination);

            // If subscribing to a document, register presence
            if (destination.startsWith("/topic/document/")) {
                String docId = destination.replace("/topic/document/", "")
                        .replace("/presence", "");
                presenceService.userOpenedDocument(user.getName(), docId);
            }
        }
    }

    /**
     * Fired when a client disconnects (after STOMP DISCONNECT or TCP close)
     */
    @EventListener
    public void handleSessionDisconnect(SessionDisconnectEvent event) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(event.getMessage());
        Principal user = accessor.getUser();

        if (user != null) {
            String userId = user.getName();
            String sessionId = accessor.getSessionId();

            // Get what room/doc this user was in
            String currentRoom = (String) accessor.getSessionAttributes()
                    .getOrDefault("currentRoom", null);

            if (currentRoom != null) {
                // Notify room that user left
                pushNotificationService.broadcastToRoom(currentRoom,
                        ChatMessage.builder()
                                .type(ChatMessage.MessageType.LEAVE)
                                .sender(userId)
                                .room(currentRoom)
                                .content(userId + " left the room")
                                .timestamp(Instant.now())
                                .build());
            }

            presenceService.userDisconnected(userId, sessionId);
            log.info("WebSocket disconnected: userId={}, sessionId={}", userId, sessionId);
        }
    }
}
```

### Presence Service

```java
package com.example.ws.service;

import com.example.ws.dto.PresenceUpdate;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Service
@RequiredArgsConstructor
public class PresenceService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final PushNotificationService pushNotificationService;

    // Local session tracking: sessionId → userId
    private final ConcurrentHashMap<String, String> sessionToUser = new ConcurrentHashMap<>();

    public void userConnected(String userId, String sessionId) {
        sessionToUser.put(sessionId, userId);

        // Track in Redis for distributed presence
        String key = "presence:user:" + userId;
        redisTemplate.opsForSet().add(key, sessionId);
        redisTemplate.expire(key, Duration.ofHours(24));
    }

    public void userDisconnected(String userId, String sessionId) {
        sessionToUser.remove(sessionId);

        String key = "presence:user:" + userId;
        redisTemplate.opsForSet().remove(key, sessionId);

        // If no more sessions, user is fully offline
        Long remaining = redisTemplate.opsForSet().size(key);
        if (remaining == null || remaining == 0) {
            log.info("User fully offline: {}", userId);
        }
    }

    public void userOpenedDocument(String userId, String documentId) {
        String key = "presence:doc:" + documentId;
        redisTemplate.opsForSet().add(key, userId);
        redisTemplate.expire(key, Duration.ofHours(1));

        broadcastPresenceForDocument(documentId);
    }

    public void userClosedDocument(String userId, String documentId) {
        String key = "presence:doc:" + documentId;
        redisTemplate.opsForSet().remove(key, userId);
        broadcastPresenceForDocument(documentId);
    }

    public void userJoinedRoom(String userId, String room) {
        String key = "presence:room:" + room;
        redisTemplate.opsForSet().add(key, userId);
        redisTemplate.expire(key, Duration.ofHours(4));
    }

    @SuppressWarnings("unchecked")
    public Set<String> getUsersInDocument(String documentId) {
        String key = "presence:doc:" + documentId;
        Set<Object> members = redisTemplate.opsForSet().members(key);
        Set<String> users = new HashSet<>();
        if (members != null) {
            members.forEach(m -> users.add(m.toString()));
        }
        return users;
    }

    public boolean isUserOnline(String userId) {
        String key = "presence:user:" + userId;
        Long count = redisTemplate.opsForSet().size(key);
        return count != null && count > 0;
    }

    private void broadcastPresenceForDocument(String documentId) {
        Set<String> users = getUsersInDocument(documentId);

        Set<PresenceUpdate.UserPresence> presences = new HashSet<>();
        String[] colors = {"#E74C3C", "#3498DB", "#2ECC71", "#F39C12", "#9B59B6"};
        int i = 0;
        for (String userId : users) {
            presences.add(PresenceUpdate.UserPresence.builder()
                    .userId(userId)
                    .userName(userId)  // In real app, fetch display name
                    .color(colors[i++ % colors.length])
                    .build());
        }

        pushNotificationService.broadcastPresenceUpdate(documentId,
                PresenceUpdate.builder()
                        .documentId(documentId)
                        .activeUsers(presences)
                        .timestamp(java.time.Instant.now())
                        .build());
    }
}
```

---

## 10. Reconnection and Heartbeat {#reconnect-heartbeat}

### Heartbeat Configuration

```java
package com.example.ws.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketTransportRegistration;

@Configuration
@EnableWebSocketMessageBroker
public class HeartbeatConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue")
                // Send heartbeat every 10 seconds
                // Client must respond within 20 seconds
                .setHeartbeatValue(new long[]{10000L, 20000L});

        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
        registration
                .setMessageSizeLimit(512 * 1024)    // 512 KB per message
                .setSendBufferSizeLimit(1024 * 1024) // 1 MB send buffer
                .setSendTimeLimit(20 * 1000)          // 20 second send timeout
                .setTimeToFirstMessage(30 * 1000);    // 30 seconds to send first message
    }
}
```

---

## 11. WebSocket vs SSE {#websocket-vs-sse}

### Server-Sent Events (SSE) Implementation

```java
package com.example.ws.controller;

import com.example.ws.dto.ChatMessage;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

/**
 * SSE is simpler than WebSocket for server-to-client only scenarios.
 * Uses standard HTTP; works through proxies; automatic reconnection.
 * Use when: live feed, notifications, progress updates (no client→server messages needed)
 */
@Slf4j
@RestController
@RequestMapping("/api/sse")
@RequiredArgsConstructor
public class SseController {

    // Map of userId → sink for pushing events
    private final ConcurrentHashMap<String, Sinks.Many<ServerSentEvent<Object>>>
            userSinks = new ConcurrentHashMap<>();

    /**
     * Client connects here to receive live events
     * GET /api/sse/events
     * Accept: text/event-stream
     */
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Object>> streamEvents(
            @RequestHeader(value = "X-User-Id", required = false) String userId) {

        if (userId == null) userId = "anonymous-" + UUID.randomUUID();

        Sinks.Many<ServerSentEvent<Object>> sink = Sinks.many().multicast().onBackpressureBuffer();
        userSinks.put(userId, sink);

        final String finalUserId = userId;
        log.info("SSE connection established: userId={}", finalUserId);

        return sink.asFlux()
                // Keep-alive comment every 30 seconds (prevents proxy timeout)
                .mergeWith(Flux.interval(Duration.ofSeconds(30))
                        .map(i -> ServerSentEvent.builder()
                                .comment("keep-alive")
                                .build()))
                .doOnCancel(() -> {
                    userSinks.remove(finalUserId);
                    log.info("SSE connection closed: userId={}", finalUserId);
                })
                .doOnError(e -> userSinks.remove(finalUserId));
    }

    /**
     * Push an event to a specific user (called from services)
     */
    public void pushToUser(String userId, String eventType, Object data) {
        Sinks.Many<ServerSentEvent<Object>> sink = userSinks.get(userId);
        if (sink != null) {
            ServerSentEvent<Object> event = ServerSentEvent.builder(data)
                    .id(UUID.randomUUID().toString())
                    .event(eventType)
                    .build();
            sink.tryEmitNext(event);
        }
    }

    /**
     * Broadcast to all connected SSE clients
     */
    public void broadcast(String eventType, Object data) {
        ServerSentEvent<Object> event = ServerSentEvent.builder(data)
                .id(UUID.randomUUID().toString())
                .event(eventType)
                .build();

        userSinks.values().forEach(sink -> sink.tryEmitNext(event));
        log.debug("SSE broadcast to {} clients: {}", userSinks.size(), eventType);
    }
}
```

### SSE Client (JavaScript)

```javascript
// SSE client - much simpler than WebSocket
const eventSource = new EventSource('/api/sse/events', {
    withCredentials: true  // Include auth cookies
});

eventSource.addEventListener('chat', (e) => {
    const message = JSON.parse(e.data);
    console.log('Chat:', message);
});

eventSource.addEventListener('notification', (e) => {
    const notification = JSON.parse(e.data);
    console.log('Notification:', notification);
});

eventSource.onopen = () => console.log('SSE connected');
eventSource.onerror = () => {
    console.log('SSE error - browser will auto-reconnect after 3 seconds');
};
```

---

## 12. Real Example: Real-time Collaborative Document Editing {#real-example}

### Document Entity

```java
package com.example.ws.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "documents")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Document {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(name = "owner_id", nullable = false)
    private String ownerId;

    @Version
    @Builder.Default
    private long version = 0L;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private Instant updatedAt;
}
```

### Operational Transform for Conflict Resolution

```java
package com.example.ws.collab;

import com.example.ws.dto.DocumentEdit;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * Simplified Operational Transform (OT) for collaborative editing.
 * In production, use a library like ShareDB or Yjs (via WebSocket bridge).
 */
@Slf4j
@Service
public class OperationalTransformService {

    /**
     * Transform edit B given that edit A was applied first.
     * Used when two clients make concurrent edits.
     */
    public DocumentEdit transform(DocumentEdit editB, DocumentEdit editA) {
        int posA = editA.getPosition();
        int lenA = editA.getLength();
        int insertA = editA.getInsertedText() != null ? editA.getInsertedText().length() : 0;

        int posB = editB.getPosition();

        // Simple transformation rules:
        // If A is entirely before B, shift B's position by A's net change
        if (posA + lenA <= posB) {
            int netChange = insertA - lenA;
            return DocumentEdit.builder()
                    .documentId(editB.getDocumentId())
                    .userId(editB.getUserId())
                    .position(posB + netChange)
                    .length(editB.getLength())
                    .insertedText(editB.getInsertedText())
                    .version(editB.getVersion())
                    .timestamp(editB.getTimestamp())
                    .build();
        }

        // A and B overlap or B is before A - return B unchanged
        // (A sophisticated OT engine would handle overlap cases)
        return editB;
    }

    /**
     * Apply an edit to document content
     */
    public String applyEdit(String content, DocumentEdit edit) {
        if (content == null) content = "";

        int pos = Math.min(edit.getPosition(), content.length());
        int end = Math.min(pos + edit.getLength(), content.length());

        StringBuilder sb = new StringBuilder(content);
        sb.replace(pos, end, edit.getInsertedText() != null ? edit.getInsertedText() : "");
        return sb.toString();
    }
}
```

### Collaborative Document Service

```java
package com.example.ws.service;

import com.example.ws.collab.OperationalTransformService;
import com.example.ws.dto.DocumentEdit;
import com.example.ws.entity.Document;
import com.example.ws.repository.DocumentRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Service
@RequiredArgsConstructor
public class CollaborativeDocumentService {

    private final DocumentRepository documentRepository;
    private final OperationalTransformService otService;
    private final SimpMessagingTemplate messagingTemplate;
    private final RedisTemplate<String, Object> redisTemplate;
    private final PresenceService presenceService;

    // In-memory pending edits buffer (flushed to DB periodically)
    private final ConcurrentHashMap<String, List<DocumentEdit>> pendingEdits =
            new ConcurrentHashMap<>();

    /**
     * Process an incoming document edit
     */
    @Transactional
    public DocumentEdit processEdit(String documentId, DocumentEdit edit) {
        Document document = documentRepository.findById(documentId)
                .orElseThrow(() -> new RuntimeException("Document not found: " + documentId));

        // Get any concurrent edits that happened since client's version
        List<DocumentEdit> concurrentEdits = getPendingEditsSince(
                documentId, edit.getVersion());

        // Transform edit against concurrent operations
        DocumentEdit transformedEdit = edit;
        for (DocumentEdit concurrent : concurrentEdits) {
            if (!concurrent.getUserId().equals(edit.getUserId())) {
                transformedEdit = otService.transform(transformedEdit, concurrent);
            }
        }

        // Apply to document content
        String newContent = otService.applyEdit(document.getContent(), transformedEdit);
        document.setContent(newContent);
        documentRepository.save(document);

        // Store edit in buffer
        transformedEdit.setVersion(document.getVersion());
        bufferEdit(documentId, transformedEdit);

        // Store in Redis for cross-instance distribution
        cacheEditInRedis(documentId, transformedEdit);

        return transformedEdit;
    }

    /**
     * Get document with current content (for initial load)
     */
    @Transactional(readOnly = true)
    public Document getDocument(String documentId) {
        return documentRepository.findById(documentId)
                .orElseThrow(() -> new RuntimeException("Document not found: " + documentId));
    }

    /**
     * Handle cursor position updates (for presence display)
     */
    public void updateCursorPosition(String documentId, String userId, int position) {
        String redisKey = "cursor:" + documentId + ":" + userId;
        redisTemplate.opsForValue().set(redisKey, position, Duration.ofMinutes(5));

        // Broadcast cursor position to other collaborators
        messagingTemplate.convertAndSend(
                "/topic/document/" + documentId + "/cursors",
                Map.of(
                        "userId", userId,
                        "position", position,
                        "timestamp", Instant.now()
                )
        );
    }

    private List<DocumentEdit> getPendingEditsSince(String documentId, long sinceVersion) {
        return pendingEdits.getOrDefault(documentId, Collections.emptyList())
                .stream()
                .filter(e -> e.getVersion() > sinceVersion)
                .sorted(Comparator.comparingLong(DocumentEdit::getVersion))
                .toList();
    }

    private void bufferEdit(String documentId, DocumentEdit edit) {
        pendingEdits.computeIfAbsent(documentId, k -> new ArrayList<>()).add(edit);

        // Keep only last 100 edits per document
        List<DocumentEdit> edits = pendingEdits.get(documentId);
        if (edits.size() > 100) {
            edits.subList(0, edits.size() - 100).clear();
        }
    }

    private void cacheEditInRedis(String documentId, DocumentEdit edit) {
        String key = "doc:edits:" + documentId;
        redisTemplate.opsForList().rightPush(key, edit);
        redisTemplate.expire(key, Duration.ofHours(1));
        // Keep only last 200 edits in Redis
        redisTemplate.opsForList().trim(key, -200, -1);
    }

    /**
     * Periodic save: flush buffered edits to ensure durability
     */
    @Scheduled(fixedDelay = 5000)
    @Transactional
    public void flushPendingEdits() {
        pendingEdits.forEach((docId, edits) -> {
            if (!edits.isEmpty()) {
                log.debug("Flushing {} edits for document: {}", edits.size(), docId);
                // In production: apply all pending edits as a batch
            }
        });
    }
}
```

### Collaborative Document WebSocket Controller

```java
package com.example.ws.controller;

import com.example.ws.dto.DocumentEdit;
import com.example.ws.dto.PresenceUpdate;
import com.example.ws.entity.Document;
import com.example.ws.service.CollaborativeDocumentService;
import com.example.ws.service.PresenceService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.annotation.SendToUser;
import org.springframework.messaging.simp.annotation.SubscribeMapping;
import org.springframework.stereotype.Controller;

import java.security.Principal;
import java.time.Instant;

@Slf4j
@Controller
@RequiredArgsConstructor
public class CollaborativeDocController {

    private final CollaborativeDocumentService documentService;
    private final PresenceService presenceService;

    /**
     * When user subscribes to /app/document/{id}, send current state
     */
    @SubscribeMapping("/document/{id}/subscribe")
    public Document subscribeToDocument(
            @DestinationVariable String id,
            Principal principal) {

        log.info("User {} subscribed to document: {}", principal.getName(), id);
        presenceService.userOpenedDocument(principal.getName(), id);
        return documentService.getDocument(id);
    }

    /**
     * Handle an edit operation
     * Transforms and broadcasts to all collaborators
     */
    @MessageMapping("/document/{id}/edit")
    @SendTo("/topic/document/{id}")
    public DocumentEdit handleEdit(
            @DestinationVariable String id,
            @Payload DocumentEdit edit,
            Principal principal) {

        edit.setUserId(principal.getName());
        edit.setTimestamp(Instant.now());

        return documentService.processEdit(id, edit);
    }

    /**
     * Handle cursor position update
     * Broadcast to other collaborators only (not the sender)
     */
    @MessageMapping("/document/{id}/cursor")
    public void handleCursorUpdate(
            @DestinationVariable String id,
            @Payload java.util.Map<String, Object> cursorData,
            Principal principal) {

        int position = ((Number) cursorData.getOrDefault("position", 0)).intValue();
        documentService.updateCursorPosition(id, principal.getName(), position);
    }

    /**
     * Handle exception: send error back to the user only
     */
    @MessageExceptionHandler
    @SendToUser("/queue/doc-errors")
    public String handleDocumentError(Exception ex, Principal principal) {
        log.warn("Document operation error for {}: {}", principal.getName(), ex.getMessage());
        return ex.getMessage();
    }
}
```

### Chat Service

```java
package com.example.ws.service;

import com.example.ws.dto.ChatMessage;
import com.example.ws.repository.ChatMessageRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class ChatService {

    private final ChatMessageRepository repository;

    @Transactional
    public void saveMessage(ChatMessage message) {
        repository.save(com.example.ws.entity.ChatMessageEntity.fromDto(message));
    }

    @Transactional(readOnly = true)
    @Cacheable(value = "chat-history", key = "#room + ':' + #limit")
    public List<ChatMessage> getRecentMessages(String room, int limit) {
        return repository.findRecentByRoom(room, limit);
    }
}
```

### Application Configuration

```yaml
# application.yml
spring:
  application:
    name: collaborative-editor

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/collab_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASS:postgres}

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER:http://localhost:9000}

# WebSocket tuning
server:
  tomcat:
    threads:
      max: 200
    max-connections: 10000
    accept-count: 100
  servlet:
    session:
      timeout: 30m

# Allow larger WebSocket messages for documents
spring:
  web:
    websocket:
      max-text-message-buffer-size: 1048576  # 1 MB
      max-binary-message-buffer-size: 1048576

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,websocket
```

---

## 13. Summary {#summary}

| Feature | How | Key Classes |
|---------|-----|------------|
| **WebSocket Setup** | `@EnableWebSocketMessageBroker` | `WebSocketMessageBrokerConfigurer` |
| **Receive messages** | `@MessageMapping("/path")` | `StompHeaderAccessor` |
| **Broadcast to topic** | `@SendTo("/topic/...")` | `SimpMessagingTemplate` |
| **Send to user** | `@SendToUser("/queue/...")` | `convertAndSendToUser()` |
| **Server push** | `SimpMessagingTemplate.convertAndSend()` | Inject anywhere |
| **Initial data on subscribe** | `@SubscribeMapping` | Returns data directly |
| **JWT auth** | `ChannelInterceptor` on CONNECT | `JwtDecoder` |
| **Security rules** | `MessageMatcherDelegatingAuthorizationManager` | `@EnableWebSocketSecurity` |
| **Multi-instance** | Redis/RabbitMQ broker relay | `enableStompBrokerRelay()` |
| **Connection events** | `@EventListener(SessionConnectedEvent)` | `SessionDisconnectEvent` |
| **SSE alternative** | `Flux<ServerSentEvent<T>>` (WebFlux) | `SinkManySpec` |
| **Collaborative OT** | Operational Transform algorithm | Custom or ShareDB |

### WebSocket vs SSE Decision Matrix

| Factor | WebSocket | SSE |
|--------|-----------|-----|
| Direction | Bidirectional | Server → Client only |
| Protocol | WS/WSS | HTTP/HTTPS |
| Reconnection | Manual | Automatic |
| Proxy support | Varies | Excellent |
| Browser support | All modern | All modern |
| Complexity | Higher | Lower |
| Use case | Chat, gaming, collaboration | Notifications, feeds, dashboards |

### Production Checklist

```
[ ] Use RabbitMQ (with STOMP plugin) or Redis for multi-instance broker relay
[ ] Validate JWT on CONNECT, not just HTTP
[ ] Set message size limits (prevent DoS)
[ ] Implement heartbeat to detect dead connections
[ ] Add rate limiting on WebSocket messages
[ ] Implement reconnect with exponential backoff on client
[ ] Monitor active WebSocket connections via Actuator
[ ] Use @Async for operations triggered by WS messages
[ ] Set session timeout and enforce it
[ ] Test with 1000+ concurrent connections before production
```

---

> **Congratulations!** You have completed the advanced Spring Boot series. This course covered everything from core Java fundamentals through cloud deployments, DevOps, security, database patterns, and real-time features. Build something amazing!
