# Part 032: Spring WebFlux - Reactive Programming

## เนื้อหาในส่วนนี้
- Reactive Programming Concepts
- Project Reactor: Mono และ Flux
- WebFlux vs MVC เมื่อไรควรใช้อะไร
- Reactive Controllers
- Reactive Repositories (R2DBC)
- Error Handling ใน Reactive
- WebClient (Reactive HTTP Client)
- Server-Sent Events (SSE)
- WebSocket
- Complete Reactive API Example

---

## 1. Reactive Programming Concepts

```
Traditional (Blocking):
Thread → Request → Wait for DB → Wait for Network → Response
(Thread is blocked during wait)

Reactive (Non-blocking):
Thread → Request → Subscribe to async operations → Thread free
         ↑ Data comes when ready, callback invoked
         
Benefits:
- Better resource utilization (fewer threads needed)
- Handle more concurrent requests
- Backpressure support
- Better for I/O-heavy workloads
```

### Maven Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<!-- R2DBC for reactive database access -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-h2</artifactId>
</dependency>

<!-- Testing -->
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 2. Project Reactor: Mono and Flux

```java
import reactor.core.publisher.*;
import reactor.core.scheduler.Schedulers;
import java.time.Duration;
import java.util.*;

public class ReactorBasics {
    
    // === Mono - 0 or 1 item ===
    public void monoExamples() {
        // Create
        Mono<String> empty = Mono.empty();
        Mono<String> just = Mono.just("Hello");
        Mono<String> fromSupplier = Mono.fromSupplier(() -> "From Supplier");
        Mono<String> fromCallable = Mono.fromCallable(() -> "From Callable");
        Mono<String> error = Mono.error(new RuntimeException("Error"));
        Mono<String> defer = Mono.defer(() -> Mono.just("Lazy evaluation"));
        
        // Transform
        Mono<Integer> length = just.map(String::length);
        Mono<String> upper = just.map(String::toUpperCase);
        
        // Async transform (flatMap)
        Mono<String> asyncOp = just.flatMap(s -> 
            Mono.fromSupplier(() -> s + " World"));
        
        // Filter (if condition fails, becomes empty)
        Mono<String> filtered = just.filter(s -> s.length() > 3);
        
        // Default value if empty
        Mono<String> withDefault = empty.defaultIfEmpty("Default Value");
        
        // Switch to another Mono if empty
        Mono<String> switched = empty.switchIfEmpty(Mono.just("Alternative"));
        
        // Handle errors
        Mono<String> withFallback = error
            .onErrorReturn("Fallback")
            .onErrorResume(e -> Mono.just("Resume from error: " + e.getMessage()));
        
        // Log each step
        Mono<String> logged = just
            .doOnSubscribe(s -> System.out.println("Subscribed"))
            .doOnNext(v -> System.out.println("Next: " + v))
            .doOnSuccess(v -> System.out.println("Success: " + v))
            .doOnError(e -> System.out.println("Error: " + e.getMessage()))
            .doOnTerminate(() -> System.out.println("Terminated"));
        
        // Block (subscribe synchronously - use only in tests or main())
        String result = just.block();
        System.out.println(result);
        
        // Subscribe asynchronously
        just.subscribe(
            value -> System.out.println("Got: " + value),
            error2 -> System.err.println("Error: " + error2.getMessage()),
            () -> System.out.println("Complete")
        );
    }
    
    // === Flux - 0 to N items ===
    public void fluxExamples() {
        // Create
        Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);
        Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));
        Flux<Integer> range = Flux.range(1, 10);
        Flux<Long> interval = Flux.interval(Duration.ofSeconds(1)); // Ticks every second
        Flux<Integer> empty = Flux.empty();
        
        // Create programmatically
        Flux<String> custom = Flux.create(sink -> {
            sink.next("item1");
            sink.next("item2");
            sink.complete();
        });
        
        // Transform
        Flux<String> strings = numbers.map(n -> "Number: " + n);
        
        // Async transform
        Flux<String> async = numbers.flatMap(n -> 
            Mono.fromSupplier(() -> "Result " + n)
                .subscribeOn(Schedulers.boundedElastic()));
        
        // Filter
        Flux<Integer> even = numbers.filter(n -> n % 2 == 0);
        
        // Reduce
        Mono<Integer> sum = numbers.reduce(0, Integer::sum);
        
        // Collect
        Mono<List<Integer>> list = numbers.collectList();
        Mono<Map<Integer, String>> map = numbers.collectMap(n -> n, n -> "val" + n);
        
        // Combine
        Flux<String> combined = Flux.concat(fromList, Flux.just("d", "e"));
        Flux<String> merged = Flux.merge(fromList, Flux.just("x", "y"));
        Flux<String> zipped = Flux.zip(fromList, Flux.just("1", "2", "3"))
            .map(t -> t.getT1() + t.getT2());
        
        // Take/skip
        Flux<Integer> first3 = numbers.take(3);
        Flux<Integer> skip2 = numbers.skip(2);
        
        // Buffer
        Flux<List<Integer>> buffered = numbers.buffer(2); // [1,2], [3,4], [5]
        
        // Window
        Flux<Flux<Integer>> windowed = numbers.window(3);
        
        // Error handling
        Flux<Integer> withError = numbers
            .map(n -> {
                if (n == 3) throw new RuntimeException("Oops at 3");
                return n;
            })
            .onErrorContinue((e, v) -> System.out.println("Skipping " + v + ": " + e.getMessage()));
        
        // Subscribe
        numbers.subscribe(
            n -> System.out.print(n + " "),
            e -> System.err.println("Error: " + e.getMessage()),
            () -> System.out.println("\nDone!")
        );
        
        // Block
        List<Integer> blocked = numbers.collectList().block();
    }
}
```

