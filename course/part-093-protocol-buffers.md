# Part 093: Protocol Buffers and gRPC Deep Dive

## Introduction

Protocol Buffers (protobuf) is a language-neutral binary serialization format developed by Google. gRPC is an RPC framework that uses protobuf for message encoding and HTTP/2 for transport, enabling efficient, strongly-typed service-to-service communication.

---

## Project Setup

```xml
<!-- pom.xml -->
<properties>
    <protobuf.version>3.25.1</protobuf.version>
    <grpc.version>1.60.0</grpc.version>
    <os-maven-plugin.version>1.7.1</os-maven-plugin.version>
    <protobuf-maven-plugin.version>0.6.1</protobuf-maven-plugin.version>
</properties>

<dependencies>
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>3.0.0.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java-util</artifactId>
        <version>${protobuf.version}</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>${os-maven-plugin.version}</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>${protobuf-maven-plugin.version}</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

## Protobuf 3 Syntax

### Basic Message Types

```protobuf
// src/main/proto/common.proto
syntax = "proto3";

package com.example.grpc;

option java_package = "com.example.grpc.proto";
option java_outer_classname = "CommonProto";
option java_multiple_files = true;

// Scalar types
message Address {
  string street = 1;
  string city = 2;
  string state = 3;
  string zip_code = 4;
  string country = 5;
}

// Enum
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // Always have a zero value
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

// oneof - mutually exclusive fields
message PaymentMethod {
  oneof method {
    CreditCard credit_card = 1;
    BankTransfer bank_transfer = 2;
    Crypto crypto = 3;
  }
}

message CreditCard {
  string number = 1;
  string expiry = 2;
  string cvv = 3;
}

message BankTransfer {
  string bank_code = 1;
  string account_number = 2;
}

message Crypto {
  string wallet_address = 1;
  string currency = 2;
}

// map field
message ProductInventory {
  map<string, int32> stock_by_warehouse = 1; // warehouse_id -> quantity
}

// repeated field (list)
message OrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  Address shipping_address = 3;
  PaymentMethod payment_method = 4;
  map<string, string> metadata = 5;  // custom key-value pairs
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double unit_price = 3;
  string currency = 4;
}

message OrderResponse {
  string order_id = 1;
  OrderStatus status = 2;
  double total_amount = 3;
  string created_at = 4;   // ISO 8601
  string estimated_delivery = 5;
}
```

### Well-Known Types

```protobuf
// src/main/proto/user.proto
syntax = "proto3";

package com.example.grpc;

option java_package = "com.example.grpc.proto";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";  // for nullable scalars
import "google/protobuf/empty.proto";
import "google/protobuf/any.proto";

message User {
  string id = 1;
  string email = 2;
  string name = 3;

  // Using Timestamp instead of string for dates
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;

  // Nullable fields using wrapper types (different from proto3 default behavior)
  google.protobuf.StringValue phone = 6;  // nullable string
  google.protobuf.Int32Value age = 7;     // nullable int

  repeated string roles = 8;
  bool active = 9;
}

message GetUserRequest {
  string user_id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  bool has_next_page = 3;
  string next_page_token = 4;
}
```

---

## gRPC Service Definitions

### All Four Service Types

```protobuf
// src/main/proto/chat.proto
syntax = "proto3";

package com.example.grpc;

option java_package = "com.example.grpc.proto";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

message ChatMessage {
  string message_id = 1;
  string room_id = 2;
  string sender_id = 3;
  string sender_name = 4;
  string content = 5;
  MessageType type = 6;
  google.protobuf.Timestamp sent_at = 7;
  repeated string mentioned_users = 8;
}

enum MessageType {
  MESSAGE_TYPE_UNSPECIFIED = 0;
  MESSAGE_TYPE_TEXT = 1;
  MESSAGE_TYPE_IMAGE = 2;
  MESSAGE_TYPE_FILE = 3;
  MESSAGE_TYPE_SYSTEM = 4;
}

message JoinRoomRequest {
  string room_id = 1;
  string user_id = 2;
  string display_name = 3;
}

