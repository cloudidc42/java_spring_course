# Part 014: Lambda Expressions & Functional Programming
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Lambda Syntax](#lambda-syntax)
2. [Closure และ Effectively Final](#closure-และ-effectively-final)
3. [Method References (4 Types)](#method-references-4-types)
4. [Higher-Order Functions](#higher-order-functions)
5. [Currying และ Partial Application](#currying-และ-partial-application)
6. [Function Composition](#function-composition)
7. [โปรแกรมตัวอย่างจริง: Pipeline Builder](#โปรแกรมตัวอย่างจริง-pipeline-builder)

---

## Lambda Syntax

```java
import java.util.*;
import java.util.function.*;

public class LambdaSyntax {
    
    public static void main(String[] args) {
        // Anonymous class (before lambda)
        Comparator<String> oldStyle = new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return a.compareTo(b);
            }
        };
        
        // Lambda equivalent
        Comparator<String> byLength = (String a, String b) -> a.compareTo(b);
        
        // Type inference (compiler infers types)
        Comparator<String> inferred = (a, b) -> a.compareTo(b);
        
        // Single parameter (no parens needed)
        Predicate<String> isEmpty = s -> s.isEmpty();
        
        // No parameters
        Runnable r = () -> System.out.println("Running!");
        
        // Block body (multiple statements)
        Function<Integer, String> describe = n -> {
            if (n < 0) return "negative";
            if (n == 0) return "zero";
            return "positive";
        };
        
        // Examples
        List<String> words = new ArrayList<>(Arrays.asList("banana", "apple", "cherry", "date"));
        words.sort(inferred);
        System.out.println("Sorted: " + words);
        
        words.sort((a, b) -> Integer.compare(a.length(), b.length()));
        System.out.println("By length: " + words);
        
        System.out.println("isEmpty test: " + isEmpty.test(""));
        System.out.println("describe(42): " + describe.apply(42));
        System.out.println("describe(-5): " + describe.apply(-5));
        
        r.run();
        
        // Lambda in variable
        BiFunction<Integer, Integer, Integer> add = (x, y) -> x + y;
        BiFunction<Integer, Integer, Integer> mul = (x, y) -> x * y;
        System.out.println("add(3,4): " + add.apply(3, 4));
        System.out.println("mul(3,4): " + mul.apply(3, 4));
    }
}
```

---

## Closure และ Effectively Final

```java
import java.util.*;
import java.util.function.*;

public class ClosureDemo {
    
    public static void main(String[] args) {
        // Capturing local variable (must be effectively final)
        String greeting = "Hello";
        Function<String, String> greet = name -> greeting + ", " + name + "!";
        System.out.println(greet.apply("World"));
        
        // greeting = "Hi";  // COMPILE ERROR: greeting must be effectively final
        
        // Final variable explicitly
        final int multiplier = 3;
        Function<Integer, Integer> triple = n -> n * multiplier;
        System.out.println("Triple 7: " + triple.apply(7));
        
        // Capturing instance variable (OK - no restriction)
        Counter counter = new Counter();
        Runnable increment = () -> counter.count++;  // Instance field: OK
        increment.run();
        increment.run();
        System.out.println("Counter: " + counter.count);
        
        // Common pattern: capture array to mutate
        int[] total = {0};  // array is effectively final (reference), but content can change
        List<Integer> nums = List.of(1, 2, 3, 4, 5);
        nums.forEach(n -> total[0] += n);
        System.out.println("Total: " + total[0]);
        
        // Closure in loop (classic gotcha)
        List<Supplier<Integer>> suppliers = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            final int captured = i;  // must capture final copy
            suppliers.add(() -> captured);
        }
        System.out.print("Suppliers: ");
        suppliers.forEach(s -> System.out.print(s.get() + " "));
        System.out.println();
        
        // Thread-safety note: shared mutable state is dangerous
        // Better: use AtomicInteger for thread-safe counters
        java.util.concurrent.atomic.AtomicInteger atomicCount = 
            new java.util.concurrent.atomic.AtomicInteger(0);
        Runnable safeIncrement = atomicCount::incrementAndGet;
        safeIncrement.run();
        safeIncrement.run();
        System.out.println("AtomicCount: " + atomicCount.get());
    }
    
    static class Counter {
        int count = 0;
    }
}
```

---

## Method References (4 Types)

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class MethodReferences {
    
    static class StringUtils {
        static String reverse(String s) {
            return new StringBuilder(s).reverse().toString();
        }
        
        static boolean isPalindrome(String s) {
            String clean = s.toLowerCase().replaceAll("[^a-z0-9]", "");
            return clean.equals(reverse(clean));
        }
    }
    
    static class Printer {
        private String prefix;
        
        Printer(String prefix) { this.prefix = prefix; }
        
        void print(String text) {
            System.out.println(prefix + text);
        }
    }
    
    public static void main(String[] args) {
        // Type 1: Static method reference (Class::staticMethod)
        Function<String, String> reverse = StringUtils::reverse;
        Predicate<String> isPalin = StringUtils::isPalindrome;
        Function<String, Integer> toInt = Integer::parseInt;
        
        System.out.println("reverse('hello'): " + reverse.apply("hello"));
        System.out.println("isPalindrome('racecar'): " + isPalin.test("racecar"));
        System.out.println("parseInt('42'): " + toInt.apply("42"));
        
        // Type 2: Instance method of specific object (instance::method)
        Printer printer = new Printer(">> ");
        Consumer<String> print = printer::print;
        print.accept("Hello from method reference!");
        
        String prefix = "PREFIX: ";
        Function<String, String> addPrefix = prefix::concat;
        System.out.println(addPrefix.apply("World"));
        
        // Type 3: Instance method of arbitrary instance (Class::instanceMethod)
        // The first parameter becomes "this"
        Function<String, String> toUpper = String::toUpperCase;
        Predicate<String> isEmpty = String::isEmpty;
        BiPredicate<String, String> startsWith = String::startsWith;
        
        List<String> words = List.of("hello", "world", "java", "stream");
        List<String> upper = words.stream().map(String::toUpperCase).collect(Collectors.toList());
        System.out.println("Upper: " + upper);
        System.out.println("startsWith 'he': " + startsWith.test("hello", "he"));
        
        // Type 4: Constructor reference (Class::new)
        Supplier<ArrayList<String>> listFactory = ArrayList::new;
        Function<String, StringBuilder> sbFactory = StringBuilder::new;
        BiFunction<String, Integer, String[]> arrayFactory = 
            (s, n) -> new String[n];  // lambda (no direct constructor ref for this)
        
        ArrayList<String> newList = listFactory.get();
        newList.add("test");
        System.out.println("New list: " + newList);
        
        StringBuilder sb = sbFactory.apply("Initial");
        System.out.println("StringBuilder: " + sb);
        
        // Practical examples
        System.out.println("\n--- Practical Method References ---");
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Diana");
        
        names.stream()
            .map(String::toLowerCase)
            .sorted()
            .forEach(System.out::println);  // Type 2 (System.out is specific instance)
        
        // Collect using constructor reference
        Map<String, Integer> nameLengths = names.stream()
            .collect(Collectors.toMap(
                Function.identity(),   // identity: s -> s
                String::length         // Type 3
            ));
        System.out.println("Name lengths: " + nameLengths);
        
        // Summary of 4 types
        System.out.println("\n=== 4 Types Summary ===");
        System.out.println("1. Static:    Class::staticMethod  -> Math::sqrt");
        System.out.println("2. Bound:     obj::method          -> System.out::println");
        System.out.println("3. Unbound:   Class::instanceMethod -> String::toUpperCase");
        System.out.println("4. Constructor: Class::new         -> ArrayList::new");
    }
}
```

---

## Higher-Order Functions

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class HigherOrderFunctions {
    
    // Function that takes a function as parameter
    static <T, R> List<R> transform(List<T> list, Function<T, R> f) {
        return list.stream().map(f).collect(Collectors.toList());
    }
    
    // Function that returns a function
    static Function<Integer, Integer> multiplier(int factor) {
        return n -> n * factor;
    }
    
    static Predicate<String> hasMinLength(int min) {
        return s -> s.length() >= min;
    }
    
    // Memoization: cache results of expensive function
    static <T, R> Function<T, R> memoize(Function<T, R> fn) {
        Map<T, R> cache = new HashMap<>();
        return input -> cache.computeIfAbsent(input, fn);
    }
    
    // Retry: retry function on failure
    static <T> T retry(int times, Supplier<T> supplier) {
        int attempts = 0;
        Exception lastEx = null;
        while (attempts < times) {
            try {
                return supplier.get();
            } catch (Exception e) {
                lastEx = e;
                attempts++;
                System.out.println("Attempt " + attempts + " failed: " + e.getMessage());
            }
        }
        throw new RuntimeException("All " + times + " attempts failed", lastEx);
    }
    
    // Decorator: add behavior around a function
    static <T> Consumer<T> logged(String name, Consumer<T> fn) {
        return input -> {
            System.out.println("Calling " + name + " with: " + input);
            fn.accept(input);
            System.out.println("Done " + name);
        };
    }
    
    // Timing decorator
    static <T, R> Function<T, R> timed(String name, Function<T, R> fn) {
        return input -> {
            long start = System.nanoTime();
            R result = fn.apply(input);
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            System.out.printf("%s(%s) took %d ms%n", name, input, elapsed);
            return result;
        };
    }
    
    public static void main(String[] args) {
        // Higher-order functions
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);
        
        List<Integer> doubled = transform(numbers, n -> n * 2);
        List<String> asString = transform(numbers, n -> "num" + n);
        System.out.println("Doubled: " + doubled);
        System.out.println("As string: " + asString);
        
        // Functions that return functions
        Function<Integer, Integer> triple = multiplier(3);
        Function<Integer, Integer> tenTimes = multiplier(10);
        System.out.println("triple(5): " + triple.apply(5));
        System.out.println("tenTimes(5): " + tenTimes.apply(5));
        
        Predicate<String> minLen5 = hasMinLength(5);
        List<String> words = List.of("hi", "hello", "world", "Java");
        System.out.println("Min 5 chars: " + 
            words.stream().filter(minLen5).collect(Collectors.toList()));
        
        // Memoization
        Function<Integer, Long> fibonacci = memoize(n -> {
            if (n <= 1) return (long) n;
            // Note: self-reference in memoize is tricky - simplified version
            long a = 0, b = 1;
            for (int i = 2; i <= n; i++) {
                long c = a + b;
                a = b;
                b = c;
            }
            return b;
        });
        
        System.out.println("\nMemoized Fibonacci:");
        for (int i : new int[]{10, 20, 30, 10, 20}) {
            System.out.println("fib(" + i + ") = " + fibonacci.apply(i));
        }
        
        // Timed decorator
        Function<Integer, Long> timedFib = timed("fibonacci", fibonacci);
        timedFib.apply(50);
        
        // Logged decorator
        Consumer<String> loggedPrint = logged("printer", System.out::println);
        loggedPrint.accept("Hello decorated world!");
        
        // Retry
        System.out.println("\n--- Retry Demo ---");
        Random rng = new Random();
        int[] attempts = {0};
        try {
            int result = retry(3, () -> {
                attempts[0]++;
                if (attempts[0] < 3) throw new RuntimeException("Temporary failure");
                return 42;
            });
            System.out.println("Result: " + result);
        } catch (RuntimeException e) {
            System.out.println("Failed: " + e.getMessage());
        }
    }
}
```

---

## Currying และ Partial Application

```java
import java.util.function.*;

public class CurryingDemo {
    
    // Currying: transform f(a,b) to f(a)(b)
    static <A, B, C> Function<A, Function<B, C>> curry(BiFunction<A, B, C> biFunction) {
        return a -> b -> biFunction.apply(a, b);
    }
    
    // Uncurry: transform f(a)(b) to f(a,b)
    static <A, B, C> BiFunction<A, B, C> uncurry(Function<A, Function<B, C>> curried) {
        return (a, b) -> curried.apply(a).apply(b);
    }
    
    // Partial application: fix some arguments
    static <A, B, C> Function<B, C> partial(BiFunction<A, B, C> fn, A a) {
        return b -> fn.apply(a, b);
    }
    
    @FunctionalInterface
    interface TriFunction<A, B, C, D> {
        D apply(A a, B b, C c);
    }
    
    static <A, B, C, D> Function<A, Function<B, Function<C, D>>> 
    curry3(TriFunction<A, B, C, D> fn) {
        return a -> b -> c -> fn.apply(a, b, c);
    }
    
    public static void main(String[] args) {
        // Basic currying
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        BiFunction<String, String, String> concat = (a, b) -> a + b;
        
        Function<Integer, Function<Integer, Integer>> curriedAdd = curry(add);
        Function<String, Function<String, String>> curriedConcat = curry(concat);
        
        Function<Integer, Integer> add5 = curriedAdd.apply(5);
        Function<Integer, Integer> add10 = curriedAdd.apply(10);
        Function<String, String> greet = curriedConcat.apply("Hello, ");
        
        System.out.println("add5(3): " + add5.apply(3));
        System.out.println("add5(7): " + add5.apply(7));
        System.out.println("add10(3): " + add10.apply(3));
        System.out.println("greet('World'): " + greet.apply("World!"));
        
        // Partial application
        BiFunction<Double, Double, Double> power = Math::pow;
        Function<Double, Double> square = partial(power, 2.0);
        Function<Double, Double> cube = partial(power, 3.0);
        
        // Wait, this is actually base^exp, let's fix to exp^base
        BiFunction<Double, Double, Double> powerFlipped = (exp, base) -> Math.pow(base, exp);
        Function<Double, Double> squares = partial(powerFlipped, 2.0);
        Function<Double, Double> cubes = partial(powerFlipped, 3.0);
        
        System.out.println("\nsquare(5): " + squares.apply(5.0));
        System.out.println("cube(3): " + cubes.apply(3.0));
        
        // Partial with 3 arguments
        TriFunction<String, Integer, String, String> formatStr = 
            (prefix, num, suffix) -> prefix + num + suffix;
        
        Function<String, Function<Integer, Function<String, String>>> curried3 = 
            curry3(formatStr);
        
        Function<Integer, Function<String, String>> withBracket = curried3.apply("[");
        Function<String, String> bracketNum = withBracket.apply(42);
        
        System.out.println("Curried3: " + bracketNum.apply("]"));
        System.out.println("Reuse: " + withBracket.apply(100).apply("]"));
        
        // Practical: tax calculator
        TriFunction<Double, Double, Double, Double> calcTax = 
            (rate, discount, amount) -> amount * (1 - discount) * (1 + rate);
        
        var curriedTax = curry3(calcTax);
        
        // Fix tax rate at 7% VAT
        var withVAT = curriedTax.apply(0.07);
        
        // Fix discount at 10%
        var withVATAndDiscount = withVAT.apply(0.10);
        
        System.out.println("\nTax calculator:");
        double[] amounts = {1000.0, 2500.0, 5000.0};
        for (double amt : amounts) {
            System.out.printf("  Amount: %.0f, After 10%% discount + 7%% VAT: %.2f%n",
                amt, withVATAndDiscount.apply(amt));
        }
    }
}
```

---

## Function Composition

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class FunctionComposition {
    
    // Chain functions together (pipe)
    @SafeVarargs
    static <T> Function<T, T> pipe(Function<T, T>... fns) {
        return Arrays.stream(fns)
            .reduce(Function.identity(), Function::andThen);
    }
    
    // Compose: apply in reverse order
    @SafeVarargs
    static <T> Function<T, T> compose(Function<T, T>... fns) {
        return Arrays.stream(fns)
            .reduce(Function.identity(), Function::compose);
    }
    
    // Safe function: wrap in try-catch
    static <T, R> Function<T, Optional<R>> safe(Function<T, R> fn) {
        return input -> {
            try {
                return Optional.of(fn.apply(input));
            } catch (Exception e) {
                return Optional.empty();
            }
        };
    }
    
    public static void main(String[] args) {
        // String processing pipeline
        Function<String, String> trim = String::trim;
        Function<String, String> lower = String::toLowerCase;
        Function<String, String> removeSpaces = s -> s.replaceAll("\\s+", "_");
        Function<String, String> addPrefix = s -> "user_" + s;
        
        Function<String, String> normalize = pipe(trim, lower, removeSpaces, addPrefix);
        
        String[] inputs = {"  Alice Smith  ", " Bob JONES", "Charlie  Brown  "};
        for (String input : inputs) {
            System.out.printf("'%s' -> '%s'%n", input, normalize.apply(input));
        }
        
        // Math pipeline
        Function<Integer, Integer> doubleIt = n -> n * 2;
        Function<Integer, Integer> addTen = n -> n + 10;
        Function<Integer, Integer> square = n -> n * n;
        
        Function<Integer, Integer> pipeline = pipe(doubleIt, addTen, square);
        System.out.println("\nMath pipeline (double, +10, square):");
        System.out.println("5 -> " + pipeline.apply(5));  // (5*2+10)^2 = 400
        System.out.println("3 -> " + pipeline.apply(3));  // (3*2+10)^2 = 256
        
        // Predicate composition
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<Integer> isLt100 = n -> n < 100;
        
        Predicate<Integer> combined = isPositive.and(isEven).and(isLt100);
        
        List<Integer> nums = List.of(-5, 0, 2, 7, 42, 150, 99, 100);
        System.out.println("\nPositive, even, < 100:");
        System.out.println(nums.stream().filter(combined).collect(Collectors.toList()));
        
        // Safe function
        Function<String, Optional<Integer>> safeParseInt = safe(Integer::parseInt);
        
        String[] strs = {"42", "abc", "100", "xyz", "7"};
        System.out.println("\nSafe parseInt:");
        for (String s : strs) {
            safeParseInt.apply(s).ifPresentOrElse(
                n -> System.out.println("  '" + s + "' -> " + n),
                () -> System.out.println("  '" + s + "' -> failed")
            );
        }
        
        // Function<T,R>.andThen and compose
        Function<String, Integer> length = String::length;
        Function<Integer, Boolean> isEvenLen = n -> n % 2 == 0;
        
        Function<String, Boolean> hasEvenLength = length.andThen(isEvenLen);
        
        List<String> words = List.of("hi", "hello", "hey", "world");
        System.out.println("\nHas even length:");
        words.forEach(w -> System.out.println("  " + w + ": " + hasEvenLength.apply(w)));
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Pipeline Builder

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class DataPipelineBuilder<T> {
    
    private final List<Function<List<T>, List<T>>> stages = new ArrayList<>();
    private final String name;
    
    DataPipelineBuilder(String name) { this.name = name; }
    
    // Filter stage
    DataPipelineBuilder<T> filter(Predicate<T> predicate) {
        stages.add(list -> list.stream().filter(predicate).collect(Collectors.toList()));
        return this;
    }
    
    // Sort stage
    DataPipelineBuilder<T> sort(Comparator<T> comparator) {
        stages.add(list -> list.stream().sorted(comparator).collect(Collectors.toList()));
        return this;
    }
    
    // Limit stage
    DataPipelineBuilder<T> limit(int n) {
        stages.add(list -> list.stream().limit(n).collect(Collectors.toList()));
        return this;
    }
    
    // Custom stage
    DataPipelineBuilder<T> transform(Function<List<T>, List<T>> fn) {
        stages.add(fn);
        return this;
    }
    
    // Execute pipeline
    List<T> execute(List<T> input) {
        System.out.println("Pipeline: " + name);
        System.out.println("  Input: " + input.size() + " items");
        List<T> result = input;
        for (int i = 0; i < stages.size(); i++) {
            result = stages.get(i).apply(result);
            System.out.println("  Stage " + (i+1) + ": " + result.size() + " items");
        }
        return result;
    }
    
    record Product(String name, String category, double price, double rating, int stock) {}
    
    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop A", "Electronics", 35000, 4.5, 10),
            new Product("Laptop B", "Electronics", 28000, 4.2, 5),
            new Product("T-Shirt", "Clothing", 299, 4.0, 100),
            new Product("Jeans", "Clothing", 1500, 4.3, 30),
            new Product("Phone X", "Electronics", 25000, 4.7, 15),
            new Product("Keyboard", "Electronics", 1200, 4.1, 0),  // out of stock
            new Product("Sneakers", "Clothing", 2500, 4.6, 20),
            new Product("Tablet", "Electronics", 18000, 3.9, 8),
            new Product("Hoodie", "Clothing", 890, 4.4, 50),
            new Product("Monitor", "Electronics", 8000, 4.3, 3)
        );
        
        // Pipeline 1: Top electronics in stock, sorted by rating
        DataPipelineBuilder<Product> pipeline1 = new DataPipelineBuilder<>("Top Electronics");
        List<Product> top = pipeline1
            .filter(p -> p.category().equals("Electronics"))
            .filter(p -> p.stock() > 0)
            .sort(Comparator.comparingDouble(Product::rating).reversed())
            .limit(3)
            .execute(products);
        
        System.out.println("\nTop 3 Electronics (in stock, by rating):");
        top.forEach(p -> System.out.printf("  %s: rating=%.1f, price=%.0f%n",
            p.name(), p.rating(), p.price()));
        
        // Pipeline 2: Affordable items sorted by value (rating/price)
        DataPipelineBuilder<Product> pipeline2 = new DataPipelineBuilder<>("Best Value");
        List<Product> bestValue = pipeline2
            .filter(p -> p.price() < 5000)
            .filter(p -> p.stock() > 0)
            .sort(Comparator.comparingDouble(
                p -> -(p.rating() / (p.price() / 1000))))  // rating per 1000 baht
            .limit(5)
            .execute(products);
        
        System.out.println("\nBest Value (under 5000, by rating/price):");
        bestValue.forEach(p -> System.out.printf("  %s: %.2f value/1000 (%.0f @ %.1f*)%n",
            p.name(), p.rating() / (p.price() / 1000), p.price(), p.rating()));
        
        // Functional composition pattern
        Predicate<Product> inStock = p -> p.stock() > 0;
        Predicate<Product> highRating = p -> p.rating() >= 4.3;
        Predicate<Product> affordable = p -> p.price() < 3000;
        
        List<Product> recommended = products.stream()
            .filter(inStock.and(highRating).and(affordable))
            .sorted(Comparator.comparingDouble(Product::rating).reversed())
            .collect(Collectors.toList());
        
        System.out.println("\nRecommended (in stock, rating>=4.3, under 3000):");
        recommended.forEach(p -> System.out.printf("  %s: %.0f (%.1f*)%n",
            p.name(), p.price(), p.rating()));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Lambda syntax (all forms)  
✅ Closure และ effectively final  
✅ Method references (4 types: static, bound, unbound, constructor)  
✅ Higher-order functions (memoize, retry, timed decorator)  
✅ Currying  
✅ Partial application  
✅ Function composition (andThen, compose, pipe)  
✅ Safe function wrapper  
✅ Complete Pipeline Builder  

---

## ขั้นตอนต่อไป

**Part 015:** Java I/O & NIO.2  
- File operations (Files, Path, Paths)  
- Reading/writing files (BufferedReader, BufferedWriter)  
- Streams I/O  
- File watching (WatchService)  
- Serialization  

---

*Part 014 | Java & Spring Boot Course | สร้างโดย Claude Code*
