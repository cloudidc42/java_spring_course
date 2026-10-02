# Part 103: JVM Internals and Optimization

## เนื้อหาในส่วนนี้
- JVM Architecture Overview
- ClassLoader System
- Memory Model (Heap, Stack, Metaspace)
- Garbage Collection Algorithms
- JVM Flags for Tuning
- JIT Compilation
- Java Memory Model (JMM) and Concurrency
- Profiling and Diagnostics
- JVM Tools

---

## 1. JVM Architecture

```
JVM Internal Architecture:

┌─────────────────────────────────────────┐
│           Class Loader Subsystem         │
│  Bootstrap → Extension → Application    │
└──────────────────────┬──────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│           Runtime Data Areas             │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Heap    │  │  Stack   │  │ Meta   │ │
│  │ Young Gen│  │ (per     │  │ Space  │ │
│  │ Old Gen  │  │ thread)  │  │        │ │
│  └──────────┘  └──────────┘  └────────┘ │
│  ┌──────────────────────────────────────┐│
│  │   Program Counter (per thread)       ││
│  └──────────────────────────────────────┘│
└──────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────┐
│        Execution Engine                  │
│  JIT Compiler + Interpreter + GC         │
└──────────────────────────────────────────┘
```

---

## 2. Memory Management

```java
// Heap memory demo
public class HeapMemoryDemo {
    
    // Young Generation (Eden + Survivor S0/S1)
    // - New objects start here
    // - Minor GC: fast, frequent
    // - Short-lived objects die here
    
    // Old Generation (Tenured)
    // - Survived multiple Minor GCs
    // - Major GC: slower, less frequent
    
    // Metaspace (outside heap in Java 8+)
    // - Class metadata (was PermGen in Java 7-)
    // - Grows dynamically (no PermGen OutOfMemory)
    
    public static void heapDemo() {
        // Object lifecycle:
        List<byte[]> young = new ArrayList<>();  // Starts in Eden
        
        for (int i = 0; i < 100; i++) {
            young.add(new byte[1024]);  // 1KB each
            if (young.size() > 50) {
                young.remove(0);  // Short-lived → dies in Young Gen
            }
        }
        
        // Long-lived objects → promoted to Old Gen
        byte[] longLived = new byte[10 * 1024 * 1024];  // 10MB
        System.out.println("Long-lived object: " + longLived.length + " bytes");
    }
    
    public static void main(String[] args) {
        // JVM flags to observe:
        // -Xms512m         initial heap
        // -Xmx2g           max heap
        // -XX:NewRatio=3   Old:Young = 3:1 → Young = 25% of heap
        // -XX:SurvivorRatio=8  Eden:Survivor = 8:1 → each Survivor = 1/10 Young
        
        System.out.println("Heap: " + Runtime.getRuntime().totalMemory() / 1024 / 1024 + "MB");
        System.out.println("Max Heap: " + Runtime.getRuntime().maxMemory() / 1024 / 1024 + "MB");
        System.out.println("Free Heap: " + Runtime.getRuntime().freeMemory() / 1024 / 1024 + "MB");
    }
}
```

---

## 3. Garbage Collection Algorithms

```bash
# G1GC (Default since Java 9+): Balanced throughput/latency
# Best for: most applications, heaps 4GB+
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=4m \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -jar myapp.jar

# ZGC (Java 15+): Sub-millisecond pauses
# Best for: latency-sensitive, very large heaps (TB scale)
java -XX:+UseZGC \
     -XX:ZCollectionInterval=5 \
     -XX:ZAllocationSpikeTolerance=5 \
     -jar myapp.jar

# Shenandoah (Java 12+, Red Hat): Low pause concurrent
# Best for: consistent low pause, OpenJDK builds
java -XX:+UseShenandoahGC \
     -XX:ShenandoahUnloadClassesFrequency=1 \
     -jar myapp.jar

# SerialGC: Single-threaded, minimal overhead
# Best for: small applications, single-core containers
java -XX:+UseSerialGC -jar myapp.jar

# ParallelGC: High throughput, stop-the-world
# Best for: batch processing, non-interactive
java -XX:+UseParallelGC \
     -XX:ParallelGCThreads=4 \
     -jar myapp.jar
```