message JoinRoomResponse {
  bool success = 1;
  string room_name = 2;
  repeated string current_members = 3;
  repeated ChatMessage recent_messages = 4;
}

message SendMessageRequest {
  ChatMessage message = 1;
}

message SendMessageResponse {
  string message_id = 1;
  bool delivered = 2;
  google.protobuf.Timestamp server_timestamp = 3;
}

message RoomHistoryRequest {
  string room_id = 1;
  int32 limit = 2;
  string before_message_id = 3;
}

message FileChunk {
  string upload_id = 1;
  bytes data = 2;
  int32 chunk_number = 3;
  bool is_last = 4;
  string filename = 5;
  string content_type = 6;
}

message UploadResponse {
  string file_url = 1;
  int64 total_bytes = 2;
}

message TypingIndicator {
  string room_id = 1;
  string user_id = 2;
  bool is_typing = 3;
}

service ChatService {
  // Unary: join a room
  rpc JoinRoom(JoinRoomRequest) returns (JoinRoomResponse);

  // Server streaming: subscribe to room messages
  rpc SubscribeToRoom(JoinRoomRequest) returns (stream ChatMessage);

  // Client streaming: upload a file in chunks
  rpc UploadFile(stream FileChunk) returns (UploadResponse);

  // Bidirectional streaming: live chat
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);

  // Another unary
  rpc GetRoomHistory(RoomHistoryRequest) returns (stream ChatMessage);
}
```

---

## Server Implementation

### Unary RPC

```java
// src/main/java/com/example/grpc/server/ChatGrpcService.java
package com.example.grpc.server;

import com.example.grpc.proto.*;
import com.google.protobuf.Timestamp;
import io.grpc.Status;
import io.grpc.StatusRuntimeException;
import io.grpc.stub.StreamObserver;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.devh.boot.grpc.server.service.GrpcService;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.List;

@Slf4j
@GrpcService
@RequiredArgsConstructor
public class ChatGrpcService extends ChatServiceGrpc.ChatServiceImplBase {

    // In-memory storage for demo (use Redis/DB in production)
    private final Map<String, List<StreamObserver<ChatMessage>>> roomSubscribers =
        new ConcurrentHashMap<>();
    private final Map<String, List<ChatMessage>> roomHistory = new ConcurrentHashMap<>();

    private final ChatRoomService chatRoomService;

