# Part 038: GraphQL with Spring Boot

## Introduction

GraphQL is a query language for APIs that gives clients precise control over the data they receive. Instead of multiple REST endpoints returning fixed shapes, GraphQL exposes a single endpoint where clients describe exactly what they need. Spring for GraphQL (introduced in Spring Boot 2.7/3.x) provides annotation-driven integration with the GraphQL Java engine.

---

## 1. GraphQL Concepts

### GraphQL vs REST

| Aspect | REST | GraphQL |
|--------|------|---------|
| Endpoints | Many (`/users`, `/posts`) | Single (`/graphql`) |
| Data shape | Fixed by server | Defined by client |
| Over-fetching | Common | Eliminated |
| Under-fetching | N+1 requests | Single query |
| Versioning | `v1/`, `v2/` | Evolve schema with deprecation |
| Real-time | Polling or WebSocket separate | Subscriptions built in |
| Introspection | OpenAPI/Swagger | Built-in schema introspection |

### Core Operations

```graphql
# Query: read data
query {
  user(id: "1") {
    name
    email
    posts {
      title
      publishedAt
    }
  }
}

# Mutation: write data
mutation {
  createPost(input: {
    title: "Hello GraphQL"
    content: "..."
    authorId: "1"
  }) {
    id
    title
    createdAt
  }
}

# Subscription: real-time data
subscription {
  postAdded {
    id
    title
    author {
      name
    }
  }
}
```

---

## 2. Spring for GraphQL Setup

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-graphql</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- For WebSocket subscriptions -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- GraphiQL UI (auto-configures at /graphiql) -->
    <!-- Included in spring-boot-starter-graphql -->
</dependencies>
```

### application.yaml

```yaml
spring:
  graphql:
    graphiql:
      enabled: true          # Enable GraphiQL UI at /graphiql
      path: /graphiql
    path: /graphql           # GraphQL endpoint
    websocket:
      path: /graphql-ws      # WebSocket path for subscriptions
    schema:
      locations: classpath:graphql/   # Location of .graphqls files
      file-extensions: .graphqls,.gqls
      printer:
        enabled: true        # Print schema to logs on startup
    cors:
      allowed-origins: "http://localhost:3000"

logging:
  level:
    org.springframework.graphql: DEBUG
```

---

## 3. GraphQL Schema Definition Language (SDL)

### Schema File

```graphql
# src/main/resources/graphql/schema.graphqls

# ─── Scalars ──────────────────────────────────────────────────────────────
scalar DateTime
scalar URL

# ─── Enums ────────────────────────────────────────────────────────────────
enum PostStatus {
    DRAFT
    PUBLISHED
    ARCHIVED
}

enum SortDirection {
    ASC
    DESC
}

# ─── Types ────────────────────────────────────────────────────────────────
type User {
    id: ID!
    username: String!
    email: String!
    bio: String
    avatarUrl: URL
    posts(first: Int, after: String): PostConnection!
    followersCount: Int!
    followingCount: Int!
    createdAt: DateTime!
}

type Post {
    id: ID!
    title: String!
    content: String!
    excerpt: String
    status: PostStatus!
    author: User!
    tags: [Tag!]!
    comments(first: Int, after: String): CommentConnection!
    viewCount: Int!
    likeCount: Int!
    createdAt: DateTime!
    updatedAt: DateTime!
    publishedAt: DateTime
}

type Tag {
    id: ID!
    name: String!
    postsCount: Int!
}

type Comment {
    id: ID!
    content: String!
    author: User!
    post: Post!
    createdAt: DateTime!
}

# ─── Pagination (Relay Connection Pattern) ────────────────────────────────
type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type PostEdge {
    node: Post!
    cursor: String!
}

