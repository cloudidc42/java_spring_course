# Part 039: gRPC with Spring Boot

## Introduction

gRPC is a high-performance, open-source RPC framework developed by Google. It uses Protocol Buffers (protobuf) as its interface description language and HTTP/2 as the transport protocol. Compared to REST, gRPC offers: strict contract-first design, efficient binary serialization, bidirectional streaming, and code generation in 10+ languages. It excels in internal microservice communication where performance and type safety matter.

---

## 1. gRPC Concepts vs REST

| Feature | REST | gRPC |
|---------|------|------|
| Protocol | HTTP/1.1 | HTTP/2 |
| Payload format | JSON (text) | Protocol Buffers (binary) |
| Schema | Optional (OpenAPI) | Required (.proto) |
| Code generation | Optional | Built-in |
| Streaming | Limited (SSE, WebSocket separate) | Built-in (4 types) |
| Browser support | Native | Requires grpc-web proxy |
| Performance | Good | Excellent (2-10x faster) |
| Human-readable | Yes | No (binary) |
| Contract versioning | Loose (field names) | Strict (field numbers) |

### When to Use gRPC

**Choose gRPC when:**
- Internal service-to-service communication
- Performance-critical paths (high-throughput, low-latency)
- Polyglot microservices (Java talking to Go, Python, etc.)
- Streaming data (logs, events, file uploads)
- Strong type safety is required

**Choose REST when:**
- Public APIs consumed by browsers
- Simple CRUD operations
- Team unfamiliar with protobuf
- External third-party integrations

---

## 2. Protocol Buffers (.proto Files)

### Basic Proto Syntax

```protobuf
// src/main/proto/user.proto
syntax = "proto3";

package com.example.grpc.user;

option java_multiple_files = true;
option java_package = "com.example.grpc.user";
option java_outer_classname = "UserProto";

// Import timestamp for common types
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";

// ─── Messages ─────────────────────────────────────────────────────────────

message User {
  string id = 1;
  string username = 2;
  string email = 3;
  string full_name = 4;
  UserStatus status = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  repeated string roles = 8;
  map<string, string> metadata = 9;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;  // Proto3: always include a 0-value
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
  string full_name = 4;
}

message GetUserRequest {
  string id = 1;
}

message UpdateUserRequest {
  string id = 1;
  string full_name = 2;
  string email = 3;
  google.protobuf.FieldMask update_mask = 4;  // Partial updates
}

message DeleteUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;       // e.g., "status=ACTIVE"
  string order_by = 4;     // e.g., "created_at desc"
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

// ─── Service ────────────────────────────────────────────────────────────────

service UserService {
  // Unary: single request → single response
  rpc CreateUser (CreateUserRequest) returns (User);
  rpc GetUser (GetUserRequest) returns (User);
  rpc UpdateUser (UpdateUserRequest) returns (User);
  rpc DeleteUser (DeleteUserRequest) returns (google.protobuf.Empty);

  // Server streaming: single request → stream of responses
  rpc ListUsers (ListUsersRequest) returns (stream User);

  // Client streaming: stream of requests → single response
  rpc BatchCreateUsers (stream CreateUserRequest) returns (ListUsersResponse);

  // Bidirectional streaming: stream of requests ↔ stream of responses
  rpc WatchUsers (stream GetUserRequest) returns (stream User);
}
```

### File Upload Proto

```protobuf
// src/main/proto/file_upload.proto
syntax = "proto3";

package com.example.grpc.file;

option java_multiple_files = true;
option java_package = "com.example.grpc.file";

import "google/protobuf/timestamp.proto";

message FileChunk {
  string file_id = 1;       // UUID assigned before upload starts
  string file_name = 2;
  string content_type = 3;
  int64 total_size = 4;
  int32 chunk_number = 5;
  int32 total_chunks = 6;
  bytes data = 7;            // Chunk payload (max ~4MB per message)
}

message UploadProgress {
  string file_id = 1;
  int32 chunks_received = 2;
  int32 total_chunks = 3;
  int64 bytes_received = 4;
  UploadStatus status = 5;
  string message = 6;
}

enum UploadStatus {
  UPLOAD_STATUS_UNSPECIFIED = 0;
  UPLOAD_STATUS_IN_PROGRESS = 1;
  UPLOAD_STATUS_COMPLETE = 2;
  UPLOAD_STATUS_FAILED = 3;
}

message FileInfo {
  string file_id = 1;
  string file_name = 2;
  string content_type = 3;
  int64 size_bytes = 4;
  string download_url = 5;
  google.protobuf.Timestamp uploaded_at = 6;
  string checksum = 7;
}

message DownloadRequest {
  string file_id = 1;
  int32 chunk_size_bytes = 2;  // Optional: requested chunk size
}

service FileUploadService {
  // Client streaming: client sends chunks, server responds once with result
  rpc Upload (stream FileChunk) returns (FileInfo);

  // Server streaming: server streams file chunks to client
  rpc Download (DownloadRequest) returns (stream FileChunk);

  // Bidirectional: client streams chunks, server streams progress
  rpc UploadWithProgress (stream FileChunk) returns (stream UploadProgress);

  // Unary: get file metadata
  rpc GetFileInfo (DownloadRequest) returns (FileInfo);
}
```

