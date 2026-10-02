# Part 017: Modern Java Features (Java 8-21)
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Java 8 Features](#java-8-features)
2. [Java 9-11 Features](#java-9-11-features)
3. [Java 14-16 Features](#java-14-16-features)
4. [Java 17 Features](#java-17-features)
5. [Java 21 Features](#java-21-features)
6. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Java 8 Features

```java
import java.time.*;
import java.time.format.*;
import java.time.temporal.*;
import java.util.*;

public class Java8Features {
    
    public static void main(String[] args) {
        // 1. Default/Static Interface methods (covered in Part 009)
        
        // 2. Optional (covered in Part 013)
        Optional<String> opt = Optional.of("Hello");
        System.out.println(opt.map(String::toUpperCase).orElse("empty"));
        
        // 3. Lambda + Streams (covered in Parts 013-014)
        
        // 4. New Date/Time API (most important new feature)
        System.out.println("=== Java 8 Date/Time API ===");
        
        // LocalDate: date without time zone
        LocalDate today = LocalDate.now();
        LocalDate birthday = LocalDate.of(1990, 5, 15);
        LocalDate nextYear = today.plusYears(1);
        
        System.out.println("Today: " + today);
        System.out.println("Birthday: " + birthday);
        System.out.println("Age: " + ChronoUnit.YEARS.between(birthday, today));
        System.out.println("Next year: " + nextYear);
        System.out.println("Day of week: " + today.getDayOfWeek());
        System.out.println("Is leap year: " + today.isLeapYear());
        
        // LocalTime: time without date/zone
        LocalTime now = LocalTime.now();
        LocalTime noon = LocalTime.of(12, 0, 0);
        LocalTime addHours = now.plusHours(3);
        
        System.out.println("\nNow: " + now.truncatedTo(ChronoUnit.SECONDS));
        System.out.println("Noon: " + noon);
        System.out.println("In 3 hours: " + addHours.truncatedTo(ChronoUnit.SECONDS));
        System.out.println("Is before noon: " + now.isBefore(noon));
        
        // LocalDateTime: date + time
        LocalDateTime dateTime = LocalDateTime.now();
        LocalDateTime meeting = LocalDateTime.of(2025, 1, 15, 14, 30);
        
        System.out.println("\nDateTime: " + dateTime.truncatedTo(ChronoUnit.SECONDS));
        System.out.println("Meeting: " + meeting);
        
        // ZonedDateTime: with timezone
        ZonedDateTime bangkokTime = ZonedDateTime.now(ZoneId.of("Asia/Bangkok"));
        ZonedDateTime tokyoTime = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
        ZonedDateTime londonTime = ZonedDateTime.now(ZoneId.of("Europe/London"));
        
        DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
        System.out.println("\nBangkok: " + bangkokTime.format(fmt));
        System.out.println("Tokyo: " + tokyoTime.format(fmt));
        System.out.println("London: " + londonTime.format(fmt));
        
        // Period: date-based duration
        Period age = Period.between(birthday, today);
        System.out.printf("\nAge: %d years, %d months, %d days%n",
            age.getYears(), age.getMonths(), age.getDays());
        
        // Duration: time-based duration
        LocalDateTime start = LocalDateTime.of(2024, 1, 1, 9, 0);
        LocalDateTime end = LocalDateTime.of(2024, 1, 1, 17, 30);
        Duration workDay = Duration.between(start, end);
        System.out.println("Work duration: " + workDay.toHours() + "h " + 
            workDay.toMinutesPart() + "m");
        
        // Formatting and Parsing
        DateTimeFormatter dateFormatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        DateTimeFormatter timeFormatter = DateTimeFormatter.ofPattern("HH:mm:ss");
        DateTimeFormatter dtFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm");
        
        System.out.println("\nFormatted date: " + today.format(dateFormatter));
        System.out.println("Parsed: " + LocalDate.parse("25/12/2024", dateFormatter));
        
        // Instant: machine time
        Instant instant = Instant.now();
        System.out.println("\nInstant: " + instant);
        System.out.println("Epoch milli: " + instant.toEpochMilli());
    }
}
```

---

## Java 9-11 Features

```java
import java.util.*;
import java.util.stream.*;
import java.net.http.*;
import java.net.URI;

public class Java9to11Features {
    
    public static void main(String[] args) throws Exception {
        // Java 9: Collection Factory Methods
        System.out.println("=== Java 9: Collection Factory Methods ===");
        List<String> immutableList = List.of("a", "b", "c");
        Set<Integer> immutableSet = Set.of(1, 2, 3, 4, 5);
        Map<String, Integer> immutableMap = Map.of("one", 1, "two", 2, "three", 3);
        Map<String, Integer> mapLarger = Map.ofEntries(
            Map.entry("a", 1),
            Map.entry("b", 2),
            Map.entry("c", 3)
        );
        
        System.out.println("List: " + immutableList);
        System.out.println("Set: " + new TreeSet<>(immutableSet));
        System.out.println("Map: " + new TreeMap<>(immutableMap));
        
        // Java 9: Stream improvements
        System.out.println("\n=== Java 9: Stream.ofNullable ===");
        Stream.ofNullable("hello").forEach(System.out::println);
        Stream.ofNullable(null).forEach(System.out::println);  // empty stream
        
        System.out.println("takeWhile: " + 
            Stream.of(2, 4, 6, 7, 8).takeWhile(n -> n % 2 == 0).collect(Collectors.toList()));
        System.out.println("dropWhile: " + 
            Stream.of(2, 4, 6, 7, 8).dropWhile(n -> n % 2 == 0).collect(Collectors.toList()));
        
        // Java 9: Optional improvements
        System.out.println("\n=== Java 9: Optional.ifPresentOrElse ===");
        Optional.of("value").ifPresentOrElse(
            v -> System.out.println("Present: " + v),
            () -> System.out.println("Empty"));
        Optional.<String>empty().ifPresentOrElse(
            v -> System.out.println("Present: " + v),
            () -> System.out.println("Empty"));
        
        Optional<String> opt = Optional.<String>empty().or(() -> Optional.of("fallback"));
        System.out.println("or(): " + opt.get());
        
        // Java 10: var (local variable type inference)
        System.out.println("\n=== Java 10: var ===");
        var list = new ArrayList<String>();  // inferred as ArrayList<String>
        list.add("hello");
        list.add("world");
        
        var map = new HashMap<String, List<Integer>>();
        map.put("numbers", List.of(1, 2, 3));
        
        for (var entry : map.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }
        
        // Java 11: String methods
        System.out.println("\n=== Java 11: String Methods ===");
        System.out.println("'  hello  '.isBlank(): " + "  hello  ".isBlank());
        System.out.println("''.isBlank(): " + "".isBlank());
        System.out.println("'  hello  '.strip(): '" + "  hello  ".strip() + "'");
        System.out.println("'  hello  '.stripLeading(): '" + "  hello  ".stripLeading() + "'");
        System.out.println("'  hello  '.stripTrailing(): '" + "  hello  ".stripTrailing() + "'");
        System.out.println("'ha'.repeat(3): " + "ha".repeat(3));
        
        "line1\nline2\nline3".lines()
            .map(l -> "  [" + l + "]")
            .forEach(System.out::println);
        
        // Java 11: HTTP Client
        System.out.println("\n=== Java 11: HttpClient ===");
        // Just showing the API, not making actual requests
        HttpClient client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(java.time.Duration.ofSeconds(10))
            .build();
        
        System.out.println("HttpClient created: " + client.version());
        
        // Example of how to use (would need real URL):
        // HttpRequest request = HttpRequest.newBuilder()
        //     .uri(URI.create("https://api.example.com/data"))
        //     .GET()
        //     .build();
        // HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        // System.out.println(response.body());
    }
}
```

---

## Java 14-16 Features

```java
public class Java14to16Features {
    
    public static void main(String[] args) {
        // Java 14: switch expressions (stable)
        System.out.println("=== Java 14: Switch Expression ===");
        
        String[] days = {"MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY", "SATURDAY", "SUNDAY"};
        for (String day : days) {
            String type = switch (day) {
                case "MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY" -> "Weekday";
                case "SATURDAY", "SUNDAY" -> "Weekend";
                default -> "Unknown";
            };
            System.out.println(day + " -> " + type);
        }
        
        // switch with yield
        int points = 85;
        String grade = switch (points / 10) {
            case 10, 9 -> "A";
            case 8 -> "B";
            case 7 -> "C";
            case 6 -> {
                System.out.println("Just passed!");
                yield "D";
            }
            default -> "F";
        };
        System.out.println("Grade: " + grade);
        
        // Java 15: Text Blocks
        System.out.println("\n=== Java 15: Text Blocks ===");
        String json = """
                {
                    "name": "Alice",
                    "age": 30,
                    "city": "Bangkok"
                }
                """;
        System.out.println("JSON:");
        System.out.println(json);
        
        String html = """
                <html>
                    <body>
                        <h1>Hello World</h1>
                    </body>
                </html>
                """;
        System.out.println("HTML:");
        System.out.println(html);
        
        String sql = """
                SELECT u.name, u.email, COUNT(o.id) as order_count
                FROM users u
                LEFT JOIN orders o ON u.id = o.user_id
                WHERE u.active = true
                GROUP BY u.id, u.name, u.email
                HAVING COUNT(o.id) > 0
                ORDER BY order_count DESC
                """;
        System.out.println("SQL:");
        System.out.println(sql);
        
        // Java 16: Records
        System.out.println("=== Java 16: Records ===");
        record Point(double x, double y) {
            // Compact constructor with validation
            Point {
                if (Double.isNaN(x) || Double.isNaN(y)) {
                    throw new IllegalArgumentException("Coordinates must be valid numbers");
                }
            }
            
            // Additional methods
            double distanceTo(Point other) {
                return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
            }
            
            Point translate(double dx, double dy) {
                return new Point(x + dx, y + dy);
            }
            
            static Point origin() { return new Point(0, 0); }
        }
        
        record Person(String name, int age) implements Comparable<Person> {
            Person {
                if (name == null || name.isBlank()) throw new IllegalArgumentException("Name required");
                if (age < 0) throw new IllegalArgumentException("Age must be non-negative");
            }
            
            @Override
            public int compareTo(Person other) {
                return Integer.compare(this.age, other.age);
            }
        }
        
        Point origin = Point.origin();
        Point p = new Point(3, 4);
        System.out.println("Origin: " + origin);
        System.out.println("Point: " + p);
        System.out.println("Distance: " + origin.distanceTo(p));
        System.out.println("Translated: " + p.translate(1, 1));
        
        Person alice = new Person("Alice", 30);
        System.out.println("Person: " + alice);
        System.out.println("Alice's name: " + alice.name());
        
        // Java 16: instanceof pattern matching
        System.out.println("\n=== Java 16: Pattern Matching instanceof ===");
        Object[] objects = {"Hello", 42, 3.14, true, null, new int[]{1,2,3}};
        
        for (Object obj : objects) {
            if (obj instanceof String s) {
                System.out.println("String of length " + s.length() + ": " + s);
            } else if (obj instanceof Integer i) {
                System.out.println("Integer: " + i + " (squared: " + i*i + ")");
            } else if (obj instanceof Double d) {
                System.out.printf("Double: %.2f%n", d);
            } else if (obj instanceof Boolean b) {
                System.out.println("Boolean: " + b);
            } else if (obj == null) {
                System.out.println("null value");
            } else if (obj instanceof int[] arr) {
                System.out.println("int array of length " + arr.length);
            }
        }
    }
}
```

---

## Java 17 Features

```java
public class Java17Features {
    
    // Sealed Classes
    sealed interface Shape permits Circle, Rectangle, Triangle {}
    
    record Circle(double radius) implements Shape {
        double area() { return Math.PI * radius * radius; }
        double perimeter() { return 2 * Math.PI * radius; }
    }
    
    record Rectangle(double width, double height) implements Shape {
        double area() { return width * height; }
        double perimeter() { return 2 * (width + height); }
    }
    
    record Triangle(double a, double b, double c) implements Shape {
        double area() {
            double s = (a + b + c) / 2;
            return Math.sqrt(s * (s-a) * (s-b) * (s-c));
        }
        double perimeter() { return a + b + c; }
    }
    
    static String describe(Shape shape) {
        return switch (shape) {
            case Circle c -> String.format("Circle with radius %.1f, area=%.2f", c.radius(), c.area());
            case Rectangle r -> String.format("Rectangle %sx%s, area=%.2f", r.width(), r.height(), r.area());
            case Triangle t -> String.format("Triangle (%.1f,%.1f,%.1f), area=%.2f", t.a(), t.b(), t.c(), t.area());
        };  // exhaustive - no default needed!
    }
    
    // Sealed class with mixed types
    sealed interface Notification permits EmailNotification, SmsNotification, PushNotification {}
    
    record EmailNotification(String to, String subject, String body) implements Notification {}
    record SmsNotification(String phone, String message) implements Notification {}
    final class PushNotification implements Notification {
        final String deviceToken;
        final String title;
        final String body;
        final java.util.Map<String, String> data;
        
        PushNotification(String deviceToken, String title, String body, java.util.Map<String, String> data) {
            this.deviceToken = deviceToken;
            this.title = title;
            this.body = body;
            this.data = data;
        }
        
        @Override
        public String toString() {
            return "Push{to=" + deviceToken.substring(0, 8) + "..., title=" + title + "}";
        }
    }
    
    static void sendNotification(Notification notification) {
        switch (notification) {
            case EmailNotification e -> 
                System.out.printf("Email to %s: [%s] %s%n", e.to(), e.subject(), e.body());
            case SmsNotification s -> 
                System.out.printf("SMS to %s: %s%n", s.phone(), s.message());
            case PushNotification p -> 
                System.out.printf("Push to device ...%s: %s - %s%n", 
                    p.deviceToken.substring(p.deviceToken.length()-4), p.title, p.body);
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Java 17: Sealed Classes ===");
        
        Shape[] shapes = {
            new Circle(5),
            new Rectangle(4, 6),
            new Triangle(3, 4, 5)
        };
        
        for (Shape shape : shapes) {
            System.out.println(describe(shape));
        }
        
        System.out.println("\n=== Notification System ===");
        Notification[] notifications = {
            new EmailNotification("alice@example.com", "Order Confirmed", "Your order #123 is confirmed"),
            new SmsNotification("0812345678", "OTP: 456789"),
            new PushNotification("device_token_abc123", "New Message", "You have a new message", 
                java.util.Map.of("type", "chat", "id", "msg_456"))
        };
        
        for (Notification n : notifications) {
            sendNotification(n);
        }
    }
}
```

---

## Java 21 Features

```java
import java.util.*;
import java.util.concurrent.*;

public class Java21Features {
    
    // Pattern matching in switch (Java 21 stable)
    sealed interface Animal permits Dog, Cat, Bird {}
    record Dog(String name, String breed) implements Animal {}
    record Cat(String name, boolean isIndoor) implements Animal {}
    record Bird(String name, String species) implements Animal {}
    
    static String describeAnimal(Animal animal) {
        return switch (animal) {
            case Dog d when d.breed().equals("Husky") -> 
                d.name() + " is a beautiful Husky!";
            case Dog d -> 
                d.name() + " is a " + d.breed();
            case Cat c when c.isIndoor() -> 
                c.name() + " is an indoor cat";
            case Cat c -> 
                c.name() + " is an outdoor cat";
            case Bird b -> 
                b.name() + " is a " + b.species();
        };
    }
    
    // Record patterns (Java 21)
    sealed interface Expr permits Num, Add, Mul {}
    record Num(int value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
    
    static int evaluate(Expr expr) {
        return switch (expr) {
            case Num(int n) -> n;
            case Add(Expr l, Expr r) -> evaluate(l) + evaluate(r);
            case Mul(Expr l, Expr r) -> evaluate(l) * evaluate(r);
        };
    }
    
    static String format(Expr expr) {
        return switch (expr) {
            case Num(int n) -> String.valueOf(n);
            case Add(Expr l, Expr r) -> "(" + format(l) + " + " + format(r) + ")";
            case Mul(Expr l, Expr r) -> "(" + format(l) + " * " + format(r) + ")";
        };
    }
    
    public static void main(String[] args) throws Exception {
        System.out.println("=== Java 21: Pattern Matching Switch ===");
        
        Animal[] animals = {
            new Dog("Max", "Husky"),
            new Dog("Rex", "German Shepherd"),
            new Cat("Whiskers", true),
            new Cat("Shadow", false),
            new Bird("Tweety", "Canary")
        };
        
        for (Animal a : animals) {
            System.out.println("  " + describeAnimal(a));
        }
        
        System.out.println("\n=== Java 21: Record Patterns ===");
        // Expression: (2 + 3) * (4 + 5)
        Expr expr = new Mul(
            new Add(new Num(2), new Num(3)),
            new Add(new Num(4), new Num(5))
        );
        System.out.println("Expression: " + format(expr));
        System.out.println("Result: " + evaluate(expr));
        
        // Another: 1 + 2 * 3
        Expr expr2 = new Add(new Num(1), new Mul(new Num(2), new Num(3)));
        System.out.println("Expression: " + format(expr2));
        System.out.println("Result: " + evaluate(expr2));
        
        // Java 21: Virtual Threads (Project Loom)
        System.out.println("\n=== Java 21: Virtual Threads ===");
        
        // Traditional thread
        long startTraditional = System.currentTimeMillis();
        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            Thread t = new Thread(() -> {
                try { Thread.sleep(10); } 
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
            threads.add(t);
            t.start();
        }
        for (Thread t : threads) t.join();
        System.out.println("100 traditional threads: " + 
            (System.currentTimeMillis() - startTraditional) + "ms");
        
        // Virtual threads (much lighter, millions possible)
        long startVirtual = System.currentTimeMillis();
        List<Thread> virtualThreads = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            Thread vt = Thread.ofVirtual().start(() -> {
                try { Thread.sleep(10); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
            virtualThreads.add(vt);
        }
        for (Thread vt : virtualThreads) vt.join();
        System.out.println("1000 virtual threads: " + 
            (System.currentTimeMillis() - startVirtual) + "ms");
        
        // Virtual thread executor
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();
            for (int i = 0; i < 10; i++) {
                final int task = i;
                futures.add(executor.submit(() -> {
                    Thread.sleep(50);
                    return "Task " + task + " on " + Thread.currentThread();
                }));
            }
            for (Future<String> f : futures) {
                System.out.println("  " + f.get());
            }
        }
        
        // Java 21: Sequenced Collections
        System.out.println("\n=== Java 21: Sequenced Collections ===");
        SequencedCollection<String> seq = new ArrayList<>(List.of("A", "B", "C", "D"));
        System.out.println("First: " + seq.getFirst());
        System.out.println("Last: " + seq.getLast());
        seq.addFirst("Z");
        seq.addLast("Z");
        System.out.println("After: " + seq);
        System.out.println("Reversed: " + seq.reversed());
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Type-Safe DSL with Modern Java

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class TypeSafeDSL {
    
    // Result type (like Rust Result or Haskell Either)
    sealed interface Result<T> permits Result.Ok, Result.Err {
        record Ok<T>(T value) implements Result<T> {}
        record Err<T>(String error) implements Result<T> {}
        
        static <T> Result<T> ok(T value) { return new Ok<>(value); }
        static <T> Result<T> err(String error) { return new Err<>(error); }
        
        default boolean isOk() { return this instanceof Ok; }
        default boolean isErr() { return this instanceof Err; }
        
        default T getOrThrow() {
            return switch (this) {
                case Ok<T> ok -> ok.value();
                case Err<T> err -> throw new RuntimeException(err.error());
            };
        }
        
        default T getOrDefault(T defaultValue) {
            return switch (this) {
                case Ok<T> ok -> ok.value();
                case Err<T> err -> defaultValue;
            };
        }
        
        default <R> Result<R> map(Function<T, R> mapper) {
            return switch (this) {
                case Ok<T> ok -> {
                    try { yield Result.ok(mapper.apply(ok.value())); }
                    catch (Exception e) { yield Result.err(e.getMessage()); }
                }
                case Err<T> err -> Result.err(err.error());
            };
        }
        
        default <R> Result<R> flatMap(Function<T, Result<R>> mapper) {
            return switch (this) {
                case Ok<T> ok -> mapper.apply(ok.value());
                case Err<T> err -> Result.err(err.error());
            };
        }
        
        default void ifOk(Consumer<T> consumer) {
            if (this instanceof Ok<T> ok) consumer.accept(ok.value());
        }
        
        default void ifErr(Consumer<String> consumer) {
            if (this instanceof Err<T> err) consumer.accept(err.error());
        }
    }
    
    // Validation using Result
    record UserRegistration(String username, String email, String password) {}
    
    static Result<String> validateUsername(String username) {
        if (username == null || username.isBlank())
            return Result.err("Username is required");
        if (username.length() < 3)
            return Result.err("Username must be at least 3 characters");
        if (username.length() > 20)
            return Result.err("Username must not exceed 20 characters");
        if (!username.matches("[a-zA-Z0-9_]+"))
            return Result.err("Username can only contain letters, numbers, and underscores");
        return Result.ok(username.toLowerCase());
    }
    
    static Result<String> validateEmail(String email) {
        if (email == null || email.isBlank())
            return Result.err("Email is required");
        if (!email.contains("@") || !email.contains("."))
            return Result.err("Invalid email format");
        return Result.ok(email.toLowerCase());
    }
    
    static Result<String> validatePassword(String password) {
        if (password == null || password.length() < 8)
            return Result.err("Password must be at least 8 characters");
        if (!password.chars().anyMatch(Character::isUpperCase))
            return Result.err("Password must contain uppercase letter");
        if (!password.chars().anyMatch(Character::isDigit))
            return Result.err("Password must contain a digit");
        return Result.ok("***");  // don't return actual password
    }
    
    static Result<UserRegistration> validateRegistration(String username, String email, String password) {
        return validateUsername(username)
            .flatMap(u -> validateEmail(email)
                .flatMap(e -> validatePassword(password)
                    .map(p -> new UserRegistration(u, e, p))));
    }
    
    public static void main(String[] args) {
        System.out.println("=== Result Type DSL ===");
        
        String[][] testCases = {
            {"alice_123", "alice@example.com", "SecurePass1"},
            {"ab", "alice@example.com", "SecurePass1"},
            {"alice", "invalid-email", "SecurePass1"},
            {"alice", "alice@example.com", "weak"},
            {"alice", "alice@example.com", "nouppercase1"},
            {"valid_user", "user@test.com", "StrongPass9"}
        };
        
        for (String[] tc : testCases) {
            Result<UserRegistration> result = validateRegistration(tc[0], tc[1], tc[2]);
            switch (result) {
                case Result.Ok<UserRegistration> ok -> 
                    System.out.printf("✓ Registered: %s (%s)%n", ok.value().username(), ok.value().email());
                case Result.Err<UserRegistration> err ->
                    System.out.printf("✗ Failed [%s, %s, %s]: %s%n", tc[0], tc[1], tc[2], err.error());
            }
        }
        
        // Chain operations
        System.out.println("\n=== Chained Result Operations ===");
        Result<Integer> parsed = Result.ok("42")
            .map(Integer::parseInt)
            .map(n -> n * 2)
            .flatMap(n -> n > 100 ? Result.err("Too large") : Result.ok(n));
        
        System.out.println("Chained result: " + parsed.getOrDefault(-1));
        
        Result<Integer> failed = Result.ok("not-a-number")
            .map(Integer::parseInt);
        
        failed.ifErr(e -> System.out.println("Error: " + e));
        System.out.println("Default: " + failed.getOrDefault(0));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Java 8: Date/Time API (LocalDate, LocalTime, ZonedDateTime, Duration)  
✅ Java 9: Collection factory methods, Stream.ofNullable, Optional.or()  
✅ Java 10: var (local variable type inference)  
✅ Java 11: String methods (isBlank, strip, lines, repeat), HttpClient  
✅ Java 14: Switch expressions (stable)  
✅ Java 15: Text blocks  
✅ Java 16: Records, instanceof pattern matching  
✅ Java 17: Sealed classes  
✅ Java 21: Pattern matching switch, record patterns, virtual threads, SequencedCollections  
✅ Result type DSL using modern Java  

---

## ขั้นตอนต่อไป

**Part 018:** Design Patterns in Java  
- Creational: Singleton, Factory, Builder, Prototype  
- Structural: Adapter, Decorator, Facade, Proxy  
- Behavioral: Strategy, Observer, Command, Template Method  

---

*Part 017 | Java & Spring Boot Course | สร้างโดย Claude Code*