type CommentConnection {
    edges: [CommentEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type CommentEdge {
    node: Comment!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}

# ─── Queries ──────────────────────────────────────────────────────────────
type Query {
    # User queries
    user(id: ID!): User
    users(first: Int = 10, after: String, search: String): UserConnection!
    me: User

    # Post queries
    post(id: ID!): Post
    posts(
        first: Int = 10
        after: String
        status: PostStatus
        authorId: ID
        tagId: ID
        search: String
        sortBy: String = "createdAt"
        sortDir: SortDirection = DESC
    ): PostConnection!

    # Tag queries
    tags: [Tag!]!
    tag(id: ID!): Tag
}

# ─── Mutations ────────────────────────────────────────────────────────────
type Mutation {
    # Auth
    register(input: RegisterInput!): AuthPayload!
    login(email: String!, password: String!): AuthPayload!

    # User
    updateProfile(input: UpdateProfileInput!): User!
    deleteAccount: Boolean!

    # Post
    createPost(input: CreatePostInput!): Post!
    updatePost(id: ID!, input: UpdatePostInput!): Post!
    deletePost(id: ID!): Boolean!
    publishPost(id: ID!): Post!

    # Comment
    addComment(postId: ID!, content: String!): Comment!
    deleteComment(id: ID!): Boolean!
}

# ─── Subscriptions ────────────────────────────────────────────────────────
type Subscription {
    postAdded: Post!
    commentAdded(postId: ID!): Comment!
    postUpdated(id: ID!): Post!
}

# ─── Input Types ──────────────────────────────────────────────────────────
input RegisterInput {
    username: String!
    email: String!
    password: String!
}

input CreatePostInput {
    title: String!
    content: String!
    status: PostStatus = DRAFT
    tagIds: [ID!]
}

input UpdatePostInput {
    title: String
    content: String
    status: PostStatus
    tagIds: [ID!]
}

input UpdateProfileInput {
    bio: String
    avatarUrl: String
}

# ─── Response Types ───────────────────────────────────────────────────────
type AuthPayload {
    token: String!
    user: User!
}

type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
}

type UserEdge {
    node: User!
    cursor: String!
}
```

---

## 4. @QueryMapping, @MutationMapping, @SubscriptionMapping

### Query Controller

```java
// src/main/java/com/example/graphql/controller/UserController.java
package com.example.graphql.controller;

import com.example.graphql.model.User;
import com.example.graphql.service.UserService;
import graphql.relay.Connection;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.MutationMapping;
import org.springframework.graphql.data.method.annotation.QueryMapping;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;

import java.util.List;
import java.util.Optional;

@Controller
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @QueryMapping
    public Optional<User> user(@Argument String id) {
        return userService.findById(id);
    }

    @QueryMapping
    @PreAuthorize("isAuthenticated()")
    public User me(java.security.Principal principal) {
        return userService.findByUsername(principal.getName());
    }

    @QueryMapping
    public Connection<User> users(
            @Argument int first,
            @Argument String after,
            @Argument String search) {
        return userService.findAll(first, after, search);
    }
}
```

### Post Controller

```java
// src/main/java/com/example/graphql/controller/PostController.java
package com.example.graphql.controller;

import com.example.graphql.model.*;
import com.example.graphql.service.PostService;
import graphql.relay.Connection;
import org.springframework.graphql.data.method.annotation.*;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;

import java.util.Optional;

@Controller
public class PostController {

    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @QueryMapping
    public Optional<Post> post(@Argument String id) {
        return postService.findById(id);
    }

    @QueryMapping
    public Connection<Post> posts(
            @Argument int first,
            @Argument String after,
            @Argument PostStatus status,
            @Argument String authorId,
            @Argument String tagId,
            @Argument String search,
            @Argument String sortBy,
            @Argument SortDirection sortDir) {
        return postService.findAll(first, after, status, authorId, tagId, search, sortBy, sortDir);
    }

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Post createPost(@Argument CreatePostInput input,
                           java.security.Principal principal) {
        return postService.create(input, principal.getName());
    }

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Post updatePost(@Argument String id,
                           @Argument UpdatePostInput input,
                           java.security.Principal principal) {
        return postService.update(id, input, principal.getName());
    }

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public boolean deletePost(@Argument String id,
                              java.security.Principal principal) {
        postService.delete(id, principal.getName());
        return true;
    }

