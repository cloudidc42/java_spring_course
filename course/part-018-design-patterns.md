# Part 018: Design Patterns in Java
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Creational Patterns](#creational-patterns)
2. [Structural Patterns](#structural-patterns)
3. [Behavioral Patterns](#behavioral-patterns)
4. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Creational Patterns

### Singleton Pattern

```java
// Thread-safe Singleton (3 approaches)
public class SingletonPatterns {
    
    // Approach 1: Eager initialization
    static class EagerSingleton {
        private static final EagerSingleton INSTANCE = new EagerSingleton();
        private EagerSingleton() {}
        static EagerSingleton getInstance() { return INSTANCE; }
    }
    
    // Approach 2: Double-checked locking
    static class LazySingleton {
        private static volatile LazySingleton instance;
        private LazySingleton() {}
        
        static LazySingleton getInstance() {
            if (instance == null) {
                synchronized (LazySingleton.class) {
                    if (instance == null) {
                        instance = new LazySingleton();
                    }
                }
            }
            return instance;
        }
    }
    
    // Approach 3: Enum (best way - thread-safe, serialization-safe)
    enum EnumSingleton {
        INSTANCE;
        
        private int value = 0;
        
        void setValue(int v) { this.value = v; }
        int getValue() { return value; }
    }
    
    // Practical: Configuration singleton
    static class AppConfig {
        private static final AppConfig INSTANCE = new AppConfig();
        private final java.util.Map<String, String> settings = new java.util.HashMap<>();
        
        private AppConfig() {
            // Load defaults
            settings.put("server.port", "8080");
            settings.put("db.url", "jdbc:h2:mem:test");
            settings.put("app.name", "MyApp");
        }
        
        static AppConfig getInstance() { return INSTANCE; }
        
        String get(String key) { return settings.get(key); }
        void set(String key, String value) { settings.put(key, value); }
    }
    
    public static void main(String[] args) {
        // All approaches
        System.out.println("Eager: " + (EagerSingleton.getInstance() == EagerSingleton.getInstance()));
        System.out.println("Lazy: " + (LazySingleton.getInstance() == LazySingleton.getInstance()));
        System.out.println("Enum: " + (EnumSingleton.INSTANCE == EnumSingleton.INSTANCE));
        
        // Config
        AppConfig config = AppConfig.getInstance();
        System.out.println("Port: " + config.get("server.port"));
        config.set("server.port", "9090");
        System.out.println("Updated port: " + AppConfig.getInstance().get("server.port"));
        
        EnumSingleton.INSTANCE.setValue(42);
        System.out.println("Enum value: " + EnumSingleton.INSTANCE.getValue());
    }
}
```

### Factory Pattern

```java
import java.util.*;

public class FactoryPatterns {
    
    // Simple Factory
    interface Animal {
        void speak();
        String getName();
    }
    
    record Dog(String name) implements Animal {
        @Override public void speak() { System.out.println(name + ": Woof!"); }
        @Override public String getName() { return name; }
    }
    
    record Cat(String name) implements Animal {
        @Override public void speak() { System.out.println(name + ": Meow!"); }
        @Override public String getName() { return name; }
    }
    
    record Bird(String name) implements Animal {
        @Override public void speak() { System.out.println(name + ": Tweet!"); }
        @Override public String getName() { return name; }
    }
    
    // Simple Factory
    static class AnimalFactory {
        static Animal create(String type, String name) {
            return switch (type.toLowerCase()) {
                case "dog" -> new Dog(name);
                case "cat" -> new Cat(name);
                case "bird" -> new Bird(name);
                default -> throw new IllegalArgumentException("Unknown animal: " + type);
            };
        }
    }
    
    // Factory Method Pattern (extensible)
    interface NotificationFactory {
        Notification createNotification(String message);
        
        default void notify(String message) {
            Notification n = createNotification(message);
            n.send();
        }
    }
    
    interface Notification {
        void send();
    }
    
    static class EmailNotificationFactory implements NotificationFactory {
        private final String recipient;
        EmailNotificationFactory(String recipient) { this.recipient = recipient; }
        
        @Override
        public Notification createNotification(String message) {
            return () -> System.out.println("Email to " + recipient + ": " + message);
        }
    }
    
    static class SlackNotificationFactory implements NotificationFactory {
        private final String channel;
        SlackNotificationFactory(String channel) { this.channel = channel; }
        
        @Override
        public Notification createNotification(String message) {
            return () -> System.out.println("Slack #" + channel + ": " + message);
        }
    }
    
    // Abstract Factory (creates families of objects)
    interface UIFactory {
        Button createButton();
        TextField createTextField();
    }
    
    interface Button { void render(); void onClick(); }
    interface TextField { void render(); String getValue(); }
    
    static class MaterialUIFactory implements UIFactory {
        @Override
        public Button createButton() {
            return new Button() {
                @Override public void render() { System.out.println("[Material Button]"); }
                @Override public void onClick() { System.out.println("Material button clicked"); }
            };
        }
        
        @Override
        public TextField createTextField() {
            return new TextField() {
                private String value = "";
                @Override public void render() { System.out.println("[Material TextField]"); }
                @Override public String getValue() { return value; }
            };
        }
    }
    
    static class BootstrapUIFactory implements UIFactory {
        @Override
        public Button createButton() {
            return new Button() {
                @Override public void render() { System.out.println("[Bootstrap Button]"); }
                @Override public void onClick() { System.out.println("Bootstrap button clicked"); }
            };
        }
        
        @Override
        public TextField createTextField() {
            return new TextField() {
                private String value = "";
                @Override public void render() { System.out.println("[Bootstrap TextField]"); }
                @Override public String getValue() { return value; }
            };
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Simple Factory ===");
        Animal[] animals = {
            AnimalFactory.create("dog", "Max"),
            AnimalFactory.create("cat", "Luna"),
            AnimalFactory.create("bird", "Tweety")
        };
        for (Animal a : animals) a.speak();
        
        System.out.println("\n=== Factory Method ===");
        List<NotificationFactory> factories = List.of(
            new EmailNotificationFactory("admin@example.com"),
            new SlackNotificationFactory("alerts")
        );
        factories.forEach(f -> f.notify("Server is down!"));
        
        System.out.println("\n=== Abstract Factory ===");
        UIFactory[] uiFactories = {new MaterialUIFactory(), new BootstrapUIFactory()};
        for (UIFactory factory : uiFactories) {
            Button btn = factory.createButton();
            TextField tf = factory.createTextField();
            btn.render();
            tf.render();
            btn.onClick();
        }
    }
}
```

### Builder Pattern

```java
import java.util.*;

public class BuilderPattern {
    
    // Complex object: HTTP Request
    static class HttpRequest {
        private final String method;
        private final String url;
        private final Map<String, String> headers;
        private final Map<String, String> params;
        private final String body;
        private final int timeoutMs;
        
        private HttpRequest(Builder builder) {
            this.method = builder.method;
            this.url = builder.url;
            this.headers = Collections.unmodifiableMap(new HashMap<>(builder.headers));
            this.params = Collections.unmodifiableMap(new HashMap<>(builder.params));
            this.body = builder.body;
            this.timeoutMs = builder.timeoutMs;
        }
        
        static Builder builder(String method, String url) {
            return new Builder(method, url);
        }
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append(method).append(" ").append(url);
            if (!params.isEmpty()) {
                sb.append("?");
                params.forEach((k,v) -> sb.append(k).append("=").append(v).append("&"));
                sb.deleteCharAt(sb.length() - 1);
            }
            sb.append("\n");
            headers.forEach((k,v) -> sb.append(k).append(": ").append(v).append("\n"));
            if (body != null) sb.append("\n").append(body);
            return sb.toString();
        }
        
        static class Builder {
            private final String method;
            private final String url;
            private final Map<String, String> headers = new LinkedHashMap<>();
            private final Map<String, String> params = new LinkedHashMap<>();
            private String body;
            private int timeoutMs = 5000;
            
            private Builder(String method, String url) {
                this.method = method;
                this.url = url;
            }
            
            Builder header(String key, String value) { headers.put(key, value); return this; }
            Builder contentType(String type) { return header("Content-Type", type); }
            Builder bearer(String token) { return header("Authorization", "Bearer " + token); }
            Builder param(String key, String value) { params.put(key, value); return this; }
            Builder body(String body) { this.body = body; return this; }
            Builder timeout(int ms) { this.timeoutMs = ms; return this; }
            
            HttpRequest build() {
                Objects.requireNonNull(method, "Method required");
                Objects.requireNonNull(url, "URL required");
                return new HttpRequest(this);
            }
        }
    }
    
    public static void main(String[] args) {
        // Builder: fluent API
        HttpRequest getRequest = HttpRequest.builder("GET", "https://api.example.com/users")
            .bearer("eyJhbGciOiJIUzI1NiJ9...")
            .param("page", "1")
            .param("size", "10")
            .param("sort", "name")
            .timeout(3000)
            .build();
        
        System.out.println("=== GET Request ===");
        System.out.println(getRequest);
        
        HttpRequest postRequest = HttpRequest.builder("POST", "https://api.example.com/users")
            .bearer("eyJhbGciOiJIUzI1NiJ9...")
            .contentType("application/json")
            .header("Accept", "application/json")
            .body("{\"name\":\"Alice\",\"email\":\"alice@example.com\"}")
            .build();
        
        System.out.println("=== POST Request ===");
        System.out.println(postRequest);
    }
}
```

---

## Structural Patterns

### Decorator Pattern

```java
import java.util.*;

public class DecoratorPattern {
    
    interface TextFormatter {
        String format(String text);
    }
    
    // Base
    static class PlainText implements TextFormatter {
        @Override
        public String format(String text) { return text; }
    }
    
    // Decorators
    static abstract class TextDecorator implements TextFormatter {
        protected final TextFormatter wrapped;
        TextDecorator(TextFormatter wrapped) { this.wrapped = wrapped; }
    }
    
    static class BoldDecorator extends TextDecorator {
        BoldDecorator(TextFormatter f) { super(f); }
        @Override public String format(String text) { return "<b>" + wrapped.format(text) + "</b>"; }
    }
    
    static class ItalicDecorator extends TextDecorator {
        ItalicDecorator(TextFormatter f) { super(f); }
        @Override public String format(String text) { return "<i>" + wrapped.format(text) + "</i>"; }
    }
    
    static class UppercaseDecorator extends TextDecorator {
        UppercaseDecorator(TextFormatter f) { super(f); }
        @Override public String format(String text) { return wrapped.format(text).toUpperCase(); }
    }
    
    static class TrimDecorator extends TextDecorator {
        TrimDecorator(TextFormatter f) { super(f); }
        @Override public String format(String text) { return wrapped.format(text.trim()); }
    }
    
    // Logging decorator
    interface DataService {
        String fetchData(String key);
        void saveData(String key, String data);
    }
    
    static class DatabaseService implements DataService {
        private Map<String, String> db = new HashMap<>();
        
        @Override
        public String fetchData(String key) { return db.getOrDefault(key, null); }
        
        @Override
        public void saveData(String key, String data) { db.put(key, data); }
    }
    
    static class LoggingDecorator implements DataService {
        private final DataService wrapped;
        
        LoggingDecorator(DataService service) { this.wrapped = service; }
        
        @Override
        public String fetchData(String key) {
            System.out.println("[LOG] Fetching: " + key);
            String result = wrapped.fetchData(key);
            System.out.println("[LOG] Result: " + result);
            return result;
        }
        
        @Override
        public void saveData(String key, String data) {
            System.out.println("[LOG] Saving: " + key + " = " + data);
            wrapped.saveData(key, data);
            System.out.println("[LOG] Saved successfully");
        }
    }
    
    static class CachingDecorator implements DataService {
        private final DataService wrapped;
        private final Map<String, String> cache = new HashMap<>();
        
        CachingDecorator(DataService service) { this.wrapped = service; }
        
        @Override
        public String fetchData(String key) {
            if (cache.containsKey(key)) {
                System.out.println("[CACHE] Hit for: " + key);
                return cache.get(key);
            }
            String result = wrapped.fetchData(key);
            if (result != null) cache.put(key, result);
            return result;
        }
        
        @Override
        public void saveData(String key, String data) {
            wrapped.saveData(key, data);
            cache.put(key, data);
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Text Formatter Decorators ===");
        TextFormatter plain = new PlainText();
        TextFormatter bold = new BoldDecorator(plain);
        TextFormatter boldItalic = new ItalicDecorator(bold);
        TextFormatter uppercase = new UppercaseDecorator(new TrimDecorator(bold));
        
        System.out.println(plain.format("Hello World"));
        System.out.println(bold.format("Hello World"));
        System.out.println(boldItalic.format("Hello World"));
        System.out.println(uppercase.format("  hello world  "));
        
        System.out.println("\n=== DataService Decorators ===");
        DataService service = new CachingDecorator(
            new LoggingDecorator(
                new DatabaseService()
            )
        );
        
        service.saveData("user:1", "Alice");
        service.fetchData("user:1");  // log + cache miss
        service.fetchData("user:1");  // cache hit
        service.fetchData("user:2");  // log + miss + null
    }
}
```

### Adapter & Facade Patterns

```java
public class AdapterAndFacade {
    
    // === ADAPTER ===
    // Old interface
    interface LegacyLogger {
        void logInfo(String msg);
        void logError(String msg);
    }
    
    // New interface we want to use
    interface Logger {
        void info(String message);
        void warn(String message);
        void error(String message);
    }
    
    // Adapter: makes LegacyLogger work as Logger
    static class LegacyLoggerAdapter implements Logger {
        private final LegacyLogger legacy;
        
        LegacyLoggerAdapter(LegacyLogger legacy) { this.legacy = legacy; }
        
        @Override public void info(String message) { legacy.logInfo("[INFO] " + message); }
        @Override public void warn(String message) { legacy.logInfo("[WARN] " + message); }
        @Override public void error(String message) { legacy.logError("[ERROR] " + message); }
    }
    
    // === FACADE ===
    // Subsystems
    static class VideoConverter { String convert(String file, String format) { return file + "." + format; } }
    static class AudioExtractor { String extract(String file) { return "audio_" + file; } }
    static class Subtitler { String addSubtitles(String file, String lang) { return file + "_" + lang; } }
    static class Uploader { String upload(String file, String dest) { return dest + "/" + file; } }
    
    // Facade: simple interface for complex process
    static class VideoProcessingFacade {
        private final VideoConverter converter = new VideoConverter();
        private final AudioExtractor extractor = new AudioExtractor();
        private final Subtitler subtitler = new Subtitler();
        private final Uploader uploader = new Uploader();
        
        String processVideo(String inputFile, String outputFormat, String subtitleLang, String destination) {
            System.out.println("Processing: " + inputFile);
            String converted = converter.convert(inputFile, outputFormat);
            String audio = extractor.extract(converted);
            String withSubs = subtitler.addSubtitles(converted, subtitleLang);
            String result = uploader.upload(withSubs, destination);
            System.out.println("Done: " + result);
            return result;
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Adapter Pattern ===");
        LegacyLogger legacyLogger = new LegacyLogger() {
            @Override public void logInfo(String msg) { System.out.println("[LEGACY INFO] " + msg); }
            @Override public void logError(String msg) { System.err.println("[LEGACY ERROR] " + msg); }
        };
        
        Logger logger = new LegacyLoggerAdapter(legacyLogger);
        logger.info("Server started");
        logger.warn("High memory usage");
        logger.error("Connection failed");
        
        System.out.println("\n=== Facade Pattern ===");
        VideoProcessingFacade facade = new VideoProcessingFacade();
        facade.processVideo("movie.avi", "mp4", "en", "s3://bucket/videos");
    }
}
```

---

## Behavioral Patterns

### Command Pattern

```java
import java.util.*;

public class CommandPattern {
    
    interface Command {
        void execute();
        void undo();
    }
    
    // Text editor commands
    static class TextEditor {
        private StringBuilder text = new StringBuilder();
        
        void insertText(int pos, String t) {
            text.insert(pos, t);
        }
        
        void deleteText(int start, int end) {
            text.delete(start, end);
        }
        
        String getText() { return text.toString(); }
    }
    
    static class InsertCommand implements Command {
        private final TextEditor editor;
        private final int position;
        private final String text;
        
        InsertCommand(TextEditor editor, int pos, String text) {
            this.editor = editor;
            this.position = pos;
            this.text = text;
        }
        
        @Override
        public void execute() {
            editor.insertText(position, text);
            System.out.println("Inserted '" + text + "' at " + position);
        }
        
        @Override
        public void undo() {
            editor.deleteText(position, position + text.length());
            System.out.println("Undone insert of '" + text + "'");
        }
    }
    
    // Command history (undo/redo)
    static class CommandHistory {
        private final Deque<Command> history = new ArrayDeque<>();
        private final Deque<Command> redoStack = new ArrayDeque<>();
        
        void execute(Command command) {
            command.execute();
            history.push(command);
            redoStack.clear();
        }
        
        void undo() {
            if (!history.isEmpty()) {
                Command cmd = history.pop();
                cmd.undo();
                redoStack.push(cmd);
            } else {
                System.out.println("Nothing to undo");
            }
        }
        
        void redo() {
            if (!redoStack.isEmpty()) {
                Command cmd = redoStack.pop();
                cmd.execute();
                history.push(cmd);
            } else {
                System.out.println("Nothing to redo");
            }
        }
    }
    
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        CommandHistory history = new CommandHistory();
        
        history.execute(new InsertCommand(editor, 0, "Hello"));
        System.out.println("Text: '" + editor.getText() + "'");
        
        history.execute(new InsertCommand(editor, 5, " World"));
        System.out.println("Text: '" + editor.getText() + "'");
        
        history.execute(new InsertCommand(editor, 11, "!"));
        System.out.println("Text: '" + editor.getText() + "'");
        
        System.out.println("\nUndo:");
        history.undo();
        System.out.println("Text: '" + editor.getText() + "'");
        
        history.undo();
        System.out.println("Text: '" + editor.getText() + "'");
        
        System.out.println("\nRedo:");
        history.redo();
        System.out.println("Text: '" + editor.getText() + "'");
    }
}
```

### Template Method Pattern

```java
public class TemplateMethodPattern {
    
    // Abstract template
    static abstract class DataProcessor {
        // Template method (defines the algorithm skeleton)
        final void process(String source) {
            System.out.println("=== Processing: " + source + " ===");
            String raw = readData(source);
            String cleaned = cleanData(raw);
            String processed = processData(cleaned);
            saveResult(processed);
            System.out.println("Done!\n");
        }
        
        // Steps to be implemented by subclasses
        protected abstract String readData(String source);
        protected abstract String processData(String data);
        
        // Default implementations (can be overridden)
        protected String cleanData(String data) {
            return data.trim().toLowerCase();
        }
        
        protected void saveResult(String result) {
            System.out.println("Saving: " + result.substring(0, Math.min(50, result.length())));
        }
    }
    
    static class CsvProcessor extends DataProcessor {
        @Override
        protected String readData(String source) {
            System.out.println("Reading CSV from: " + source);
            return "  Alice,30,Engineering  \n  Bob,25,Marketing  ";
        }
        
        @Override
        protected String cleanData(String data) {
            // Override to handle CSV specifically
            return data.trim().replace("  ", "");
        }
        
        @Override
        protected String processData(String data) {
            System.out.println("Processing CSV...");
            String[] lines = data.split("\n");
            StringBuilder result = new StringBuilder("Processed " + lines.length + " records:\n");
            for (String line : lines) {
                String[] parts = line.trim().split(",");
                if (parts.length >= 3) {
                    result.append("  ").append(parts[0].trim())
                          .append(" (").append(parts[2].trim()).append(")\n");
                }
            }
            return result.toString();
        }
    }
    
    static class JsonProcessor extends DataProcessor {
        @Override
        protected String readData(String source) {
            System.out.println("Reading JSON from: " + source);
            return "[{\"name\":\"Alice\"},{\"name\":\"Bob\"}]";
        }
        
        @Override
        protected String processData(String data) {
            System.out.println("Processing JSON...");
            // Simplified JSON processing
            int count = (data.split("name").length - 1);
            return "Processed " + count + " JSON objects";
        }
    }
    
    public static void main(String[] args) {
        DataProcessor csv = new CsvProcessor();
        DataProcessor json = new JsonProcessor();
        
        csv.process("data/employees.csv");
        json.process("api/users.json");
    }
}
```

---

## โปรแกรมตัวอย่างจริง: E-Commerce Pattern Integration

```java
import java.util.*;
import java.util.function.*;

public class ECommercePatterns {
    
    // Domain models
    record Product(String id, String name, double price) {}
    record CartItem(Product product, int quantity) {
        double subtotal() { return product.price() * quantity; }
    }
    
    // Strategy: Discount strategies
    interface DiscountStrategy {
        double apply(double total);
        String description();
        
        static DiscountStrategy noDiscount() {
            return new DiscountStrategy() {
                @Override public double apply(double total) { return total; }
                @Override public String description() { return "No discount"; }
            };
        }
        
        static DiscountStrategy percentage(double pct) {
            return new DiscountStrategy() {
                @Override public double apply(double total) { return total * (1 - pct/100); }
                @Override public String description() { return pct + "% off"; }
            };
        }
        
        static DiscountStrategy flat(double amount) {
            return new DiscountStrategy() {
                @Override public double apply(double total) { return Math.max(0, total - amount); }
                @Override public String description() { return "Flat " + amount + " off"; }
            };
        }
        
        static DiscountStrategy freeShipping() {
            return new DiscountStrategy() {
                private static final double SHIPPING = 50.0;
                @Override public double apply(double total) { return total; }
                @Override public String description() { return "Free shipping"; }
            };
        }
    }
    
    // Observer: Order events
    interface OrderObserver {
        void onOrderPlaced(String orderId, double total);
    }
    
    // Command: Cart operations
    interface CartCommand {
        void execute(List<CartItem> cart);
        void undo(List<CartItem> cart);
    }
    
    // Facade: Shopping Cart
    static class ShoppingCart {
        private final List<CartItem> items = new ArrayList<>();
        private DiscountStrategy discountStrategy = DiscountStrategy.noDiscount();
        private final List<OrderObserver> observers = new ArrayList<>();
        private final Deque<CartCommand> commandHistory = new ArrayDeque<>();
        private static int orderCounter = 1000;
        
        // Observer management
        void addObserver(OrderObserver o) { observers.add(o); }
        
        // Strategy
        void setDiscount(DiscountStrategy strategy) {
            this.discountStrategy = strategy;
        }
        
        // Command-based operations
        void addItem(Product product, int qty) {
            CartCommand cmd = new CartCommand() {
                @Override
                public void execute(List<CartItem> cart) {
                    cart.stream()
                        .filter(i -> i.product().id().equals(product.id()))
                        .findFirst()
                        .ifPresentOrElse(
                            existing -> {
                                cart.remove(existing);
                                cart.add(new CartItem(product, existing.quantity() + qty));
                            },
                            () -> cart.add(new CartItem(product, qty))
                        );
                    System.out.println("Added " + qty + "x " + product.name());
                }
                @Override
                public void undo(List<CartItem> cart) {
                    cart.removeIf(i -> i.product().id().equals(product.id()));
                    System.out.println("Undone: removed " + product.name());
                }
            };
            cmd.execute(items);
            commandHistory.push(cmd);
        }
        
        void undoLast() {
            if (!commandHistory.isEmpty()) {
                commandHistory.pop().undo(items);
            }
        }
        
        double subtotal() {
            return items.stream().mapToDouble(CartItem::subtotal).sum();
        }
        
        double total() {
            return discountStrategy.apply(subtotal());
        }
        
        String checkout() {
            if (items.isEmpty()) return "Cart is empty!";
            
            String orderId = "ORD-" + (++orderCounter);
            double total = total();
            
            // Notify observers
            observers.forEach(o -> o.onOrderPlaced(orderId, total));
            
            String receipt = buildReceipt(orderId, total);
            items.clear();
            commandHistory.clear();
            return receipt;
        }
        
        private String buildReceipt(String orderId, double total) {
            StringBuilder sb = new StringBuilder();
            sb.append("Order #").append(orderId).append("\n");
            sb.append("─────────────────────────────\n");
            items.forEach(item -> sb.append(String.format("  %-20s x%-2d %8.2f%n",
                item.product().name(), item.quantity(), item.subtotal())));
            sb.append("─────────────────────────────\n");
            sb.append(String.format("  Subtotal:            %8.2f%n", subtotal()));
            sb.append(String.format("  Discount (%s): %8.2f%n", 
                discountStrategy.description(), subtotal() - total));
            sb.append(String.format("  TOTAL:               %8.2f%n", total));
            return sb.toString();
        }
        
        void printCart() {
            System.out.println("Cart (" + items.size() + " items, discount: " + 
                discountStrategy.description() + "):");
            items.forEach(i -> System.out.printf("  %s x%d = %.2f%n",
                i.product().name(), i.quantity(), i.subtotal()));
            System.out.printf("  Subtotal: %.2f, Total: %.2f%n", subtotal(), total());
        }
    }
    
    public static void main(String[] args) {
        // Setup products
        Product laptop = new Product("P001", "Laptop", 35000);
        Product mouse = new Product("P002", "Mouse", 850);
        Product keyboard = new Product("P003", "Keyboard", 1500);
        Product monitor = new Product("P004", "Monitor", 8000);
        
        ShoppingCart cart = new ShoppingCart();
        
        // Add observers
        cart.addObserver((id, total) -> System.out.println("[EMAIL] Order " + id + " placed! Total: " + total));
        cart.addObserver((id, total) -> System.out.println("[SMS] Your order " + id + " is confirmed"));
        cart.addObserver((id, total) -> {
            if (total > 10000) System.out.println("[LOYALTY] Earned " + (int)(total * 0.05) + " points!");
        });
        
        // Add items
        cart.addItem(laptop, 1);
        cart.addItem(mouse, 2);
        cart.addItem(keyboard, 1);
        
        // Print cart
        cart.printCart();
        
        // Apply discount strategy
        cart.setDiscount(DiscountStrategy.percentage(10));
        System.out.println("\nAfter 10% discount:");
        cart.printCart();
        
        // Undo
        System.out.println("\nUndo last add:");
        cart.undoLast();
        cart.printCart();
        
        // Add monitor and checkout
        cart.addItem(monitor, 1);
        cart.setDiscount(DiscountStrategy.flat(500));
        System.out.println("\nFinal cart with flat 500 discount:");
        cart.printCart();
        
        System.out.println("\n=== CHECKOUT ===");
        System.out.println(cart.checkout());
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Singleton (eager, lazy, enum)  
✅ Factory Method & Abstract Factory  
✅ Builder Pattern (fluent API)  
✅ Decorator Pattern (stacking behaviors)  
✅ Adapter Pattern  
✅ Facade Pattern  
✅ Command Pattern (undo/redo)  
✅ Template Method Pattern  
✅ Observer Pattern  
✅ Strategy Pattern  
✅ Complete E-Commerce integration  

---

## ขั้นตอนต่อไป

**Part 019:** Testing with JUnit 5 & Mockito  
- JUnit 5 annotations and assertions  
- Parameterized tests  
- Mockito mocking and stubbing  
- Test-driven development (TDD)  

---

*Part 018 | Java & Spring Boot Course | สร้างโดย Claude Code*
