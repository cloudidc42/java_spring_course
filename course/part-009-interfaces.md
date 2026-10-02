# Part 009: Interfaces & Abstract Classes (Advanced)
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Interface คืออะไร?](#interface-คืออะไร)
2. [การสร้างและใช้ Interface](#การสร้างและใช้-interface)
3. [Default Methods](#default-methods)
4. [Static Methods ใน Interface](#static-methods-ใน-interface)
5. [Functional Interface](#functional-interface)
6. [Multiple Inheritance with Interfaces](#multiple-inheritance-with-interfaces)
7. [Sealed Classes (Java 17+)](#sealed-classes-java-17)
8. [Interface vs Abstract Class](#interface-vs-abstract-class)
9. [Design Patterns ด้วย Interfaces](#design-patterns-ด้วย-interfaces)
10. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)

---

## Interface คืออะไร?

Interface คือ contract ที่กำหนดว่า class ต้องมี method อะไรบ้าง

```
Interface = Contract (สัญญา)
- บอกว่า "ต้องทำอะไรได้"
- ไม่บอกว่า "ทำอย่างไร"
- Class ที่ implements ต้อง implement ทุก method
```

### Interface vs Abstract Class

```
Abstract Class:
- extends: 1 class เท่านั้น
- มี state (fields)
- มี constructor
- ใช้เมื่อต้องการ code sharing

Interface:
- implements: หลาย interface ได้
- ไม่มี instance state (Java 8+ มี static/default)
- ไม่มี constructor
- ใช้เมื่อต้องการ define behavior contract
```

---

## การสร้างและใช้ Interface

### Interface พื้นฐาน

```java
// Interface: Printable
public interface Printable {
    // Abstract methods (public abstract โดยอัตโนมัติ)
    void print();
    String getContent();
    
    // Constants (public static final โดยอัตโนมัติ)
    int MAX_PAGES = 1000;
}

// Interface: Saveable
public interface Saveable {
    void save(String filename);
    boolean load(String filename);
}

// Interface: Resizable
public interface Resizable {
    void resize(double factor);
    double getSize();
}

// Class implements multiple interfaces
public class Document implements Printable, Saveable {
    private String title;
    private String content;
    private int pageCount;
    
    public Document(String title, String content) {
        this.title = title;
        this.content = content;
        this.pageCount = (int) Math.ceil(content.length() / 500.0);
    }
    
    @Override
    public void print() {
        System.out.println("╔══════════════════════════════╗");
        System.out.println("║ Document: " + title);
        System.out.println("║ Pages: " + pageCount);
        System.out.println("╠══════════════════════════════╣");
        System.out.println(content);
        System.out.println("╚══════════════════════════════╝");
    }
    
    @Override
    public String getContent() {
        return content;
    }
    
    @Override
    public void save(String filename) {
        System.out.println("Saving '" + title + "' to " + filename);
        // actual file saving...
    }
    
    @Override
    public boolean load(String filename) {
        System.out.println("Loading from " + filename);
        return true;
    }
    
    public static void main(String[] args) {
        Document doc = new Document("Java Guide", "Java is a programming language...");
        doc.print();
        doc.save("java_guide.txt");
        
        // Polymorphism via interface
        Printable printable = doc;
        printable.print();
        
        // Check interface implementation
        System.out.println("Is Printable: " + (doc instanceof Printable));
        System.out.println("Is Saveable: " + (doc instanceof Saveable));
        System.out.println("Is Resizable: " + (doc instanceof Resizable));
    }
}
```

### Built-in Java Interfaces

```java
import java.util.*;

public class BuiltInInterfaces {
    
    // Comparable: เปรียบเทียบ/เรียงลำดับ
    static class Student implements Comparable<Student> {
        String name;
        double gpa;
        
        Student(String name, double gpa) {
            this.name = name;
            this.gpa = gpa;
        }
        
        @Override
        public int compareTo(Student other) {
            return Double.compare(other.gpa, this.gpa);  // descending
        }
        
        @Override
        public String toString() { return name + "(" + gpa + ")"; }
    }
    
    // Iterable: ใช้กับ for-each loop
    static class NumberRange implements Iterable<Integer> {
        private final int start;
        private final int end;
        
        NumberRange(int start, int end) {
            this.start = start;
            this.end = end;
        }
        
        @Override
        public Iterator<Integer> iterator() {
            return new Iterator<>() {
                private int current = start;
                
                @Override
                public boolean hasNext() { return current <= end; }
                
                @Override
                public Integer next() {
                    if (!hasNext()) throw new NoSuchElementException();
                    return current++;
                }
            };
        }
    }
    
    // Cloneable
    static class Config implements Cloneable {
        String host;
        int port;
        
        Config(String host, int port) {
            this.host = host;
            this.port = port;
        }
        
        @Override
        public Config clone() {
            try {
                return (Config) super.clone();
            } catch (CloneNotSupportedException e) {
                throw new RuntimeException(e);
            }
        }
        
        @Override
        public String toString() { return host + ":" + port; }
    }
    
    public static void main(String[] args) {
        // Comparable
        List<Student> students = new ArrayList<>(Arrays.asList(
            new Student("Alice", 3.8),
            new Student("Bob", 3.5),
            new Student("Charlie", 3.9),
            new Student("Diana", 3.7)
        ));
        
        Collections.sort(students);
        System.out.println("Sorted by GPA (desc): " + students);
        
        // Iterable
        NumberRange range = new NumberRange(1, 10);
        System.out.print("Range: ");
        for (int n : range) {
            System.out.print(n + " ");
        }
        System.out.println();
        
        // Cloneable
        Config original = new Config("localhost", 8080);
        Config copy = original.clone();
        copy.port = 9090;
        System.out.println("Original: " + original);
        System.out.println("Copy: " + copy);
    }
}
```

---

## Default Methods

```java
public interface Collection<E> {
    // Abstract methods
    void add(E element);
    boolean remove(E element);
    boolean contains(E element);
    int size();
    
    // Default methods (Java 8+)
    default boolean isEmpty() {
        return size() == 0;
    }
    
    default void addAll(E... elements) {
        for (E e : elements) {
            add(e);
        }
    }
    
    default void printAll() {
        System.out.println("Collection with " + size() + " elements");
    }
}

// การใช้ default method จริงๆ ใน Java
interface Greeting {
    String getGreeting(String name);
    
    // Default method
    default void greet(String name) {
        System.out.println(getGreeting(name));
    }
    
    default void greetAll(String... names) {
        for (String name : names) {
            greet(name);
        }
    }
    
    // Static factory method
    static Greeting formal() {
        return name -> "Dear " + name + ", I hope this message finds you well.";
    }
    
    static Greeting casual() {
        return name -> "Hey " + name + "!";
    }
}

// Implementation 1
class ThaiGreeting implements Greeting {
    @Override
    public String getGreeting(String name) {
        return "สวัสดี คุณ" + name + " ครับ/ค่ะ";
    }
}

// Implementation 2
class EnglishGreeting implements Greeting {
    @Override
    public String getGreeting(String name) {
        return "Hello, " + name + "!";
    }
    
    // Override default method
    @Override
    public void greetAll(String... names) {
        System.out.println("Greeting " + names.length + " people:");
        for (String name : names) {
            greet(name);
        }
    }
}

// Default method conflict resolution
interface A {
    default String hello() { return "Hello from A"; }
}

interface B {
    default String hello() { return "Hello from B"; }
}

// class C implements A, B {
//     @Override
//     public String hello() {
//         return A.super.hello() + " and " + B.super.hello(); // resolve conflict
//     }
// }

class DefaultMethodDemo {
    public static void main(String[] args) {
        ThaiGreeting thai = new ThaiGreeting();
        thai.greet("สมชาย");
        thai.greetAll("Alice", "Bob", "Charlie");
        
        System.out.println();
        EnglishGreeting eng = new EnglishGreeting();
        eng.greetAll("Alice", "Bob", "Charlie");
        
        // Static factory
        Greeting formal = Greeting.formal();
        Greeting casual = Greeting.casual();
        formal.greet("Mr. Smith");
        casual.greet("Mike");
    }
}
```

---

## Static Methods ใน Interface

```java
public interface MathOperations {
    // Static utility methods ใน interface
    
    static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    static int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }
    
    static int[] range(int start, int end, int step) {
        int size = (end - start) / step;
        int[] arr = new int[size];
        for (int i = 0; i < size; i++) {
            arr[i] = start + i * step;
        }
        return arr;
    }
    
    static double clamp(double value, double min, double max) {
        return Math.max(min, Math.min(max, value));
    }
}

// Validator interface example
public interface Validator<T> {
    boolean validate(T value);
    
    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }
    
    default Validator<T> or(Validator<T> other) {
        return value -> this.validate(value) || other.validate(value);
    }
    
    default Validator<T> negate() {
        return value -> !this.validate(value);
    }
    
    static <T> Validator<T> of(Validator<T> validator) {
        return validator;
    }
}

class ValidatorDemo {
    public static void main(String[] args) {
        // Static methods
        System.out.println("isPrime(17): " + MathOperations.isPrime(17));
        System.out.println("gcd(48,18): " + MathOperations.gcd(48, 18));
        System.out.println("clamp(150, 0, 100): " + MathOperations.clamp(150, 0, 100));
        
        // Composable validators
        Validator<String> notNull = s -> s != null;
        Validator<String> notEmpty = s -> !s.isEmpty();
        Validator<String> minLength = s -> s.length() >= 8;
        Validator<String> hasUpperCase = s -> s.chars().anyMatch(Character::isUpperCase);
        Validator<String> hasDigit = s -> s.chars().anyMatch(Character::isDigit);
        
        Validator<String> passwordValidator = notNull
            .and(notEmpty)
            .and(minLength)
            .and(hasUpperCase)
            .and(hasDigit);
        
        String[] passwords = {"", "simple", "Simple1", "ValidPass1", "NoDigit!"};
        System.out.println("\nPassword validation:");
        for (String pwd : passwords) {
            System.out.printf("'%s': %s%n", pwd, 
                passwordValidator.validate(pwd) ? "✓ Valid" : "✗ Invalid");
        }
    }
}
```

---

## Functional Interface

```java
import java.util.function.*;
import java.util.List;

public class FunctionalInterfaces {
    
    // Custom functional interface
    @FunctionalInterface
    interface Transformer<T, R> {
        R transform(T input);
        
        default <V> Transformer<T, V> andThen(Transformer<R, V> after) {
            return input -> after.transform(this.transform(input));
        }
    }
    
    @FunctionalInterface
    interface TriFunction<A, B, C, R> {
        R apply(A a, B b, C c);
    }
    
    public static void main(String[] args) {
        // Built-in functional interfaces
        
        // Function<T, R>: T -> R
        Function<String, Integer> strToLen = String::length;
        Function<Integer, String> intToStr = i -> "Number: " + i;
        Function<String, String> transform = strToLen.andThen(intToStr);
        System.out.println(transform.apply("Hello"));  // Number: 5
        
        // Predicate<T>: T -> boolean
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<Integer> isPositiveEven = isPositive.and(isEven);
        
        System.out.println("isPositiveEven(4): " + isPositiveEven.test(4));  // true
        System.out.println("isPositiveEven(-2): " + isPositiveEven.test(-2)); // false
        
        // Consumer<T>: T -> void
        Consumer<String> printer = System.out::println;
        Consumer<String> upperPrinter = s -> System.out.println(s.toUpperCase());
        Consumer<String> both = printer.andThen(upperPrinter);
        both.accept("hello");
        
        // Supplier<T>: () -> T
        Supplier<String> greeting = () -> "Hello, World!";
        Supplier<Double> random = Math::random;
        System.out.println(greeting.get());
        System.out.printf("Random: %.4f%n", random.get());
        
        // BiFunction<T, U, R>: (T, U) -> R
        BiFunction<String, Integer, String> repeat = 
            (s, n) -> s.repeat(n);
        System.out.println(repeat.apply("Ha", 3));  // HaHaHa
        
        // UnaryOperator<T>: T -> T
        UnaryOperator<String> trim = String::trim;
        UnaryOperator<String> upper = String::toUpperCase;
        UnaryOperator<String> process = trim.andThen(upper);
        System.out.println(process.apply("  hello world  "));
        
        // BinaryOperator<T>: (T, T) -> T
        BinaryOperator<Integer> add = Integer::sum;
        BinaryOperator<Integer> max = Integer::max;
        System.out.println("add: " + add.apply(3, 7));
        System.out.println("max: " + max.apply(3, 7));
        
        // Custom functional interface
        Transformer<String, String[]> splitter = s -> s.split(",");
        Transformer<String[], List<String>> toList = List::of;
        // Composed (partial - needs adjustments for types)
        
        TriFunction<Integer, Integer, Integer, Integer> triAdd = 
            (a, b, c) -> a + b + c;
        System.out.println("TriAdd(1,2,3): " + triAdd.apply(1, 2, 3));
        
        // Method references
        System.out.println("\nMethod References:");
        Function<String, Integer> parse = Integer::parseInt;
        System.out.println("Parse '42': " + parse.apply("42"));
        
        List<String> names = List.of("Alice", "Bob", "Charlie");
        names.forEach(System.out::println);
    }
}
```

---

## Multiple Inheritance with Interfaces

```java
public class MultipleInheritance {
    
    // เรือสามารถว่ายน้ำและบรรทุกสินค้าได้
    interface Swimmable {
        void swim();
        default String getMovementType() { return "swimming"; }
    }
    
    interface Flyable {
        void fly();
        default String getMovementType() { return "flying"; }
    }
    
    interface Cargo {
        void loadCargo(String item);
        void unloadCargo();
        int getCargoWeight();
    }
    
    // Flying Boat implements ทั้ง Swimmable และ Flyable
    static class FlyingBoat implements Swimmable, Flyable, Cargo {
        private String name;
        private java.util.List<String> cargo = new java.util.ArrayList<>();
        
        FlyingBoat(String name) {
            this.name = name;
        }
        
        @Override
        public void swim() {
            System.out.println(name + " is sailing on water");
        }
        
        @Override
        public void fly() {
            System.out.println(name + " is flying through air");
        }
        
        // ต้อง override เพราะมีความขัดแย้ง
        @Override
        public String getMovementType() {
            return Swimmable.super.getMovementType() + " and " + 
                   Flyable.super.getMovementType();
        }
        
        @Override
        public void loadCargo(String item) {
            cargo.add(item);
            System.out.println("Loaded: " + item);
        }
        
        @Override
        public void unloadCargo() {
            System.out.println("Unloaded: " + cargo);
            cargo.clear();
        }
        
        @Override
        public int getCargoWeight() { return cargo.size() * 100; }
    }
    
    // Interface inheritance hierarchy
    interface Animal {
        String getName();
        void breathe();
    }
    
    interface Pet extends Animal {
        String getOwner();
        void play();
    }
    
    interface ServiceAnimal extends Animal {
        String getJob();
        void performDuty();
    }
    
    // ServicePet implements both hierarchies
    static class ServiceDog implements Pet, ServiceAnimal {
        private String name, owner, job;
        
        ServiceDog(String name, String owner, String job) {
            this.name = name;
            this.owner = owner;
            this.job = job;
        }
        
        @Override public String getName() { return name; }
        @Override public void breathe() { System.out.println(name + " breathes"); }
        @Override public String getOwner() { return owner; }
        @Override public void play() { System.out.println(name + " plays fetch!"); }
        @Override public String getJob() { return job; }
        @Override public void performDuty() {
            System.out.println(name + " performs " + job + " duties");
        }
    }
    
    public static void main(String[] args) {
        FlyingBoat boat = new FlyingBoat("Seaplane Alpha");
        boat.swim();
        boat.fly();
        System.out.println("Movement: " + boat.getMovementType());
        boat.loadCargo("Medical Supplies");
        boat.loadCargo("Food Packages");
        System.out.println("Cargo weight: " + boat.getCargoWeight() + "kg");
        boat.unloadCargo();
        
        System.out.println();
        ServiceDog dog = new ServiceDog("Max", "John", "Guide");
        dog.breathe();
        dog.play();
        dog.performDuty();
        System.out.println("Owner: " + dog.getOwner());
        
        // Polymorphism
        Pet pet = dog;
        ServiceAnimal service = dog;
        Animal animal = dog;
        System.out.println("\nAll are " + animal.getName());
    }
}
```

---

## Sealed Classes (Java 17+)

```java
public class SealedClassesDemo {
    
    // Sealed interface: จำกัด implementation
    sealed interface Result<T> permits Success, Failure, Loading {}
    
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error, Throwable cause) implements Result<T> {
        Failure(String error) { this(error, null); }
    }
    record Loading<T>() implements Result<T> {}
    
    // Process Result
    static <T> String processResult(Result<T> result) {
        return switch (result) {
            case Success<T> s -> "Success: " + s.value();
            case Failure<T> f -> "Error: " + f.error() + 
                (f.cause() != null ? " (" + f.cause().getMessage() + ")" : "");
            case Loading<T> l -> "Loading...";
        };
    }
    
    // Sealed class hierarchy for Shape
    sealed abstract class Shape permits Circle, Rectangle, Triangle {
        abstract double area();
        abstract double perimeter();
    }
    
    final class Circle extends Shape {
        final double radius;
        Circle(double radius) { this.radius = radius; }
        @Override public double area() { return Math.PI * radius * radius; }
        @Override public double perimeter() { return 2 * Math.PI * radius; }
    }
    
    final class Rectangle extends Shape {
        final double width, height;
        Rectangle(double w, double h) { width = w; height = h; }
        @Override public double area() { return width * height; }
        @Override public double perimeter() { return 2 * (width + height); }
    }
    
    non-sealed class Triangle extends Shape {
        final double a, b, c;
        Triangle(double a, double b, double c) { this.a = a; this.b = b; this.c = c; }
        @Override public double area() {
            double s = (a + b + c) / 2;
            return Math.sqrt(s * (s - a) * (s - b) * (s - c));
        }
        @Override public double perimeter() { return a + b + c; }
    }
    
    public static void main(String[] args) {
        // Result type usage
        Result<String> success = new Success<>("Data loaded successfully");
        Result<String> failure = new Failure<>("Network error");
        Result<String> loading = new Loading<>();
        
        System.out.println(processResult(success));
        System.out.println(processResult(failure));
        System.out.println(processResult(loading));
        
        // Simulate API call
        Result<Integer> apiResult = fetchData(true);
        System.out.println("\nAPI Result: " + processResult(apiResult));
    }
    
    static Result<Integer> fetchData(boolean success) {
        if (success) return new Success<>(42);
        return new Failure<>("Server error", new RuntimeException("Connection refused"));
    }
}
```

---

## Interface vs Abstract Class

```java
public class InterfaceVsAbstract {
    
    // Use Case 1: Interface - define capabilities
    interface Drawable {
        void draw();
        default void drawBorder() {
            System.out.println("Drawing default border");
        }
    }
    
    interface Resizable {
        void resize(double factor);
    }
    
    interface Serializable {
        String serialize();
        void deserialize(String data);
    }
    
    // Use Case 2: Abstract Class - share code
    static abstract class BaseShape implements Drawable {
        protected String color;
        protected double x, y;
        
        BaseShape(String color, double x, double y) {
            this.color = color;
            this.x = x;
            this.y = y;
        }
        
        // Shared implementation
        public void moveTo(double newX, double newY) {
            this.x = newX;
            this.y = newY;
        }
        
        public void setColor(String color) {
            this.color = color;
        }
        
        // Template method
        @Override
        public final void draw() {
            System.out.println("Drawing " + color + " " + getType() + 
                " at (" + x + "," + y + ")");
            drawDetail();
        }
        
        protected abstract String getType();
        protected abstract void drawDetail();
    }
    
    // Concrete class
    static class Rectangle extends BaseShape 
            implements Resizable, Serializable {
        private double width, height;
        
        Rectangle(String color, double x, double y, double w, double h) {
            super(color, x, y);
            this.width = w;
            this.height = h;
        }
        
        @Override
        protected String getType() { return "Rectangle"; }
        
        @Override
        protected void drawDetail() {
            System.out.printf("  Size: %.1f x %.1f%n", width, height);
        }
        
        @Override
        public void resize(double factor) {
            width *= factor;
            height *= factor;
        }
        
        @Override
        public String serialize() {
            return String.format("{type:rect,x:%s,y:%s,w:%s,h:%s,color:%s}",
                x, y, width, height, color);
        }
        
        @Override
        public void deserialize(String data) {
            // parse JSON-like data
            System.out.println("Deserialized from: " + data);
        }
    }
    
    public static void main(String[] args) {
        Rectangle rect = new Rectangle("Blue", 0, 0, 100, 50);
        rect.draw();
        rect.drawBorder();
        rect.resize(1.5);
        rect.draw();
        System.out.println("Serialized: " + rect.serialize());
        
        // Summary
        System.out.println("\nInterface vs Abstract Class:");
        System.out.println("Drawable is interface: " + Drawable.class.isInterface());
        System.out.println("BaseShape is abstract: " + 
            java.lang.reflect.Modifier.isAbstract(BaseShape.class.getModifiers()));
    }
}
```

---

## Design Patterns ด้วย Interfaces

### Strategy Pattern

```java
import java.util.Arrays;

public class StrategyPattern {
    
    // Strategy Interface
    interface SortStrategy {
        void sort(int[] arr);
        default String getName() { return this.getClass().getSimpleName(); }
    }
    
    // Concrete strategies
    static class BubbleSort implements SortStrategy {
        @Override
        public void sort(int[] arr) {
            int n = arr.length;
            for (int i = 0; i < n - 1; i++) {
                for (int j = 0; j < n - i - 1; j++) {
                    if (arr[j] > arr[j + 1]) {
                        int temp = arr[j];
                        arr[j] = arr[j + 1];
                        arr[j + 1] = temp;
                    }
                }
            }
        }
    }
    
    static class SelectionSort implements SortStrategy {
        @Override
        public void sort(int[] arr) {
            int n = arr.length;
            for (int i = 0; i < n - 1; i++) {
                int minIdx = i;
                for (int j = i + 1; j < n; j++) {
                    if (arr[j] < arr[minIdx]) minIdx = j;
                }
                int temp = arr[minIdx];
                arr[minIdx] = arr[i];
                arr[i] = temp;
            }
        }
    }
    
    static class JavaBuiltInSort implements SortStrategy {
        @Override
        public void sort(int[] arr) {
            Arrays.sort(arr);
        }
    }
    
    // Context
    static class Sorter {
        private SortStrategy strategy;
        
        Sorter(SortStrategy strategy) {
            this.strategy = strategy;
        }
        
        void setStrategy(SortStrategy strategy) {
            this.strategy = strategy;
        }
        
        int[] sort(int[] arr) {
            int[] copy = arr.clone();
            long start = System.nanoTime();
            strategy.sort(copy);
            long time = System.nanoTime() - start;
            System.out.printf("%-20s sorted in %d ns%n", 
                strategy.getName() + ":", time);
            return copy;
        }
    }
    
    public static void main(String[] args) {
        int[] data = {64, 34, 25, 12, 22, 11, 90, 55, 43, 77};
        System.out.println("Original: " + Arrays.toString(data));
        
        Sorter sorter = new Sorter(new BubbleSort());
        int[] result1 = sorter.sort(data);
        System.out.println("Bubble: " + Arrays.toString(result1));
        
        sorter.setStrategy(new SelectionSort());
        int[] result2 = sorter.sort(data);
        System.out.println("Selection: " + Arrays.toString(result2));
        
        sorter.setStrategy(new JavaBuiltInSort());
        int[] result3 = sorter.sort(data);
        System.out.println("Built-in: " + Arrays.toString(result3));
        
        // Lambda as strategy
        sorter.setStrategy(arr -> {
            // Insertion sort
            for (int i = 1; i < arr.length; i++) {
                int key = arr[i];
                int j = i - 1;
                while (j >= 0 && arr[j] > key) {
                    arr[j + 1] = arr[j];
                    j--;
                }
                arr[j + 1] = key;
            }
        });
        int[] result4 = sorter.sort(data);
        System.out.println("Insertion (lambda): " + Arrays.toString(result4));
    }
}
```

### Observer Pattern

```java
import java.util.ArrayList;
import java.util.List;

public class ObserverPattern {
    
    // Observer interface
    interface EventListener<T> {
        void onEvent(T event);
    }
    
    // Event
    record StockEvent(String symbol, double oldPrice, double newPrice) {
        double percentChange() { return (newPrice - oldPrice) / oldPrice * 100; }
    }
    
    // Subject
    static class StockMarket {
        private final List<EventListener<StockEvent>> listeners = new ArrayList<>();
        private final java.util.Map<String, Double> prices = new java.util.HashMap<>();
        
        void subscribe(EventListener<StockEvent> listener) {
            listeners.add(listener);
        }
        
        void unsubscribe(EventListener<StockEvent> listener) {
            listeners.remove(listener);
        }
        
        void updatePrice(String symbol, double newPrice) {
            double oldPrice = prices.getOrDefault(symbol, newPrice);
            prices.put(symbol, newPrice);
            
            StockEvent event = new StockEvent(symbol, oldPrice, newPrice);
            listeners.forEach(listener -> listener.onEvent(event));
        }
    }
    
    public static void main(String[] args) {
        StockMarket market = new StockMarket();
        
        // Observers
        EventListener<StockEvent> logger = event -> {
            System.out.printf("[LOG] %s: %.2f -> %.2f (%.1f%%)%n",
                event.symbol(), event.oldPrice(), event.newPrice(), 
                event.percentChange());
        };
        
        EventListener<StockEvent> alertSystem = event -> {
            if (Math.abs(event.percentChange()) > 5) {
                System.out.printf("[ALERT] %s moved %.1f%%!%n",
                    event.symbol(), event.percentChange());
            }
        };
        
        EventListener<StockEvent> portfolio = event -> {
            System.out.printf("[PORTFOLIO] Update for %s%n", event.symbol());
        };
        
        market.subscribe(logger);
        market.subscribe(alertSystem);
        market.subscribe(portfolio);
        
        System.out.println("=== Market Updates ===");
        market.updatePrice("AAPL", 150.00);
        market.updatePrice("AAPL", 158.00);  // +5.3% - alert!
        market.updatePrice("GOOGL", 2800.00);
        market.updatePrice("GOOGL", 2650.00); // -5.4% - alert!
        
        // Unsubscribe portfolio
        System.out.println("\n[Portfolio unsubscribed]");
        market.unsubscribe(portfolio);
        market.updatePrice("AAPL", 160.00);
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Payment System

```java
// เฉลย
public class PaymentSystem {
    
    interface PaymentProcessor {
        boolean processPayment(double amount);
        String getPaymentMethod();
        
        default String formatAmount(double amount) {
            return String.format("%.2f THB", amount);
        }
    }
    
    static class CreditCardProcessor implements PaymentProcessor {
        private String cardNumber;
        private double limit;
        private double used;
        
        CreditCardProcessor(String cardNumber, double limit) {
            this.cardNumber = cardNumber;
            this.limit = limit;
        }
        
        @Override
        public boolean processPayment(double amount) {
            if (used + amount > limit) {
                System.out.println("Credit limit exceeded");
                return false;
            }
            used += amount;
            System.out.printf("Credit card payment: %s%n", formatAmount(amount));
            return true;
        }
        
        @Override
        public String getPaymentMethod() { return "Credit Card (*" + cardNumber.substring(cardNumber.length()-4) + ")"; }
    }
    
    static class PromptPayProcessor implements PaymentProcessor {
        private String phone;
        
        PromptPayProcessor(String phone) { this.phone = phone; }
        
        @Override
        public boolean processPayment(double amount) {
            System.out.printf("PromptPay to %s: %s%n", phone, formatAmount(amount));
            return true;
        }
        
        @Override
        public String getPaymentMethod() { return "PromptPay (" + phone + ")"; }
    }
    
    public static void main(String[] args) {
        PaymentProcessor credit = new CreditCardProcessor("4111111111111234", 50000);
        PaymentProcessor promptpay = new PromptPayProcessor("0812345678");
        
        credit.processPayment(1500.00);
        credit.processPayment(49000.00);
        credit.processPayment(100.00);  // exceeded
        
        promptpay.processPayment(2500.00);
        
        System.out.println("\nMethods available:");
        for (PaymentProcessor p : new PaymentProcessor[]{credit, promptpay}) {
            System.out.println("  " + p.getPaymentMethod());
        }
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Interface พื้นฐาน  
✅ Built-in Interfaces (Comparable, Iterable, Cloneable)  
✅ Default Methods  
✅ Static Methods ใน Interface  
✅ Functional Interface  
✅ Multiple Interface Implementation  
✅ Sealed Classes (Java 17+)  
✅ Interface vs Abstract Class  
✅ Strategy Pattern  
✅ Observer Pattern  

---

## ขั้นตอนต่อไป

**Part 010:** Exception Handling  
เราจะเรียนรู้:
- try-catch-finally
- Checked vs Unchecked Exceptions
- Custom Exceptions
- try-with-resources
- Exception chaining

---

*Part 009 | Java & Spring Boot Course | สร้างโดย Claude Code*