    @MutationMapping
    @PreAuthorize("isAuthenticated()")
    public Post publishPost(@Argument String id,
                            java.security.Principal principal) {
        return postService.publish(id, principal.getName());
    }
}
```

---

## 5. @SchemaMapping and DataFetcher

### Schema Mapping for Nested Fields

```java
// src/main/java/com/example/graphql/controller/PostSchemaController.java
package com.example.graphql.controller;

import com.example.graphql.model.*;
import com.example.graphql.service.UserService;
import com.example.graphql.service.CommentService;
import graphql.relay.Connection;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;

@Controller
public class PostSchemaController {

    private final UserService userService;
    private final CommentService commentService;

    public PostSchemaController(UserService userService, CommentService commentService) {
        this.userService = userService;
        this.commentService = commentService;
    }

    // Resolve Post.author: called for each Post in the result
    @SchemaMapping(typeName = "Post", field = "author")
    public User getAuthor(Post post) {
        // ⚠️ This causes N+1 problem! Use DataLoader instead (see Section 6)
        return userService.findById(post.getAuthorId()).orElseThrow();
    }

    // Resolve Post.comments with pagination
    @SchemaMapping(typeName = "Post", field = "comments")
    public Connection<Comment> getComments(Post post,
                                            @Argument int first,
                                            @Argument String after) {
        return commentService.findByPostId(post.getId(), first, after);
    }

    // Resolve User.posts with pagination
    @SchemaMapping(typeName = "User", field = "posts")
    public Connection<Post> getUserPosts(User user,
                                         @Argument int first,
                                         @Argument String after) {
        return postService.findByAuthorId(user.getId(), first, after);
    }

    // Resolve User.followersCount
    @SchemaMapping(typeName = "User", field = "followersCount")
    public int getFollowersCount(User user) {
        return userService.countFollowers(user.getId());
    }
}
```

---

## 6. N+1 Problem and DataLoader

### The N+1 Problem

```
Query: posts { author { name } }
→ Fetches N posts
→ For each post, fetches 1 author → N database queries
→ Total: 1 + N queries

With DataLoader:
→ Fetches N posts
→ Batches all author IDs into 1 query
→ Total: 2 queries
```

### DataLoader Configuration

```java
// src/main/java/com/example/graphql/loader/UserDataLoader.java
package com.example.graphql.loader;

import com.example.graphql.model.User;
import com.example.graphql.repository.UserRepository;
import org.dataloader.BatchLoaderEnvironment;
import org.springframework.graphql.execution.BatchLoaderRegistry;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.function.Function;
import java.util.stream.Collectors;

@Component
public class UserDataLoader {

    public UserDataLoader(BatchLoaderRegistry registry, UserRepository userRepository) {
        // Register batch loader: given Set<String> of IDs, return Map<String, User>
        registry.forTypePair(String.class, User.class)
            .withName("userLoader")
            .registerBatchLoader((ids, env) -> {
                // Single query to fetch all users by their IDs
                List<User> users = userRepository.findAllById(ids);

                // Map by ID for O(1) lookup
                Map<String, User> userMap = users.stream()
                    .collect(Collectors.toMap(User::getId, Function.identity()));

                // Return in same order as requested IDs
                return Mono.just(userMap);
            });
    }
}
```

### Using DataLoader in Controller

```java
// src/main/java/com/example/graphql/controller/PostSchemaControllerWithLoader.java
package com.example.graphql.controller;

import com.example.graphql.model.Post;
import com.example.graphql.model.User;
import org.dataloader.DataLoader;
import org.springframework.graphql.data.method.annotation.SchemaMapping;
import org.springframework.stereotype.Controller;

import java.util.concurrent.CompletableFuture;

@Controller
public class PostSchemaControllerWithLoader {

    // DataLoader batches all author IDs from the current GraphQL execution
    // into a single database query
    @SchemaMapping(typeName = "Post", field = "author")
    public CompletableFuture<User> getAuthor(Post post,
                                              DataLoader<String, User> userLoader) {
        // Returns a future; DataLoader defers the actual load
        // and batches all pending IDs
        return userLoader.load(post.getAuthorId());
    }
}
```

### Multiple DataLoaders

```java
// src/main/java/com/example/graphql/loader/AllDataLoaders.java
package com.example.graphql.loader;

