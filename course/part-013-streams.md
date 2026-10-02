# Part 013: Java Streams API
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Streams คืออะไร?](#streams-คืออะไร)
2. [Creating Streams](#creating-streams)
3. [Intermediate Operations](#intermediate-operations)
4. [Terminal Operations](#terminal-operations)
5. [Collectors](#collectors)
6. [FlatMap และ Optional](#flatmap-และ-optional)
7. [Parallel Streams](#parallel-streams)
8. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Streams คืออะไร?

Stream คือ pipeline สำหรับ process ข้อมูลแบบ functional และ lazy

```
Collection/Array/I-O
        ↓
    Source (Stream.of, list.stream(), etc.)
        ↓
    Intermediate ops (lazy, return Stream)
        ↓ filter → map → sorted → ...
    Terminal op (eager, returns result)
        ↓
    Result (List, count, sum, Optional, etc.)
```

```java
// Imperative style
List<String> result1 = new ArrayList<>();
for (String name : names) {
    if (name.startsWith("A")) {
        result1.add(name.toUpperCase());
    }
}

// Stream style (declarative)
List<String> result2 = names.stream()
    .filter(name -> name.startsWith("A"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

---

## Creating Streams

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class StreamCreation {
    
    public static void main(String[] args) {
        // From Collection
        List<String> list = List.of("a", "b", "c");
        Stream<String> fromList = list.stream();
        
        // From Array
        String[] arr = {"x", "y", "z"};
        Stream<String> fromArray = Arrays.stream(arr);
        Stream<String> fromArrayRange = Arrays.stream(arr, 1, 3);  // [1,3)
        
        // Stream.of
        Stream<Integer> ofStream = Stream.of(1, 2, 3, 4, 5);
        
        // Empty stream
        Stream<String> empty = Stream.empty();
        
        // Stream.builder
        Stream.Builder<String> builder = Stream.builder();
        builder.add("A");
        builder.add("B");
        builder.add("C");
        Stream<String> built = builder.build();
        
        // Stream.iterate (infinite, Java 8)
        Stream<Integer> evens = Stream.iterate(0, n -> n + 2).limit(10);
        System.out.println("Even numbers: " + evens.collect(Collectors.toList()));
        
        // Stream.iterate with predicate (Java 9)
        Stream<Integer> smallNums = Stream.iterate(1, n -> n <= 100, n -> n * 2);
        System.out.println("Powers of 2 <= 100: " + smallNums.collect(Collectors.toList()));
        
        // Stream.generate
        Stream<Double> randoms = Stream.generate(Math::random).limit(5);
        System.out.println("Randoms: " + randoms.map(d -> String.format("%.3f", d)).collect(Collectors.toList()));
        
        // Stream.concat
        Stream<String> s1 = Stream.of("a", "b");
        Stream<String> s2 = Stream.of("c", "d");
        Stream<String> combined = Stream.concat(s1, s2);
        System.out.println("Concatenated: " + combined.collect(Collectors.toList()));
        
        // IntStream, LongStream, DoubleStream
        IntStream range = IntStream.range(1, 11);           // [1,10]
        IntStream rangeClosed = IntStream.rangeClosed(1, 10); // [1,10]
        System.out.println("Range: " + range.boxed().collect(Collectors.toList()));
        
        // From string chars
        "Hello".chars().forEach(c -> System.out.print((char)c + " "));
        System.out.println();
    }
}
```

---

## Intermediate Operations

```java
import java.util.*;
import java.util.stream.*;

public class IntermediateOps {
    
    record Person(String name, int age, String city, double salary) {}
    
    public static void main(String[] args) {
        List<Person> people = List.of(
            new Person("Alice", 30, "Bangkok", 85000),
            new Person("Bob", 25, "Chiang Mai", 65000),
            new Person("Charlie", 35, "Bangkok", 95000),
            new Person("Diana", 28, "Phuket", 70000),
            new Person("Eve", 32, "Bangkok", 80000),
            new Person("Frank", 27, "Chiang Mai", 60000),
            new Person("Grace", 29, "Phuket", 75000),
            new Person("Henry", 33, "Bangkok", 90000)
        );
        
        System.out.println("=== filter ===");
        // filter: keep elements matching predicate
        List<Person> bangkokPeople = people.stream()
            .filter(p -> p.city().equals("Bangkok"))
            .collect(Collectors.toList());
        bangkokPeople.forEach(p -> System.out.println("  " + p.name()));
        
        System.out.println("=== map ===");
        // map: transform each element
        List<String> names = people.stream()
            .map(Person::name)
            .collect(Collectors.toList());
        System.out.println("  Names: " + names);
        
        List<Double> salaries = people.stream()
            .map(p -> p.salary() * 1.1)  // 10% raise
            .collect(Collectors.toList());
        System.out.println("  Salaries after raise: " + salaries.stream()
            .map(s -> String.format("%.0f", s))
            .collect(Collectors.joining(", ")));
        
        System.out.println("=== mapToInt/Long/Double ===");
        int totalAge = people.stream()
            .mapToInt(Person::age)
            .sum();
        double avgSalary = people.stream()
            .mapToDouble(Person::salary)
            .average()
            .orElse(0);
        System.out.printf("  Total age: %d, Avg salary: %.2f%n", totalAge, avgSalary);
        
        System.out.println("=== sorted ===");
        people.stream()
            .sorted(Comparator.comparingInt(Person::age))
            .forEach(p -> System.out.printf("  %s (age %d)%n", p.name(), p.age()));
        
        System.out.println("=== distinct ===");
        List<String> cities = people.stream()
            .map(Person::city)
            .distinct()
            .sorted()
            .collect(Collectors.toList());
        System.out.println("  Unique cities: " + cities);
        
        System.out.println("=== limit & skip ===");
        List<Person> top3 = people.stream()
            .sorted(Comparator.comparingDouble(Person::salary).reversed())
            .limit(3)
            .collect(Collectors.toList());
        System.out.println("  Top 3 earners: " + top3.stream().map(Person::name).collect(Collectors.toList()));
        
        List<Person> page2 = people.stream()
            .sorted(Comparator.comparing(Person::name))
            .skip(3)
            .limit(3)
            .collect(Collectors.toList());
        System.out.println("  Page 2: " + page2.stream().map(Person::name).collect(Collectors.toList()));
        
        System.out.println("=== peek (debug) ===");
        List<String> result = people.stream()
            .filter(p -> p.salary() > 75000)
            .peek(p -> System.out.println("  [debug] " + p.name()))
            .map(Person::name)
            .collect(Collectors.toList());
        System.out.println("  High earners: " + result);
        
        System.out.println("=== takeWhile & dropWhile (Java 9) ===");
        List<Integer> nums = List.of(2, 4, 6, 7, 8, 10);
        List<Integer> taken = nums.stream().takeWhile(n -> n % 2 == 0).collect(Collectors.toList());
        List<Integer> dropped = nums.stream().dropWhile(n -> n % 2 == 0).collect(Collectors.toList());
        System.out.println("  takeWhile even: " + taken);
        System.out.println("  dropWhile even: " + dropped);
    }
}
```

---

## Terminal Operations

```java
import java.util.*;
import java.util.stream.*;

public class TerminalOps {
    
    public static void main(String[] args) {
        List<Integer> nums = List.of(5, 3, 8, 1, 9, 2, 7, 4, 6, 10);
        
        // collect - most versatile
        List<Integer> sorted = nums.stream().sorted().collect(Collectors.toList());
        System.out.println("Sorted: " + sorted);
        
        // count
        long count = nums.stream().filter(n -> n > 5).count();
        System.out.println("Count > 5: " + count);
        
        // min/max
        Optional<Integer> min = nums.stream().min(Integer::compare);
        Optional<Integer> max = nums.stream().max(Integer::compare);
        System.out.println("Min: " + min.get() + ", Max: " + max.get());
        
        // sum/average (primitive streams)
        int sum = nums.stream().mapToInt(Integer::intValue).sum();
        OptionalDouble avg = nums.stream().mapToDouble(Integer::doubleValue).average();
        System.out.println("Sum: " + sum + ", Avg: " + avg.getAsDouble());
        
        // reduce
        Optional<Integer> product = nums.stream().reduce((a, b) -> a * b);
        int sumWithIdentity = nums.stream().reduce(0, Integer::sum);
        System.out.println("Product: " + product.get());
        System.out.println("Sum (reduce): " + sumWithIdentity);
        
        // findFirst / findAny
        Optional<Integer> firstBig = nums.stream().filter(n -> n > 7).findFirst();
        System.out.println("First > 7: " + firstBig.get());
        
        // anyMatch / allMatch / noneMatch
        boolean anyNegative = nums.stream().anyMatch(n -> n < 0);
        boolean allPositive = nums.stream().allMatch(n -> n > 0);
        boolean noneAbove100 = nums.stream().noneMatch(n -> n > 100);
        System.out.println("Any negative: " + anyNegative);
        System.out.println("All positive: " + allPositive);
        System.out.println("None above 100: " + noneAbove100);
        
        // forEach
        System.out.print("forEach: ");
        nums.stream().filter(n -> n % 2 == 0).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // toArray
        Integer[] arr = nums.stream().sorted().toArray(Integer[]::new);
        System.out.println("toArray: " + Arrays.toString(arr));
        
        // IntStream summaryStatistics
        IntSummaryStatistics stats = nums.stream()
            .mapToInt(Integer::intValue)
            .summaryStatistics();
        System.out.printf("Stats: count=%d, sum=%d, min=%d, max=%d, avg=%.1f%n",
            stats.getCount(), (long)stats.getSum(), stats.getMin(), 
            stats.getMax(), stats.getAverage());
    }
}
```

---

## Collectors

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class StreamCollectors {
    
    record Employee(String name, String dept, double salary, int year) {}
    
    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 90000, 2020),
            new Employee("Bob", "Engineering", 85000, 2019),
            new Employee("Charlie", "Marketing", 75000, 2021),
            new Employee("Diana", "HR", 70000, 2018),
            new Employee("Eve", "Engineering", 95000, 2020),
            new Employee("Frank", "Marketing", 80000, 2022),
            new Employee("Grace", "HR", 65000, 2019),
            new Employee("Henry", "Engineering", 88000, 2021)
        );
        
        System.out.println("=== toList / toSet / toUnmodifiable ===");
        List<String> names = employees.stream()
            .map(Employee::name)
            .collect(Collectors.toList());
        System.out.println("Names: " + names);
        
        Set<String> depts = employees.stream()
            .map(Employee::dept)
            .collect(Collectors.toSet());
        System.out.println("Departments: " + new TreeSet<>(depts));
        
        System.out.println("\n=== joining ===");
        String csv = employees.stream()
            .map(Employee::name)
            .collect(Collectors.joining(", "));
        System.out.println("CSV: " + csv);
        
        String table = employees.stream()
            .map(e -> e.name() + "(" + e.dept() + ")")
            .collect(Collectors.joining(" | ", "[ ", " ]"));
        System.out.println("Table: " + table);
        
        System.out.println("\n=== toMap ===");
        Map<String, Double> nameSalaryMap = employees.stream()
            .collect(Collectors.toMap(Employee::name, Employee::salary));
        System.out.println("Alice's salary: " + nameSalaryMap.get("Alice"));
        
        // Handle duplicates with merge function
        Map<String, Double> deptMaxSalary = employees.stream()
            .collect(Collectors.toMap(
                Employee::dept,
                Employee::salary,
                Double::max  // merge: keep maximum
            ));
        System.out.println("Max salary per dept: " + 
            new TreeMap<>(deptMaxSalary));
        
        System.out.println("\n=== groupingBy ===");
        Map<String, List<Employee>> byDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept));
        byDept.entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .forEach((k, v) -> {
                System.out.println("  " + k + ": " + 
                    v.stream().map(Employee::name).collect(Collectors.joining(", ")));
            });
        
        // groupingBy with downstream collector
        Map<String, Long> countByDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
        System.out.println("\nCount by dept: " + new TreeMap<>(countByDept));
        
        Map<String, Double> avgSalaryByDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept, 
                Collectors.averagingDouble(Employee::salary)));
        System.out.println("Avg salary by dept: " + 
            new TreeMap<>(avgSalaryByDept).entrySet().stream()
                .map(e -> e.getKey() + "=" + String.format("%.0f", e.getValue()))
                .collect(Collectors.joining(", ", "{", "}")));
        
        Map<String, Optional<Employee>> topEarnerByDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept,
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));
        System.out.println("Top earner by dept:");
        new TreeMap<>(topEarnerByDept).forEach((dept, emp) -> 
            System.out.printf("  %s: %s ($%.0f)%n", 
                dept, emp.get().name(), emp.get().salary()));
        
        System.out.println("\n=== partitioningBy ===");
        Map<Boolean, List<Employee>> seniorJunior = employees.stream()
            .collect(Collectors.partitioningBy(e -> e.year() <= 2020));
        System.out.println("Senior (2020 or earlier): " + 
            seniorJunior.get(true).stream().map(Employee::name).collect(Collectors.toList()));
        System.out.println("Junior (after 2020): " + 
            seniorJunior.get(false).stream().map(Employee::name).collect(Collectors.toList()));
        
        System.out.println("\n=== summingDouble / summarizingDouble ===");
        double totalSalary = employees.stream()
            .collect(Collectors.summingDouble(Employee::salary));
        System.out.printf("Total salary: %.0f%n", totalSalary);
        
        DoubleSummaryStatistics salaryStats = employees.stream()
            .collect(Collectors.summarizingDouble(Employee::salary));
        System.out.printf("Salary stats: count=%d, sum=%.0f, avg=%.0f, min=%.0f, max=%.0f%n",
            salaryStats.getCount(), salaryStats.getSum(), salaryStats.getAverage(),
            salaryStats.getMin(), salaryStats.getMax());
        
        System.out.println("\n=== teeing (Java 12) ===");
        record MinMax(Employee min, Employee max) {}
        MinMax minMax = employees.stream()
            .collect(Collectors.teeing(
                Collectors.minBy(Comparator.comparingDouble(Employee::salary)),
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
                (min, max) -> new MinMax(min.get(), max.get())
            ));
        System.out.printf("Lowest: %s ($%.0f)%n", minMax.min().name(), minMax.min().salary());
        System.out.printf("Highest: %s ($%.0f)%n", minMax.max().name(), minMax.max().salary());
    }
}
```

---

## FlatMap และ Optional

```java
import java.util.*;
import java.util.stream.*;