---

## 3. Spring Boot gRPC Server Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<properties>
    <grpc.version>1.59.0</grpc.version>
    <protobuf.version>3.24.4</protobuf.version>
    <grpc-spring-boot-starter.version>3.1.0.RELEASE</grpc-spring-boot-starter.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- gRPC Spring Boot Starter (LogNet) -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>${grpc-spring-boot-starter.version}</version>
    </dependency>

    <!-- Protobuf runtime -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>${protobuf.version}</version>
    </dependency>
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java-util</artifactId>
        <version>${protobuf.version}</version>
    </dependency>

    <!-- For separate client/server projects -->
    <!-- <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-server-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-client-spring-boot-starter</artifactId>
    </dependency> -->
</dependencies>
```

### Maven Plugin for Protobuf Code Generation

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
                <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
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

### application.yaml for gRPC Server

```yaml
# application.yaml
grpc:
  server:
    port: 9090
    max-inbound-message-size: 10MB
    max-inbound-metadata-size: 8KB
    enable-keep-alive: true
    keep-alive-time: 30s
    keep-alive-timeout: 5s

spring:
  application:
    name: user-grpc-service
```

---

## 4. gRPC Service Types: Unary, Server, Client, Bidirectional

### Unary RPC (Single Request/Response)

```java
// src/main/java/com/example/grpc/service/UserGrpcService.java
package com.example.grpc.service;

import com.example.grpc.user.*;
import com.example.grpc.repository.UserRepository;
import com.example.grpc.mapper.UserMapper;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@GrpcService   // Marks this as a gRPC service bean
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    private static final Logger log = LoggerFactory.getLogger(UserGrpcService.class);

    private final UserRepository userRepository;
    private final UserMapper userMapper;

    public UserGrpcService(UserRepository userRepository, UserMapper userMapper) {
        this.userRepository = userRepository;
        this.userMapper = userMapper;
    }

    // ─── Unary RPC ────────────────────────────────────────────────────────

    @Override
    public void createUser(CreateUserRequest request,
                           StreamObserver<User> responseObserver) {
        try {
            log.info("Creating user: {}", request.getUsername());

            // Validate
            if (request.getUsername().isBlank()) {
                responseObserver.onError(Status.INVALID_ARGUMENT
                    .withDescription("Username is required")
                    .asRuntimeException());
                return;
            }

            if (userRepository.existsByEmail(request.getEmail())) {
                responseObserver.onError(Status.ALREADY_EXISTS
                    .withDescription("Email already registered: " + request.getEmail())
                    .asRuntimeException());
                return;
            }

            // Create user
            com.example.grpc.model.User user = userMapper.fromProto(request);
            com.example.grpc.model.User saved = userRepository.save(user);

            // Send single response
            responseObserver.onNext(userMapper.toProto(saved));
            responseObserver.onCompleted();

        } catch (Exception e) {
            log.error("Error creating user", e);
            responseObserver.onError(Status.INTERNAL
                .withDescription("Internal error: " + e.getMessage())
                .withCause(e)
                .asRuntimeException());
        }
    }

    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<User> responseObserver) {
        userRepository.findById(request.getId())
            .ifPresentOrElse(
                user -> {
                    responseObserver.onNext(userMapper.toProto(user));
                    responseObserver.onCompleted();
                },
                () -> responseObserver.onError(Status.NOT_FOUND
                    .withDescription("User not found: " + request.getId())
                    .asRuntimeException())
            );
    }

    @Override
    public void updateUser(UpdateUserRequest request,
                           StreamObserver<User> responseObserver) {
        userRepository.findById(request.getId())
            .ifPresentOrElse(
                user -> {
                    // Apply field mask (partial update)
                    if (request.hasUpdateMask()) {
                        for (String path : request.getUpdateMask().getPathsList()) {
                            switch (path) {
                                case "full_name" -> user.setFullName(request.getFullName());
                                case "email" -> user.setEmail(request.getEmail());
                            }
                        }
                    } else {
                        // Full update
                        user.setFullName(request.getFullName());
                        user.setEmail(request.getEmail());
                    }
                    responseObserver.onNext(userMapper.toProto(userRepository.save(user)));
                    responseObserver.onCompleted();
                },
                () -> responseObserver.onError(Status.NOT_FOUND
                    .withDescription("User not found: " + request.getId())
                    .asRuntimeException())
            );
    }

    @Override
    public void deleteUser(DeleteUserRequest request,
                           StreamObserver<com.google.protobuf.Empty> responseObserver) {
        if (!userRepository.existsById(request.getId())) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("User not found: " + request.getId())
                .asRuntimeException());
            return;
        }
        userRepository.deleteById(request.getId());
        responseObserver.onNext(com.google.protobuf.Empty.getDefaultInstance());
        responseObserver.onCompleted();
    }

    // ─── Server Streaming RPC ─────────────────────────────────────────────

    @Override
    public void listUsers(ListUsersRequest request,
                          StreamObserver<User> responseObserver) {
        try {
            int pageSize = request.getPageSize() > 0 ? request.getPageSize() : 20;

            // Stream users one by one
            userRepository.findAll().stream()
                .filter(u -> matchesFilter(u, request.getFilter()))
                .limit(pageSize)
                .map(userMapper::toProto)
                .forEach(responseObserver::onNext);

            responseObserver.onCompleted();

        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription(e.getMessage()).asRuntimeException());
        }
    }

    // ─── Client Streaming RPC ─────────────────────────────────────────────

    @Override
    public StreamObserver<CreateUserRequest> batchCreateUsers(
            StreamObserver<ListUsersResponse> responseObserver) {

        return new StreamObserver<>() {
            private final java.util.List<User> created = new java.util.ArrayList<>();
            private int count = 0;

            @Override
            public void onNext(CreateUserRequest request) {
                // Called for each message from client
                try {
                    com.example.grpc.model.User user = userMapper.fromProto(request);
                    com.example.grpc.model.User saved = userRepository.save(user);
                    created.add(userMapper.toProto(saved));
                    count++;
                    log.debug("Batch created user {}: {}", count, request.getUsername());
                } catch (Exception e) {
                    log.error("Failed to create user in batch: {}", request.getUsername(), e);
                }
            }

            @Override
            public void onError(Throwable t) {
                log.error("Batch create stream error", t);
            }

            @Override
            public void onCompleted() {
                // Send single response when client stream ends
                ListUsersResponse response = ListUsersResponse.newBuilder()
                    .addAllUsers(created)
                    .setTotalCount(created.size())
                    .build();
                responseObserver.onNext(response);
                responseObserver.onCompleted();
            }
        };
    }

    // ─── Bidirectional Streaming RPC ──────────────────────────────────────

    @Override
    public StreamObserver<GetUserRequest> watchUsers(
            StreamObserver<User> responseObserver) {

        return new StreamObserver<>() {
            @Override
            public void onNext(GetUserRequest request) {
                // For each requested ID, stream back the user
                userRepository.findById(request.getId())
                    .ifPresentOrElse(
                        user -> responseObserver.onNext(userMapper.toProto(user)),
                        () -> log.warn("User not found in watch: {}", request.getId())
                    );
            }

            @Override
            public void onError(Throwable t) {
                log.error("Watch stream error", t);
            }

            @Override
            public void onCompleted() {
                responseObserver.onCompleted();
            }
        };
    }

    private boolean matchesFilter(com.example.grpc.model.User user, String filter) {
        if (filter == null || filter.isBlank()) return true;
        if (filter.contains("status=ACTIVE")) {
            return user.getStatus() == com.example.grpc.model.UserStatus.ACTIVE;
        }
        return true;
    }
}
```

---

## 5. Protobuf Code Generation with Maven Plugin

```bash
# Generate code from .proto files
mvn generate-sources

