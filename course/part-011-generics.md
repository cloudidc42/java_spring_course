# Part 011: Generics & Type Safety
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Generics คืออะไร?](#generics-คืออะไร)
2. [Generic Classes](#generic-classes)
3. [Generic Methods](#generic-methods)
4. [Bounded Type Parameters](#bounded-type-parameters)
5. [Wildcards](#wildcards)
6. [Generic Interfaces](#generic-interfaces)
7. [Type Erasure](#type-erasure)
8. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Generics คืออะไร?

Generics ทำให้ code ทำงานกับ type ต่างๆ ได้อย่าง type-safe โดยไม่ต้อง cast

```java
// Before Generics (Java 1.4 and older)
List listOld = new ArrayList();
listOld.add("Hello");
listOld.add(42);         // โปรแกรม compile ผ่าน แต่ runtime error!
String s = (String) listOld.get(1);  // ClassCastException!

// With Generics (Java 5+)
List<String> listNew = new ArrayList<>();
listNew.add("Hello");
// listNew.add(42);  // compile error - type safe!
String t = listNew.get(0);  // ไม่ต้อง cast
```

---

## Generic Classes

### Generic Class พื้นฐาน

```java
public class Box<T> {
    private T value;
    
    public Box(T value) {
        this.value = value;
    }
    
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
    
    @Override
    public String toString() { return "Box[" + value + "]"; }
    
    public static void main(String[] args) {
        Box<String> strBox = new Box<>("Hello");
        Box<Integer> intBox = new Box<>(42);
        Box<Double> dblBox = new Box<>(3.14);
        
        System.out.println(strBox);  // Box[Hello]
        System.out.println(intBox);  // Box[42]
        System.out.println(dblBox);  // Box[3.14]
        
        // Type inference with diamond operator
        Box<String> inferredBox = new Box<>("World");
    }
}

// Multiple type parameters
public class Pair<K, V> {
    private final K key;
    private final V value;
    
    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }
    
    public K getKey() { return key; }
    public V getValue() { return value; }
    
    public static <K, V> Pair<K, V> of(K key, V value) {
        return new Pair<>(key, value);
    }
    
    @Override
    public String toString() { return "(" + key + ", " + value + ")"; }
    
    public static void main(String[] args) {
        Pair<String, Integer> nameAge = Pair.of("Alice", 30);
        Pair<String, String> config = Pair.of("host", "localhost");
        Pair<Integer, List<String>> indexed = Pair.of(1, List.of("a", "b", "c"));
        
        System.out.println(nameAge);     // (Alice, 30)
        System.out.println(config);      // (host, localhost)
        System.out.println(indexed);     // (1, [a, b, c])
    }
}
```

### Generic Stack Implementation

```java
import java.util.*;

public class GenericStack<T> {
    private final Object[] elements;
    private int size;
    
    @SuppressWarnings("unchecked")
    public GenericStack(int capacity) {
        elements = new Object[capacity];
    }
    
    public void push(T element) {
        if (size == elements.length) {
            throw new IllegalStateException("Stack is full");
        }
        elements[size++] = element;
    }
    
    @SuppressWarnings("unchecked")
    public T pop() {
        if (isEmpty()) throw new NoSuchElementException("Stack is empty");
        T element = (T) elements[--size];
        elements[size] = null;  // help GC
        return element;
    }
    
    @SuppressWarnings("unchecked")
    public T peek() {
        if (isEmpty()) throw new NoSuchElementException("Stack is empty");
        return (T) elements[size - 1];
    }
    
    public boolean isEmpty() { return size == 0; }
    public int size() { return size; }
    
    @Override
    @SuppressWarnings("unchecked")
    public String toString() {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < size; i++) {
            if (i > 0) sb.append(", ");
            sb.append(elements[i]);
        }
        sb.append("] (top=").append(size > 0 ? elements[size-1] : "empty").append(")");
        return sb.toString();
    }
    
    public static void main(String[] args) {
        GenericStack<Integer> intStack = new GenericStack<>(10);
        intStack.push(1);
        intStack.push(2);
        intStack.push(3);
        System.out.println("Stack: " + intStack);
        System.out.println("Pop: " + intStack.pop());
        System.out.println("Stack: " + intStack);
        
        GenericStack<String> strStack = new GenericStack<>(5);
        strStack.push("A");
        strStack.push("B");
        strStack.push("C");
        System.out.println("String Stack: " + strStack);
    }
}
```

---

## Generic Methods

```java
import java.util.*;
import java.util.function.*;

public class GenericMethods {
    
    // Generic method
    public static <T> T getFirst(List<T> list) {
        if (list == null || list.isEmpty()) return null;
        return list.get(0);
    }
    
    public static <T> T getLast(List<T> list) {
        if (list == null || list.isEmpty()) return null;
        return list.get(list.size() - 1);
    }
    
    // Swap elements
    public static <T> void swap(T[] arr, int i, int j) {
        T temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    // Filter generic list
    public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (predicate.test(item)) result.add(item);
        }
        return result;
    }
    
    // Map generic list
    public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
        List<R> result = new ArrayList<>();
        for (T item : list) {
            result.add(mapper.apply(item));
        }
        return result;
    }
    
    // Reduce
    public static <T, R> R reduce(List<T> list, R identity, BiFunction<R, T, R> accumulator) {
        R result = identity;
        for (T item : list) {
            result = accumulator.apply(result, item);
        }
        return result;
    }
    
    // Convert array to list
    @SafeVarargs
    public static <T> List<T> listOf(T... elements) {
        return new ArrayList<>(Arrays.asList(elements));
    }
    
    // Zip two lists
    public static <A, B> List<Pair<A, B>> zip(List<A> listA, List<B> listB) {
        List<Pair<A, B>> result = new ArrayList<>();
        int size = Math.min(listA.size(), listB.size());
        for (int i = 0; i < size; i++) {
            result.add(new Pair<>(listA.get(i), listB.get(i)));
        }
        return result;
    }
    
    record Pair<A, B>(A first, B second) {}
    
    public static void main(String[] args) {
        List<Integer> nums = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        System.out.println("First: " + getFirst(nums));
        System.out.println("Last: " + getLast(nums));
        
        List<Integer> evens = filter(nums, n -> n % 2 == 0);
        System.out.println("Evens: " + evens);
        
        List<String> strs = map(nums, n -> "num_" + n);
        System.out.println("Mapped: " + strs);
        
        int sum = reduce(nums, 0, Integer::sum);
        System.out.println("Sum: " + sum);
        
        List<String> names = listOf("Alice", "Bob", "Charlie");
        List<Integer> ages = listOf(25, 30, 35);
        List<Pair<String, Integer>> zipped = zip(names, ages);
        System.out.println("Zipped: " + zipped);
        
        // Swap
        Integer[] arr = {1, 2, 3, 4, 5};
        swap(arr, 0, 4);
        System.out.println("Swapped: " + Arrays.toString(arr));
    }
}
```

---

## Bounded Type Parameters

```java
import java.util.*;

public class BoundedTypeParams {
    
    // Upper bounded: T extends Number
    public static <T extends Number> double sum(List<T> list) {
        double total = 0;
        for (T n : list) {
            total += n.doubleValue();
        }
        return total;
    }
    
    public static <T extends Number & Comparable<T>> T max(List<T> list) {
        if (list.isEmpty()) throw new NoSuchElementException();
        T result = list.get(0);
        for (T item : list) {
            if (item.compareTo(result) > 0) result = item;
        }
        return result;
    }
    
    // Multiple bounds
    public static <T extends Comparable<T>> List<T> sorted(List<T> list) {
        List<T> copy = new ArrayList<>(list);
        Collections.sort(copy);
        return copy;
    }
    
    // Recursive bound (CRTP pattern)
    public static <T extends Comparable<T>> T clamp(T value, T min, T max) {
        if (value.compareTo(min) < 0) return min;
        if (value.compareTo(max) > 0) return max;
        return value;
    }
    
    // Generic class with bounded parameter
    static class SortedList<T extends Comparable<T>> {
        private final List<T> items = new ArrayList<>();
        
        public void add(T item) {
            items.add(item);
            Collections.sort(items);
        }
        
        public T min() { return items.isEmpty() ? null : items.get(0); }
        public T max() { return items.isEmpty() ? null : items.get(items.size() - 1); }
        
        @Override
        public String toString() { return items.toString(); }
    }
    
    public static void main(String[] args) {
        List<Integer> ints = List.of(3, 1, 4, 1, 5, 9, 2, 6);
        List<Double> doubles = List.of(3.14, 2.71, 1.41);
        
        System.out.println("Sum ints: " + sum(ints));
        System.out.println("Sum doubles: " + sum(doubles));
        System.out.println("Max int: " + max(ints));
        System.out.println("Sorted: " + sorted(ints));
        
        System.out.println("clamp(15, 0, 10): " + clamp(15, 0, 10));
        System.out.println("clamp(5, 0, 10): " + clamp(5, 0, 10));
        System.out.println("clamp(-3, 0, 10): " + clamp(-3, 0, 10));
        
        SortedList<String> sortedList = new SortedList<>();
        sortedList.add("Banana");
        sortedList.add("Apple");
        sortedList.add("Cherry");
        sortedList.add("Date");
        System.out.println("SortedList: " + sortedList);
        System.out.println("Min: " + sortedList.min() + ", Max: " + sortedList.max());
    }
}
```

---

## Wildcards

```java
import java.util.*;

public class Wildcards {
    
    // Unbounded wildcard: List<?>
    // "list of unknown type"
    public static void printList(List<?> list) {
        System.out.print("[");
        for (int i = 0; i < list.size(); i++) {
            if (i > 0) System.out.print(", ");
            System.out.print(list.get(i));
        }
        System.out.println("]");
    }
    
    // Upper bounded wildcard: List<? extends Number>
    // "list of Number or any subtype" - can READ but not ADD
    public static double sumOfList(List<? extends Number> list) {
        double sum = 0;
        for (Number n : list) {
            sum += n.doubleValue();
        }
        return sum;
    }
    
    // Lower bounded wildcard: List<? super Integer>
    // "list of Integer or any supertype" - can ADD but reading returns Object
    public static void addNumbers(List<? super Integer> list, int count) {
        for (int i = 1; i <= count; i++) {
            list.add(i);
        }
    }
    
    // PECS principle: Producer Extends, Consumer Super
    public static <T> void copy(List<? extends T> source,     // Producer
                                List<? super T> destination) { // Consumer
        for (T item : source) {
            destination.add(item);
        }
    }
    
    public static void main(String[] args) {
        List<Integer> ints = List.of(1, 2, 3);
        List<Double> dbls = List.of(1.1, 2.2, 3.3);
        List<String> strs = List.of("a", "b", "c");
        
        // Unbounded wildcard
        printList(ints);
        printList(dbls);
        printList(strs);
        
        // Upper bounded
        System.out.println("Sum ints: " + sumOfList(ints));
        System.out.println("Sum doubles: " + sumOfList(dbls));
        
        // Lower bounded
        List<Number> numList = new ArrayList<>();
        addNumbers(numList, 5);
        System.out.println("numList: " + numList);
        
        // Copy (PECS)
        List<Integer> source = List.of(10, 20, 30);
        List<Number> destination = new ArrayList<>();
        copy(source, destination);
        System.out.println("Destination: " + destination);
        
        // Wildcard comparison
        List<Integer> list1 = Arrays.asList(1, 2, 3);
        List<Integer> list2 = Arrays.asList(1, 2, 3);
        System.out.println("Equal: " + list1.equals(list2));
    }
}
```

---

## Generic Interfaces

```java
import java.util.*;
import java.util.function.*;

public class GenericInterfaces {
    
    // Generic Repository Interface
    interface Repository<T, ID> {
        T save(T entity);
        Optional<T> findById(ID id);
        List<T> findAll();
        void delete(ID id);
        boolean exists(ID id);
        int count();
    }
    
    // Entity
    record Product(Long id, String name, double price, int stock) {}
    
    // In-memory implementation
    static class ProductRepository implements Repository<Product, Long> {
        private final Map<Long, Product> store = new HashMap<>();
        private long nextId = 1;
        
        @Override
        public Product save(Product product) {
            Product toSave = product.id() == null 
                ? new Product(nextId++, product.name(), product.price(), product.stock())
                : product;
            store.put(toSave.id(), toSave);
            return toSave;
        }
        
        @Override
        public Optional<Product> findById(Long id) {
            return Optional.ofNullable(store.get(id));
        }
        
        @Override
        public List<Product> findAll() { return new ArrayList<>(store.values()); }
        
        @Override
        public void delete(Long id) { store.remove(id); }
        
        @Override
        public boolean exists(Long id) { return store.containsKey(id); }
        
        @Override
        public int count() { return store.size(); }
        
        // Extra methods
        public List<Product> findByPriceLessThan(double maxPrice) {
            return store.values().stream()
                .filter(p -> p.price() < maxPrice)
                .toList();
        }
    }
    
    // Generic Service
    static class CrudService<T, ID> {
        private final Repository<T, ID> repository;
        
        CrudService(Repository<T, ID> repository) {
            this.repository = repository;
        }
        
        public T create(T entity) {
            return repository.save(entity);
        }
        
        public T getOrThrow(ID id) {
            return repository.findById(id)
                .orElseThrow(() -> new NoSuchElementException("Not found: " + id));
        }
        
        public List<T> getAll() {
            return repository.findAll();
        }
        
        public void remove(ID id) {
            if (!repository.exists(id)) {
                throw new NoSuchElementException("Not found: " + id);
            }
            repository.delete(id);
        }
    }
    
    public static void main(String[] args) {
        ProductRepository repo = new ProductRepository();
        CrudService<Product, Long> service = new CrudService<>(repo);
        
        // Create products
        Product p1 = service.create(new Product(null, "Laptop", 25000, 10));
        Product p2 = service.create(new Product(null, "Mouse", 500, 50));
        Product p3 = service.create(new Product(null, "Keyboard", 1500, 30));
        
        System.out.println("Created: " + p1);
        System.out.println("Count: " + repo.count());
        
        // Find
        Optional<Product> found = repo.findById(2L);
        found.ifPresent(p -> System.out.println("Found: " + p.name()));
        
        // List all
        System.out.println("\nAll products:");
        service.getAll().forEach(p -> 
            System.out.printf("  [%d] %s - %.2f (stock: %d)%n", 
                p.id(), p.name(), p.price(), p.stock()));
        
        // Custom query
        System.out.println("\nProducts under 2000:");
        repo.findByPriceLessThan(2000)
            .forEach(p -> System.out.println("  " + p.name() + ": " + p.price()));
        
        // Delete
        service.remove(2L);
        System.out.println("\nAfter delete, count: " + repo.count());
        
        // Throws if not found
        try {
            service.getOrThrow(999L);
        } catch (NoSuchElementException e) {
            System.out.println("Expected error: " + e.getMessage());
        }
    }
}
```

---

## Type Erasure

```java
import java.lang.reflect.*;
import java.util.*;

public class TypeErasure {
    
    // Type erasure: generic info removed at compile time
    public static void main(String[] args) {
        List<String> stringList = new ArrayList<>();
        List<Integer> intList = new ArrayList<>();
        
        // Same class at runtime!
        System.out.println("Same class: " + 
            (stringList.getClass() == intList.getClass()));  // true
        System.out.println("Class: " + stringList.getClass());  // ArrayList
        
        // Cannot do: instanceof with generic type
        // if (stringList instanceof List<String>) {}  // compile error
        if (stringList instanceof List<?>) {}  // OK (unbounded wildcard)
        
        // Cannot create generic array directly
        // T[] arr = new T[10];  // compile error
        // List<String>[] arr = new List<String>[10];  // compile error
        
        // Workaround: use Object array or Class<T>
        Object[] objArr = new Object[10];
        
        // Reflection to get generic info at runtime
        demonstrateReflection();
    }
    
    static class Container<T> {
        private T value;
        Container(T value) { this.value = value; }
        T get() { return value; }
    }
    
    static void demonstrateReflection() {
        try {
            // Can get generic type info from field/method declarations
            class Example {
                List<String> stringList;
                Map<String, Integer> map;
            }
            
            Field listField = Example.class.getDeclaredField("stringList");
            Type genericType = listField.getGenericType();
            System.out.println("\nField type: " + genericType);
            
            if (genericType instanceof ParameterizedType pt) {
                System.out.println("Raw type: " + pt.getRawType());
                System.out.println("Type args: " + 
                    Arrays.toString(pt.getActualTypeArguments()));
            }
            
            Field mapField = Example.class.getDeclaredField("map");
            Type mapGenericType = mapField.getGenericType();
            if (mapGenericType instanceof ParameterizedType pt) {
                System.out.println("Map type args: " + 
                    Arrays.toString(pt.getActualTypeArguments()));
            }
        } catch (NoSuchFieldException e) {
            System.out.println("Field not found: " + e.getMessage());
        }
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Generic Cache System

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;

public class GenericCacheSystem {
    
    // Generic Cache interface
    interface Cache<K, V> {
        void put(K key, V value);
        Optional<V> get(K key);
        void invalidate(K key);
        void clear();
        int size();
        boolean containsKey(K key);
    }
    
    // TTL Cache Entry
    static class CacheEntry<V> {
        final V value;
        final long expiresAt;
        
        CacheEntry(V value, long ttlMillis) {
            this.value = value;
            this.expiresAt = System.currentTimeMillis() + ttlMillis;
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() > expiresAt;
        }
    }
    
    // TTL Cache implementation
    static class TTLCache<K, V> implements Cache<K, V> {
        private final Map<K, CacheEntry<V>> store = new LinkedHashMap<>();
        private final long defaultTtlMillis;
        private final int maxSize;
        
        TTLCache(int maxSize, long defaultTtlMillis) {
            this.maxSize = maxSize;
            this.defaultTtlMillis = defaultTtlMillis;
        }
        
        @Override
        public void put(K key, V value) {
            cleanup();
            if (store.size() >= maxSize) {
                // Evict oldest entry
                K oldest = store.keySet().iterator().next();
                store.remove(oldest);
            }
            store.put(key, new CacheEntry<>(value, defaultTtlMillis));
        }
        
        @Override
        public Optional<V> get(K key) {
            CacheEntry<V> entry = store.get(key);
            if (entry == null) return Optional.empty();
            if (entry.isExpired()) {
                store.remove(key);
                return Optional.empty();
            }
            return Optional.of(entry.value);
        }
        
        @Override
        public void invalidate(K key) { store.remove(key); }
        
        @Override
        public void clear() { store.clear(); }
        
        @Override
        public int size() { 
            cleanup();
            return store.size(); 
        }
        
        @Override
        public boolean containsKey(K key) {
            return get(key).isPresent();
        }
        
        // Get or compute value
        public V getOrCompute(K key, Function<K, V> loader) {
            return get(key).orElseGet(() -> {
                V value = loader.apply(key);
                put(key, value);
                return value;
            });
        }
        
        private void cleanup() {
            store.entrySet().removeIf(e -> e.getValue().isExpired());
        }
    }
    
    // Service using cache
    static class UserService {
        private final TTLCache<Long, String> userCache = 
            new TTLCache<>(100, 5000);  // 100 entries, 5 second TTL
        
        private final Map<Long, String> database = new HashMap<>(Map.of(
            1L, "Alice", 2L, "Bob", 3L, "Charlie"
        ));
        
        private int dbHits = 0;
        
        public String getUserName(Long id) {
            return userCache.getOrCompute(id, this::loadFromDatabase);
        }
        
        private String loadFromDatabase(Long id) {
            dbHits++;
            System.out.println("  [DB] Loading user " + id);
            return database.getOrDefault(id, "Unknown");
        }
        
        public int getDbHits() { return dbHits; }
        
        public void invalidateUser(Long id) {
            userCache.invalidate(id);
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        UserService service = new UserService();
        
        System.out.println("=== Cache Demo ===");
        
        // First access - loads from DB
        System.out.println("First accesses (should hit DB):");
        System.out.println("User 1: " + service.getUserName(1L));
        System.out.println("User 2: " + service.getUserName(2L));
        System.out.println("User 1: " + service.getUserName(1L));  // cached
        System.out.println("DB hits: " + service.getDbHits());     // should be 2
        
        System.out.println("\nSecond accesses (from cache):");
        System.out.println("User 1: " + service.getUserName(1L));
        System.out.println("User 2: " + service.getUserName(2L));
        System.out.println("DB hits: " + service.getDbHits());     // still 2
        
        // Invalidate
        System.out.println("\nAfter invalidation:");
        service.invalidateUser(1L);
        System.out.println("User 1: " + service.getUserName(1L));  // loads again
        System.out.println("DB hits: " + service.getDbHits());     // now 3
        
        // Generic cache with different types
        TTLCache<String, List<Integer>> listCache = new TTLCache<>(50, 10000);
        listCache.put("nums", Arrays.asList(1, 2, 3, 4, 5));
        listCache.put("odds", Arrays.asList(1, 3, 5, 7, 9));
        
        System.out.println("\nList cache:");
        System.out.println("nums: " + listCache.get("nums").orElse(Collections.emptyList()));
        System.out.println("odds: " + listCache.get("odds").orElse(Collections.emptyList()));
        System.out.println("evens: " + listCache.get("evens").orElse(Collections.emptyList()));
        System.out.println("Cache size: " + listCache.size());
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Generics concept และประโยชน์  
✅ Generic Classes (Box, Pair, Stack)  
✅ Generic Methods (filter, map, reduce, zip)  
✅ Bounded Type Parameters (extends, multiple bounds)  
✅ Wildcards (?, ? extends, ? super)  
✅ PECS Principle  
✅ Generic Interfaces (Repository pattern)  
✅ Type Erasure  
✅ Generic Cache System  

---

## ขั้นตอนต่อไป

**Part 012:** Java Collections Framework (Advanced)  
เราจะเรียนรู้:
- HashMap, TreeMap, LinkedHashMap internals
- HashSet, TreeSet, LinkedHashSet
- Collections utility methods (advanced)
- Concurrent collections
- Iterable & Iterator pattern

---

*Part 011 | Java & Spring Boot Course | สร้างโดย Claude Code*