---

## 3. Reactive Controllers

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;
import reactor.core.publisher.*;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/reactive/products")
public class ReactiveProductController {
    
    private final ReactiveProductService productService;
    
    ReactiveProductController(ReactiveProductService productService) {
        this.productService = productService;
    }
    
    // GET all - returns Flux (stream of items)
    @GetMapping
    public Flux<ReactiveProduct> getAllProducts() {
        return productService.findAll();
    }
    
    // GET by id - returns Mono (0 or 1 item)
    @GetMapping("/{id}")
    public Mono<ResponseEntity<ReactiveProduct>> getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    // POST - create
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<ReactiveProduct> createProduct(@RequestBody Mono<CreateProductReq> request) {
        return request.flatMap(productService::create);
    }
    
    // PUT - update
    @PutMapping("/{id}")
    public Mono<ResponseEntity<ReactiveProduct>> updateProduct(
            @PathVariable Long id,
            @RequestBody Mono<CreateProductReq> request) {
        
        return request
            .flatMap(req -> productService.update(id, req))
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
    
    // DELETE
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> deleteProduct(@PathVariable Long id) {
        return productService.delete(id)
            .then(Mono.just(ResponseEntity.<Void>noContent().build()))
            .onErrorReturn(ResponseEntity.notFound().build());
    }
    
    // GET stream (Server-Sent Events)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ReactiveProduct> streamProducts() {
        return productService.findAll()
            .delayElements(Duration.ofMillis(500)); // Simulate slow stream
    }
    
    // GET with filtering
    @GetMapping("/search")
    public Flux<ReactiveProduct> searchProducts(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        return productService.search(category, page, size);
    }
}

record ReactiveProduct(Long id, String name, double price, String category) {}
record CreateProductReq(String name, double price, String category) {}

@org.springframework.stereotype.Service
class ReactiveProductService {
    
    private final Map<Long, ReactiveProduct> store = new ConcurrentHashMap<>(Map.of(
        1L, new ReactiveProduct(1L, "Laptop", 999.99, "Electronics"),
        2L, new ReactiveProduct(2L, "Phone", 599.99, "Electronics"),
        3L, new ReactiveProduct(3L, "Desk", 299.99, "Furniture")
    ));
    private long nextId = 4L;
    
    Flux<ReactiveProduct> findAll() {
        return Flux.fromIterable(store.values());
    }
    
    Mono<ReactiveProduct> findById(Long id) {
        return Mono.justOrEmpty(store.get(id));
    }
    
    Mono<ReactiveProduct> create(CreateProductReq req) {
        return Mono.fromSupplier(() -> {
            long id = nextId++;
            ReactiveProduct product = new ReactiveProduct(id, req.name(), req.price(), req.category());
            store.put(id, product);
            return product;
        });
    }
    
