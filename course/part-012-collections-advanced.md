# Part 012: Java Collections Framework (Advanced)
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Map Implementations](#map-implementations)
2. [Set Implementations](#set-implementations)
3. [Deque & Blocking Queues](#deque--blocking-queues)
4. [Collections Utility Methods](#collections-utility-methods)
5. [Iterator Pattern](#iterator-pattern)
6. [Comparable vs Comparator](#comparable-vs-comparator)
7. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Map Implementations

### HashMap vs LinkedHashMap vs TreeMap

```java
import java.util.*;

public class MapComparison {
    
    public static void main(String[] args) {
        // HashMap: O(1) ops, no order guarantee
        Map<String, Integer> hashMap = new HashMap<>();
        hashMap.put("banana", 2);
        hashMap.put("apple", 5);
        hashMap.put("cherry", 3);
        hashMap.put("date", 1);
        System.out.println("HashMap (no order): " + hashMap);
        
        // LinkedHashMap: O(1) ops, insertion order
        Map<String, Integer> linkedMap = new LinkedHashMap<>();
        linkedMap.put("banana", 2);
        linkedMap.put("apple", 5);
        linkedMap.put("cherry", 3);
        linkedMap.put("date", 1);
        System.out.println("LinkedHashMap (insertion order): " + linkedMap);
        
        // TreeMap: O(log n) ops, sorted order
        Map<String, Integer> treeMap = new TreeMap<>();
        treeMap.put("banana", 2);
        treeMap.put("apple", 5);
        treeMap.put("cherry", 3);
        treeMap.put("date", 1);
        System.out.println("TreeMap (sorted): " + treeMap);
        
        // LRU Cache with LinkedHashMap
        Map<String, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
                return size() > 3;  // max 3 entries
            }
        };
        lruCache.put("a", "Apple");
        lruCache.put("b", "Banana");
        lruCache.put("c", "Cherry");
        System.out.println("\nLRU Cache (3 items): " + lruCache);
        lruCache.get("a");  // access "a", moves to recent
        lruCache.put("d", "Date");  // evicts "b" (least recently used)
        System.out.println("After access 'a', add 'd': " + lruCache);
    }
}
```

### HashMap Operations (Advanced)

```java
import java.util.*;

public class HashMapAdvanced {
    
    public static void main(String[] args) {
        Map<String, Integer> wordCount = new HashMap<>();
        String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
        
        // getOrDefault
        for (String word : words) {
            wordCount.put(word, wordCount.getOrDefault(word, 0) + 1);
        }
        System.out.println("Word count (getOrDefault): " + wordCount);
        
        // putIfAbsent
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 90);
        scores.putIfAbsent("Alice", 100);  // no-op, already exists
        scores.putIfAbsent("Bob", 85);     // adds Bob
        System.out.println("Scores: " + scores);
        
        // computeIfAbsent (create if missing)
        Map<String, List<String>> groups = new HashMap<>();
        String[] names = {"Alice", "Bob", "Anna", "Charlie", "Ben"};
        for (String name : names) {
            groups.computeIfAbsent(
                String.valueOf(name.charAt(0)),
                k -> new ArrayList<>()
            ).add(name);
        }
        System.out.println("Groups: " + groups);
        
        // compute (update value)
        Map<String, Integer> counter = new HashMap<>();
        String[] items = {"a", "b", "a", "c", "b", "a"};
        for (String item : items) {
            counter.compute(item, (k, v) -> v == null ? 1 : v + 1);
        }
        System.out.println("Counter: " + counter);
        
        // merge (combine old and new value)
        Map<String, Integer> map1 = new HashMap<>(Map.of("a", 1, "b", 2));
        Map<String, Integer> map2 = new HashMap<>(Map.of("b", 3, "c", 4));
        map2.forEach((k, v) -> map1.merge(k, v, Integer::sum));
        System.out.println("Merged: " + map1);
        
        // replaceAll
        Map<String, Integer> prices = new HashMap<>(Map.of("apple", 100, "banana", 50));
        prices.replaceAll((k, v) -> (int)(v * 1.1));  // 10% price increase
        System.out.println("Prices after 10% increase: " + prices);
        
        // Iteration patterns
        System.out.println("\nIteration patterns:");
        Map<String, Integer> data = Map.of("x", 1, "y", 2, "z", 3);
        
        // forEach
        data.forEach((k, v) -> System.out.print(k + "=" + v + " "));
        System.out.println();
        
        // entrySet
        for (Map.Entry<String, Integer> entry : data.entrySet()) {
            System.out.print(entry.getKey() + "->" + entry.getValue() + " ");
        }
        System.out.println();
        
        // keySet + get (slower, avoid)
        for (String key : data.keySet()) {
            System.out.print(key + ":" + data.get(key) + " ");
        }
        System.out.println();
    }
}
```

### TreeMap Navigation

```java
import java.util.*;

public class TreeMapNavigation {
    
    public static void main(String[] args) {
        TreeMap<Integer, String> treeMap = new TreeMap<>();
        int[] keys = {50, 20, 80, 10, 30, 60, 90};
        String[] vals = {"E", "B", "H", "A", "C", "F", "I"};
        
        for (int i = 0; i < keys.length; i++) {
            treeMap.put(keys[i], vals[i]);
        }
        
        System.out.println("TreeMap: " + treeMap);
        System.out.println("First key: " + treeMap.firstKey());
        System.out.println("Last key: " + treeMap.lastKey());
        System.out.println("Floor 45: " + treeMap.floorKey(45));   // <= 45
        System.out.println("Ceiling 45: " + treeMap.ceilingKey(45)); // >= 45
        System.out.println("Lower 50: " + treeMap.lowerKey(50));    // < 50
        System.out.println("Higher 50: " + treeMap.higherKey(50));  // > 50
        
        System.out.println("headMap(<50): " + treeMap.headMap(50));     // < 50
        System.out.println("tailMap(>=50): " + treeMap.tailMap(50));    // >= 50
        System.out.println("subMap(20-60): " + treeMap.subMap(20, 60)); // [20, 60)
        
        System.out.println("Descending: " + treeMap.descendingMap());
        
        // NavigableMap: pollFirstEntry/pollLastEntry
        TreeMap<Integer, String> copy = new TreeMap<>(treeMap);
        System.out.println("\nPoll first: " + copy.pollFirstEntry());
        System.out.println("Poll last: " + copy.pollLastEntry());
        System.out.println("Remaining: " + copy);
    }
}
```

---

## Set Implementations

```java
import java.util.*;

public class SetImplementations {
    
    public static void main(String[] args) {
        // HashSet: O(1) ops, no order
        Set<String> hashSet = new HashSet<>(Arrays.asList("banana", "apple", "cherry", "apple"));
        System.out.println("HashSet (no dups, no order): " + hashSet);
        
        // LinkedHashSet: insertion order
        Set<String> linkedSet = new LinkedHashSet<>(Arrays.asList("banana", "apple", "cherry", "apple"));
        System.out.println("LinkedHashSet (insertion order): " + linkedSet);
        
        // TreeSet: sorted order
        Set<String> treeSet = new TreeSet<>(Arrays.asList("banana", "apple", "cherry", "apple"));
        System.out.println("TreeSet (sorted): " + treeSet);
        
        // Set operations
        Set<Integer> setA = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
        Set<Integer> setB = new HashSet<>(Arrays.asList(3, 4, 5, 6, 7));
        
        // Union
        Set<Integer> union = new HashSet<>(setA);
        union.addAll(setB);
        System.out.println("\nUnion: " + new TreeSet<>(union));
        
        // Intersection
        Set<Integer> intersection = new HashSet<>(setA);
        intersection.retainAll(setB);
        System.out.println("Intersection: " + new TreeSet<>(intersection));
        
        // Difference (A - B)
        Set<Integer> difference = new HashSet<>(setA);
        difference.removeAll(setB);
        System.out.println("Difference (A-B): " + new TreeSet<>(difference));
        
        // Symmetric difference
        Set<Integer> symDiff = new HashSet<>(union);
        symDiff.removeAll(intersection);
        System.out.println("Symmetric diff: " + new TreeSet<>(symDiff));
        
        // TreeSet navigation
        TreeSet<Integer> nav = new TreeSet<>(Arrays.asList(10, 20, 30, 40, 50));
        System.out.println("\nTreeSet: " + nav);
        System.out.println("Floor 25: " + nav.floor(25));    // <= 25
        System.out.println("Ceiling 25: " + nav.ceiling(25)); // >= 25
        System.out.println("headSet(<30): " + nav.headSet(30));
        System.out.println("tailSet(>=30): " + nav.tailSet(30));
        System.out.println("subSet(20-40): " + nav.subSet(20, true, 40, false));
    }
}
```

---

## Deque & Blocking Queues

```java
import java.util.*;
import java.util.concurrent.*;

public class DequeAndQueues {
    
    public static void main(String[] args) throws InterruptedException {
        // ArrayDeque as Stack
        Deque<String> stack = new ArrayDeque<>();
        stack.push("first");
        stack.push("second");
        stack.push("third");
        System.out.println("Stack: " + stack);
        System.out.println("Pop: " + stack.pop());
        System.out.println("Peek: " + stack.peek());
        
        // ArrayDeque as Queue
        Deque<String> queue = new ArrayDeque<>();
        queue.offer("first");
        queue.offer("second");
        queue.offer("third");
        System.out.println("\nQueue: " + queue);
        System.out.println("Poll: " + queue.poll());
        System.out.println("Peek: " + queue.peek());
        
        // Priority Queue (min-heap by default)
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        minHeap.addAll(Arrays.asList(5, 3, 8, 1, 9, 2));
        System.out.print("\nMin-heap order: ");
        while (!minHeap.isEmpty()) System.out.print(minHeap.poll() + " ");
        
        // Max-heap using reverseOrder comparator
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        maxHeap.addAll(Arrays.asList(5, 3, 8, 1, 9, 2));
        System.out.print("\nMax-heap order: ");
        while (!maxHeap.isEmpty()) System.out.print(maxHeap.poll() + " ");
        
        // Custom priority: Task scheduler
        record Task(String name, int priority) implements Comparable<Task> {
            @Override
            public int compareTo(Task other) {
                return Integer.compare(other.priority, this.priority); // high priority first
            }
        }
        
        PriorityQueue<Task> taskQueue = new PriorityQueue<>();
        taskQueue.add(new Task("Low priority task", 1));
        taskQueue.add(new Task("Critical bug fix", 10));
        taskQueue.add(new Task("Regular feature", 5));
        taskQueue.add(new Task("Security patch", 9));
        taskQueue.add(new Task("UI improvement", 3));
        
        System.out.println("\n\nTask execution order:");
        while (!taskQueue.isEmpty()) {
            Task t = taskQueue.poll();
            System.out.printf("  [Priority %d] %s%n", t.priority(), t.name());
        }
        
        // BlockingQueue
        BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(3);
        blockingQueue.put("item1");
        blockingQueue.put("item2");
        blockingQueue.put("item3");
        
        // offer with timeout returns false if full
        boolean offered = blockingQueue.offer("item4", 100, TimeUnit.MILLISECONDS);
        System.out.println("\nOffer to full queue: " + offered);  // false
        
        System.out.println("Poll: " + blockingQueue.poll());
        System.out.println("Queue: " + blockingQueue);
    }
}
```

---

## Collections Utility Methods

```java
import java.util.*;

public class CollectionsUtils {
    
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(5, 3, 8, 1, 9, 2, 7, 4, 6));
        
        System.out.println("Original: " + list);
        
        // Sort
        List<Integer> sorted = new ArrayList<>(list);
        Collections.sort(sorted);
        System.out.println("Sorted: " + sorted);
        
        // Reverse sort
        Collections.sort(sorted, Collections.reverseOrder());
        System.out.println("Reverse sorted: " + sorted);
        
        // Binary search (requires sorted)
        Collections.sort(list);
        int idx = Collections.binarySearch(list, 7);
        System.out.println("Index of 7: " + idx);
        
        // Shuffle
        List<Integer> shuffled = new ArrayList<>(Arrays.asList(1,2,3,4,5));
        Collections.shuffle(shuffled, new Random(42));
        System.out.println("Shuffled: " + shuffled);
        
        // Min/Max
        System.out.println("Min: " + Collections.min(list));
        System.out.println("Max: " + Collections.max(list));
        
        // Frequency
        List<String> fruits = Arrays.asList("apple","banana","apple","cherry","apple","banana");
        System.out.println("Frequency of 'apple': " + Collections.frequency(fruits, "apple"));
        
        // Reverse
        List<Integer> toReverse = new ArrayList<>(Arrays.asList(1,2,3,4,5));
        Collections.reverse(toReverse);
        System.out.println("Reversed: " + toReverse);
        
        // Fill
        List<String> filled = new ArrayList<>(Arrays.asList("", "", "", "", ""));
        Collections.fill(filled, "placeholder");
        System.out.println("Filled: " + filled);
        
        // Copy
        List<Integer> dest = new ArrayList<>(Arrays.asList(0,0,0,0,0,0,0,0,0));
        Collections.copy(dest, list);
        System.out.println("Copied: " + dest);
        
        // Swap
        List<String> swapList = new ArrayList<>(Arrays.asList("A","B","C","D","E"));
        Collections.swap(swapList, 0, 4);
        System.out.println("Swapped: " + swapList);
        
        // Rotate
        List<Integer> rotateList = new ArrayList<>(Arrays.asList(1,2,3,4,5));
        Collections.rotate(rotateList, 2);
        System.out.println("Rotated by 2: " + rotateList);
        
        // Unmodifiable
        List<String> immutable = Collections.unmodifiableList(new ArrayList<>(Arrays.asList("a","b","c")));
        try {
            immutable.add("d");
        } catch (UnsupportedOperationException e) {
            System.out.println("Cannot modify unmodifiable list");
        }
        
        // disjoint (no common elements)
        Set<Integer> s1 = new HashSet<>(Arrays.asList(1, 2, 3));
        Set<Integer> s2 = new HashSet<>(Arrays.asList(4, 5, 6));
        Set<Integer> s3 = new HashSet<>(Arrays.asList(3, 4, 5));
        System.out.println("Disjoint s1,s2: " + Collections.disjoint(s1, s2)); // true
        System.out.println("Disjoint s1,s3: " + Collections.disjoint(s1, s3)); // false
        
        // nCopies
        List<String> copies = Collections.nCopies(5, "hello");
        System.out.println("nCopies: " + copies);
        
        // singleton
        Set<String> singleton = Collections.singleton("only");
        System.out.println("Singleton: " + singleton);
        
        // emptyList/emptySet/emptyMap
        List<Object> emptyList = Collections.emptyList();
        System.out.println("Empty list: " + emptyList + " (size: " + emptyList.size() + ")");
    }
}
```

---

## Iterator Pattern

```java
import java.util.*;

public class IteratorPattern {
    
    // Custom Iterable: Binary Tree
    static class BinaryTree<T> implements Iterable<T> {
        private Node<T> root;
        
        static class Node<T> {
            T value;
            Node<T> left, right;
            Node(T value) { this.value = value; }
        }
        
        public void insert(T value, Comparator<T> cmp) {
            root = insert(root, value, cmp);
        }
        
        private Node<T> insert(Node<T> node, T value, Comparator<T> cmp) {
            if (node == null) return new Node<>(value);
            if (cmp.compare(value, node.value) < 0)
                node.left = insert(node.left, value, cmp);
            else if (cmp.compare(value, node.value) > 0)
                node.right = insert(node.right, value, cmp);
            return node;
        }
        
        // In-order iterator (sorted)
        @Override
        public Iterator<T> iterator() {
            return new InOrderIterator();
        }
        
        class InOrderIterator implements Iterator<T> {
            private final Deque<Node<T>> stack = new ArrayDeque<>();
            
            InOrderIterator() {
                pushLeft(root);
            }
            
            private void pushLeft(Node<T> node) {
                while (node != null) {
                    stack.push(node);
                    node = node.left;
                }
            }
            
            @Override
            public boolean hasNext() { return !stack.isEmpty(); }
            
            @Override
            public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                Node<T> current = stack.pop();
                T value = current.value;
                pushLeft(current.right);
                return value;
            }
        }
    }
    
    // Lazy infinite sequence
    static class FibonacciSequence implements Iterable<Long> {
        private final int limit;
        
        FibonacciSequence(int limit) { this.limit = limit; }
        
        @Override
        public Iterator<Long> iterator() {
            return new Iterator<>() {
                private long a = 0, b = 1;
                private int count = 0;
                
                @Override
                public boolean hasNext() { return count < limit; }
                
                @Override
                public Long next() {
                    long value = a;
                    long next = a + b;
                    a = b;
                    b = next;
                    count++;
                    return value;
                }
            };
        }
    }
    
    public static void main(String[] args) {
        // Binary tree iterator
        BinaryTree<Integer> tree = new BinaryTree<>();
        int[] values = {5, 3, 7, 1, 4, 6, 8};
        for (int v : values) tree.insert(v, Integer::compare);
        
        System.out.print("In-order (sorted): ");
        for (int v : tree) System.out.print(v + " ");
        System.out.println();
        
        // Fibonacci
        System.out.print("Fibonacci: ");
        for (long f : new FibonacciSequence(10)) {
            System.out.print(f + " ");
        }
        System.out.println();
        
        // ListIterator: can go forward and backward
        List<String> list = new ArrayList<>(Arrays.asList("A","B","C","D","E"));
        ListIterator<String> it = list.listIterator(list.size());  // start from end
        System.out.print("Reverse: ");
        while (it.hasPrevious()) System.out.print(it.previous() + " ");
        System.out.println();
        
        // Modify while iterating (using iterator.remove())
        List<Integer> nums = new ArrayList<>(Arrays.asList(1,2,3,4,5,6,7,8,9,10));
        Iterator<Integer> iter = nums.iterator();
        while (iter.hasNext()) {
            if (iter.next() % 2 == 0) iter.remove();  // remove evens
        }
        System.out.println("After removing evens: " + nums);
    }
}
```

---

## Comparable vs Comparator

```java
import java.util.*;

public class ComparableVsComparator {
    
    // Comparable: natural ordering built into the class
    static class Employee implements Comparable<Employee> {
        String name;
        int salary;
        String department;
        
        Employee(String name, int salary, String department) {
            this.name = name;
            this.salary = salary;
            this.department = department;
        }
        
        // Natural order: by name
        @Override
        public int compareTo(Employee other) {
            return this.name.compareTo(other.name);
        }
        
        @Override
        public String toString() {
            return name + "($" + salary + "," + department + ")";
        }
    }
    
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(Arrays.asList(
            new Employee("Charlie", 70000, "Engineering"),
            new Employee("Alice", 90000, "Marketing"),
            new Employee("Bob", 80000, "Engineering"),
            new Employee("Diana", 75000, "HR"),
            new Employee("Eve", 85000, "Engineering")
        ));
        
        // Natural order (Comparable)
        Collections.sort(employees);
        System.out.println("By name (natural): " + employees);
        
        // Comparator: external ordering strategies
        Comparator<Employee> bySalary = Comparator.comparingInt(e -> e.salary);
        Comparator<Employee> bySalaryDesc = bySalary.reversed();
        Comparator<Employee> byDeptThenName = Comparator
            .comparing((Employee e) -> e.department)
            .thenComparing(e -> e.name);
        Comparator<Employee> byDeptThenSalaryDesc = Comparator
            .comparing((Employee e) -> e.department)
            .thenComparing(Comparator.comparingInt((Employee e) -> e.salary).reversed());
        
        employees.sort(bySalary);
        System.out.println("By salary (asc): " + employees);
        
        employees.sort(bySalaryDesc);
        System.out.println("By salary (desc): " + employees);
        
        employees.sort(byDeptThenName);
        System.out.println("By dept then name: " + employees);
        
        employees.sort(byDeptThenSalaryDesc);
        System.out.println("By dept, salary desc: " + employees);
        
        // Null-safe comparator
        List<String> withNulls = new ArrayList<>(Arrays.asList("B", null, "A", null, "C"));
        withNulls.sort(Comparator.nullsFirst(Comparator.naturalOrder()));
        System.out.println("\nNulls first: " + withNulls);
        
        withNulls.sort(Comparator.nullsLast(Comparator.naturalOrder()));
        System.out.println("Nulls last: " + withNulls);
        
        // TreeMap with custom comparator (reverse key order)
        Map<String, Integer> reverseMap = new TreeMap<>(Comparator.reverseOrder());
        reverseMap.put("banana", 1);
        reverseMap.put("apple", 2);
        reverseMap.put("cherry", 3);
        System.out.println("\nTreeMap reverse order: " + reverseMap);
        
        // TreeSet with custom comparator
        TreeSet<Employee> empSet = new TreeSet<>(bySalaryDesc);
        empSet.addAll(employees);
        System.out.println("\nTreeSet by salary desc: " + empSet);
    }
}
```

---

## โปรแกรมตัวอย่างจริง: E-Commerce Inventory System

```java
import java.util.*;
import java.util.stream.*;

public class InventorySystem {
    
    enum Category { ELECTRONICS, CLOTHING, FOOD, BOOKS }
    
    record Product(String id, String name, Category category, 
                   double price, int quantity) {
        double totalValue() { return price * quantity; }
    }
    
    static class Inventory {
        // Multiple data structures for different operations
        private final Map<String, Product> byId = new LinkedHashMap<>();
        private final Map<Category, TreeSet<Product>> byCategory = new EnumMap<>(Category.class);
        private final TreeMap<Double, Set<Product>> byPrice = new TreeMap<>();
        
        private static final Comparator<Product> BY_NAME = 
            Comparator.comparing(Product::name);
        
        public Inventory() {
            for (Category cat : Category.values()) {
                byCategory.put(cat, new TreeSet<>(BY_NAME));
            }
        }
        
        public void addProduct(Product product) {
            byId.put(product.id(), product);
            byCategory.get(product.category()).add(product);
            byPrice.computeIfAbsent(product.price(), k -> new HashSet<>()).add(product);
        }
        
        public Optional<Product> findById(String id) {
            return Optional.ofNullable(byId.get(id));
        }
        
        public Set<Product> findByCategory(Category category) {
            return Collections.unmodifiableSet(byCategory.get(category));
        }
        
        public List<Product> findByPriceRange(double min, double max) {
            return byPrice.subMap(min, true, max, true)
                .values().stream()
                .flatMap(Set::stream)
                .sorted(Comparator.comparingDouble(Product::price))
                .collect(Collectors.toList());
        }
        
        public List<Product> findAffordable(double budget) {
            return byPrice.headMap(budget, true).values().stream()
                .flatMap(Set::stream)
                .sorted(Comparator.comparingDouble(Product::price).reversed())
                .collect(Collectors.toList());
        }
        
        public Map<Category, DoubleSummaryStatistics> priceStatsByCategory() {
            Map<Category, DoubleSummaryStatistics> stats = new EnumMap<>(Category.class);
            for (Category cat : Category.values()) {
                DoubleSummaryStatistics s = byCategory.get(cat).stream()
                    .mapToDouble(Product::price)
                    .summaryStatistics();
                stats.put(cat, s);
            }
            return stats;
        }
        
        public List<Product> getLowStock(int threshold) {
            return byId.values().stream()
                .filter(p -> p.quantity() <= threshold)
                .sorted(Comparator.comparingInt(Product::quantity))
                .collect(Collectors.toList());
        }
        
        public double totalInventoryValue() {
            return byId.values().stream()
                .mapToDouble(Product::totalValue)
                .sum();
        }
        
        public void printReport() {
            System.out.println("=== Inventory Report ===");
            System.out.printf("Total products: %d%n", byId.size());
            System.out.printf("Total value: $%.2f%n", totalInventoryValue());
            
            System.out.println("\nBy Category:");
            byCategory.forEach((cat, products) -> {
                System.out.printf("  %s (%d items):%n", cat, products.size());
                products.forEach(p -> 
                    System.out.printf("    - %s: $%.2f x %d = $%.2f%n",
                        p.name(), p.price(), p.quantity(), p.totalValue()));
            });
            
            System.out.println("\nPrice Stats by Category:");
            priceStatsByCategory().forEach((cat, stats) -> {
                if (stats.getCount() > 0) {
                    System.out.printf("  %s: min=$%.2f, max=$%.2f, avg=$%.2f%n",
                        cat, stats.getMin(), stats.getMax(), stats.getAverage());
                }
            });
            
            System.out.println("\nLow Stock (<=5):");
            getLowStock(5).forEach(p -> 
                System.out.printf("  ⚠️  %s: only %d left%n", p.name(), p.quantity()));
        }
    }
    
    public static void main(String[] args) {
        Inventory inventory = new Inventory();
        
        // Add products
        inventory.addProduct(new Product("E001", "Laptop Pro", Category.ELECTRONICS, 35000, 10));
        inventory.addProduct(new Product("E002", "Wireless Mouse", Category.ELECTRONICS, 850, 50));
        inventory.addProduct(new Product("E003", "USB-C Hub", Category.ELECTRONICS, 1200, 3));
        inventory.addProduct(new Product("C001", "T-Shirt", Category.CLOTHING, 299, 100));
        inventory.addProduct(new Product("C002", "Jeans", Category.CLOTHING, 1500, 30));
        inventory.addProduct(new Product("C003", "Sneakers", Category.CLOTHING, 2500, 2));
        inventory.addProduct(new Product("F001", "Organic Coffee", Category.FOOD, 450, 20));
        inventory.addProduct(new Product("F002", "Green Tea", Category.FOOD, 180, 5));
        inventory.addProduct(new Product("B001", "Clean Code", Category.BOOKS, 750, 15));
        inventory.addProduct(new Product("B002", "Java 17", Category.BOOKS, 950, 8));
        
        inventory.printReport();
        
        System.out.println("\n=== Queries ===");
        System.out.println("Products 300-1000 baht: " + 
            inventory.findByPriceRange(300, 1000).stream()
                .map(Product::name).collect(Collectors.joining(", ")));
        
        System.out.println("Affordable with 1000 baht: " + 
            inventory.findAffordable(1000).stream()
                .map(p -> p.name() + "($" + (int)p.price() + ")")
                .collect(Collectors.joining(", ")));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ HashMap vs LinkedHashMap vs TreeMap  
✅ HashMap advanced ops (computeIfAbsent, merge, replaceAll)  
✅ TreeMap navigation methods  
✅ HashSet vs LinkedHashSet vs TreeSet  
✅ Set operations (union, intersection, difference)  
✅ PriorityQueue and custom priorities  
✅ Collections utility methods  
✅ Iterator and custom iterables  
✅ Comparable vs Comparator  
✅ Complete Inventory System  

---

## ขั้นตอนต่อไป

**Part 013:** Java Streams API  
- Stream creation and operations
- Intermediate operations (filter, map, flatMap, distinct, sorted)
- Terminal operations (collect, reduce, count, min, max)
- Collectors (toList, toMap, groupingBy, partitioningBy)
- Parallel streams

---

*Part 012 | Java & Spring Boot Course | สร้างโดย Claude Code*