    @Override
    public void joinRoom(JoinRoomRequest request,
                         StreamObserver<JoinRoomResponse> responseObserver) {
        try {
            log.info("User {} joining room {}", request.getUserId(), request.getRoomId());

            // Validate request
            if (request.getRoomId().isBlank()) {
                responseObserver.onError(Status.INVALID_ARGUMENT
                    .withDescription("room_id cannot be empty")
                    .asRuntimeException());
                return;
            }

            // Build response
            JoinRoomResponse response = chatRoomService.joinRoom(request);

            responseObserver.onNext(response);
            responseObserver.onCompleted();

        } catch (RoomNotFoundException e) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("Room not found: " + request.getRoomId())
                .withCause(e)
                .asRuntimeException());
        } catch (Exception e) {
            log.error("Error joining room", e);
            responseObserver.onError(Status.INTERNAL
                .withDescription("Internal server error")
                .withCause(e)
                .asRuntimeException());
        }
    }

    // Server streaming: push messages as they arrive
    @Override
    public void subscribeToRoom(JoinRoomRequest request,
                                StreamObserver<ChatMessage> responseObserver) {
        String roomId = request.getRoomId();
        String userId = request.getUserId();

        log.info("User {} subscribing to room {}", userId, roomId);

        // Send recent history
        List<ChatMessage> history = roomHistory.getOrDefault(roomId, List.of());
        int start = Math.max(0, history.size() - 50);
        history.subList(start, history.size()).forEach(responseObserver::onNext);

        // Register subscriber for real-time messages
        roomSubscribers.computeIfAbsent(roomId, k -> new CopyOnWriteArrayList<>())
            .add(responseObserver);

        // The stream stays open until the client disconnects
        // In production, you'd use a proper cleanup mechanism
    }

    // Client streaming: receive file chunks, return summary
    @Override
    public StreamObserver<FileChunk> uploadFile(StreamObserver<UploadResponse> responseObserver) {
        return new StreamObserver<>() {
            private final List<byte[]> chunks = new CopyOnWriteArrayList<>();
            private long totalBytes = 0;
            private String filename = "";
            private String contentType = "";
            private String uploadId = UUID.randomUUID().toString();

            @Override
            public void onNext(FileChunk chunk) {
                log.debug("Received chunk {} for upload {}", chunk.getChunkNumber(), uploadId);
                chunks.add(chunk.getData().toByteArray());
                totalBytes += chunk.getData().size();
                filename = chunk.getFilename();
                contentType = chunk.getContentType();

                if (chunk.getIsLast()) {
                    log.info("Last chunk received for {}", filename);
                }
            }

            @Override
            public void onError(Throwable t) {
                log.error("Upload error for {}: {}", filename, t.getMessage());
                // Clean up partial upload
            }

            @Override
            public void onCompleted() {
                try {
                    // Combine chunks and store file
                    byte[] fileData = combineChunks(chunks);
                    String fileUrl = chatRoomService.storeFile(uploadId, filename,
                        contentType, fileData);

                    responseObserver.onNext(UploadResponse.newBuilder()
                        .setFileUrl(fileUrl)
                        .setTotalBytes(totalBytes)
                        .build());
                    responseObserver.onCompleted();
                    log.info("File upload complete: {} ({} bytes)", filename, totalBytes);
                } catch (Exception e) {
                    responseObserver.onError(Status.INTERNAL
                        .withDescription("Failed to store file: " + e.getMessage())
                        .asRuntimeException());
                }
            }

            private byte[] combineChunks(List<byte[]> chunks) {
                int total = chunks.stream().mapToInt(b -> b.length).sum();
                byte[] combined = new byte[total];
                int offset = 0;
                for (byte[] chunk : chunks) {
                    System.arraycopy(chunk, 0, combined, offset, chunk.length);
                    offset += chunk.length;
                }
                return combined;
            }
        };
    }

    // Bidirectional streaming: full-duplex chat
    @Override
    public StreamObserver<ChatMessage> chat(StreamObserver<ChatMessage> responseObserver) {
        String sessionId = UUID.randomUUID().toString();
        log.info("New chat session: {}", sessionId);

        return new StreamObserver<>() {
            private String roomId;
            private String userId;

            @Override
            public void onNext(ChatMessage incomingMessage) {
                log.debug("Received message from {} in room {}",
                    incomingMessage.getSenderId(), incomingMessage.getRoomId());

                roomId = incomingMessage.getRoomId();
                userId = incomingMessage.getSenderId();

                // Enrich message with server timestamp
                ChatMessage serverMessage = incomingMessage.toBuilder()
                    .setMessageId(UUID.randomUUID().toString())
                    .setSentAt(Timestamp.newBuilder()
                        .setSeconds(Instant.now().getEpochSecond())
                        .setNanos(Instant.now().getNano())
                        .build())
                    .build();

                // Store in history
                roomHistory.computeIfAbsent(roomId, k -> new CopyOnWriteArrayList<>())
                    .add(serverMessage);

                // Broadcast to all subscribers in the room
                broadcastToRoom(roomId, serverMessage, responseObserver);
            }

            @Override
            public void onError(Throwable t) {
                log.error("Chat stream error for session {}: {}", sessionId, t.getMessage());
                cleanupSession();
            }

            @Override
            public void onCompleted() {
                log.info("Chat session {} completed", sessionId);
                cleanupSession();
                responseObserver.onCompleted();
            }

            private void cleanupSession() {
                if (roomId != null) {
                    List<StreamObserver<ChatMessage>> subscribers =
                        roomSubscribers.get(roomId);
                    if (subscribers != null) {
                        subscribers.remove(responseObserver);
                    }
                }
            }
        };
    }

    private void broadcastToRoom(String roomId, ChatMessage message,
                                  StreamObserver<ChatMessage> sender) {
        List<StreamObserver<ChatMessage>> subscribers = roomSubscribers.get(roomId);
        if (subscribers == null) return;

        List<StreamObserver<ChatMessage>> dead = new CopyOnWriteArrayList<>();
        for (StreamObserver<ChatMessage> subscriber : subscribers) {
            try {
                subscriber.onNext(message);
            } catch (Exception e) {
                log.warn("Failed to send to subscriber, removing: {}", e.getMessage());
                dead.add(subscriber);
            }
        }
        subscribers.removeAll(dead);
    }
}
```

---

## Client Implementation

```java
// src/main/java/com/example/grpc/client/ChatGrpcClient.java
package com.example.grpc.client;