    Mono<ReactiveProduct> update(Long id, CreateProductReq req) {
        return Mono.fromSupplier(() -> {
            if (!store.containsKey(id)) return null;
            ReactiveProduct updated = new ReactiveProduct(id, req.name(), req.price(), req.category());
            store.put(id, updated);
            return updated;
        }).filter(Objects::nonNull);
    }
    
    Mono<Void> delete(Long id) {
        return Mono.fromRunnable(() -> {
            if (!store.containsKey(id)) throw new RuntimeException("Not found: " + id);
            store.remove(id);
        });
    }
    
    Flux<ReactiveProduct> search(String category, int page, int size) {
        return findAll()
            .filter(p -> category == null || p.category().equals(category))
            .skip((long) page * size)
            .take(size);
    }
}
```

---

## 4. Error Handling in Reactive

```java
import reactor.core.publisher.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;

@RestController
@RequestMapping("/api/reactive/error-handling")
public class ReactiveErrorController {
    
    // === onErrorReturn - fallback value ===
    @GetMapping("/fallback/{id}")
    public Mono<String> withFallback(@PathVariable Long id) {
        return Mono.fromCallable(() -> {
                if (id > 100) throw new IllegalArgumentException("ID too large");
                return "Product " + id;
            })
            .onErrorReturn("Default Product");
    }
    
    // === onErrorResume - switch to alternative Mono/Flux ===
    @GetMapping("/resume/{id}")
    public Mono<ReactiveProduct> withResume(@PathVariable Long id) {
        return fetchFromPrimary(id)
            .onErrorResume(e -> {
                System.out.println("Primary failed, trying secondary: " + e.getMessage());
                return fetchFromSecondary(id);
            })
            .onErrorResume(e -> {
                System.out.println("Secondary failed, using default");
                return Mono.just(new ReactiveProduct(id, "Default", 0, "Unknown"));
            });
    }
    
    // === onErrorMap - transform exception type ===
    @GetMapping("/transform/{id}")
    public Mono<ReactiveProduct> withErrorTransform(@PathVariable Long id) {
        return fetchFromPrimary(id)
            .onErrorMap(IllegalArgumentException.class, 
                e -> new ResponseStatusException(HttpStatus.BAD_REQUEST, e.getMessage()))
            .onErrorMap(RuntimeException.class, 
                e -> new ResponseStatusException(HttpStatus.INTERNAL_SERVER_ERROR, "Server error"));
    }
    
    // === onErrorContinue - skip erroring items in Flux ===
    @GetMapping("/continue")
    public Flux<String> withErrorContinue() {
        return Flux.just(1, 2, 3, 0, 5)
            .map(n -> {
                if (n == 0) throw new ArithmeticException("Division by zero");
                return "100 / " + n + " = " + (100 / n);
            })
            .onErrorContinue((e, v) -> 
                System.out.println("Skipping " + v + ": " + e.getMessage()));
    }
    
    // === doOnError - side effects (logging) ===
    @GetMapping("/logging/{id}")
    public Mono<ReactiveProduct> withErrorLogging(@PathVariable Long id) {
        return fetchFromPrimary(id)
            .doOnError(e -> System.err.println("Error for id " + id + ": " + e.getMessage()))
            .doOnError(IllegalArgumentException.class, 
                e -> System.err.println("Validation error: " + e.getMessage()));
    }
    
    // === retry ===
    @GetMapping("/retry/{id}")
    public Mono<ReactiveProduct> withRetry(@PathVariable Long id) {
        return fetchUnreliable(id)
            .retry(3) // retry up to 3 times
            .onErrorReturn(new ReactiveProduct(id, "After retry failed", 0, "Unknown"));
    }
    
    // === retryWhen with backoff ===
    @GetMapping("/retry-backoff/{id}")
    public Mono<ReactiveProduct> withRetryBackoff(@PathVariable Long id) {
        return fetchUnreliable(id)
            .retryWhen(reactor.util.retry.Retry
                .backoff(3, Duration.ofMillis(100))
                .maxBackoff(Duration.ofSeconds(2))
                .jitter(0.5)
                .filter(e -> e instanceof RuntimeException))
            .onErrorReturn(new ReactiveProduct(id, "Retry exhausted", 0, "Unknown"));
    }
    
    private Mono<ReactiveProduct> fetchFromPrimary(Long id) {
        if (id <= 0) throw new IllegalArgumentException("Invalid ID");
        return Mono.justOrEmpty(id > 10 ? null : new ReactiveProduct(id, "Product " + id, 99.0, "Cat"));
    }
    