```java
// GC tuning in Spring Boot containers
public class GcTuning {
    
    public static void main(String[] args) {
        // Print GC logs (Java 11+)
        // -Xlog:gc*:file=/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
        
        // GC notification listener
        for (java.lang.management.GarbageCollectorMXBean gc :
                java.lang.management.ManagementFactory.getGarbageCollectorMXBeans()) {
            System.out.println("GC: " + gc.getName() + 
                ", Collections: " + gc.getCollectionCount() +
                ", Time: " + gc.getCollectionTime() + "ms");
        }
        
        // Memory notification
        java.lang.management.MemoryMXBean memBean = 
            java.lang.management.ManagementFactory.getMemoryMXBean();
        java.lang.management.MemoryUsage heap = memBean.getHeapMemoryUsage();
        
        System.out.printf("Heap: used=%dMB, committed=%dMB, max=%dMB%n",
            heap.getUsed() / 1_000_000,
            heap.getCommitted() / 1_000_000,
            heap.getMax() / 1_000_000);
    }
}
```

---

## 4. JIT Compilation

```java
public class JitDemo {
    
    // JIT compilation tiers:
    // Tier 0: Interpreter (slowest)
    // Tier 1: C1 simple JIT (client compiler)
    // Tier 2: C1 with profiling
    // Tier 3: C1 with full profiling
    // Tier 4: C2 aggressive optimization (server compiler)
    
    // JIT optimizations:
    // - Inlining: replace method calls with method body
    // - Loop unrolling: reduce loop overhead
    // - Dead code elimination
    // - Escape analysis: stack-allocate non-escaping objects
    // - Branch prediction
    
    // Hot method detection: >10,000 invocations → JIT compiled
    
    public static int sum(int n) {
        int total = 0;
        for (int i = 0; i <= n; i++) {
            total += i;
        }
        return total;
    }
    
    public static void main(String[] args) {
        // Warm up JIT
        for (int i = 0; i < 100_000; i++) {
            sum(1000);  // After ~10,000 calls, JIT compiles this method
        }
        
        // Now runs at native speed
        long start = System.nanoTime();
        int result = sum(1_000_000);
        long duration = System.nanoTime() - start;
        
        System.out.println("Result: " + result + " in " + duration + "ns");
        
        // JVM flags for JIT tuning:
        // -XX:CompileThreshold=10000       invocations before JIT
        // -XX:+PrintCompilation            print JIT activity
        // -XX:+PrintInlining               print inlining decisions
        // -XX:-TieredCompilation           use only C2
        // -XX:+UseStringDeduplication      deduplicate String objects
    }
}
```

---

## 5. Java Memory Model (JMM)