import com.example.grpc.proto.*;
import com.google.protobuf.Timestamp;
import io.grpc.stub.StreamObserver;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class ChatGrpcClient {

    @GrpcClient("chat-service")
    private ChatServiceGrpc.ChatServiceBlockingStub blockingStub;

    @GrpcClient("chat-service")
    private ChatServiceGrpc.ChatServiceStub asyncStub;

    /**
     * Unary call: join a room synchronously
     */
    public JoinRoomResponse joinRoom(String roomId, String userId, String displayName) {
        JoinRoomRequest request = JoinRoomRequest.newBuilder()
            .setRoomId(roomId)
            .setUserId(userId)
            .setDisplayName(displayName)
            .build();

        return blockingStub.joinRoom(request);
    }

    /**
     * Server streaming: subscribe to a room's messages
     */
    public void subscribeToRoom(String roomId, String userId,
                                 MessageHandler handler) {
        JoinRoomRequest request = JoinRoomRequest.newBuilder()
            .setRoomId(roomId)
            .setUserId(userId)
            .build();

        asyncStub.subscribeToRoom(request, new StreamObserver<>() {
            @Override
            public void onNext(ChatMessage message) {
                log.debug("Received message: {}", message.getContent());
                handler.handle(message);
            }

            @Override
            public void onError(Throwable t) {
                log.error("Stream error: {}", t.getMessage());
                handler.onError(t);
            }

            @Override
            public void onCompleted() {
                log.info("Room subscription completed");
                handler.onCompleted();
            }
        });
    }

    /**
     * Client streaming: upload a file in chunks
     */
    public UploadResponse uploadFile(String filename, String contentType,
                                      byte[] fileData) throws InterruptedException {
        final UploadResponse[] result = new UploadResponse[1];
        final Throwable[] error = new Throwable[1];
        CountDownLatch latch = new CountDownLatch(1);

        StreamObserver<FileChunk> requestObserver = asyncStub.uploadFile(
            new StreamObserver<>() {
                @Override
                public void onNext(UploadResponse response) {
                    result[0] = response;
                }

                @Override
                public void onError(Throwable t) {
                    error[0] = t;
                    latch.countDown();
                }

                @Override
                public void onCompleted() {
                    latch.countDown();
                }
            });

        // Send file in 64KB chunks
        int chunkSize = 64 * 1024;
        int totalChunks = (int) Math.ceil((double) fileData.length / chunkSize);

        for (int i = 0; i < totalChunks; i++) {
            int start = i * chunkSize;
            int end = Math.min(start + chunkSize, fileData.length);
            boolean isLast = (i == totalChunks - 1);

            FileChunk chunk = FileChunk.newBuilder()
                .setUploadId(UUID.randomUUID().toString())
                .setData(com.google.protobuf.ByteString.copyFrom(fileData, start, end - start))
                .setChunkNumber(i)
                .setIsLast(isLast)
                .setFilename(filename)
                .setContentType(contentType)
                .build();

            requestObserver.onNext(chunk);
            log.debug("Sent chunk {}/{}", i + 1, totalChunks);
        }

        requestObserver.onCompleted();

        if (!latch.await(60, TimeUnit.SECONDS)) {
            throw new RuntimeException("Upload timed out");
        }
        if (error[0] != null) {
            throw new RuntimeException("Upload failed", error[0]);
        }

        return result[0];
    }

    /**
     * Bidirectional streaming: live chat session
     */
    public StreamObserver<ChatMessage> startChatSession(MessageHandler handler) {
        return asyncStub.chat(new StreamObserver<>() {
            @Override
            public void onNext(ChatMessage message) {
                handler.handle(message);
            }

            @Override
            public void onError(Throwable t) {
                log.error("Chat session error: {}", t.getMessage());
                handler.onError(t);
            }

            @Override
            public void onCompleted() {
                log.info("Chat session completed");
                handler.onCompleted();
            }
        });
    }

    public ChatMessage buildMessage(String roomId, String senderId,
                                     String senderName, String content) {
        return ChatMessage.newBuilder()
            .setRoomId(roomId)
            .setSenderId(senderId)
            .setSenderName(senderName)
            .setContent(content)
            .setType(MessageType.MESSAGE_TYPE_TEXT)
            .setSentAt(Timestamp.newBuilder()
                .setSeconds(Instant.now().getEpochSecond())
                .build())
            .build();
    }

    @FunctionalInterface
    interface MessageHandler {
        void handle(ChatMessage message);
        default void onError(Throwable t) { log.error("Error: {}", t.getMessage()); }
        default void onCompleted() { log.info("Completed"); }
    }
}
```

---

## gRPC Interceptors

### Server-side Auth Interceptor

```java
// src/main/java/com/example/grpc/interceptor/AuthInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.springframework.core.annotation.Order;