    private Mono<ReactiveProduct> fetchFromSecondary(Long id) {
        return Mono.just(new ReactiveProduct(id, "Secondary " + id, 79.0, "Cat"));
    }
    
    private int callCount = 0;
    private Mono<ReactiveProduct> fetchUnreliable(Long id) {
        return Mono.fromCallable(() -> {
            callCount++;
            if (callCount % 3 != 0) throw new RuntimeException("Temporary failure #" + callCount);
            return new ReactiveProduct(id, "Success after " + callCount + " attempts", 99.0, "Cat");
        });
    }
}
```

---

## 5. WebClient - Reactive HTTP Client

```java
import org.springframework.web.reactive.function.client.*;
import org.springframework.stereotype.Service;
import reactor.core.publisher.*;
import org.springframework.http.*;
import java.time.Duration;

@Service
public class ReactiveHttpClient {
    
    private final WebClient webClient;
    
    ReactiveHttpClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://jsonplaceholder.typicode.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .codecs(config -> config.defaultCodecs().maxInMemorySize(16 * 1024 * 1024))
            .build();
    }
    
    // Simple GET
    public Mono<JsonPost> getPost(Long id) {
        return webClient.get()
            .uri("/posts/{id}", id)
            .retrieve()
            .bodyToMono(JsonPost.class);
    }
    
    // GET all
    public Flux<JsonPost> getAllPosts() {
        return webClient.get()
            .uri("/posts")
            .retrieve()
            .bodyToFlux(JsonPost.class);
    }
    
    // GET with query params
    public Flux<JsonPost> getPostsByUser(Long userId) {
        return webClient.get()
            .uri(uri -> uri.path("/posts")
                .queryParam("userId", userId)
                .build())
            .retrieve()
            .bodyToFlux(JsonPost.class);
    }
    
    // POST
    public Mono<JsonPost> createPost(JsonPost post) {
        return webClient.post()
            .uri("/posts")
            .bodyValue(post)
            .retrieve()
            .bodyToMono(JsonPost.class);
    }
    
    // With headers
    public Mono<JsonPost> getPostWithAuth(Long id, String token) {
        return webClient.get()
            .uri("/posts/{id}", id)
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
            .retrieve()
            .bodyToMono(JsonPost.class);
    }
    
    // Handle HTTP errors
    public Mono<JsonPost> getPostWithErrorHandling(Long id) {
        return webClient.get()
            .uri("/posts/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, 
                response -> response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(
                        new RuntimeException("Client error: " + body))))
            .onStatus(HttpStatusCode::is5xxServerError,
                response -> Mono.error(
                    new RuntimeException("Server error: " + response.statusCode())))
            .bodyToMono(JsonPost.class)
            .timeout(Duration.ofSeconds(5))
            .retry(2);
    }
    
    // Exchange (full control)
    public Mono<ResponseEntity<JsonPost>> getPostResponse(Long id) {
        return webClient.get()
            .uri("/posts/{id}", id)
            .retrieve()
            .toEntity(JsonPost.class);
    }
    
    // Parallel requests
    public Mono<List<JsonPost>> getMultiplePosts(List<Long> ids) {
        return Flux.fromIterable(ids)
            .flatMap(id -> getPost(id).onErrorResume(e -> Mono.empty()))
            .collectList();
    }
    
    // Sequential requests with dependency
    public Mono<String> getUserFirstPost(Long userId) {
        return getPostsByUser(userId)
            .next()  // Get first item
            .map(post -> "User " + userId + " first post: " + post.title());
    }
}

record JsonPost(Long id, Long userId, String title, String body) {}

// WebClient configuration
@org.springframework.context.annotation.Configuration
class WebClientConfig {
    
    @org.springframework.context.annotation.Bean
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder()
            .filter(logRequest())
            .filter(logResponse());
    }
    
    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            System.out.println("Request: " + request.method() + " " + request.url());
            return Mono.just(request);
        });
    }
    
    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            System.out.println("Response status: " + response.statusCode());
            return Mono.just(response);
        });
    }
}
```

---

## 6. Server-Sent Events (SSE)

```java
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.*;
import java.time.*;
import java.util.concurrent.atomic.AtomicLong;

@RestController
@RequestMapping("/api/sse")
public class SSEController {
    