public class FlatMapAndOptional {
    
    record Order(String id, List<String> items, double total) {}
    
    public static void main(String[] args) {
        // flatMap: flatten nested streams
        List<List<Integer>> nested = List.of(
            List.of(1, 2, 3),
            List.of(4, 5),
            List.of(6, 7, 8, 9)
        );
        
        List<Integer> flat = nested.stream()
            .flatMap(Collection::stream)
            .collect(Collectors.toList());
        System.out.println("Flattened: " + flat);
        
        // flatMap with String splitting
        List<String> sentences = List.of(
            "Hello World Java",
            "Stream API is great",
            "Learning is fun"
        );
        
        List<String> words = sentences.stream()
            .flatMap(s -> Arrays.stream(s.split(" ")))
            .collect(Collectors.toList());
        System.out.println("Words: " + words);
        
        long uniqueWords = sentences.stream()
            .flatMap(s -> Arrays.stream(s.split(" ")))
            .map(String::toLowerCase)
            .distinct()
            .count();
        System.out.println("Unique words: " + uniqueWords);
        
        // flatMap with Orders
        List<Order> orders = List.of(
            new Order("O1", List.of("Apple", "Banana", "Cherry"), 250),
            new Order("O2", List.of("Durian", "Elderberry"), 180),
            new Order("O3", List.of("Fig", "Grape", "Apple"), 320)
        );
        
        List<String> allItems = orders.stream()
            .flatMap(o -> o.items().stream())
            .distinct()
            .sorted()
            .collect(Collectors.toList());
        System.out.println("All unique items: " + allItems);
        
        // Optional operations
        System.out.println("\n=== Optional ===");
        
        Optional<String> present = Optional.of("Hello");
        Optional<String> empty = Optional.empty();
        Optional<String> nullable = Optional.ofNullable(null);
        
        System.out.println("isPresent: " + present.isPresent());
        System.out.println("isEmpty: " + empty.isEmpty());
        
        // map
        Optional<Integer> length = present.map(String::length);
        System.out.println("Length: " + length.get());
        
        // flatMap (avoid Optional<Optional<T>>)
        Optional<Optional<String>> nested2 = Optional.of(Optional.of("nested"));
        Optional<String> flat2 = Optional.of(Optional.of("nested")).flatMap(o -> o);
        System.out.println("Flat optional: " + flat2.get());
        
        // orElse / orElseGet / orElseThrow
        String val1 = empty.orElse("default");
        String val2 = empty.orElseGet(() -> "computed default");
        System.out.println("orElse: " + val1);
        System.out.println("orElseGet: " + val2);
        
        try {
            empty.orElseThrow(() -> new RuntimeException("Value is empty"));
        } catch (RuntimeException e) {
            System.out.println("orElseThrow: " + e.getMessage());
        }
        
        // ifPresent / ifPresentOrElse (Java 9)
        present.ifPresent(v -> System.out.println("ifPresent: " + v));
        empty.ifPresentOrElse(
            v -> System.out.println("Has value: " + v),
            () -> System.out.println("ifPresentOrElse: empty")
        );
        
        // filter
        Optional<String> longStr = present.filter(s -> s.length() > 3);
        System.out.println("Filtered (len>3): " + longStr.isPresent());
        
        // or (Java 9): fallback to another Optional
        Optional<String> result = empty.or(() -> Optional.of("fallback"));
        System.out.println("or(): " + result.get());
        
        // stream() (Java 9)
        long count = present.stream().filter(s -> !s.isEmpty()).count();
        System.out.println("stream count: " + count);
    }
}
```

---

## Parallel Streams

```java
import java.util.*;
import java.util.stream.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ParallelStreams {
    
    public static void main(String[] args) throws InterruptedException {
        List<Integer> largeList = IntStream.rangeClosed(1, 10_000_000)
            .boxed()
            .collect(Collectors.toList());
        
        // Sequential vs Parallel
        long start = System.currentTimeMillis();
        long seqSum = largeList.stream()
            .mapToLong(Integer::longValue)
            .sum();
        long seqTime = System.currentTimeMillis() - start;
        
        start = System.currentTimeMillis();
        long parSum = largeList.parallelStream()
            .mapToLong(Integer::longValue)
            .sum();
        long parTime = System.currentTimeMillis() - start;
        
        System.out.printf("Sequential: %d ms (sum=%d)%n", seqTime, seqSum);
        System.out.printf("Parallel: %d ms (sum=%d)%n", parTime, parSum);
        
        // ⚠️ Parallel streams gotchas
        System.out.println("\n=== Gotchas ===");
        
        // 1. Non-thread-safe operations
        List<Integer> unsafeResult = new ArrayList<>();
        try {
            IntStream.range(0, 1000).parallel()
                .forEach(unsafeResult::add);  // UNSAFE!
        } catch (Exception e) {
            System.out.println("Error with non-thread-safe: " + e.getMessage());
        }
        // Use thread-safe collector instead:
        List<Integer> safeResult = IntStream.range(0, 1000).parallel()
            .boxed()
            .collect(Collectors.toList());  // Safe!
        System.out.println("Safe result size: " + safeResult.size());
        
        // 2. Order might not be preserved
        System.out.print("Parallel forEach (unordered): ");
        IntStream.range(1, 6).parallel().forEach(i -> System.out.print(i + " "));
        System.out.println();
        
        System.out.print("Parallel forEachOrdered: ");
        IntStream.range(1, 6).parallel().forEachOrdered(i -> System.out.print(i + " "));
        System.out.println();
        
        // 3. Good use: CPU-intensive work
        long expensiveSeq = measureTime(() -> 
            IntStream.rangeClosed(1, 1_000_000).filter(ParallelStreams::isPrime).count()
        );
        long expensivePar = measureTime(() ->
            IntStream.rangeClosed(1, 1_000_000).parallel().filter(ParallelStreams::isPrime).count()
        );
        System.out.printf("\nPrime count (sequential): %d ms%n", expensiveSeq);
        System.out.printf("Prime count (parallel): %d ms%n", expensivePar);
    }
    
    static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    static long measureTime(Runnable task) {
        long start = System.currentTimeMillis();
        task.run();
        return System.currentTimeMillis() - start;
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Sales Analytics System

```java
import java.util.*;
import java.util.stream.*;

public class SalesAnalytics {
    
    enum Region { NORTH, SOUTH, EAST, WEST, CENTRAL }
    
    record Sale(String id, String product, String salesperson, 
                Region region, double amount, int month, int year) {}
    
    static class Analytics {
        private final List<Sale> sales;
        
        Analytics(List<Sale> sales) { this.sales = sales; }
        
        // Total revenue
        double totalRevenue() {
            return sales.stream().mapToDouble(Sale::amount).sum();
        }
        
        // Revenue by region
        Map<Region, Double> revenueByRegion() {
            return sales.stream()
                .collect(Collectors.groupingBy(Sale::region,
                    Collectors.summingDouble(Sale::amount)));
        }
        
        // Top N salespeople
        List<Map.Entry<String, Double>> topSalespeople(int n) {
            return sales.stream()
                .collect(Collectors.groupingBy(Sale::salesperson,
                    Collectors.summingDouble(Sale::amount)))
                .entrySet().stream()
                .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                .limit(n)
                .collect(Collectors.toList());
        }
        
        // Monthly trend
        Map<Integer, Double> monthlyRevenue(int year) {
            return sales.stream()
                .filter(s -> s.year() == year)
                .collect(Collectors.groupingBy(Sale::month,
                    Collectors.summingDouble(Sale::amount)));
        }
        
        // Product performance
        Map<String, DoubleSummaryStatistics> productStats() {
            return sales.stream()
                .collect(Collectors.groupingBy(Sale::product,
                    Collectors.summarizingDouble(Sale::amount)));
        }
        
        // Best selling product per region
        Map<Region, String> bestProductPerRegion() {
            return sales.stream()
                .collect(Collectors.groupingBy(Sale::region,
                    Collectors.collectingAndThen(
                        Collectors.groupingBy(Sale::product,
                            Collectors.summingDouble(Sale::amount)),
                        map -> map.entrySet().stream()
                            .max(Map.Entry.comparingByValue())
                            .map(Map.Entry::getKey)
                            .orElse("N/A")
                    )));
        }
        
        // Sales above threshold
        List<Sale> highValueSales(double threshold) {
            return sales.stream()
                .filter(s -> s.amount() >= threshold)
                .sorted(Comparator.comparingDouble(Sale::amount).reversed())
                .collect(Collectors.toList());
        }
        
        void printReport() {
            System.out.println("╔══════════════════════════════════════════╗");
            System.out.printf("║  Total Revenue: $%,.0f              ║%n", totalRevenue());
            System.out.println("╠══════════════════════════════════════════╣");
            
            System.out.println("║  Revenue by Region:");
            revenueByRegion().entrySet().stream()
                .sorted(Map.Entry.<Region, Double>comparingByValue().reversed())
                .forEach(e -> System.out.printf("║    %-8s: $%,10.0f           ║%n", 
                    e.getKey(), e.getValue()));
            
            System.out.println("╠══════════════════════════════════════════╣");
            System.out.println("║  Top 3 Salespeople:");
            topSalespeople(3).forEach((e, idx) -> 
                System.out.printf("║    %-12s: $%,10.0f       ║%n", 
                    e.getKey(), e.getValue()));
            
            System.out.println("╠══════════════════════════════════════════╣");
            System.out.println("║  Best Product per Region:");
            bestProductPerRegion().entrySet().stream()
                .sorted(Map.Entry.comparingByKey())
                .forEach(e -> System.out.printf("║    %-8s: %-20s     ║%n",
                    e.getKey(), e.getValue()));
            
            System.out.println("╚══════════════════════════════════════════╝");
        }
    }
    
    // Generate sample data
    static List<Sale> generateData() {
        Random rng = new Random(42);
        String[] products = {"Laptop", "Phone", "Tablet", "Monitor", "Keyboard"};
        String[] people = {"Alice", "Bob", "Charlie", "Diana", "Eve"};
        Region[] regions = Region.values();
        
        List<Sale> sales = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            sales.add(new Sale(
                "S" + String.format("%03d", i),
                products[rng.nextInt(products.length)],
                people[rng.nextInt(people.length)],
                regions[rng.nextInt(regions.length)],
                5000 + rng.nextDouble() * 45000,
                1 + rng.nextInt(12),
                2024
            ));
        }
        return sales;
    }
    
    public static void main(String[] args) {
        List<Sale> salesData = generateData();
        Analytics analytics = new Analytics(salesData);
        
        analytics.printReport();
        
        System.out.println("\nHigh Value Sales (>40000):");
        analytics.highValueSales(40000).stream()
            .limit(5)
            .forEach(s -> System.out.printf("  %s: %s by %s in %s - $%.0f%n",
                s.id(), s.product(), s.salesperson(), s.region(), s.amount()));
        
        System.out.println("\nProduct Stats:");
        analytics.productStats().entrySet().stream()
            .sorted(Map.Entry.comparingByKey())
            .forEach(e -> {
                DoubleSummaryStatistics stats = e.getValue();
                System.out.printf("  %-8s: count=%d, total=$%,.0f, avg=$%,.0f%n",
                    e.getKey(), stats.getCount(), stats.getSum(), stats.getAverage());
            });
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Stream creation (from list, array, generate, iterate)  
✅ Intermediate ops (filter, map, sorted, distinct, limit, peek, takeWhile)  
✅ Terminal ops (collect, count, min/max, reduce, anyMatch, findFirst)  
✅ Collectors (toList, toMap, groupingBy, partitioningBy, joining, summarizing, teeing)  
✅ FlatMap for nested structures  
✅ Optional (of, empty, map, flatMap, orElse, filter)  
✅ Parallel Streams (benefits and pitfalls)  
✅ Complete Sales Analytics System  

---

## ขั้นตอนต่อไป

**Part 014:** Lambda Expressions & Functional Programming  
- Lambda syntax  
- Closure and effectively final  
- Method references (4 types)  
- Higher-order functions  
- Currying and partial application

---

*Part 013 | Java & Spring Boot Course | สร้างโดย Claude Code*