@Slf4j
@GrpcGlobalServerInterceptor
@RequiredArgsConstructor
@Order(10)
public class AuthInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_HEADER =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    // Context key to pass user info to service implementations
    public static final Context.Key<String> USER_ID_KEY =
        Context.key("userId");
    public static final Context.Key<String> USER_ROLE_KEY =
        Context.key("userRole");

    private final JwtService jwtService;

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        log.debug("Intercepting call to: {}", methodName);

        // Skip auth for health checks and public endpoints
        if (isPublicMethod(methodName)) {
            return next.startCall(call, headers);
        }

        String authHeader = headers.get(AUTH_HEADER);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED.withDescription("Missing or invalid auth token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }

        String token = authHeader.substring(7);
        try {
            JwtClaims claims = jwtService.validateToken(token);

            // Add user info to context
            Context ctx = Context.current()
                .withValue(USER_ID_KEY, claims.getUserId())
                .withValue(USER_ROLE_KEY, claims.getRole());

            return Contexts.interceptCall(ctx, call, headers, next);

        } catch (JwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid token: " + e.getMessage()), new Metadata());
            return new ServerCall.Listener<>() {};
        }
    }

    private boolean isPublicMethod(String methodName) {
        return methodName.contains("JoinRoom") || methodName.contains("Health");
    }
}
```

### Logging Interceptor

```java
// src/main/java/com/example/grpc/interceptor/LoggingInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.springframework.core.annotation.Order;

@Slf4j
@GrpcGlobalServerInterceptor
@RequiredArgsConstructor
@Order(20)
public class LoggingInterceptor implements ServerInterceptor {

    private final MeterRegistry meterRegistry;

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();

        log.info("gRPC call started: {}", methodName);

        ServerCall<ReqT, RespT> wrappedCall = new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                long duration = System.currentTimeMillis() - startTime;
                log.info("gRPC call completed: {} | Status: {} | Duration: {}ms",
                    methodName, status.getCode(), duration);

                // Record metrics
                Timer.builder("grpc.server.calls")
                    .tag("method", methodName)
                    .tag("status", status.getCode().name())
                    .register(meterRegistry)
                    .record(duration, java.util.concurrent.TimeUnit.MILLISECONDS);

                super.close(status, trailers);
            }
        };

        return next.startCall(wrappedCall, headers);
    }
}
```

### Client-side Interceptor

```java
// src/main/java/com/example/grpc/interceptor/ClientAuthInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import lombok.RequiredArgsConstructor;
import net.devh.boot.grpc.client.interceptor.GrpcGlobalClientInterceptor;

@GrpcGlobalClientInterceptor
@RequiredArgsConstructor
public class ClientAuthInterceptor implements ClientInterceptor {