    // Simple SSE stream
    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamEvents() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(tick -> "Event #" + tick + " at " + LocalTime.now())
            .take(20);
    }
    
    // SSE with typed events
    @GetMapping(value = "/typed", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockPrice>> streamStockPrices() {
        AtomicLong counter = new AtomicLong();
        
        return Flux.interval(Duration.ofMillis(500))
            .map(tick -> {
                long id = counter.incrementAndGet();
                String symbol = id % 2 == 0 ? "AAPL" : "GOOGL";
                double price = symbol.equals("AAPL") 
                    ? 150 + Math.random() * 10 
                    : 130 + Math.random() * 10;
                
                return ServerSentEvent.<StockPrice>builder()
                    .id(String.valueOf(id))
                    .event("stock-update")
                    .data(new StockPrice(symbol, price, LocalDateTime.now()))
                    .comment("price update for " + symbol)
                    .retry(Duration.ofSeconds(3))
                    .build();
            })
            .take(50);
    }
    
    // SSE from a stream that can be cancelled
    @GetMapping(value = "/live-metrics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<SystemMetric>> streamMetrics() {
        return Flux.interval(Duration.ofSeconds(2))
            .map(tick -> {
                Runtime rt = Runtime.getRuntime();
                long usedMem = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
                long totalMem = rt.totalMemory() / (1024 * 1024);
                
                return ServerSentEvent.<SystemMetric>builder()
                    .event("metric")
                    .data(new SystemMetric(
                        usedMem, totalMem,
                        Runtime.getRuntime().availableProcessors(),
                        tick
                    ))
                    .build();
            })
            .doOnCancel(() -> System.out.println("Client disconnected from metrics stream"));
    }
}

record StockPrice(String symbol, double price, LocalDateTime timestamp) {}
record SystemMetric(long usedMemoryMB, long totalMemoryMB, int cpuCores, long tick) {}

// Client-side JavaScript:
/*
const eventSource = new EventSource('/api/sse/typed');

eventSource.addEventListener('stock-update', (event) => {
    const data = JSON.parse(event.data);
    console.log(`${data.symbol}: $${data.price.toFixed(2)}`);
});

eventSource.onerror = (error) => {
    console.error('SSE error:', error);
    eventSource.close();
};
*/
```

---

## 7. Reactive WebSocket

```java
import org.springframework.web.reactive.socket.*;
import org.springframework.web.reactive.socket.server.support.HandlerMapping;
import org.springframework.context.annotation.*;
import reactor.core.publisher.*;
import java.time.*;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Configuration
public class WebSocketConfig {
    
    @Bean
    public HandlerMapping webSocketHandlerMapping(EchoWebSocketHandler echo,
                                                    ChatWebSocketHandler chat) {
        Map<String, WebSocketHandler> handlers = Map.of(
            "/ws/echo", echo,
            "/ws/chat", chat
        );
        
        org.springframework.web.reactive.handler.SimpleUrlHandlerMapping mapping = 
            new org.springframework.web.reactive.handler.SimpleUrlHandlerMapping();
        mapping.setOrder(10);
        mapping.setUrlMap(handlers);
        return mapping;
    }
    
    @Bean
    public org.springframework.web.reactive.socket.server.support.WebSocketHandlerAdapter 
            handlerAdapter() {
        return new org.springframework.web.reactive.socket.server.support.WebSocketHandlerAdapter();
    }
}

// Echo WebSocket Handler
@org.springframework.stereotype.Component
class EchoWebSocketHandler implements WebSocketHandler {
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        System.out.println("New WebSocket connection: " + session.getId());
        
        Flux<WebSocketMessage> responses = session.receive()
            .map(message -> {
                String received = message.getPayloadAsText();
                System.out.println("Received: " + received);
                return "Echo: " + received;
            })
            .map(session::textMessage)
            .doOnTerminate(() -> System.out.println("Connection closed: " + session.getId()));
        
        return session.send(responses);
    }
}

// Chat Room WebSocket Handler
@org.springframework.stereotype.Component
class ChatWebSocketHandler implements WebSocketHandler {
    
    // All connected sessions
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    
    // Message sink for broadcasting
    private final Sinks.Many<String> messageSink = 
        Sinks.many().multicast().onBackpressureBuffer();
    
    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String sessionId = session.getId();
        sessions.put(sessionId, session);
        