# Generated code locations:
# target/generated-sources/protobuf/java/    → Message classes
# target/generated-sources/protobuf/grpc-java/ → Service stubs
```

### Generated Classes Overview

```java
// Generated by protoc (DO NOT EDIT):
// UserServiceGrpc.java — contains:
//   - UserServiceGrpc.UserServiceImplBase (server base class)
//   - UserServiceGrpc.UserServiceBlockingStub (synchronous client)
//   - UserServiceGrpc.UserServiceFutureStub (async client)
//   - UserServiceGrpc.UserServiceStub (async streaming client)

// User.java — the protobuf message class
// CreateUserRequest.java, GetUserRequest.java, etc.
```

---

## 6. gRPC Client Setup

### Client Configuration

```yaml
# application.yaml (client service)
grpc:
  client:
    user-service:
      address: static://localhost:9090
      enable-keep-alive: true
      keep-alive-without-calls: true
      negotiation-type: plaintext   # or TLS

    # Multiple services
    order-service:
      address: static://order-service:9090
      negotiation-type: tls
```

### Injecting gRPC Client

```java
// src/main/java/com/example/grpc/client/UserServiceClient.java
package com.example.grpc.client;

import com.example.grpc.user.*;
import io.grpc.StatusRuntimeException;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Service
public class UserServiceClient {

    private static final Logger log = LoggerFactory.getLogger(UserServiceClient.class);

    // Inject blocking stub (synchronous)
    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceBlockingStub blockingStub;

    // Inject async stub (for streaming)
    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceStub asyncStub;

    // ─── Unary call ───────────────────────────────────────────────────────

    public User createUser(String username, String email, String password) {
        CreateUserRequest request = CreateUserRequest.newBuilder()
            .setUsername(username)
            .setEmail(email)
            .setPassword(password)
            .build();

        try {
            return blockingStub.createUser(request);
        } catch (StatusRuntimeException e) {
            log.error("Failed to create user: {} - {}", e.getStatus(), e.getMessage());
            throw e;
        }
    }