import com.example.graphql.model.*;
import com.example.graphql.repository.*;
import org.springframework.graphql.execution.BatchLoaderRegistry;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

@Component
public class AllDataLoaders {

    public AllDataLoaders(BatchLoaderRegistry registry,
                          UserRepository userRepository,
                          TagRepository tagRepository,
                          CommentRepository commentRepository) {

        // User loader
        registry.forTypePair(String.class, User.class)
            .withName("userLoader")
            .registerBatchLoader((ids, env) -> {
                Map<String, User> map = userRepository.findAllById(ids)
                    .stream().collect(Collectors.toMap(User::getId, Function.identity()));
                return Mono.just(map);
            });

        // Tags loader (returns List<Tag> per post ID)
        registry.forTypePair(String.class, List.class)
            .withName("postTagsLoader")
            .registerBatchLoader((postIds, env) -> {
                Map<String, List<Tag>> tagsByPostId = tagRepository.findByPostIdIn(postIds)
                    .stream().collect(Collectors.groupingBy(Tag::getPostId));
                return Mono.just(tagsByPostId);
            });
    }
}
```

---

## 7. Error Handling in GraphQL

### Custom Error Handler

```java
// src/main/java/com/example/graphql/exception/GraphQLExceptionHandler.java
package com.example.graphql.exception;

import graphql.ErrorClassification;
import graphql.GraphQLError;
import graphql.schema.DataFetchingEnvironment;
import org.springframework.graphql.execution.DataFetcherExceptionResolverAdapter;
import org.springframework.graphql.execution.ErrorType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.stereotype.Component;

@Component
public class GraphQLExceptionHandler extends DataFetcherExceptionResolverAdapter {

    @Override
    protected GraphQLError resolveToSingleError(Throwable ex, DataFetchingEnvironment env) {
        if (ex instanceof NotFoundException e) {
            return GraphQLError.newError()
                .errorType(ErrorType.NOT_FOUND)
                .message(e.getMessage())
                .path(env.getExecutionStepInfo().getPath())
                .location(env.getField().getSourceLocation())
                .build();
        }

        if (ex instanceof ValidationException e) {
            return GraphQLError.newError()
                .errorType(ErrorType.BAD_REQUEST)
                .message(e.getMessage())
                .extensions(Map.of(
                    "field", e.getField(),
                    "code", e.getCode()
                ))
                .build();
        }

        if (ex instanceof AccessDeniedException) {
            return GraphQLError.newError()
                .errorType(ErrorType.FORBIDDEN)
                .message("You don't have permission to perform this action")
                .build();
        }

        // Default: internal server error (don't expose stack trace)
        return GraphQLError.newError()
            .errorType(ErrorType.INTERNAL_ERROR)
            .message("An internal error occurred")
            .build();
    }
}
```

### Custom Domain Exceptions

```java
// src/main/java/com/example/graphql/exception/NotFoundException.java
package com.example.graphql.exception;

public class NotFoundException extends RuntimeException {
    public NotFoundException(String type, String id) {
        super(type + " not found: " + id);
    }
}

// src/main/java/com/example/graphql/exception/ValidationException.java
package com.example.graphql.exception;

public class ValidationException extends RuntimeException {
    private final String field;
    private final String code;

    public ValidationException(String field, String message, String code) {
        super(message);
        this.field = field;
        this.code = code;
    }

    public String getField() { return field; }
    public String getCode() { return code; }
}
```

### GraphQL Error Response Format

```json
{
  "data": null,
  "errors": [
    {
      "message": "Post not found: 999",
      "locations": [{"line": 2, "column": 3}],
      "path": ["post"],
      "extensions": {
        "classification": "NOT_FOUND"
      }
    }
  ]
}
```

---

## 8. Pagination with Connections

### Relay-Style Cursor Pagination Service

```java
// src/main/java/com/example/graphql/service/CursorPaginationService.java
package com.example.graphql.service;

