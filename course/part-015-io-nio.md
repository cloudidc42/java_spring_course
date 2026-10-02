# Part 015: Java I/O & NIO.2
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Java I/O Overview](#java-io-overview)
2. [File Operations (java.nio.file)](#file-operations-javaniofile)
3. [Reading and Writing Text Files](#reading-and-writing-text-files)
4. [Reading and Writing Binary Files](#reading-and-writing-binary-files)
5. [File System Operations](#file-system-operations)
6. [WatchService](#watchservice)
7. [Object Serialization](#object-serialization)
8. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Java I/O Overview

```
java.io (Classic)              java.nio.file (Modern, Java 7+)
───────────────────            ───────────────────────────────
File                    →      Path, Paths, Files
FileInputStream         →      Files.newInputStream()
FileOutputStream        →      Files.newOutputStream()
BufferedReader          →      Files.newBufferedReader()
BufferedWriter          →      Files.newBufferedWriter()
Scanner                 →      Files.readAllLines() / readString()
FileWriter              →      Files.writeString() / write()
```

---

## File Operations (java.nio.file)

```java
import java.nio.file.*;
import java.io.*;
import java.util.*;

public class FileOperations {
    
    public static void main(String[] args) throws IOException {
        // Path creation
        Path current = Paths.get(".");
        Path absolute = Paths.get("/home/user/data");
        Path relative = Paths.get("docs", "report.txt");
        Path fromUri = Path.of("README.md");
        
        System.out.println("Current: " + current.toAbsolutePath());
        System.out.println("Relative: " + relative);
        
        // Path operations
        Path base = Paths.get("/home/user/projects/myapp");
        System.out.println("Parent: " + base.getParent());
        System.out.println("FileName: " + base.getFileName());
        System.out.println("Root: " + base.getRoot());
        System.out.println("Name count: " + base.getNameCount());
        System.out.println("Name(1): " + base.getName(1));
        
        // Resolve (combine paths)
        Path config = base.resolve("config/app.properties");
        System.out.println("Resolved: " + config);
        
        // Normalize (remove ./ and ../)
        Path messy = Paths.get("/home/user/../user/./projects");
        System.out.println("Normalized: " + messy.normalize());
        
        // Check path
        Path file = Paths.get("test.txt");
        System.out.println("\nFile checks:");
        System.out.println("exists: " + Files.exists(file));
        System.out.println("isFile: " + Files.isRegularFile(file));
        System.out.println("isDirectory: " + Files.isDirectory(file));
        System.out.println("isReadable: " + Files.isReadable(file));
        System.out.println("isWritable: " + Files.isWritable(file));
        
        // Create and delete
        Path tempDir = Files.createTempDirectory("myapp_");
        System.out.println("\nTemp dir: " + tempDir);
        
        Path tempFile = Files.createTempFile(tempDir, "test_", ".txt");
        System.out.println("Temp file: " + tempFile);
        
        // File metadata
        if (Files.exists(tempFile)) {
            System.out.println("Size: " + Files.size(tempFile));
            System.out.println("Last modified: " + Files.getLastModifiedTime(tempFile));
        }
        
        // Cleanup
        Files.delete(tempFile);
        Files.delete(tempDir);
        System.out.println("Cleaned up temp files");
    }
}
```

---

## Reading and Writing Text Files

```java
import java.nio.file.*;
import java.nio.charset.*;
import java.io.*;
import java.util.*;
import java.util.stream.*;

public class TextFileOperations {
    
    public static void main(String[] args) throws IOException {
        Path path = Paths.get("sample.txt");
        
        // ======================
        // WRITING
        // ======================
        
        // Method 1: writeString (simplest, Java 11+)
        Files.writeString(path, "Hello, World!\nLine 2\nLine 3\n");
        System.out.println("Written with writeString");
        
        // Method 2: write with List<String>
        List<String> lines = List.of("Alice,30,Engineering", "Bob,25,Marketing", "Charlie,35,HR");
        Files.write(path, lines, StandardCharsets.UTF_8, 
            StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
        System.out.println("Written with write(List)");
        
        // Method 3: BufferedWriter (for large files)
        try (BufferedWriter writer = Files.newBufferedWriter(path, StandardCharsets.UTF_8)) {
            writer.write("Name,Age,Department");
            writer.newLine();
            for (String line : lines) {
                writer.write(line);
                writer.newLine();
            }
        }
        System.out.println("Written with BufferedWriter");
        
        // Method 4: PrintWriter (printf formatting)
        try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(path))) {
            pw.println("Name,Age,Department");
            pw.printf("%-10s,%3d,%-12s%n", "Alice", 30, "Engineering");
            pw.printf("%-10s,%3d,%-12s%n", "Bob", 25, "Marketing");
        }
        System.out.println("Written with PrintWriter");
        
        // Append mode
        Files.writeString(path, "\nExtra line\n", 
            StandardOpenOption.APPEND);
        
        // ======================
        // READING
        // ======================
        System.out.println("\n=== Reading ===");
        
        // Method 1: readString (entire file, Java 11+)
        String content = Files.readString(path);
        System.out.println("readString:\n" + content);
        
        // Method 2: readAllLines (all lines as List)
        List<String> allLines = Files.readAllLines(path, StandardCharsets.UTF_8);
        System.out.println("readAllLines: " + allLines.size() + " lines");
        allLines.forEach(l -> System.out.println("  " + l));
        
        // Method 3: lines() as Stream (lazy, good for large files)
        System.out.println("\nlines() stream:");
        try (Stream<String> lineStream = Files.lines(path)) {
            lineStream
                .filter(l -> !l.isEmpty())
                .map(String::toUpperCase)
                .forEach(System.out::println);
        }
        
        // Method 4: BufferedReader (classic, for large files)
        try (BufferedReader reader = Files.newBufferedReader(path)) {
            String line;
            int lineNum = 1;
            while ((line = reader.readLine()) != null) {
                System.out.printf("[%02d] %s%n", lineNum++, line);
            }
        }
        
        // CSV parsing example
        System.out.println("\n=== CSV Parsing ===");
        record Person(String name, int age, String dept) {}
        
        List<Person> people = Files.readAllLines(path).stream()
            .skip(1)  // skip header
            .filter(l -> !l.isBlank())
            .map(l -> l.trim().split(","))
            .filter(parts -> parts.length >= 3)
            .map(parts -> new Person(
                parts[0].trim(),
                Integer.parseInt(parts[1].trim()),
                parts[2].trim()))
            .collect(Collectors.toList());
        
        people.forEach(p -> System.out.printf("  %s, %d, %s%n", p.name(), p.age(), p.dept()));
        
        // Cleanup
        Files.delete(path);
    }
}
```

---

## Reading and Writing Binary Files

```java
import java.nio.file.*;
import java.io.*;
import java.nio.*;
import java.nio.channels.*;
import java.util.*;

public class BinaryFileOperations {
    
    public static void main(String[] args) throws IOException {
        Path binaryFile = Paths.get("data.bin");
        
        // Write binary data
        try (DataOutputStream dos = new DataOutputStream(
                new BufferedOutputStream(Files.newOutputStream(binaryFile)))) {
            dos.writeInt(42);
            dos.writeDouble(3.14159);
            dos.writeBoolean(true);
            dos.writeUTF("Hello Binary");
            dos.writeLong(System.currentTimeMillis());
        }
        System.out.println("Written binary file: " + Files.size(binaryFile) + " bytes");
        
        // Read binary data
        try (DataInputStream dis = new DataInputStream(
                new BufferedInputStream(Files.newInputStream(binaryFile)))) {
            System.out.println("Int: " + dis.readInt());
            System.out.println("Double: " + dis.readDouble());
            System.out.println("Boolean: " + dis.readBoolean());
            System.out.println("String: " + dis.readUTF());
            System.out.println("Long: " + dis.readLong());
        }
        
        // Read all bytes
        byte[] bytes = Files.readAllBytes(binaryFile);
        System.out.println("Total bytes: " + bytes.length);
        
        // NIO ByteBuffer
        Path nioFile = Paths.get("nio_data.bin");
        try (FileChannel channel = FileChannel.open(nioFile, 
                StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            buffer.putInt(100);
            buffer.putDouble(2.718);
            buffer.putLong(999L);
            buffer.flip();  // switch to read mode
            channel.write(buffer);
        }
        
        try (FileChannel channel = FileChannel.open(nioFile, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            channel.read(buffer);
            buffer.flip();  // switch to read mode
            System.out.println("\nNIO read:");
            System.out.println("Int: " + buffer.getInt());
            System.out.println("Double: " + buffer.getDouble());
            System.out.println("Long: " + buffer.getLong());
        }
        
        // Memory-mapped file (fast for large files)
        try (RandomAccessFile raf = new RandomAccessFile(nioFile.toFile(), "r");
             FileChannel fc = raf.getChannel()) {
            MappedByteBuffer mmap = fc.map(FileChannel.MapMode.READ_ONLY, 0, fc.size());
            System.out.println("\nMemory-mapped size: " + mmap.limit() + " bytes");
        }
        
        // Cleanup
        Files.deleteIfExists(binaryFile);
        Files.deleteIfExists(nioFile);
    }
}
```

---

## File System Operations

```java
import java.nio.file.*;
import java.nio.file.attribute.*;
import java.io.*;
import java.util.*;
import java.util.stream.*;

public class FileSystemOps {
    
    public static void main(String[] args) throws IOException {
        Path workDir = Files.createTempDirectory("fsops_");
        
        // Create directory structure
        Path src = workDir.resolve("src/main/java");
        Path test = workDir.resolve("src/test/java");
        Path resources = workDir.resolve("src/main/resources");
        
        Files.createDirectories(src);
        Files.createDirectories(test);
        Files.createDirectories(resources);
        
        // Create files
        Files.writeString(src.resolve("Main.java"), "public class Main {}");
        Files.writeString(src.resolve("Service.java"), "public class Service {}");
        Files.writeString(test.resolve("MainTest.java"), "class MainTest {}");
        Files.writeString(resources.resolve("application.properties"), "server.port=8080");
        
        System.out.println("Created directory structure");
        
        // List directory
        System.out.println("\nDirect children of src:");
        try (DirectoryStream<Path> stream = Files.newDirectoryStream(workDir.resolve("src"))) {
            for (Path entry : stream) {
                System.out.println("  " + entry.getFileName() + 
                    (Files.isDirectory(entry) ? "/" : ""));
            }
        }
        
        // Walk file tree
        System.out.println("\nAll files:");
        try (Stream<Path> walk = Files.walk(workDir)) {
            walk.filter(Files::isRegularFile)
                .forEach(p -> System.out.println("  " + workDir.relativize(p)));
        }
        
        // Find files by glob pattern
        System.out.println("\nJava files:");
        try (Stream<Path> found = Files.find(workDir, 10, 
                (path, attr) -> path.toString().endsWith(".java"))) {
            found.forEach(p -> System.out.println("  " + workDir.relativize(p)));
        }
        
        // Copy
        Path destDir = Files.createTempDirectory("fsdest_");
        Files.copy(src.resolve("Main.java"), destDir.resolve("Main.java"));
        System.out.println("\nCopied Main.java to " + destDir.getFileName());
        
        // Copy directory recursively
        copyDirectory(workDir.resolve("src"), destDir.resolve("src"));
        System.out.println("Copied entire src/ directory");
        
        // Move/Rename
        Path renamed = workDir.resolve("src/main/java/App.java");
        Files.move(src.resolve("Main.java"), renamed, StandardCopyOption.REPLACE_EXISTING);
        System.out.println("Renamed Main.java to App.java");
        
        // File attributes
        Path appFile = renamed;
        BasicFileAttributes attrs = Files.readAttributes(appFile, BasicFileAttributes.class);
        System.out.println("\nFile attributes for App.java:");
        System.out.println("  Size: " + attrs.size() + " bytes");
        System.out.println("  Created: " + attrs.creationTime());
        System.out.println("  Modified: " + attrs.lastModifiedTime());
        System.out.println("  Regular file: " + attrs.isRegularFile());
        
        // Cleanup
        deleteDirectory(workDir);
        deleteDirectory(destDir);
        System.out.println("\nCleaned up");
    }
    
    static void copyDirectory(Path src, Path dest) throws IOException {
        Files.createDirectories(dest);
        try (Stream<Path> stream = Files.walk(src)) {
            stream.forEach(path -> {
                try {
                    Path target = dest.resolve(src.relativize(path));
                    if (Files.isDirectory(path)) {
                        Files.createDirectories(target);
                    } else {
                        Files.copy(path, target, StandardCopyOption.REPLACE_EXISTING);
                    }
                } catch (IOException e) {
                    throw new UncheckedIOException(e);
                }
            });
        }
    }
    
    static void deleteDirectory(Path dir) throws IOException {
        try (Stream<Path> walk = Files.walk(dir)) {
            walk.sorted(Comparator.reverseOrder())  // delete files before dirs
                .forEach(path -> {
                    try { Files.delete(path); }
                    catch (IOException e) { /* ignore */ }
                });
        }
    }
}
```

---

## WatchService

```java
import java.nio.file.*;
import java.io.*;
import java.util.concurrent.*;

public class FileWatcherDemo {
    
    public static void main(String[] args) throws IOException, InterruptedException {
        Path watchDir = Files.createTempDirectory("watch_");
        System.out.println("Watching directory: " + watchDir);
        
        WatchService watcher = FileSystems.getDefault().newWatchService();
        watchDir.register(watcher, 
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY,
            StandardWatchEventKinds.ENTRY_DELETE);
        
        // Simulate file changes in background
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        executor.schedule(() -> {
            try {
                Path file = watchDir.resolve("test.txt");
                Files.writeString(file, "Initial content");
                Thread.sleep(200);
                Files.writeString(file, "Modified content", StandardOpenOption.APPEND);
                Thread.sleep(200);
                Files.delete(file);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }, 500, TimeUnit.MILLISECONDS);
        
        // Watch for 5 events or 3 seconds
        int events = 0;
        long deadline = System.currentTimeMillis() + 3000;
        
        while (events < 5 && System.currentTimeMillis() < deadline) {
            WatchKey key = watcher.poll(500, TimeUnit.MILLISECONDS);
            if (key == null) continue;
            
            for (WatchEvent<?> event : key.pollEvents()) {
                WatchEvent.Kind<?> kind = event.kind();
                
                if (kind == StandardWatchEventKinds.OVERFLOW) continue;
                
                @SuppressWarnings("unchecked")
                WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
                Path changed = pathEvent.context();
                
                System.out.printf("[EVENT] %s: %s%n", kind.name(), changed);
                events++;
            }
            
            key.reset();
        }
        
        executor.shutdown();
        watcher.close();
        Files.deleteIfExists(watchDir);
        System.out.println("Watcher stopped");
    }
}
```

---

## Object Serialization

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;

public class SerializationDemo {
    
    // Serializable class
    static class User implements Serializable {
        @Serial
        private static final long serialVersionUID = 1L;
        
        private String name;
        private int age;
        private String email;
        private transient String password;  // not serialized
        
        User(String name, int age, String email, String password) {
            this.name = name;
            this.age = age;
            this.email = email;
            this.password = password;
        }
        
        @Override
        public String toString() {
            return String.format("User{name='%s', age=%d, email='%s', password='%s'}",
                name, age, email, password);
        }
    }
    
    // Custom serialization
    static class SecureUser implements Serializable {
        @Serial
        private static final long serialVersionUID = 1L;
        
        private String username;
        private String encryptedData;
        
        SecureUser(String username, String data) {
            this.username = username;
            this.encryptedData = "ENCRYPTED:" + data;
        }
        
        @Serial
        private void writeObject(ObjectOutputStream oos) throws IOException {
            oos.defaultWriteObject();
            System.out.println("Custom writeObject called");
        }
        
        @Serial
        private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
            ois.defaultReadObject();
            System.out.println("Custom readObject called");
        }
        
        @Override
        public String toString() {
            return "SecureUser{username='" + username + "', data='" + encryptedData + "'}";
        }
    }
    
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Path file = Paths.get("users.ser");
        
        // Serialize
        List<User> users = List.of(
            new User("Alice", 30, "alice@example.com", "secret123"),
            new User("Bob", 25, "bob@example.com", "password456")
        );
        
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new BufferedOutputStream(Files.newOutputStream(file)))) {
            oos.writeObject(users);
        }
        System.out.println("Serialized " + users.size() + " users");
        System.out.println("File size: " + Files.size(file) + " bytes");
        
        // Deserialize
        @SuppressWarnings("unchecked")
        List<User> loaded;
        try (ObjectInputStream ois = new ObjectInputStream(
                new BufferedInputStream(Files.newInputStream(file)))) {
            loaded = (List<User>) ois.readObject();
        }
        
        System.out.println("\nDeserialized users:");
        loaded.forEach(u -> System.out.println("  " + u));
        System.out.println("Note: password is null (transient)");
        
        // Custom serialization
        Path securePath = Paths.get("secure.ser");
        SecureUser su = new SecureUser("admin", "sensitive_data");
        
        try (ObjectOutputStream oos = new ObjectOutputStream(
                Files.newOutputStream(securePath))) {
            oos.writeObject(su);
        }
        
        try (ObjectInputStream ois = new ObjectInputStream(
                Files.newInputStream(securePath))) {
            SecureUser loaded2 = (SecureUser) ois.readObject();
            System.out.println("\nLoaded: " + loaded2);
        }
        
        // Cleanup
        Files.deleteIfExists(file);
        Files.deleteIfExists(securePath);
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Simple File-Based Config System

```java
import java.nio.file.*;
import java.util.*;
import java.io.*;
import java.util.stream.*;

public class ConfigSystem {
    
    // Simple .properties file handler
    static class Config {
        private final Path configFile;
        private final Map<String, String> props = new LinkedHashMap<>();
        
        Config(Path configFile) throws IOException {
            this.configFile = configFile;
            if (Files.exists(configFile)) {
                load();
            }
        }
        
        private void load() throws IOException {
            try (Stream<String> lines = Files.lines(configFile)) {
                lines.map(String::trim)
                    .filter(l -> !l.isEmpty() && !l.startsWith("#"))
                    .forEach(line -> {
                        int eq = line.indexOf('=');
                        if (eq > 0) {
                            String key = line.substring(0, eq).trim();
                            String value = line.substring(eq + 1).trim();
                            props.put(key, value);
                        }
                    });
            }
            System.out.println("Loaded " + props.size() + " properties from " + configFile.getFileName());
        }
        
        void save() throws IOException {
            List<String> lines = props.entrySet().stream()
                .map(e -> e.getKey() + "=" + e.getValue())
                .collect(Collectors.toList());
            Files.write(configFile, lines, StandardOpenOption.CREATE, 
                StandardOpenOption.TRUNCATE_EXISTING);
            System.out.println("Saved " + props.size() + " properties");
        }
        
        String get(String key, String defaultValue) {
            return props.getOrDefault(key, defaultValue);
        }
        
        int getInt(String key, int defaultValue) {
            try {
                return Integer.parseInt(props.getOrDefault(key, String.valueOf(defaultValue)));
            } catch (NumberFormatException e) {
                return defaultValue;
            }
        }
        
        boolean getBoolean(String key, boolean defaultValue) {
            String val = props.get(key);
            if (val == null) return defaultValue;
            return "true".equalsIgnoreCase(val) || "1".equals(val) || "yes".equalsIgnoreCase(val);
        }
        
        void set(String key, Object value) {
            props.put(key, String.valueOf(value));
        }
        
        void print() {
            System.out.println("\n=== Configuration ===");
            props.forEach((k, v) -> System.out.printf("  %-30s = %s%n", k, v));
        }
    }
    
    // Application that uses the config
    static class AppServer {
        private final Config config;
        
        AppServer(Config config) {
            this.config = config;
        }
        
        void start() {
            String host = config.get("server.host", "localhost");
            int port = config.getInt("server.port", 8080);
            boolean debug = config.getBoolean("app.debug", false);
            String dbUrl = config.get("db.url", "jdbc:h2:mem:test");
            
            System.out.println("\n=== Starting AppServer ===");
            System.out.println("Host: " + host);
            System.out.println("Port: " + port);
            System.out.println("Debug: " + debug);
            System.out.println("DB: " + dbUrl);
            System.out.println("Server started successfully!");
        }
    }
    
    public static void main(String[] args) throws IOException {
        Path configPath = Paths.get("app.properties");
        
        // Create default config
        Config config = new Config(configPath);
        
        // Set default values if not present
        if (config.get("server.host", null) == null) {
            config.set("server.host", "localhost");
            config.set("server.port", "8080");
            config.set("app.debug", "false");
            config.set("app.name", "MyApplication");
            config.set("db.url", "jdbc:postgresql://localhost:5432/mydb");
            config.set("db.pool.size", "10");
            config.set("log.level", "INFO");
            config.set("log.file", "app.log");
            config.save();
        }
        
        config.print();
        
        AppServer server = new AppServer(config);
        server.start();
        
        // Modify and save
        System.out.println("\n--- Updating config ---");
        config.set("app.debug", "true");
        config.set("server.port", "9090");
        config.save();
        
        // Reload
        Config reloaded = new Config(configPath);
        reloaded.print();
        
        // Cleanup
        Files.deleteIfExists(configPath);
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Java I/O vs NIO.2  
✅ Path, Paths, Files operations  
✅ Text file read/write (5 methods)  
✅ Binary file I/O with DataInputStream/OutputStream  
✅ NIO ByteBuffer and FileChannel  
✅ Directory traversal (walk, find, DirectoryStream)  
✅ File copy, move, delete recursively  
✅ WatchService for file monitoring  
✅ Object Serialization  
✅ Config System example  

---

## ขั้นตอนต่อไป

**Part 016:** Concurrency & Multithreading  
- Thread creation (extends Thread, implements Runnable)  
- Synchronization, volatile  
- java.util.concurrent (ExecutorService, Future, CompletableFuture)  
- Thread safety patterns  

---

*Part 015 | Java & Spring Boot Course | สร้างโดย Claude Code*