    public User getUser(String id) {
        return blockingStub.getUser(
            GetUserRequest.newBuilder().setId(id).build()
        );
    }

    // ─── Server streaming call ────────────────────────────────────────────

    public List<User> listAllUsers(String filter) {
        ListUsersRequest request = ListUsersRequest.newBuilder()
            .setFilter(filter)
            .setPageSize(100)
            .build();

        List<User> users = new ArrayList<>();
        Iterator<User> iterator = blockingStub.listUsers(request);

        while (iterator.hasNext()) {
            users.add(iterator.next());
        }

        return users;
    }

    // ─── Client streaming call ────────────────────────────────────────────

    public ListUsersResponse batchCreate(List<CreateUserRequest> requests)
            throws InterruptedException {

        CountDownLatch latch = new CountDownLatch(1);
        List<ListUsersResponse> result = new ArrayList<>();
        List<Throwable> errors = new ArrayList<>();

        io.grpc.stub.StreamObserver<ListUsersResponse> responseObserver =
            new io.grpc.stub.StreamObserver<>() {
                @Override
                public void onNext(ListUsersResponse response) {
                    result.add(response);
                }

                @Override
                public void onError(Throwable t) {
                    errors.add(t);
                    latch.countDown();
                }

                @Override
                public void onCompleted() {
                    latch.countDown();
                }
            };

        io.grpc.stub.StreamObserver<CreateUserRequest> requestObserver =
            asyncStub.batchCreateUsers(responseObserver);

        for (CreateUserRequest req : requests) {
            requestObserver.onNext(req);
        }
        requestObserver.onCompleted();

        latch.await(30, TimeUnit.SECONDS);

        if (!errors.isEmpty()) throw new RuntimeException("Batch create failed", errors.get(0));
        return result.isEmpty() ? ListUsersResponse.getDefaultInstance() : result.get(0);
    }
}
```

### REST Controller wrapping gRPC

```java
// src/main/java/com/example/grpc/web/UserWebController.java
package com.example.grpc.web;

import com.example.grpc.client.UserServiceClient;
import com.example.grpc.user.User;
import io.grpc.StatusRuntimeException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserWebController {

    private final UserServiceClient userServiceClient;

    public UserWebController(UserServiceClient userServiceClient) {
        this.userServiceClient = userServiceClient;
    }

    @PostMapping
    public ResponseEntity<?> createUser(@RequestBody Map<String, String> body) {
        try {
            User user = userServiceClient.createUser(
                body.get("username"),
                body.get("email"),
                body.get("password")
            );
            return ResponseEntity.status(HttpStatus.CREATED).body(
                Map.of("id", user.getId(), "username", user.getUsername())
            );
        } catch (StatusRuntimeException e) {
            return switch (e.getStatus().getCode()) {
                case ALREADY_EXISTS -> ResponseEntity.status(HttpStatus.CONFLICT)
                    .body(Map.of("error", e.getStatus().getDescription()));
                case INVALID_ARGUMENT -> ResponseEntity.badRequest()
                    .body(Map.of("error", e.getStatus().getDescription()));
                default -> ResponseEntity.internalServerError()
                    .body(Map.of("error", "Service unavailable"));
            };
        }
    }

    @GetMapping("/{id}")
    public ResponseEntity<?> getUser(@PathVariable String id) {
        try {
            User user = userServiceClient.getUser(id);
            return ResponseEntity.ok(Map.of(
                "id", user.getId(),
                "username", user.getUsername(),
                "email", user.getEmail()
            ));
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == io.grpc.Status.Code.NOT_FOUND) {
                return ResponseEntity.notFound().build();
            }
            throw e;
        }
    }
}
```

---

## 7. Interceptors for Auth and Logging

### Server-Side Interceptor

```java
// src/main/java/com/example/grpc/interceptor/AuthInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;

@GrpcGlobalServerInterceptor   // Applies to all gRPC services
public class AuthInterceptor implements ServerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(AuthInterceptor.class);

    // Header key for bearer token
    public static final Metadata.Key<String> AUTHORIZATION_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    // Context key to pass user info to service
    public static final Context.Key<String> USER_ID_KEY =
        Context.key("userId");

    @Value("${grpc.security.token-secret}")
    private String tokenSecret;

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        log.debug("gRPC call: {}", methodName);

        // Skip auth for health check
        if (methodName.contains("grpc.health")) {
            return next.startCall(call, headers);
        }

        String authorization = headers.get(AUTHORIZATION_KEY);
        if (authorization == null || !authorization.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Missing or invalid authorization header"), headers);
            return new ServerCall.Listener<>() {};
        }

        String token = authorization.substring(7);

        try {
            String userId = validateToken(token);

            // Add userId to gRPC context
            Context ctx = Context.current().withValue(USER_ID_KEY, userId);
            return Contexts.interceptCall(ctx, call, headers, next);

        } catch (Exception e) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid token: " + e.getMessage()), headers);
            return new ServerCall.Listener<>() {};
        }
    }

    private String validateToken(String token) {
        // JWT validation logic here (use jjwt or nimbus-jose-jwt)
        // Simplified for demo:
        if (token.equals("valid-token")) return "user-123";
        throw new IllegalArgumentException("Invalid token");
    }
}
```

### Logging Interceptor

```java
// src/main/java/com/example/grpc/interceptor/LoggingInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Duration;
import java.time.Instant;

