# Part 016: Concurrency & Multithreading
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Thread Basics](#thread-basics)
2. [Synchronization](#synchronization)
3. [volatile Keyword](#volatile-keyword)
4. [ExecutorService](#executorservice)
5. [Future & CompletableFuture](#future--completablefuture)
6. [Concurrent Collections](#concurrent-collections)
7. [Thread Safety Patterns](#thread-safety-patterns)
8. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Thread Basics

```java
public class ThreadBasics {
    
    // Method 1: extends Thread
    static class CounterThread extends Thread {
        private final String name;
        private final int count;
        
        CounterThread(String name, int count) {
            this.name = name;
            this.count = count;
        }
        
        @Override
        public void run() {
            for (int i = 1; i <= count; i++) {
                System.out.printf("[%s] count=%d%n", name, i);
                try { Thread.sleep(50); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }
    
    // Method 2: implements Runnable
    static class Printer implements Runnable {
        private final String message;
        
        Printer(String message) { this.message = message; }
        
        @Override
        public void run() {
            for (int i = 0; i < 3; i++) {
                System.out.println(Thread.currentThread().getName() + ": " + message);
                try { Thread.sleep(30); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Thread Lifecycle ===");
        System.out.println("Current thread: " + Thread.currentThread().getName());
        System.out.println("Active threads: " + Thread.activeCount());
        
        // Method 1: extends Thread
        CounterThread t1 = new CounterThread("Thread-A", 3);
        CounterThread t2 = new CounterThread("Thread-B", 3);
        
        t1.start();
        t2.start();
        
        t1.join();  // wait for t1 to finish
        t2.join();  // wait for t2 to finish
        
        System.out.println("\n=== Runnable ===");
        Thread t3 = new Thread(new Printer("Hello from Runnable"), "PrinterThread");
        t3.start();
        t3.join();
        
        System.out.println("\n=== Lambda Thread ===");
        Thread t4 = new Thread(() -> {
            System.out.println("Lambda thread running: " + Thread.currentThread().getName());
        });
        t4.setName("LambdaThread");
        t4.start();
        t4.join();
        
        // Thread properties
        System.out.println("\n=== Thread Properties ===");
        Thread current = Thread.currentThread();
        System.out.println("Name: " + current.getName());
        System.out.println("ID: " + current.threadId());
        System.out.println("Priority: " + current.getPriority());
        System.out.println("isDaemon: " + current.isDaemon());
        System.out.println("State: " + current.getState());
        
        // Daemon thread
        Thread daemon = new Thread(() -> {
            while (true) {
                System.out.println("Daemon is running...");
                try { Thread.sleep(100); } catch (InterruptedException e) { break; }
            }
        });
        daemon.setDaemon(true);  // JVM won't wait for daemon threads
        daemon.start();
        Thread.sleep(250);
        System.out.println("Main done (daemon will be killed)");
    }
}
```

---

## Synchronization

```java
import java.util.*;
import java.util.concurrent.*;

public class SynchronizationDemo {
    
    // Problem: race condition without sync
    static class UnsafeCounter {
        int count = 0;
        
        void increment() { count++; }  // NOT atomic!
        int get() { return count; }
    }
    
    // Solution 1: synchronized method
    static class SafeCounter {
        private int count = 0;
        
        synchronized void increment() { count++; }
        synchronized int get() { return count; }
    }
    
    // Solution 2: synchronized block (more granular)
    static class BankAccount {
        private double balance;
        private final Object lock = new Object();
        
        BankAccount(double initial) { this.balance = initial; }
        
        void deposit(double amount) {
            synchronized (lock) {
                balance += amount;
            }
        }
        
        boolean withdraw(double amount) {
            synchronized (lock) {
                if (balance >= amount) {
                    balance -= amount;
                    return true;
                }
                return false;
            }
        }
        
        double getBalance() {
            synchronized (lock) { return balance; }
        }
    }
    
    // Solution 3: AtomicInteger (fastest for simple counters)
    static class AtomicCounter {
        private final java.util.concurrent.atomic.AtomicInteger count = 
            new java.util.concurrent.atomic.AtomicInteger(0);
        
        void increment() { count.incrementAndGet(); }
        int get() { return count.get(); }
    }
    
    // Deadlock example (and how to avoid)
    static class DeadlockDemo {
        static final Object lock1 = new Object();
        static final Object lock2 = new Object();
        
        static void thread1() throws InterruptedException {
            synchronized (lock1) {
                Thread.sleep(50);
                synchronized (lock2) {  // deadlock: t1 has lock1, wants lock2
                    System.out.println("Thread1 done");
                }
            }
        }
        
        static void thread2() throws InterruptedException {
            synchronized (lock1) {  // Fix: same lock order
                Thread.sleep(50);
                synchronized (lock2) {
                    System.out.println("Thread2 done");
                }
            }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        // Race condition demo
        System.out.println("=== Race Condition ===");
        UnsafeCounter unsafe = new UnsafeCounter();
        SafeCounter safe = new SafeCounter();
        AtomicCounter atomic = new AtomicCounter();
        
        int numThreads = 100;
        int increments = 1000;
        
        CountDownLatch latch = new CountDownLatch(numThreads);
        
        for (int t = 0; t < numThreads; t++) {
            new Thread(() -> {
                for (int i = 0; i < increments; i++) {
                    unsafe.increment();
                    safe.increment();
                    atomic.increment();
                }
                latch.countDown();
            }).start();
        }
        
        latch.await();
        
        int expected = numThreads * increments;
        System.out.println("Expected: " + expected);
        System.out.println("Unsafe: " + unsafe.get() + (unsafe.get() == expected ? " ✓" : " ✗ (race!)"));
        System.out.println("Safe: " + safe.get() + (safe.get() == expected ? " ✓" : " ✗"));
        System.out.println("Atomic: " + atomic.get() + (atomic.get() == expected ? " ✓" : " ✗"));
        
        // Bank account
        System.out.println("\n=== Bank Account (Thread-Safe) ===");
        BankAccount account = new BankAccount(1000);
        CountDownLatch bankLatch = new CountDownLatch(10);
        
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                account.deposit(100);
                bankLatch.countDown();
            }).start();
        }
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                account.withdraw(50);
                bankLatch.countDown();
            }).start();
        }
        
        bankLatch.await();
        System.out.println("Final balance: " + account.getBalance());
        System.out.println("Expected: " + (1000 + 5*100 - 5*50));
    }
}
```

---

## volatile Keyword

```java
public class VolatileDemo {
    
    // Without volatile: visibility problem
    static class StopFlag {
        private boolean running = true;           // may be cached in CPU register
        private volatile boolean runningV = true; // always reads from main memory
        
        void stop() { running = false; }
        void stopV() { runningV = false; }
        
        void runUnsafe() {
            while (running) {  // might loop forever (cached old value)
                // CPU may cache 'running' and never re-read
            }
            System.out.println("Unsafe stopped");
        }
        
        void runSafe() {
            while (runningV) {  // always reads fresh value
                // Thread will see the update
            }
            System.out.println("Safe stopped");
        }
    }
    
    // Double-checked locking (singleton pattern)
    static class Singleton {
        private static volatile Singleton instance;  // volatile is crucial here
        
        private Singleton() {}
        
        static Singleton getInstance() {
            if (instance == null) {
                synchronized (Singleton.class) {
                    if (instance == null) {  // double check
                        instance = new Singleton();
                    }
                }
            }
            return instance;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        // Singleton
        System.out.println("=== Singleton ===");
        Singleton s1 = Singleton.getInstance();
        Singleton s2 = Singleton.getInstance();
        System.out.println("Same instance: " + (s1 == s2));
        
        // Volatile flag
        System.out.println("\n=== Volatile Flag ===");
        StopFlag flag = new StopFlag();
        Thread worker = new Thread(flag::runSafe);
        worker.start();
        Thread.sleep(100);
        flag.stopV();
        worker.join(1000);
        System.out.println("Worker state: " + worker.getState());
    }
}
```

---

## ExecutorService

```java
import java.util.concurrent.*;
import java.util.*;

public class ExecutorServiceDemo {
    
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        // Fixed thread pool
        System.out.println("=== Fixed Thread Pool ===");
        ExecutorService pool = Executors.newFixedThreadPool(3);
        
        for (int i = 0; i < 8; i++) {
            final int taskId = i;
            pool.submit(() -> {
                System.out.printf("Task %d on %s%n", taskId, Thread.currentThread().getName());
                try { Thread.sleep(50); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }
        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
        
        // Cached thread pool (grows as needed)
        System.out.println("\n=== Cached Thread Pool ===");
        ExecutorService cached = Executors.newCachedThreadPool();
        List<Future<Integer>> futures = new ArrayList<>();
        
        for (int i = 0; i < 5; i++) {
            final int n = i * 10;
            Future<Integer> f = cached.submit(() -> {
                Thread.sleep(50);
                return n * n;
            });
            futures.add(f);
        }
        
        System.out.print("Squares: ");
        for (Future<Integer> f : futures) {
            System.out.print(f.get() + " ");
        }
        System.out.println();
        cached.shutdown();
        
        // ScheduledExecutorService
        System.out.println("\n=== Scheduled Executor ===");
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
        
        // One-time delay
        scheduler.schedule(() -> System.out.println("Delayed task ran!"), 100, TimeUnit.MILLISECONDS);
        
        // Repeated at fixed rate
        ScheduledFuture<?> periodic = scheduler.scheduleAtFixedRate(
            () -> System.out.println("Periodic: " + System.currentTimeMillis() % 10000),
            0, 200, TimeUnit.MILLISECONDS);
        
        Thread.sleep(700);
        periodic.cancel(false);
        scheduler.shutdown();
        scheduler.awaitTermination(1, TimeUnit.SECONDS);
        
        // invokeAll / invokeAny
        System.out.println("\n=== invokeAll ===");
        ExecutorService exec = Executors.newFixedThreadPool(3);
        List<Callable<String>> tasks = List.of(
            () -> { Thread.sleep(100); return "Task A"; },
            () -> { Thread.sleep(50);  return "Task B"; },
            () -> { Thread.sleep(150); return "Task C"; }
        );
        
        List<Future<String>> results = exec.invokeAll(tasks);
        results.forEach(f -> {
            try { System.out.println("Result: " + f.get()); }
            catch (Exception e) { e.printStackTrace(); }
        });
        
        System.out.println("\n=== invokeAny (fastest wins) ===");
        String fastest = exec.invokeAny(tasks);
        System.out.println("Fastest result: " + fastest);
        
        exec.shutdown();
    }
}
```

---

## Future & CompletableFuture

```java
import java.util.concurrent.*;
import java.util.*;
import java.util.stream.*;

public class CompletableFutureDemo {
    
    static String fetchUser(long id) {
        sleep(100);
        return "User" + id;
    }
    
    static String fetchOrders(String user) {
        sleep(150);
        return user + "_orders[O1,O2,O3]";
    }
    
    static double calculateTotal(String orders) {
        sleep(80);
        return 1500.0;
    }
    
    static void sleep(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
    
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService exec = Executors.newFixedThreadPool(8);
        
        System.out.println("=== Basic CompletableFuture ===");
        
        // supplyAsync: async computation
        CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
            sleep(100);
            return "Hello from async!";
        }, exec);
        
        System.out.println("Is done: " + cf.isDone());
        System.out.println("Result: " + cf.get());
        
        // thenApply: transform result
        CompletableFuture<Integer> lengthFuture = cf.thenApply(String::length);
        System.out.println("Length: " + lengthFuture.get());
        
        System.out.println("\n=== Chain (Sequential) ===");
        long start = System.currentTimeMillis();
        
        String result = CompletableFuture
            .supplyAsync(() -> fetchUser(42), exec)
            .thenApply(user -> fetchOrders(user))
            .thenApply(orders -> "Total: " + calculateTotal(orders))
            .get();
        
        System.out.printf("Chain result: %s (took %d ms)%n", result, 
            System.currentTimeMillis() - start);
        
        System.out.println("\n=== Parallel Tasks ===");
        start = System.currentTimeMillis();
        
        CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> fetchUser(1), exec);
        CompletableFuture<String> user2Future = CompletableFuture.supplyAsync(() -> fetchUser(2), exec);
        CompletableFuture<String> user3Future = CompletableFuture.supplyAsync(() -> fetchUser(3), exec);
        
        // Wait for all
        CompletableFuture.allOf(userFuture, user2Future, user3Future).get();
        System.out.printf("3 users fetched in %d ms: %s, %s, %s%n",
            System.currentTimeMillis() - start,
            userFuture.get(), user2Future.get(), user3Future.get());
        
        System.out.println("\n=== anyOf (first to complete) ===");
        start = System.currentTimeMillis();
        CompletableFuture<Object> fastest = CompletableFuture.anyOf(
            CompletableFuture.supplyAsync(() -> { sleep(200); return "Slow"; }, exec),
            CompletableFuture.supplyAsync(() -> { sleep(50);  return "Fast"; }, exec),
            CompletableFuture.supplyAsync(() -> { sleep(100); return "Medium"; }, exec)
        );
        System.out.printf("Fastest: %s (in %d ms)%n", fastest.get(), System.currentTimeMillis() - start);
        
        System.out.println("\n=== Error Handling ===");
        CompletableFuture<String> withError = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) throw new RuntimeException("Random failure");
                return "Success";
            }, exec)
            .exceptionally(ex -> "Recovered from: " + ex.getMessage())
            .whenComplete((val, ex) -> System.out.println("Complete: " + val));
        
        System.out.println("Result: " + withError.get());
        
        System.out.println("\n=== thenCombine (zip two results) ===");
        CompletableFuture<String> nameF = CompletableFuture.supplyAsync(() -> { sleep(80); return "Alice"; }, exec);
        CompletableFuture<Integer> ageF = CompletableFuture.supplyAsync(() -> { sleep(60); return 30; }, exec);
        
        CompletableFuture<String> combined = nameF.thenCombine(ageF, 
            (name, age) -> name + " is " + age + " years old");
        System.out.println("Combined: " + combined.get());
        
        exec.shutdown();
    }
}
```

---

## Concurrent Collections

```java
import java.util.*;
import java.util.concurrent.*;

public class ConcurrentCollections {
    
    public static void main(String[] args) throws InterruptedException {
        // ConcurrentHashMap: thread-safe HashMap
        System.out.println("=== ConcurrentHashMap ===");
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        ExecutorService exec = Executors.newFixedThreadPool(5);
        CountDownLatch latch = new CountDownLatch(100);
        
        for (int i = 0; i < 100; i++) {
            final int val = i;
            exec.submit(() -> {
                String key = "key" + (val % 10);
                map.merge(key, 1, Integer::sum);
                latch.countDown();
            });
        }
        latch.await();
        System.out.println("Map size: " + map.size() + " (expected 10)");
        System.out.println("Total count: " + map.values().stream().mapToInt(Integer::intValue).sum() + " (expected 100)");
        
        // CopyOnWriteArrayList: thread-safe for read-heavy workloads
        System.out.println("\n=== CopyOnWriteArrayList ===");
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        cowList.add("A");
        cowList.add("B");
        cowList.add("C");
        
        // Safe to iterate while modifying (no ConcurrentModificationException)
        for (String s : cowList) {
            System.out.print(s + " ");
            cowList.add("X");  // adds don't affect current iteration
        }
        System.out.println("\nFinal size: " + cowList.size());
        
        // BlockingQueue: producer-consumer pattern
        System.out.println("\n=== BlockingQueue (Producer-Consumer) ===");
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);
        
        // Producer
        Thread producer = new Thread(() -> {
            String[] items = {"task1", "task2", "task3", "task4", "task5"};
            for (String item : items) {
                try {
                    queue.put(item);
                    System.out.println("Produced: " + item);
                    Thread.sleep(50);
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
            }
        });
        
        // Consumer
        Thread consumer = new Thread(() -> {
            int consumed = 0;
            while (consumed < 5) {
                try {
                    String item = queue.poll(500, TimeUnit.MILLISECONDS);
                    if (item != null) {
                        System.out.println("  Consumed: " + item);
                        consumed++;
                        Thread.sleep(120);  // slower consumer
                    }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
            }
        });
        
        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
        
        exec.shutdown();
        System.out.println("\nDone!");
    }
}
```

---

## Thread Safety Patterns

```java
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class ThreadSafetyPatterns {
    
    // Pattern 1: Immutable objects (inherently thread-safe)
    static final class ImmutablePoint {
        private final double x, y;
        
        ImmutablePoint(double x, double y) {
            this.x = x;
            this.y = y;
        }
        
        // Return new instance instead of modifying
        ImmutablePoint translate(double dx, double dy) {
            return new ImmutablePoint(x + dx, y + dy);
        }
        
        double distanceTo(ImmutablePoint other) {
            return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
        }
        
        @Override public String toString() { return "(" + x + "," + y + ")"; }
    }
    
    // Pattern 2: ReadWriteLock (multiple readers, exclusive writer)
    static class ReadWriteCache<K, V> {
        private final ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
        private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
        private int readCount = 0, writeCount = 0;
        
        V get(K key) {
            rwLock.readLock().lock();
            try {
                readCount++;
                return map.get(key);
            } finally {
                rwLock.readLock().unlock();
            }
        }
        
        void put(K key, V value) {
            rwLock.writeLock().lock();
            try {
                writeCount++;
                map.put(key, value);
            } finally {
                rwLock.writeLock().unlock();
            }
        }
        
        void stats() {
            System.out.printf("Reads: %d, Writes: %d%n", readCount, writeCount);
        }
    }
    
    // Pattern 3: Semaphore (limit concurrent access)
    static class ConnectionPool {
        private final Semaphore semaphore;
        private final int maxConnections;
        
        ConnectionPool(int max) {
            maxConnections = max;
            semaphore = new Semaphore(max, true);  // fair
        }
        
        void execute(Runnable work) throws InterruptedException {
            semaphore.acquire();
            try {
                System.out.printf("[%s] acquired (available: %d)%n",
                    Thread.currentThread().getName(), semaphore.availablePermits());
                work.run();
            } finally {
                semaphore.release();
            }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        // Immutable
        System.out.println("=== Immutable Point ===");
        ImmutablePoint p = new ImmutablePoint(0, 0);
        ImmutablePoint p2 = p.translate(3, 4);
        System.out.println("Original: " + p);
        System.out.println("Translated: " + p2);
        System.out.println("Distance: " + p.distanceTo(p2));
        
        // ReadWriteLock
        System.out.println("\n=== ReadWriteCache ===");
        ReadWriteCache<String, String> cache = new ReadWriteCache<>();
        cache.put("user:1", "Alice");
        cache.put("user:2", "Bob");
        
        // Multiple concurrent readers
        ExecutorService readers = Executors.newFixedThreadPool(5);
        CountDownLatch latch = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            readers.submit(() -> {
                System.out.print("R:" + cache.get("user:1") + " ");
                latch.countDown();
            });
        }
        latch.await();
        System.out.println();
        cache.stats();
        readers.shutdown();
        
        // Semaphore (connection pool)
        System.out.println("\n=== Connection Pool (max 2 concurrent) ===");
        ConnectionPool pool = new ConnectionPool(2);
        ExecutorService exec = Executors.newFixedThreadPool(5);
        CountDownLatch done = new CountDownLatch(5);
        
        for (int i = 1; i <= 5; i++) {
            final int task = i;
            exec.submit(() -> {
                try {
                    pool.execute(() -> {
                        System.out.println("  Working: Task " + task);
                        try { Thread.sleep(100); } catch (InterruptedException e) {}
                    });
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    done.countDown();
                }
            });
        }
        
        done.await();
        exec.shutdown();
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Async HTTP Simulator

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class AsyncHttpSimulator {
    
    record HttpRequest(String method, String url) {}
    record HttpResponse(int statusCode, String body, long latencyMs) {}
    
    static class HttpClient {
        private final ExecutorService executor;
        private final Random rng = new Random(42);
        
        HttpClient(int poolSize) {
            executor = Executors.newFixedThreadPool(poolSize);
        }
        
        CompletableFuture<HttpResponse> get(String url) {
            return CompletableFuture.supplyAsync(() -> {
                long delay = 50 + rng.nextInt(200);
                try { Thread.sleep(delay); } 
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                
                // Simulate 90% success, 10% failure
                if (rng.nextDouble() < 0.1) {
                    throw new RuntimeException("Connection timeout for: " + url);
                }
                
                String body = "{\"url\":\"" + url + "\",\"timestamp\":" + System.currentTimeMillis() + "}";
                return new HttpResponse(200, body, delay);
            }, executor);
        }
        
        void shutdown() { executor.shutdown(); }
    }
    
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        HttpClient client = new HttpClient(10);
        
        String[] urls = {
            "https://api.example.com/users",
            "https://api.example.com/products",
            "https://api.example.com/orders",
            "https://api.example.com/inventory",
            "https://api.example.com/reports"
        };
        
        System.out.println("=== Sequential Requests ===");
        long seqStart = System.currentTimeMillis();
        for (String url : urls) {
            try {
                HttpResponse resp = client.get(url).get();
                System.out.printf("  %s -> %d (%d ms)%n", url.substring(30), resp.statusCode(), resp.latencyMs());
            } catch (ExecutionException e) {
                System.out.printf("  %s -> ERROR: %s%n", url.substring(30), e.getCause().getMessage());
            }
        }
        System.out.printf("Sequential total: %d ms%n", System.currentTimeMillis() - seqStart);
        
        System.out.println("\n=== Parallel Requests ===");
        long parStart = System.currentTimeMillis();
        
        List<CompletableFuture<HttpResponse>> futures = Arrays.stream(urls)
            .map(url -> client.get(url)
                .exceptionally(ex -> new HttpResponse(500, ex.getMessage(), 0)))
            .collect(Collectors.toList());
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
        
        long totalLatency = 0;
        int success = 0, errors = 0;
        
        for (int i = 0; i < futures.size(); i++) {
            HttpResponse resp = futures.get(i).get();
            if (resp.statusCode() == 200) {
                success++;
                totalLatency += resp.latencyMs();
                System.out.printf("  %s -> %d (%d ms)%n", 
                    urls[i].substring(30), resp.statusCode(), resp.latencyMs());
            } else {
                errors++;
                System.out.printf("  %s -> ERROR%n", urls[i].substring(30));
            }
        }
        
        long parTotal = System.currentTimeMillis() - parStart;
        System.out.printf("Parallel total: %d ms (vs %d ms sequential)%n",
            parTotal, System.currentTimeMillis() - seqStart + parTotal);
        System.out.printf("Success: %d, Errors: %d%n", success, errors);
        
        // Fan-out, fan-in pattern
        System.out.println("\n=== Fan-out/Fan-in: Aggregated Data ===");
        
        record UserData(String userId, String profile, String orders, String preferences) {}
        
        String userId = "user123";
        CompletableFuture<String> profileF = client.get("https://api.example.com/users/" + userId + "/profile")
            .exceptionally(ex -> new HttpResponse(500, "fallback_profile", 0))
            .thenApply(r -> "profile_data");
        CompletableFuture<String> ordersF = client.get("https://api.example.com/users/" + userId + "/orders")
            .exceptionally(ex -> new HttpResponse(500, "fallback_orders", 0))
            .thenApply(r -> "orders_data");
        CompletableFuture<String> prefsF = client.get("https://api.example.com/users/" + userId + "/preferences")
            .exceptionally(ex -> new HttpResponse(500, "fallback_prefs", 0))
            .thenApply(r -> "prefs_data");
        
        UserData userData = profileF
            .thenCombine(ordersF, (p, o) -> new String[]{p, o})
            .thenCombine(prefsF, (po, pref) -> new UserData(userId, po[0], po[1], pref))
            .get();
        
        System.out.printf("Aggregated user %s:%n  Profile: %s%n  Orders: %s%n  Prefs: %s%n",
            userData.userId(), userData.profile(), userData.orders(), userData.preferences());
        
        client.shutdown();
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Thread creation (extends Thread, Runnable, lambda)  
✅ Thread lifecycle, daemon threads  
✅ Synchronization (synchronized methods/blocks)  
✅ Race conditions and solutions  
✅ AtomicInteger and atomic operations  
✅ volatile keyword  
✅ ExecutorService (Fixed, Cached, Scheduled)  
✅ CompletableFuture (chain, parallel, error handling, combine)  
✅ Concurrent collections (ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue)  
✅ ReadWriteLock, Semaphore patterns  
✅ Async HTTP Simulator  

---

## ขั้นตอนต่อไป

**Part 017:** Java 8-21 Modern Features  
- Optional, Streams (review)  
- Records (Java 16+)  
- Sealed classes (Java 17+)  
- Pattern matching (instanceof, switch)  
- Text blocks (Java 15+)  
- Virtual threads (Java 21)  

---

*Part 016 | Java & Spring Boot Course | สร้างโดย Claude Code*