```java
import java.util.concurrent.atomic.*;
import java.util.concurrent.*;

public class JmmDemo {
    
    // Visibility problem: without volatile/synchronized, 
    // thread may see stale value from CPU cache
    
    // BAD: visibility not guaranteed
    static boolean running = true;  // Non-volatile!
    
    // GOOD: volatile ensures visibility
    static volatile boolean runningVolatile = true;
    
    // GOOD: AtomicBoolean for visibility + atomicity
    static AtomicBoolean runningAtomic = new AtomicBoolean(true);
    
    public static void visibilityDemo() throws InterruptedException {
        // Thread might never see running=false without volatile!
        Thread t = new Thread(() -> {
            while (runningVolatile) {
                // Do work
            }
            System.out.println("Thread stopped (saw volatile update)");
        });
        t.start();
        
        Thread.sleep(100);
        runningVolatile = false;  // Visible to other threads
        t.join();
    }
    
    // Happens-before relationships:
    // 1. Program order: a → b (same thread)
    // 2. Monitor lock: unlock → subsequent lock
    // 3. Volatile write → subsequent volatile read
    // 4. Thread start → first action of started thread
    // 5. Last action → thread.join() return
    
    // Safe publication idioms
    static class SafePublication {
        // 1. static final field (initialized in static initializer)
        static final Object OBJ = new Object();
        
        // 2. volatile field
        static volatile Object volObj;
        
        // 3. Atomic reference
        static AtomicReference<Object> atomicObj = new AtomicReference<>();
        
        // 4. Lock-protected
        static final Object lock = new Object();
        static Object lockedObj;
        
        static void init(Object o) {
            synchronized (lock) { lockedObj = o; }
        }
        
        static Object get() {
            synchronized (lock) { return lockedObj; }
        }
    }
    
    // Double-checked locking (correct with volatile)
    static class Singleton {
        private static volatile Singleton instance;
        
        static Singleton getInstance() {
            if (instance == null) {                  // First check (no lock)
                synchronized (Singleton.class) {
                    if (instance == null) {          // Second check (with lock)
                        instance = new Singleton();  // volatile write
                    }
                }
            }
            return instance;
        }
    }
    
    // Lock-free programming with CAS
    static class LockFreeCounter {
        private final AtomicLong count = new AtomicLong(0);
        
        public long increment() {
            return count.incrementAndGet();  // Atomic CAS operation
        }
        
        public long addIfLessThan(long value, long threshold) {
            long current;
            do {
                current = count.get();
                if (current >= threshold) return current;
            } while (!count.compareAndSet(current, current + value));
            return current + value;
        }
    }
}
```

---

## 6. JVM Diagnostic Tools

```bash
# jps: list Java processes
jps -l

# jstack: thread dump
jstack <pid>                    # Print thread dump to stdout
jstack -l <pid>                 # Include lock info
jstack <pid> > thread-dump.txt  # Save to file

# jmap: heap analysis
jmap -heap <pid>               # Heap summary
jmap -histo <pid>              # Object histogram
jmap -dump:format=b,file=heap.hprof <pid>  # Full heap dump

# jstat: JVM statistics
jstat -gc <pid> 1000 10         # GC stats every 1s, 10 times
jstat -gcutil <pid> 1000        # GC utilization %

# jcmd: versatile tool
jcmd <pid> VM.flags              # JVM flags
jcmd <pid> VM.system_properties  # System properties
jcmd <pid> GC.run                # Force GC
jcmd <pid> Thread.print          # Thread dump
jcmd <pid> VM.heap_info          # Heap info (Java 14+)
jcmd <pid> JFR.start name=myrecording  # Start JFR

# Java Flight Recorder (JFR)
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=filename=myapp.jfr,duration=60s \
     -jar myapp.jar

# Async-profiler (CPU + allocation profiling)
./profiler.sh -d 30 -e cpu -f profile.html <pid>
./profiler.sh -d 30 -e alloc -f alloc.html <pid>
```