@GrpcGlobalServerInterceptor
public class LoggingInterceptor implements ServerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String method = call.getMethodDescriptor().getFullMethodName();
        Instant start = Instant.now();

        ServerCall<ReqT, RespT> loggingCall = new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                Duration duration = Duration.between(start, Instant.now());
                log.info("gRPC {} completed: status={}, duration={}ms",
                    method, status.getCode(), duration.toMillis());

                if (!status.isOk()) {
                    log.warn("gRPC {} failed: {} - {}",
                        method, status.getCode(), status.getDescription());
                }

                super.close(status, trailers);
            }
        };

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(
                next.startCall(loggingCall, headers)) {

            @Override
            public void onMessage(ReqT message) {
                log.debug("gRPC {} request received", method);
                super.onMessage(message);
            }
        };
    }
}
```

### Client-Side Interceptor

```java
// src/main/java/com/example/grpc/interceptor/ClientAuthInterceptor.java
package com.example.grpc.interceptor;

import io.grpc.*;
import net.devh.boot.grpc.client.interceptor.GrpcGlobalClientInterceptor;
import org.springframework.beans.factory.annotation.Value;

@GrpcGlobalClientInterceptor
public class ClientAuthInterceptor implements ClientInterceptor {

    @Value("${grpc.client.auth-token}")
    private String authToken;

    private static final Metadata.Key<String> AUTHORIZATION_KEY =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {

            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                headers.put(AUTHORIZATION_KEY, "Bearer " + authToken);
                super.start(responseListener, headers);
            }
        };
    }
}
```

---

## 8. Error Handling with Status Codes

### gRPC Status Codes

```java
// src/main/java/com/example/grpc/exception/GrpcExceptionTranslator.java
package com.example.grpc.exception;

import io.grpc.Status;

/**
 * Maps application exceptions to gRPC status codes.
 */
public class GrpcExceptionTranslator {

    /*
     * Standard gRPC Status codes:
     * OK              - Success
     * CANCELLED       - Operation was cancelled
     * UNKNOWN         - Unknown error
     * INVALID_ARGUMENT - Client sent bad data
     * DEADLINE_EXCEEDED - Deadline expired
     * NOT_FOUND       - Requested entity not found
     * ALREADY_EXISTS  - Entity already exists
     * PERMISSION_DENIED - Not authorized
     * RESOURCE_EXHAUSTED - Resource quota exceeded
     * FAILED_PRECONDITION - System not in correct state
     * ABORTED         - Operation aborted (concurrency conflict)
     * OUT_OF_RANGE    - Value out of valid range
     * UNIMPLEMENTED   - Not implemented
     * INTERNAL        - Internal error
     * UNAVAILABLE     - Service unavailable
     * DATA_LOSS       - Unrecoverable data loss
     * UNAUTHENTICATED - No valid credentials
     */

    public static Status fromException(Exception e) {
        if (e instanceof EntityNotFoundException ex) {
            return Status.NOT_FOUND.withDescription(ex.getMessage()).withCause(e);
        }
        if (e instanceof ValidationException ex) {
            return Status.INVALID_ARGUMENT.withDescription(ex.getMessage()).withCause(e);
        }
        if (e instanceof DuplicateException ex) {
            return Status.ALREADY_EXISTS.withDescription(ex.getMessage()).withCause(e);
        }
        if (e instanceof SecurityException ex) {
            return Status.PERMISSION_DENIED.withDescription(ex.getMessage()).withCause(e);
        }
        return Status.INTERNAL.withDescription("Unexpected error").withCause(e);
    }
}
```

### Handling Status Errors on Client

```java
public User getUser(String id) {
    try {
        return blockingStub.getUser(GetUserRequest.newBuilder().setId(id).build());
    } catch (StatusRuntimeException e) {
        switch (e.getStatus().getCode()) {
            case NOT_FOUND:
                throw new UserNotFoundException("User not found: " + id);
            case INVALID_ARGUMENT:
                throw new IllegalArgumentException(e.getStatus().getDescription());
            case UNAUTHENTICATED:
                throw new UnauthorizedException("Authentication failed");
            case PERMISSION_DENIED:
                throw new AccessDeniedException("Access denied");
            case UNAVAILABLE:
                // Service is down: implement retry
                throw new ServiceUnavailableException("User service is unavailable");
            default:
                throw new RuntimeException("gRPC error: " + e.getStatus(), e);
        }
    }
}
```

---

## 9. gRPC with TLS

### Server TLS Configuration

```yaml
# application.yaml
grpc:
  server:
    port: 9443
    security:
      certificate-chain: classpath:tls/server.crt
      private-key: classpath:tls/server.key
      # For mutual TLS:
      # client-auth: REQUIRE
      # trust-cert-collection: classpath:tls/ca.crt