        System.out.println("User connected: " + sessionId);
        broadcast("System: User " + sessionId.substring(0, 8) + " joined");
        
        // Send messages from the sink to this client
        Flux<WebSocketMessage> outgoing = messageSink.asFlux()
            .map(session::textMessage);
        
        // Receive messages from this client
        Mono<Void> incoming = session.receive()
            .map(msg -> {
                String text = msg.getPayloadAsText();
                String formatted = sessionId.substring(0, 8) + ": " + text;
                broadcast(formatted);
                return formatted;
            })
            .doOnComplete(() -> {
                sessions.remove(sessionId);
                broadcast("System: User " + sessionId.substring(0, 8) + " left");
                System.out.println("User disconnected: " + sessionId);
            })
            .then();
        
        return Mono.when(session.send(outgoing), incoming);
    }
    
    private void broadcast(String message) {
        messageSink.tryEmitNext(message);
    }
}
```

---

## 8. R2DBC - Reactive Database

```java
import org.springframework.data.r2dbc.repository.Query;
import org.springframework.data.repository.reactive.*;
import org.springframework.data.annotation.*;
import org.springframework.data.relational.core.mapping.*;
import reactor.core.publisher.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.Transactional;

// Entity
@Table("users")
public class ReactiveUser {
    @Id
    private Long id;
    private String name;
    private String email;
    
    public ReactiveUser() {}
    public ReactiveUser(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name=" + name + "}";
    }
}

// Reactive Repository
public interface ReactiveUserRepository extends ReactiveCrudRepository<ReactiveUser, Long> {
    
    Flux<ReactiveUser> findByName(String name);
    Mono<ReactiveUser> findByEmail(String email);
    Flux<ReactiveUser> findByNameContainingIgnoreCase(String keyword);
    
    @Query("SELECT * FROM users WHERE email LIKE :pattern")
    Flux<ReactiveUser> findByEmailPattern(String pattern);
    
    @Query("SELECT COUNT(*) FROM users WHERE name = :name")
    Mono<Long> countByName(String name);
}

// Service
@Service
@Transactional
class ReactiveUserService {
    
    private final ReactiveUserRepository userRepository;
    
    ReactiveUserService(ReactiveUserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public Mono<ReactiveUser> create(String name, String email) {
        return userRepository.save(new ReactiveUser(name, email));
    }
    
    public Flux<ReactiveUser> findAll() {
        return userRepository.findAll();
    }
    
    public Mono<ReactiveUser> findById(Long id) {
        return userRepository.findById(id)
            .switchIfEmpty(Mono.error(new RuntimeException("User not found: " + id)));
    }
    
    public Mono<ReactiveUser> update(Long id, String newName) {
        return findById(id)
            .doOnNext(user -> user.setName(newName))
            .flatMap(userRepository::save);
    }
    
    public Mono<Void> delete(Long id) {
        return findById(id)
            .flatMap(user -> userRepository.deleteById(id));
    }
    
    // Transactional: both operations must succeed
    public Mono<ReactiveUser> createUserAndLog(String name, String email) {
        return userRepository.save(new ReactiveUser(name, email))
            .flatMap(savedUser -> {
                // Log the creation
                System.out.println("Created user: " + savedUser.getId());
                return Mono.just(savedUser);
            });
    }
}

/*
application.yml for R2DBC:
spring:
  r2dbc:
    url: r2dbc:h2:mem:///testdb
    username: sa
    password:
  sql:
    init:
      schema-locations: classpath:schema.sql
      mode: always

schema.sql:
CREATE TABLE IF NOT EXISTS users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);
*/
```

---

## สรุป Part 032

| หัวข้อ | Description |
|--------|-------------|
| Mono<T> | 0 or 1 reactive item |
| Flux<T> | 0 to N reactive items |
| @RestController | Works same as MVC, returns Mono/Flux |
| WebClient | Non-blocking HTTP client |
| SSE | Server push events (text/event-stream) |
| WebSocket | Bidirectional reactive communication |
| R2DBC | Reactive database access |

### WebFlux vs MVC: เมื่อไรใช้อะไร?
- **WebFlux**: I/O-heavy, high concurrency, microservices, streaming
- **MVC**: Traditional CRUD, simpler code, team familiar with blocking, JPA
- **Don't mix**: ไม่ควร block() ใน reactive code

---

**Part 033:** Microservices with Spring Boot