```java
// Programmatic heap dump and monitoring
import java.lang.management.*;
import javax.management.*;
import java.lang.reflect.UndeclaredThrowableException;

public class JvmMonitor {
    
    public static void heapDump(String filename) throws Exception {
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();
        ObjectName hotspotDiagnostics = 
            new ObjectName("com.sun.management:type=HotSpotDiagnostic");
        server.invoke(hotspotDiagnostics, "dumpHeap",
            new Object[]{filename, true},
            new String[]{String.class.getName(), boolean.class.getName()});
        System.out.println("Heap dump saved to: " + filename);
    }
    
    public static void printMemoryInfo() {
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        MemoryUsage nonHeap = memBean.getNonHeapMemoryUsage();
        
        System.out.printf("Heap - used: %dMB, max: %dMB (%.1f%%)%n",
            heap.getUsed() >> 20, heap.getMax() >> 20,
            100.0 * heap.getUsed() / heap.getMax());
        System.out.printf("Non-heap (Metaspace) - used: %dMB%n",
            nonHeap.getUsed() >> 20);
    }
    
    public static void detectMemoryLeak() {
        // Alert if heap > 90%
        MemoryMXBean memBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heap = memBean.getHeapMemoryUsage();
        double usagePercent = 100.0 * heap.getUsed() / heap.getMax();
        
        if (usagePercent > 90) {
            System.err.println("ALERT: Heap usage at " + usagePercent + "% - possible memory leak!");
            try {
                heapDump("/app/emergency-dump.hprof");
            } catch (Exception e) {
                System.err.println("Failed to dump heap: " + e.getMessage());
            }
        }
    }
    
    public static void printThreadInfo() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        System.out.println("Active threads: " + threadBean.getThreadCount());
        System.out.println("Peak threads: " + threadBean.getPeakThreadCount());
        System.out.println("Total started: " + threadBean.getTotalStartedThreadCount());
        
        // Find deadlocks
        long[] deadlocked = threadBean.findDeadlockedThreads();
        if (deadlocked != null) {
            System.err.println("DEADLOCK DETECTED! Threads: " + deadlocked.length);
            ThreadInfo[] info = threadBean.getThreadInfo(deadlocked, true, true);
            for (ThreadInfo ti : info) {
                System.err.println(ti);
            }
        }
    }
}
```

---

## 7. Common Performance Issues

```java
public class CommonIssues {
    
    // 1. String concatenation in loops (BAD)
    static String badConcat(List<String> items) {
        String result = "";
        for (String item : items) {
            result += item + ",";  // Creates new String each iteration!
        }
        return result;
    }
    
    // GOOD: use StringBuilder
    static String goodConcat(List<String> items) {
        StringBuilder sb = new StringBuilder();
        for (String item : items) {
            sb.append(item).append(',');
        }
        return sb.toString();
    }
    
    // 2. Boxing/Unboxing in loops (BAD)
    static long badSum(List<Integer> numbers) {
        Long sum = 0L;  // Autoboxing on every iteration!
        for (Integer n : numbers) {
            sum += n;  // Unbox Integer, add, box Long
        }
        return sum;
    }
    
    // GOOD: use primitives
    static long goodSum(List<Integer> numbers) {
        long sum = 0L;
        for (int n : numbers) {
            sum += n;  // Fast primitive operations
        }
        return sum;
    }
    
    // 3. Creating objects in tight loops (BAD)
    static void badObjectCreation() {
        for (int i = 0; i < 1_000_000; i++) {
            new java.awt.Rectangle(i, i, 100, 100);  // GC pressure!
        }
    }
    
    // GOOD: reuse objects
    static void goodObjectReuse() {
        java.awt.Rectangle rect = new java.awt.Rectangle();  // Reuse
        for (int i = 0; i < 1_000_000; i++) {
            rect.setBounds(i, i, 100, 100);  // No GC pressure
        }
    }
    
    // 4. Inefficient ArrayList iteration
    static int badLinkedListAccess(java.util.LinkedList<Integer> list) {
        int sum = 0;
        for (int i = 0; i < list.size(); i++) {
            sum += list.get(i);  // O(n) per access = O(n²) total!
        }
        return sum;
    }
    
    // GOOD: use iterator
    static int goodIterator(java.util.LinkedList<Integer> list) {
        int sum = 0;
        for (int n : list) {  // Uses iterator = O(n) total
            sum += n;
        }
        return sum;
    }
}
```

---

## สรุป Part 103

| Concept | Key Points |
|---------|-----------|
| Heap | Young Gen (Eden+S0/S1) + Old Gen |
| Metaspace | Class metadata (no PermGen OOM) |
| G1GC | Default, balanced, ~200ms pause target |
| ZGC | Sub-ms pauses, large heaps |
| JIT | Tier 0-4, hot after 10K invocations |
| JMM | volatile = visibility, synchronized = visibility+atomicity |
| jstack | Thread dump for deadlock analysis |
| jmap | Heap dump for memory leak analysis |
| JFR | Low-overhead production profiling |

---

**Part 104:** Java Security - Cryptography, TLS, Secure Coding