```

### Client TLS Configuration

```yaml
grpc:
  client:
    user-service:
      address: static://user-service:9443
      negotiation-type: tls
      security:
        trust-cert-collection: classpath:tls/ca.crt
        # For mutual TLS:
        # certificate-chain: classpath:tls/client.crt
        # private-key: classpath:tls/client.key
```

### Generate Self-Signed Certificates

```bash
# Generate CA
openssl req -new -x509 -days 365 -keyout ca.key -out ca.crt \
  -subj "/CN=My CA"

# Generate server key and CSR
openssl req -newkey rsa:2048 -nodes -keyout server.key -out server.csr \
  -subj "/CN=localhost"

# Sign server certificate
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt

# Copy to resources
cp ca.crt server.crt server.key src/main/resources/tls/
```

---

## 10. gRPC Health Checking

### Setup gRPC Health Service

```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-services</artifactId>
    <version>${grpc.version}</version>
</dependency>
```

```java
// src/main/java/com/example/grpc/health/GrpcHealthConfig.java
package com.example.grpc.health;

import io.grpc.health.v1.HealthCheckResponse;
import io.grpc.protobuf.services.HealthStatusManager;
import net.devh.boot.grpc.server.serverfactory.GrpcServerConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GrpcHealthConfig {

    @Bean
    public HealthStatusManager healthStatusManager() {
        return new HealthStatusManager();
    }

    @Bean
    public GrpcServerConfigurer serverConfigurer(HealthStatusManager healthStatusManager) {
        return serverBuilder -> serverBuilder
            .addService(healthStatusManager.getHealthService());
    }

    // Update health status dynamically (e.g., when DB is down)
    @Bean
    public HealthUpdater healthUpdater(HealthStatusManager healthStatusManager) {
        return new HealthUpdater(healthStatusManager);
    }
}
```

```java
// src/main/java/com/example/grpc/health/HealthUpdater.java
package com.example.grpc.health;

import io.grpc.health.v1.HealthCheckResponse.ServingStatus;
import io.grpc.protobuf.services.HealthStatusManager;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class HealthUpdater {

    private final HealthStatusManager healthStatusManager;

    public HealthUpdater(HealthStatusManager healthStatusManager) {
        this.healthStatusManager = healthStatusManager;
        // Set initial status
        healthStatusManager.setStatus("", ServingStatus.SERVING);
        healthStatusManager.setStatus("UserService", ServingStatus.SERVING);
    }

    public void setServing(String service) {
        healthStatusManager.setStatus(service, ServingStatus.SERVING);
    }

    public void setNotServing(String service) {
        healthStatusManager.setStatus(service, ServingStatus.NOT_SERVING);
    }
}
```

```bash
# Check health with grpc_health_probe
grpc_health_probe -addr=localhost:9090

# Or with grpcurl
grpcurl -plaintext localhost:9090 grpc.health.v1.Health/Check
```

---

## 11. Real Example: File Upload Streaming Service

### File Upload Service Implementation

```java
// src/main/java/com/example/grpc/service/FileUploadGrpcService.java
package com.example.grpc.service;

import com.example.grpc.file.*;
import com.google.protobuf.ByteString;
import com.google.protobuf.Timestamp;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;

import java.io.*;
import java.nio.file.*;
import java.security.MessageDigest;
import java.time.Instant;
import java.util.*;

@GrpcService
public class FileUploadGrpcService extends FileUploadServiceGrpc.FileUploadServiceImplBase {

    private static final Logger log = LoggerFactory.getLogger(FileUploadGrpcService.class);
    private static final int CHUNK_SIZE = 256 * 1024;  // 256KB chunks

    @Value("${app.upload-dir:/tmp/uploads}")
    private String uploadDir;

    // ─── Client Streaming: Upload ─────────────────────────────────────────