import graphql.relay.*;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class CursorPaginationService {

    public <T> Connection<T> paginate(List<T> items, int first, String after,
                                       java.util.function.Function<T, String> idExtractor,
                                       long totalCount) {
        // Decode cursor to offset
        int offset = after != null ? decodeCursor(after) : 0;

        List<T> pageItems = items.stream()
            .skip(offset)
            .limit(first)
            .collect(Collectors.toList());

        List<Edge<T>> edges = pageItems.stream()
            .map(item -> new DefaultEdge<>(item,
                new DefaultConnectionCursor(encodeCursor(idExtractor.apply(item)))))
            .collect(Collectors.toList());

        String startCursor = edges.isEmpty() ? null : edges.get(0).getCursor().getValue();
        String endCursor = edges.isEmpty() ? null : edges.get(edges.size() - 1).getCursor().getValue();

        PageInfo pageInfo = new DefaultPageInfo(
            startCursor != null ? new DefaultConnectionCursor(startCursor) : null,
            endCursor != null ? new DefaultConnectionCursor(endCursor) : null,
            offset > 0,                                    // hasPreviousPage
            offset + pageItems.size() < totalCount         // hasNextPage
        );

        return new DefaultConnection<>(edges, pageInfo);
    }

    public String encodeCursor(String id) {
        return Base64.getEncoder().encodeToString(("cursor:" + id).getBytes(StandardCharsets.UTF_8));
    }

    public int decodeCursor(String cursor) {
        try {
            String decoded = new String(Base64.getDecoder().decode(cursor), StandardCharsets.UTF_8);
            return Integer.parseInt(decoded.replace("cursor:", ""));
        } catch (Exception e) {
            return 0;
        }
    }
}
```

### Post Service with Pagination

```java
// src/main/java/com/example/graphql/service/PostService.java
package com.example.graphql.service;

import com.example.graphql.model.*;
import com.example.graphql.repository.PostRepository;
import graphql.relay.Connection;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Sinks;

import java.util.*;
import java.util.Optional;

@Service
public class PostService {

    private final PostRepository postRepository;
    private final CursorPaginationService paginationService;

    // For subscriptions
    private final Sinks.Many<Post> postSink = Sinks.many().multicast().onBackpressureBuffer();

    public PostService(PostRepository postRepository, CursorPaginationService paginationService) {
        this.postRepository = postRepository;
        this.paginationService = paginationService;
    }

    public Connection<Post> findAll(int first, String after, PostStatus status,
                                    String authorId, String tagId, String search,
                                    String sortBy, SortDirection sortDir) {
        Sort sort = Sort.by(
            sortDir == SortDirection.ASC ? Sort.Direction.ASC : Sort.Direction.DESC,
            sortBy
        );

        List<Post> posts = postRepository.findWithFilters(status, authorId, tagId, search, sort);
        long total = posts.size();

        return paginationService.paginate(posts, first, after, Post::getId, total);
    }

    public Optional<Post> findById(String id) {
        return postRepository.findById(id);
    }

    public Post create(CreatePostInput input, String username) {
        Post post = new Post();
        post.setTitle(input.getTitle());
        post.setContent(input.getContent());
        post.setStatus(input.getStatus() != null ? input.getStatus() : PostStatus.DRAFT);
        post.setAuthorId(username);

        Post saved = postRepository.save(post);

        // Notify subscriptions
        postSink.tryEmitNext(saved);

        return saved;
    }

    public Post update(String id, UpdatePostInput input, String username) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Post", id));

        if (!post.getAuthorId().equals(username)) {
            throw new AccessDeniedException("Not the author");
        }

        if (input.getTitle() != null) post.setTitle(input.getTitle());
        if (input.getContent() != null) post.setContent(input.getContent());
        if (input.getStatus() != null) post.setStatus(input.getStatus());

        Post updated = postRepository.save(post);
        postSink.tryEmitNext(updated);
        return updated;
    }

    public void delete(String id, String username) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Post", id));

        if (!post.getAuthorId().equals(username)) {
            throw new AccessDeniedException("Not the author");
        }

        postRepository.delete(post);
    }

    public Post publish(String id, String username) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Post", id));

        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(java.time.Instant.now());
        return postRepository.save(post);
    }

    // For subscriptions
    public Flux<Post> getPostAddedFlux() {
        return postSink.asFlux();
    }
}
```

---

## 9. Subscriptions with WebSocket

### Subscription Controller

```java
// src/main/java/com/example/graphql/controller/SubscriptionController.java
package com.example.graphql.controller;