    private static final Metadata.Key<String> AUTH_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    private static final Metadata.Key<String> TRACE_KEY =
        Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER);

    private final TokenProvider tokenProvider;

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {

            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                // Attach auth token
                headers.put(AUTH_KEY, "Bearer " + tokenProvider.getToken());

                // Attach trace ID for distributed tracing
                String traceId = java.util.UUID.randomUUID().toString();
                headers.put(TRACE_KEY, traceId);

                super.start(responseListener, headers);
            }
        };
    }
}
```

---

## gRPC Error Handling

```java
// src/main/java/com/example/grpc/server/ErrorHandlingService.java
package com.example.grpc.server;

import com.google.rpc.Code;
import com.google.rpc.ErrorInfo;
import com.google.rpc.Status;
import io.grpc.StatusRuntimeException;
import io.grpc.protobuf.StatusProto;
import io.grpc.stub.StreamObserver;

public class ErrorHandlingService {

    /**
     * Rich error handling with google.rpc.Status
     */
    protected <T> void handleError(StreamObserver<T> observer, Exception e) {
        if (e instanceof ValidationException ve) {
            // INVALID_ARGUMENT for client errors
            observer.onError(io.grpc.Status.INVALID_ARGUMENT
                .withDescription(ve.getMessage())
                .augmentDescription("Field: " + ve.getField())
                .asRuntimeException());

        } else if (e instanceof NotFoundException nfe) {
            observer.onError(io.grpc.Status.NOT_FOUND
                .withDescription(nfe.getMessage())
                .withCause(e)
                .asRuntimeException());

        } else if (e instanceof RateLimitException rle) {
            // Use google.rpc.Status for rich errors
            Status richStatus = Status.newBuilder()
                .setCode(Code.RESOURCE_EXHAUSTED.getNumber())
                .setMessage("Rate limit exceeded")
                .addDetails(com.google.protobuf.Any.pack(
                    ErrorInfo.newBuilder()
                        .setReason("RATE_LIMIT_EXCEEDED")
                        .setDomain("chat.example.com")
                        .putMetadata("limit", rle.getLimit())
                        .putMetadata("reset_time", rle.getResetTime())
                        .build()
                ))
                .build();

            observer.onError(StatusProto.toStatusRuntimeException(richStatus));

        } else {
            observer.onError(io.grpc.Status.INTERNAL
                .withDescription("Internal server error")
                .withCause(e)
                .asRuntimeException());
        }
    }

    /**
     * Client-side: extract rich error details
     */
    public static void handleClientError(StatusRuntimeException e) {
        io.grpc.Status status = e.getStatus();
        System.err.println("gRPC error: " + status.getCode() + " - " + status.getDescription());

        com.google.rpc.Status richStatus = StatusProto.fromThrowable(e);
        if (richStatus != null) {
            richStatus.getDetailsList().forEach(detail -> {
                if (detail.is(ErrorInfo.class)) {
                    try {
                        ErrorInfo errorInfo = detail.unpack(ErrorInfo.class);
                        System.err.println("Error reason: " + errorInfo.getReason());
                        System.err.println("Metadata: " + errorInfo.getMetadataMap());
                    } catch (Exception ex) {
                        // ignore
                    }
                }
            });
        }
    }
}
```

---

## gRPC Configuration (application.yml)

```yaml
# application.yml
grpc:
  server:
    port: 9090
    enable-reflection: true  # allows grpcurl to introspect
    security:
      enabled: false  # set true for TLS in production

  client:
    chat-service:
      address: static://localhost:9090
      negotiation-type: plaintext  # use TLS in production

spring:
  application:
    name: chat-grpc-service
```

---

## gRPC REST Transcoding

```protobuf
// src/main/proto/transcoding.proto
syntax = "proto3";

package com.example.grpc;

import "google/api/annotations.proto";
import "google/api/http.proto";