    @Override
    public StreamObserver<FileChunk> upload(StreamObserver<FileInfo> responseObserver) {
        return new StreamObserver<>() {
            private String fileId;
            private String fileName;
            private String contentType;
            private long totalSize;
            private ByteArrayOutputStream buffer = new ByteArrayOutputStream();
            private MessageDigest digest;
            private int chunksReceived = 0;

            {
                try {
                    digest = MessageDigest.getInstance("SHA-256");
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            }

            @Override
            public void onNext(FileChunk chunk) {
                // First chunk sets metadata
                if (fileId == null) {
                    fileId = chunk.getFileId().isEmpty()
                        ? UUID.randomUUID().toString()
                        : chunk.getFileId();
                    fileName = chunk.getFileName();
                    contentType = chunk.getContentType();
                    totalSize = chunk.getTotalSize();
                    log.info("Upload started: fileId={}, name={}, size={}",
                        fileId, fileName, totalSize);
                }

                byte[] data = chunk.getData().toByteArray();
                buffer.writeBytes(data);
                digest.update(data);
                chunksReceived++;

                log.debug("Received chunk {}/{} for file {}",
                    chunksReceived, chunk.getTotalChunks(), fileId);
            }

            @Override
            public void onError(Throwable t) {
                log.error("Upload stream error for file {}", fileId, t);
                // Clean up temp data
            }

            @Override
            public void onCompleted() {
                try {
                    // Save file to disk
                    Path uploadPath = Path.of(uploadDir);
                    Files.createDirectories(uploadPath);

                    Path filePath = uploadPath.resolve(fileId + "_" + fileName);
                    Files.write(filePath, buffer.toByteArray());

                    // Calculate checksum
                    String checksum = HexFormat.of().formatHex(digest.digest());

                    log.info("Upload complete: fileId={}, bytes={}, checksum={}",
                        fileId, buffer.size(), checksum);

                    FileInfo response = FileInfo.newBuilder()
                        .setFileId(fileId)
                        .setFileName(fileName)
                        .setContentType(contentType)
                        .setSizeBytes(buffer.size())
                        .setDownloadUrl("/files/" + fileId)
                        .setChecksum(checksum)
                        .setUploadedAt(Timestamp.newBuilder()
                            .setSeconds(Instant.now().getEpochSecond())
                            .build())
                        .build();

                    responseObserver.onNext(response);
                    responseObserver.onCompleted();

                } catch (IOException e) {
                    responseObserver.onError(Status.INTERNAL
                        .withDescription("Failed to save file: " + e.getMessage())
                        .withCause(e)
                        .asRuntimeException());
                }
            }
        };
    }

    // ─── Server Streaming: Download ───────────────────────────────────────

    @Override
    public void download(DownloadRequest request,
                         StreamObserver<FileChunk> responseObserver) {
        try {
            String fileId = request.getFileId();
            int chunkSize = request.getChunkSizeBytes() > 0
                ? request.getChunkSizeBytes() : CHUNK_SIZE;

            // Find file by ID prefix
            Path uploadPath = Path.of(uploadDir);
            Optional<Path> filePath = Files.list(uploadPath)
                .filter(p -> p.getFileName().toString().startsWith(fileId))
                .findFirst();

            if (filePath.isEmpty()) {
                responseObserver.onError(Status.NOT_FOUND
                    .withDescription("File not found: " + fileId)
                    .asRuntimeException());
                return;
            }

            Path file = filePath.get();
            long fileSize = Files.size(file);
            String fileName = file.getFileName().toString().substring(fileId.length() + 1);
            long totalChunks = (fileSize + chunkSize - 1) / chunkSize;

            log.info("Streaming file download: fileId={}, size={}, chunks={}",
                fileId, fileSize, totalChunks);

            try (InputStream is = Files.newInputStream(file)) {
                byte[] buffer = new byte[chunkSize];
                int bytesRead;
                int chunkNum = 0;

                while ((bytesRead = is.read(buffer)) != -1) {
                    FileChunk chunk = FileChunk.newBuilder()
                        .setFileId(fileId)
                        .setFileName(fileName)
                        .setTotalSize(fileSize)
                        .setChunkNumber(chunkNum)
                        .setTotalChunks((int) totalChunks)
                        .setData(ByteString.copyFrom(buffer, 0, bytesRead))
                        .build();

                    responseObserver.onNext(chunk);
                    chunkNum++;
                }
            }

            responseObserver.onCompleted();
            log.info("Download complete: fileId={}", fileId);

        } catch (IOException e) {
            responseObserver.onError(Status.INTERNAL
                .withDescription("Download failed: " + e.getMessage())
                .asRuntimeException());
        }
    }

    // ─── Bidirectional Streaming: Upload with Progress ───────────────────

    @Override
    public StreamObserver<FileChunk> uploadWithProgress(
            StreamObserver<UploadProgress> responseObserver) {

        return new StreamObserver<>() {
            private String fileId;
            private int totalChunks;
            private int chunksReceived = 0;
            private long bytesReceived = 0;
            private final ByteArrayOutputStream buffer = new ByteArrayOutputStream();

            @Override
            public void onNext(FileChunk chunk) {
                if (fileId == null) {
                    fileId = chunk.getFileId().isEmpty()
                        ? UUID.randomUUID().toString()
                        : chunk.getFileId();
                    totalChunks = chunk.getTotalChunks();
                }

                byte[] data = chunk.getData().toByteArray();
                buffer.writeBytes(data);
                bytesReceived += data.length;
                chunksReceived++;

                // Send progress update for every chunk
                UploadProgress progress = UploadProgress.newBuilder()
                    .setFileId(fileId)
                    .setChunksReceived(chunksReceived)
                    .setTotalChunks(totalChunks)
                    .setBytesReceived(bytesReceived)
                    .setStatus(UploadStatus.UPLOAD_STATUS_IN_PROGRESS)
                    .setMessage(String.format("Received %d/%d chunks", chunksReceived, totalChunks))
                    .build();

                responseObserver.onNext(progress);
            }

            @Override
            public void onError(Throwable t) {
                log.error("Upload with progress stream error", t);
                responseObserver.onError(t);
            }

            @Override
            public void onCompleted() {
                // Send final complete progress
                UploadProgress complete = UploadProgress.newBuilder()
                    .setFileId(fileId)
                    .setChunksReceived(chunksReceived)
                    .setTotalChunks(totalChunks)
                    .setBytesReceived(bytesReceived)
                    .setStatus(UploadStatus.UPLOAD_STATUS_COMPLETE)
                    .setMessage("Upload complete!")
                    .build();

                responseObserver.onNext(complete);
                responseObserver.onCompleted();
            }
        };
    }
}
```

### File Upload Client

```java
// src/main/java/com/example/grpc/client/FileUploadClient.java
package com.example.grpc.client;

import com.example.grpc.file.*;
import com.google.protobuf.ByteString;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.springframework.stereotype.Service;

import java.io.*;
import java.nio.file.*;
import java.util.UUID;
import java.util.concurrent.*;

@Service
public class FileUploadClient {