import com.example.graphql.model.Comment;
import com.example.graphql.model.Post;
import com.example.graphql.service.CommentService;
import com.example.graphql.service.PostService;
import org.springframework.graphql.data.method.annotation.Argument;
import org.springframework.graphql.data.method.annotation.SubscriptionMapping;
import org.springframework.stereotype.Controller;
import reactor.core.publisher.Flux;

@Controller
public class SubscriptionController {

    private final PostService postService;
    private final CommentService commentService;

    public SubscriptionController(PostService postService, CommentService commentService) {
        this.postService = postService;
        this.commentService = commentService;
    }

    @SubscriptionMapping
    public Flux<Post> postAdded() {
        return postService.getPostAddedFlux();
    }

    @SubscriptionMapping
    public Flux<Comment> commentAdded(@Argument String postId) {
        return commentService.getCommentFlux()
            .filter(comment -> comment.getPostId().equals(postId));
    }

    @SubscriptionMapping
    public Flux<Post> postUpdated(@Argument String id) {
        return postService.getPostAddedFlux()
            .filter(post -> post.getId().equals(id));
    }
}
```

### WebSocket Configuration

```java
// src/main/java/com/example/graphql/config/GraphQLWebSocketConfig.java
package com.example.graphql.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class GraphQLWebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // Spring for GraphQL auto-configures this via spring.graphql.websocket.path
        // Manual configuration only needed for STOMP or custom protocols
    }
}
```

### Client-Side Subscription (JavaScript)

```javascript
// Using graphql-ws library
import { createClient } from 'graphql-ws';
import WebSocket from 'ws';

const client = createClient({
  url: 'ws://localhost:8080/graphql-ws',
  webSocketImpl: WebSocket,
});

// Subscribe to new posts
const unsubscribe = client.subscribe(
  {
    query: `
      subscription {
        postAdded {
          id
          title
          author { name }
          createdAt
        }
      }
    `,
  },
  {
    next: (data) => console.log('New post:', data.data.postAdded),
    error: (err) => console.error('Error:', err),
    complete: () => console.log('Subscription completed'),
  }
);

// Later: unsubscribe()
```

---

## 10. GraphQL Security

### Depth Limiting

```java
// src/main/java/com/example/graphql/config/GraphQLSecurityConfig.java
package com.example.graphql.config;

import graphql.analysis.MaxQueryDepthInstrumentation;
import graphql.analysis.MaxQueryComplexityInstrumentation;
import org.springframework.boot.autoconfigure.graphql.GraphQlSourceBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GraphQLSecurityConfig {

    @Bean
    public GraphQlSourceBuilderCustomizer graphQlSourceCustomizer() {
        return builder -> builder
            .configureGraphQl(graphQlBuilder -> graphQlBuilder
                // Prevent deeply nested queries (default: unlimited)
                .instrumentation(new MaxQueryDepthInstrumentation(10))
                // Prevent computationally expensive queries
                .instrumentation(new MaxQueryComplexityInstrumentation(100))
            );
    }
}
```

### Custom Complexity Calculator

```java
// src/main/java/com/example/graphql/config/ComplexityConfig.java
package com.example.graphql.config;

import graphql.analysis.FieldComplexityCalculator;
import graphql.analysis.FieldComplexityEnvironment;
import graphql.analysis.MaxQueryComplexityInstrumentation;
import org.springframework.boot.autoconfigure.graphql.GraphQlSourceBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ComplexityConfig {

    @Bean
    public GraphQlSourceBuilderCustomizer complexityCustomizer() {
        FieldComplexityCalculator calculator = (env, childComplexity) -> {
            // Posts query is expensive (database + pagination)
            if (env.getField().getName().equals("posts")) {
                return childComplexity + 10;
            }
            // Subscriptions
            if (env.getField().getName().startsWith("post")) {
                return childComplexity + 5;
            }
            return childComplexity + 1;
        };

        return builder -> builder
            .configureGraphQl(graphQlBuilder -> graphQlBuilder
                .instrumentation(new MaxQueryComplexityInstrumentation(50, calculator))
            );
    }
}
```

### Persisted Queries (Prevent arbitrary queries in production)

```java
// src/main/java/com/example/graphql/security/PersistedQueryFilter.java
package com.example.graphql.security;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Set;