// Allows REST clients to call gRPC services via HTTP/JSON
service UserService {
  rpc GetUser(GetUserRequest) returns (User) {
    option (google.api.http) = {
      get: "/v1/users/{user_id}"
    };
  }

  rpc CreateUser(CreateUserRequest) returns (User) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"
    };
  }

  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {
    option (google.api.http) = {
      get: "/v1/users"
    };
  }
}
```

---

## Integration Test

```java
// src/test/java/com/example/grpc/ChatGrpcServiceTest.java
package com.example.grpc;

import com.example.grpc.proto.*;
import io.grpc.testing.GrpcCleanupRule;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.junit.Rule;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.annotation.DirtiesContext;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(properties = {
    "grpc.server.port=0",
    "grpc.client.chat-service.address=in-process:test"
})
@DirtiesContext
class ChatGrpcServiceTest {

    @GrpcClient("chat-service")
    private ChatServiceGrpc.ChatServiceBlockingStub blockingStub;

    @GrpcClient("chat-service")
    private ChatServiceGrpc.ChatServiceStub asyncStub;

    @Test
    void joinRoomReturnsSuccessResponse() {
        JoinRoomRequest request = JoinRoomRequest.newBuilder()
            .setRoomId("room-1")
            .setUserId("user-1")
            .setDisplayName("Alice")
            .build();

        JoinRoomResponse response = blockingStub.joinRoom(request);

        assertThat(response.getSuccess()).isTrue();
        assertThat(response.getRoomName()).isNotBlank();
    }

    @Test
    void joinRoomWithEmptyRoomIdReturnsInvalidArgument() {
        JoinRoomRequest request = JoinRoomRequest.newBuilder()
            .setRoomId("")
            .setUserId("user-1")
            .build();

        org.junit.jupiter.api.Assertions.assertThrows(
            io.grpc.StatusRuntimeException.class,
            () -> blockingStub.joinRoom(request),
            "INVALID_ARGUMENT"
        );
    }

    @Test
    void bidiStreamingChatExchangesMessages() throws InterruptedException {
        List<ChatMessage> received = new ArrayList<>();
        CountDownLatch latch = new CountDownLatch(3);

        io.grpc.stub.StreamObserver<ChatMessage> requestObserver =
            asyncStub.chat(new io.grpc.stub.StreamObserver<>() {
                @Override
                public void onNext(ChatMessage msg) {
                    received.add(msg);
                    latch.countDown();
                }

                @Override
                public void onError(Throwable t) { latch.countDown(); }

                @Override
                public void onCompleted() {}
            });

        // Send 3 messages
        for (int i = 1; i <= 3; i++) {
            requestObserver.onNext(ChatMessage.newBuilder()
                .setRoomId("test-room")
                .setSenderId("user-1")
                .setContent("Message " + i)
                .setType(MessageType.MESSAGE_TYPE_TEXT)
                .build());
        }

        requestObserver.onCompleted();
        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(received).hasSize(3);
    }
}
```

---

## Summary

| Pattern | Proto Keyword | Java StreamObserver Usage |
|---|---|---|
| Unary | none | Single `onNext` → `onCompleted` |
| Server Streaming | `returns (stream T)` | Multiple `onNext` → `onCompleted` |
| Client Streaming | `(stream T) returns` | Return `StreamObserver<Req>` from impl |
| Bidirectional | `(stream T) returns (stream T)` | Return `StreamObserver<Req>`, use arg `StreamObserver<Resp>` |

### Key Takeaways
- Always include a zero value (0) in proto3 enums
- Use `oneof` to model exclusive choices — saves space vs. multiple optional fields
- `map<K, V>` in proto compiles to `java.util.Map`
- `repeated` compiles to `java.util.List` (immutable in generated getters)
- Use `google.protobuf.Timestamp` for dates, not strings
- Interceptors are the right place for cross-cutting concerns: logging, auth, tracing
- Rich errors use `google.rpc.Status` with `ErrorInfo` — avoid stuffing errors in descriptions

---

## Next Part Preview

**Part 094: Hexagonal Architecture (Ports & Adapters)** covers structuring a Spring Boot application so the domain core has zero framework dependencies, input/output ports as interfaces, and adapters as the only place where Spring or JPA annotations live. Includes a complete loan application system example.