    private static final int CHUNK_SIZE = 256 * 1024;  // 256KB

    @GrpcClient("file-service")
    private FileUploadServiceGrpc.FileUploadServiceStub asyncStub;

    @GrpcClient("file-service")
    private FileUploadServiceGrpc.FileUploadServiceBlockingStub blockingStub;

    public FileInfo uploadFile(Path filePath) throws IOException, InterruptedException {
        String fileId = UUID.randomUUID().toString();
        String fileName = filePath.getFileName().toString();
        String contentType = Files.probeContentType(filePath);
        long fileSize = Files.size(filePath);
        long totalChunks = (fileSize + CHUNK_SIZE - 1) / CHUNK_SIZE;

        CountDownLatch latch = new CountDownLatch(1);
        List<FileInfo> results = new CopyOnWriteArrayList<>();
        List<Throwable> errors = new CopyOnWriteArrayList<>();

        StreamObserver<FileInfo> responseObserver = new StreamObserver<>() {
            @Override public void onNext(FileInfo fi) { results.add(fi); }
            @Override public void onError(Throwable t) { errors.add(t); latch.countDown(); }
            @Override public void onCompleted() { latch.countDown(); }
        };

        StreamObserver<FileChunk> requestObserver = asyncStub.upload(responseObserver);

        try (InputStream is = Files.newInputStream(filePath)) {
            byte[] buffer = new byte[CHUNK_SIZE];
            int bytesRead;
            int chunkNum = 0;

            while ((bytesRead = is.read(buffer)) != -1) {
                FileChunk chunk = FileChunk.newBuilder()
                    .setFileId(fileId)
                    .setFileName(fileName)
                    .setContentType(contentType != null ? contentType : "application/octet-stream")
                    .setTotalSize(fileSize)
                    .setChunkNumber(chunkNum)
                    .setTotalChunks((int) totalChunks)
                    .setData(ByteString.copyFrom(buffer, 0, bytesRead))
                    .build();

                requestObserver.onNext(chunk);
                chunkNum++;
            }
        }

        requestObserver.onCompleted();
        latch.await(60, TimeUnit.SECONDS);

        if (!errors.isEmpty()) throw new RuntimeException("Upload failed", errors.get(0));
        return results.isEmpty() ? null : results.get(0);
    }

    public void downloadFile(String fileId, Path savePath) throws IOException {
        DownloadRequest request = DownloadRequest.newBuilder()
            .setFileId(fileId)
            .setChunkSizeBytes(CHUNK_SIZE)
            .build();

        try (OutputStream os = Files.newOutputStream(savePath)) {
            blockingStub.download(request).forEachRemaining(chunk -> {
                try {
                    os.write(chunk.getData().toByteArray());
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    }
}
```

---

## Summary Table

| Service Type | Client sends | Server sends | Use Case |
|-------------|-------------|-------------|---------|
| Unary | Single message | Single message | CRUD operations |
| Server streaming | Single message | Stream of messages | List, download |
| Client streaming | Stream of messages | Single message | Upload, batch insert |
| Bidirectional | Stream of messages | Stream of messages | Chat, real-time sync |

| Topic | Class/Annotation | Notes |
|-------|-----------------|-------|
| Server service | `@GrpcService` | Extends `*ImplBase` |
| Client stub (sync) | `@GrpcClient` + `BlockingStub` | Good for unary |
| Client stub (async) | `@GrpcClient` + `Stub` | Required for streaming |
| Error handling | `Status.*` codes | Map to HTTP in gateway |
| Interceptor (server) | `@GrpcGlobalServerInterceptor` | Auth, logging, metrics |
| Interceptor (client) | `@GrpcGlobalClientInterceptor` | Add auth headers |
| TLS | `security.certificate-chain` | Production required |
| Health check | `HealthStatusManager` | K8s readiness/liveness |
| Code gen | `protobuf-maven-plugin` | From `.proto` files |

---

## What's Next

**Part 040: Monitoring with Prometheus & Grafana** — Build a complete observability stack. We'll implement custom Micrometer metrics, Prometheus scraping, Grafana dashboards, ELK stack log aggregation, and distributed tracing with correlation IDs.

---

*End of Part 039: gRPC with Spring Boot*