@Component
public class PersistedQueryFilter implements Filter {

    @Value("${graphql.security.persisted-queries-only:false}")
    private boolean persistedQueriesOnly;

    // In production, load from registry
    private final Set<String> allowedQueryIds = Set.of(
        "q1-get-posts",
        "q2-get-user",
        "m1-create-post"
    );

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        if (persistedQueriesOnly && request instanceof HttpServletRequest httpRequest) {
            String queryId = httpRequest.getHeader("X-Query-Id");
            if (queryId == null || !allowedQueryIds.contains(queryId)) {
                ((HttpServletResponse) response).sendError(
                    HttpServletResponse.SC_BAD_REQUEST,
                    "Only persisted queries are allowed"
                );
                return;
            }
        }

        chain.doFilter(request, response);
    }
}
```

---

## 11. GraphiQL and Voyager Tools

### GraphiQL (auto-configured)

```yaml
# application.yaml
spring:
  graphql:
    graphiql:
      enabled: true    # Accessible at http://localhost:8080/graphiql
```

### Voyager (Schema visualization)

```java
// src/main/java/com/example/graphql/controller/VoyagerController.java
package com.example.graphql.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class VoyagerController {

    @GetMapping("/voyager")
    public String voyager() {
        return "voyager";  // voyager.html template
    }
}
```

```html
<!-- src/main/resources/templates/voyager.html -->
<!DOCTYPE html>
<html>
<head>
    <title>API Voyager</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/graphql-voyager/dist/voyager.css">
</head>
<body>
<div id="voyager">Loading...</div>
<script src="https://cdn.jsdelivr.net/npm/graphql-voyager/dist/voyager.standalone.js"></script>
<script>
    GraphQLVoyager.init(document.getElementById('voyager'), {
        introspection: function(query) {
            return fetch('/graphql', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ query })
            }).then(res => res.json());
        }
    });
</script>
</body>
</html>
```

---

## 12. Real Example: Blog API with Full CRUD

### Domain Models

```java
// src/main/java/com/example/graphql/model/Post.java
package com.example.graphql.model;

import jakarta.persistence.*;
import java.time.Instant;
import java.util.List;

@Entity
@Table(name = "posts")
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT", nullable = false)
    private String content;

    @Enumerated(EnumType.STRING)
    private PostStatus status = PostStatus.DRAFT;

    @Column(name = "author_id", nullable = false)
    private String authorId;

    @Column(name = "view_count")
    private int viewCount = 0;

    @Column(name = "like_count")
    private int likeCount = 0;

    @Column(name = "created_at")
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    @Column(name = "published_at")
    private Instant publishedAt;

    @ManyToMany
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private List<Tag> tags;

    @PrePersist
    public void prePersist() {
        createdAt = Instant.now();
        updatedAt = Instant.now();
    }

    @PreUpdate
    public void preUpdate() {
        updatedAt = Instant.now();
    }

    // Getters and setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }

    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }

    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }

    public PostStatus getStatus() { return status; }
    public void setStatus(PostStatus status) { this.status = status; }

    public String getAuthorId() { return authorId; }
    public void setAuthorId(String authorId) { this.authorId = authorId; }

    public int getViewCount() { return viewCount; }
    public void setViewCount(int viewCount) { this.viewCount = viewCount; }

    public int getLikeCount() { return likeCount; }
    public void setLikeCount(int likeCount) { this.likeCount = likeCount; }

    public Instant getCreatedAt() { return createdAt; }
    public void setCreatedAt(Instant createdAt) { this.createdAt = createdAt; }

    public Instant getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(Instant updatedAt) { this.updatedAt = updatedAt; }

    public Instant getPublishedAt() { return publishedAt; }
    public void setPublishedAt(Instant publishedAt) { this.publishedAt = publishedAt; }

    public List<Tag> getTags() { return tags; }
    public void setTags(List<Tag> tags) { this.tags = tags; }
}
```

### Repository with Custom Queries

```java
// src/main/java/com/example/graphql/repository/PostRepository.java
package com.example.graphql.repository;

import com.example.graphql.model.Post;
import com.example.graphql.model.PostStatus;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface PostRepository extends JpaRepository<Post, String> {

    @Query("""
        SELECT p FROM Post p
        LEFT JOIN p.tags t
        WHERE (:status IS NULL OR p.status = :status)
          AND (:authorId IS NULL OR p.authorId = :authorId)
          AND (:tagId IS NULL OR t.id = :tagId)
          AND (:search IS NULL OR
               LOWER(p.title) LIKE LOWER(CONCAT('%', :search, '%')) OR
               LOWER(p.content) LIKE LOWER(CONCAT('%', :search, '%')))
        """)
    List<Post> findWithFilters(
        @Param("status") PostStatus status,
        @Param("authorId") String authorId,
        @Param("tagId") String tagId,
        @Param("search") String search,
        Sort sort
    );
}
```

### Complete Integration Test

```java
// src/test/java/com/example/graphql/PostGraphQLTest.java
package com.example.graphql;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.graphql.tester.AutoConfigureHttpGraphQlTester;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.graphql.test.tester.HttpGraphQlTester;

@SpringBootTest
@AutoConfigureHttpGraphQlTester
class PostGraphQLTest {

    @Autowired
    private HttpGraphQlTester graphQlTester;

    @Test
    void shouldCreateAndFetchPost() {
        // Create a post
        String postId = graphQlTester.document("""
            mutation {
              createPost(input: {
                title: "Test Post"
                content: "Test content"
                status: DRAFT
              }) {
                id
                title
                status
              }
            }
            """)
            .execute()
            .path("createPost.id")
            .entity(String.class)
            .get();

        // Fetch the created post
        graphQlTester.document("""
            query GetPost($id: ID!) {
              post(id: $id) {
                title
                content
                status
                author { username }
              }
            }
            """)
            .variable("id", postId)
            .execute()
            .path("post.title").entity(String.class).isEqualTo("Test Post")
            .path("post.status").entity(String.class).isEqualTo("DRAFT");
    }

    @Test
    void shouldReturnNotFoundError() {
        graphQlTester.document("""
            query {
              post(id: "nonexistent") {
                title
              }
            }
            """)
            .execute()
            .path("post").valueIsNull()
            .errors()
            .satisfy(errors -> {
                assert errors.size() == 1;
                assert errors.get(0).getMessage().contains("not found");
            });
    }
}
```

---

## Summary Table

| Topic | Annotation/Class | Notes |
|-------|-----------------|-------|
| Query | `@QueryMapping` | Maps to GraphQL Query type field |
| Mutation | `@MutationMapping` | Maps to GraphQL Mutation type field |
| Subscription | `@SubscriptionMapping` | Returns `Flux<T>` |
| Nested field | `@SchemaMapping(typeName, field)` | Resolves sub-fields |
| N+1 fix | `DataLoader<K, V>` | Batch loads via `BatchLoaderRegistry` |
| Error handling | `DataFetcherExceptionResolverAdapter` | Map exceptions to GraphQL errors |
| Pagination | Relay Connection pattern | `first`, `after`, `edges`, `pageInfo` |
| Depth limit | `MaxQueryDepthInstrumentation` | Prevent deep nesting attacks |
| Complexity | `MaxQueryComplexityInstrumentation` | Prevent expensive queries |
| Testing | `HttpGraphQlTester` | `@AutoConfigureHttpGraphQlTester` |
| GraphiQL | `spring.graphql.graphiql.enabled=true` | Browser-based IDE |

---

## What's Next

**Part 039: gRPC with Spring Boot** — Build high-performance, strongly-typed service communication with Protocol Buffers and gRPC. We'll cover unary and streaming RPCs, interceptors, TLS, and implement a file upload streaming service.

---

*End of Part 038: GraphQL with Spring Boot*
