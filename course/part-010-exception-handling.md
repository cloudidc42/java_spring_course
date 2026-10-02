# Part 010: Exception Handling
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Exception คืออะไร?](#exception-คืออะไร)
2. [Exception Hierarchy](#exception-hierarchy)
3. [try-catch-finally](#try-catch-finally)
4. [Checked vs Unchecked Exceptions](#checked-vs-unchecked-exceptions)
5. [Custom Exceptions](#custom-exceptions)
6. [try-with-resources](#try-with-resources)
7. [Exception Chaining](#exception-chaining)
8. [Multi-catch และ Best Practices](#multi-catch-และ-best-practices)
9. [โปรแกรมตัวอย่างจริง](#โปรแกรมตัวอย่างจริง)

---

## Exception คืออะไร?

Exception คือ เหตุการณ์ที่ทำให้โปรแกรมทำงานผิดปกติระหว่าง runtime

```
สาเหตุหลักของ Exceptions:
1. User Input ผิดพลาด      - กรอกข้อมูลผิด, ไม่ครบ
2. Hardware/Network ล้มเหลว - ไฟดับ, เน็ตหลุด
3. Logic ผิดพลาด           - หาร 0, null pointer
4. Resource หมด            - memory, disk space
5. External System ล้มเหลว  - database, API ไม่ตอบ
```

### โปรแกรมที่ไม่มี Exception Handling

```java
public class NoExceptionHandling {
    public static void main(String[] args) {
        // ปัญหาที่ 1: หาร 0
        int result = 10 / 0;  // ArithmeticException!
        
        // ปัญหาที่ 2: null pointer
        String name = null;
        int length = name.length();  // NullPointerException!
        
        // ปัญหาที่ 3: array out of bounds
        int[] arr = new int[5];
        arr[10] = 100;  // ArrayIndexOutOfBoundsException!
        
        // ปัญหาที่ 4: type casting
        Object obj = "hello";
        Integer num = (Integer) obj;  // ClassCastException!
        
        // ปัญหาที่ 5: number format
        int x = Integer.parseInt("abc");  // NumberFormatException!
    }
}
```

### โปรแกรมที่มี Exception Handling

```java
public class WithExceptionHandling {
    public static void main(String[] args) {
        // ปัญหาที่ 1: หาร 0
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("Error: Cannot divide by zero");
        }
        
        // ปัญหาที่ 2: null pointer
        try {
            String name = null;
            int length = name.length();
        } catch (NullPointerException e) {
            System.out.println("Error: Name is null");
        }
        
        // ปัญหาที่ 3: number format
        try {
            int x = Integer.parseInt("abc");
        } catch (NumberFormatException e) {
            System.out.println("Error: 'abc' is not a valid number");
        }
        
        System.out.println("Program continues running!");
    }
}
```

---

## Exception Hierarchy

```
Throwable
├── Error (ไม่ควร catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── AssertionError
│
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── ArithmeticException
    │   ├── NumberFormatException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   └── UnsupportedOperationException
    │
    ├── IOException (Checked)
    │   ├── FileNotFoundException
    │   └── SocketException
    │
    ├── SQLException (Checked)
    ├── ClassNotFoundException (Checked)
    └── ParseException (Checked)
```

---

## try-catch-finally

### รูปแบบพื้นฐาน

```java
public class TryCatchFinally {
    public static void main(String[] args) {
        // รูปแบบที่ 1: try-catch
        try {
            int result = divide(10, 0);
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Caught: " + e.getMessage());
        }
        
        // รูปแบบที่ 2: try-catch-finally
        try {
            System.out.println("Opening connection...");
            int result = divide(10, 2);
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Math error: " + e.getMessage());
        } finally {
            System.out.println("Closing connection... (always runs)");
        }
        
        // รูปแบบที่ 3: หลาย catch
        try {
            processInput("abc", null, 5);
        } catch (NumberFormatException e) {
            System.out.println("Invalid number format: " + e.getMessage());
        } catch (NullPointerException e) {
            System.out.println("Null value found: " + e.getMessage());
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Index out of bounds: " + e.getMessage());
        } catch (Exception e) {
            System.out.println("General error: " + e.getMessage());
        }
    }
    
    static int divide(int a, int b) {
        return a / b;
    }
    
    static void processInput(String numStr, String name, int index) {
        int num = Integer.parseInt(numStr);    // NumberFormatException
        int len = name.length();               // NullPointerException
        int[] arr = new int[3];
        arr[index] = num;                      // ArrayIndexOutOfBoundsException
    }
}
```

### ลำดับการทำงานของ finally

```java
public class FinallyOrder {
    
    static int method1() {
        try {
            System.out.println("try block");
            return 1;
        } finally {
            System.out.println("finally block (runs before return)");
        }
    }
    
    static int method2() {
        try {
            System.out.println("try block");
            throw new RuntimeException("test");
        } catch (RuntimeException e) {
            System.out.println("catch block");
            return 2;
        } finally {
            System.out.println("finally block");
            // return 3;  // ถ้า return ใน finally มันจะ override catch return!
        }
    }
    
    static void method3() {
        try {
            System.out.println("method3 try");
            return;
        } finally {
            System.out.println("method3 finally");
            // Exception ใน finally overrides exception ใน try
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== method1 ===");
        System.out.println("Returns: " + method1());
        
        System.out.println("\n=== method2 ===");
        System.out.println("Returns: " + method2());
        
        System.out.println("\n=== method3 ===");
        method3();
    }
}
```

### Exception Information

```java
public class ExceptionInfo {
    public static void main(String[] args) {
        try {
            callLevel1();
        } catch (Exception e) {
            System.out.println("Message: " + e.getMessage());
            System.out.println("Class: " + e.getClass().getName());
            System.out.println("Cause: " + e.getCause());
            
            System.out.println("\nStack Trace:");
            e.printStackTrace();
            
            // Get specific stack frame
            StackTraceElement[] trace = e.getStackTrace();
            System.out.println("\nTop frame: " + trace[0]);
        }
    }
    
    static void callLevel1() { callLevel2(); }
    static void callLevel2() { callLevel3(); }
    static void callLevel3() { 
        throw new RuntimeException("Error in level 3"); 
    }
}
```

---

## Checked vs Unchecked Exceptions

```java
import java.io.*;
import java.net.*;

public class CheckedVsUnchecked {
    
    // Checked Exception: ต้อง declare หรือ handle
    static String readFile(String path) throws IOException {
        // IOException เป็น checked exception
        // ต้อง try-catch หรือ throws ใน signature
        try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                sb.append(line).append("\n");
            }
            return sb.toString();
        }
    }
    
    // Unchecked Exception: ไม่ต้อง declare
    static int divide(int a, int b) {
        // ArithmeticException เป็น unchecked
        // ไม่ต้อง throws ใน signature
        if (b == 0) {
            throw new ArithmeticException("Cannot divide by zero");
        }
        return a / b;
    }
    
    // ตัวอย่างการ convert checked -> unchecked
    static String readFileSafely(String path) {
        try {
            return readFile(path);
        } catch (IOException e) {
            throw new RuntimeException("Failed to read file: " + path, e);
        }
    }
    
    public static void main(String[] args) {
        // Checked: ต้อง handle
        try {
            String content = readFile("data.txt");
            System.out.println(content);
        } catch (IOException e) {
            System.out.println("File error: " + e.getMessage());
        }
        
        // Unchecked: อาจจะ handle หรือไม่ก็ได้
        int result = divide(10, 2);  // OK
        System.out.println("Result: " + result);
        
        // ถ้าไม่ handle unchecked, โปรแกรมจะ crash
        try {
            int bad = divide(10, 0);
        } catch (ArithmeticException e) {
            System.out.println("Handled: " + e.getMessage());
        }
        
        // Convert checked to unchecked
        String data = readFileSafely("nonexistent.txt");
    }
}
```

---

## Custom Exceptions

### สร้าง Custom Exception

```java
// Base custom exception
public class AppException extends RuntimeException {
    private final String errorCode;
    
    public AppException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public AppException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() { return errorCode; }
    
    @Override
    public String toString() {
        return "[" + errorCode + "] " + getMessage();
    }
}

// Specific exceptions
public class ValidationException extends AppException {
    private final String field;
    private final Object invalidValue;
    
    public ValidationException(String field, Object value, String reason) {
        super("VALIDATION_ERROR", 
              "Validation failed for field '" + field + "': " + reason);
        this.field = field;
        this.invalidValue = value;
    }
    
    public String getField() { return field; }
    public Object getInvalidValue() { return invalidValue; }
}

public class NotFoundException extends AppException {
    private final String resource;
    private final Object id;
    
    public NotFoundException(String resource, Object id) {
        super("NOT_FOUND", resource + " with id '" + id + "' not found");
        this.resource = resource;
        this.id = id;
    }
    
    public String getResource() { return resource; }
    public Object getId() { return id; }
}

public class BusinessException extends AppException {
    public BusinessException(String message) {
        super("BUSINESS_ERROR", message);
    }
    
    public BusinessException(String code, String message) {
        super(code, message);
    }
}

// Usage
public class CustomExceptionDemo {
    
    record User(Long id, String name, String email, int age) {}
    
    static User findUser(Long id) {
        // Simulate database lookup
        if (id == null) throw new ValidationException("id", null, "must not be null");
        if (id <= 0) throw new ValidationException("id", id, "must be positive");
        if (id == 999) throw new NotFoundException("User", id);
        
        return new User(id, "John Doe", "john@example.com", 30);
    }
    
    static User createUser(String name, String email, int age) {
        if (name == null || name.isBlank()) {
            throw new ValidationException("name", name, "must not be blank");
        }
        if (email == null || !email.contains("@")) {
            throw new ValidationException("email", email, "invalid email format");
        }
        if (age < 0 || age > 150) {
            throw new ValidationException("age", age, "must be between 0 and 150");
        }
        
        return new User(System.nanoTime(), name, email, age);
    }
    
    static void transferMoney(User from, User to, double amount) {
        if (amount <= 0) {
            throw new BusinessException("INVALID_AMOUNT", 
                "Transfer amount must be positive");
        }
        if (amount > 1_000_000) {
            throw new BusinessException("AMOUNT_EXCEEDS_LIMIT", 
                "Transfer amount exceeds daily limit");
        }
        System.out.printf("Transferred %.2f from %s to %s%n", 
            amount, from.name(), to.name());
    }
    
    public static void main(String[] args) {
        // Test findUser
        Long[] testIds = {1L, null, -1L, 999L};
        for (Long id : testIds) {
            try {
                User user = findUser(id);
                System.out.println("Found: " + user.name());
            } catch (ValidationException e) {
                System.out.println("Validation: field=" + e.getField() + 
                    ", value=" + e.getInvalidValue() + 
                    ", msg=" + e.getMessage());
            } catch (NotFoundException e) {
                System.out.println("Not found: " + e.getMessage());
            }
        }
        
        System.out.println();
        
        // Test createUser
        String[][] users = {
            {"Alice", "alice@example.com", "25"},
            {"", "bob@example.com", "30"},
            {"Charlie", "invalid-email", "35"},
            {"Diana", "diana@example.com", "-5"}
        };
        
        for (String[] data : users) {
            try {
                User user = createUser(data[0], data[1], Integer.parseInt(data[2]));
                System.out.println("Created: " + user.name());
            } catch (ValidationException e) {
                System.out.println("Failed: " + e);
            }
        }
        
        System.out.println();
        
        // Test transfer
        User alice = new User(1L, "Alice", "alice@example.com", 25);
        User bob = new User(2L, "Bob", "bob@example.com", 30);
        
        double[] amounts = {1000.0, -500.0, 2_000_000.0};
        for (double amount : amounts) {
            try {
                transferMoney(alice, bob, amount);
            } catch (BusinessException e) {
                System.out.println("Business error [" + e.getErrorCode() + "]: " + 
                    e.getMessage());
            }
        }
    }
}
```

---

## try-with-resources

### AutoCloseable Interface

```java
import java.io.*;

public class TryWithResources {
    
    // Custom resource
    static class DatabaseConnection implements AutoCloseable {
        private final String url;
        private boolean connected;
        
        DatabaseConnection(String url) {
            this.url = url;
            System.out.println("Opening connection to: " + url);
            this.connected = true;
        }
        
        String query(String sql) {
            if (!connected) throw new IllegalStateException("Not connected");
            System.out.println("Executing: " + sql);
            return "Result of: " + sql;
        }
        
        @Override
        public void close() {
            System.out.println("Closing connection to: " + url);
            connected = false;
        }
    }
    
    static class FileProcessor implements AutoCloseable {
        private final String filename;
        private boolean open;
        
        FileProcessor(String filename) throws Exception {
            System.out.println("Opening file: " + filename);
            this.filename = filename;
            this.open = true;
        }
        
        void process() {
            System.out.println("Processing: " + filename);
        }
        
        @Override
        public void close() {
            System.out.println("Closing file: " + filename);
            open = false;
        }
    }
    
    public static void main(String[] args) {
        // Before Java 7 (manual close)
        DatabaseConnection conn1 = null;
        try {
            conn1 = new DatabaseConnection("jdbc:mysql://localhost/db");
            conn1.query("SELECT * FROM users");
        } finally {
            if (conn1 != null) conn1.close();
        }
        
        System.out.println();
        
        // Java 7+ try-with-resources (auto close!)
        try (DatabaseConnection conn = new DatabaseConnection("jdbc:mysql://localhost/db")) {
            conn.query("SELECT * FROM products");
        }  // auto closes here
        
        System.out.println();
        
        // Multiple resources (closed in reverse order)
        try (DatabaseConnection conn = new DatabaseConnection("jdbc:postgresql://localhost/db");
             FileProcessor file = new FileProcessor("output.csv")) {
            
            String data = conn.query("SELECT id, name FROM users");
            file.process();
            System.out.println("Data: " + data);
            
        } catch (Exception e) {
            System.out.println("Error: " + e.getMessage());
        }
        
        System.out.println();
        
        // Real-world: File I/O
        String filename = "test.txt";
        
        // Write file
        try (PrintWriter writer = new PrintWriter(new FileWriter(filename))) {
            writer.println("Line 1");
            writer.println("Line 2");
            writer.println("Line 3");
        } catch (IOException e) {
            System.out.println("Write error: " + e.getMessage());
        }
        
        // Read file
        try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println("Read: " + line);
            }
        } catch (IOException e) {
            System.out.println("Read error: " + e.getMessage());
        }
        
        // Cleanup
        new File(filename).delete();
    }
}
```

---

## Exception Chaining

```java
public class ExceptionChaining {
    
    // Database layer
    static String queryDatabase(String sql) throws Exception {
        throw new Exception("Database connection failed");
    }
    
    // Repository layer: wraps database exception
    static String findUserById(Long id) {
        try {
            return queryDatabase("SELECT * FROM users WHERE id = " + id);
        } catch (Exception e) {
            throw new RuntimeException("Failed to find user with id: " + id, e);
        }
    }
    
    // Service layer: wraps repository exception
    static String getUserProfile(Long id) {
        try {
            return findUserById(id);
        } catch (RuntimeException e) {
            throw new RuntimeException("getUserProfile failed", e);
        }
    }
    
    // Controller: handles the chain
    public static void main(String[] args) {
        try {
            String profile = getUserProfile(123L);
        } catch (RuntimeException e) {
            System.out.println("Error: " + e.getMessage());
            
            // Walk the exception chain
            Throwable cause = e;
            int level = 0;
            while (cause != null) {
                System.out.println("Level " + level + ": " + 
                    cause.getClass().getSimpleName() + ": " + cause.getMessage());
                cause = cause.getCause();
                level++;
            }
        }
        
        // initCause example
        System.out.println("\n--- initCause example ---");
        try {
            RuntimeException ex = new RuntimeException("Outer exception");
            ex.initCause(new Exception("Root cause"));
            throw ex;
        } catch (RuntimeException e) {
            System.out.println("Exception: " + e.getMessage());
            System.out.println("Cause: " + e.getCause().getMessage());
        }
    }
}
```

---

## Multi-catch และ Best Practices

### Multi-catch (Java 7+)

```java
public class MultiCatch {
    public static void main(String[] args) {
        // ก่อน Java 7 (repetitive code)
        try {
            riskyOperation(1);
        } catch (NumberFormatException e) {
            handleError(e);
        } catch (ArrayIndexOutOfBoundsException e) {
            handleError(e);
        } catch (NullPointerException e) {
            handleError(e);
        }
        
        // Java 7+ multi-catch (cleaner!)
        for (int i = 1; i <= 4; i++) {
            try {
                riskyOperation(i);
            } catch (NumberFormatException | NullPointerException e) {
                System.out.println("Data error [" + i + "]: " + e.getMessage());
            } catch (ArrayIndexOutOfBoundsException e) {
                System.out.println("Index error [" + i + "]: " + e.getMessage());
            } catch (Exception e) {
                System.out.println("Other error [" + i + "]: " + e.getMessage());
            }
        }
    }
    
    static void riskyOperation(int type) {
        switch (type) {
            case 1 -> Integer.parseInt("not-a-number");
            case 2 -> { String s = null; s.length(); }
            case 3 -> { int[] a = new int[3]; a[10] = 1; }
            case 4 -> throw new UnsupportedOperationException("Not implemented");
        }
    }
    
    static void handleError(Exception e) {
        System.out.println("Handling: " + e.getClass().getSimpleName());
    }
}
```

### Best Practices

```java
public class ExceptionBestPractices {
    
    // ❌ BAD: Catch all exceptions silently
    static void bad1() {
        try {
            risky();
        } catch (Exception e) {
            // Empty catch - NEVER DO THIS!
        }
    }
    
    // ✅ GOOD: At minimum, log the exception
    static void good1() {
        try {
            risky();
        } catch (Exception e) {
            // Log it
            System.err.println("ERROR: " + e.getMessage());
            // or throw as unchecked
            throw new RuntimeException("Operation failed", e);
        }
    }
    
    // ❌ BAD: Too broad catch
    static void bad2() {
        try {
            int[] arr = new int[10];
            arr[0] = Integer.parseInt("123");
        } catch (Exception e) {  // Catches everything!
            System.out.println("Something went wrong");
        }
    }
    
    // ✅ GOOD: Specific catch
    static void good2() {
        try {
            int[] arr = new int[10];
            arr[0] = Integer.parseInt("123");
        } catch (NumberFormatException e) {
            System.out.println("Invalid number format");
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Array index out of bounds");
        }
    }
    
    // ❌ BAD: Exception as flow control
    static boolean bad3(String s) {
        try {
            Integer.parseInt(s);
            return true;
        } catch (NumberFormatException e) {
            return false;
        }
    }
    
    // ✅ GOOD: Explicit check
    static boolean good3(String s) {
        if (s == null) return false;
        return s.matches("-?\\d+");
    }
    
    // ❌ BAD: Lose original cause
    static void bad4(String id) {
        try {
            loadData(id);
        } catch (Exception e) {
            throw new RuntimeException("Data load failed");  // lost original!
        }
    }
    
    // ✅ GOOD: Preserve cause
    static void good4(String id) {
        try {
            loadData(id);
        } catch (Exception e) {
            throw new RuntimeException("Data load failed for id: " + id, e);  // preserved
        }
    }
    
    // ✅ GOOD: Validate before operation
    static int safeDivide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("Divisor cannot be zero");
        }
        return a / b;
    }
    
    static void risky() throws Exception {}
    static void loadData(String id) throws Exception {}
    
    public static void main(String[] args) {
        // Test good practices
        System.out.println("Is numeric '123': " + good3("123"));
        System.out.println("Is numeric 'abc': " + good3("abc"));
        
        try {
            System.out.println("safeDivide(10, 0): ");
            safeDivide(10, 0);
        } catch (IllegalArgumentException e) {
            System.out.println("Error: " + e.getMessage());
        }
        
        System.out.println("safeDivide(10, 2): " + safeDivide(10, 2));
    }
}
```

---

## โปรแกรมตัวอย่างจริง: Bank Transaction System

```java
import java.util.*;

public class BankTransactionSystem {
    
    // Custom Exception Hierarchy
    static class BankException extends RuntimeException {
        final String code;
        BankException(String code, String message) { 
            super(message); 
            this.code = code; 
        }
        BankException(String code, String message, Throwable cause) { 
            super(message, cause); 
            this.code = code; 
        }
    }
    
    static class InsufficientFundsException extends BankException {
        final double required, available;
        InsufficientFundsException(double required, double available) {
            super("INSUFFICIENT_FUNDS", 
                String.format("Required: %.2f, Available: %.2f", required, available));
            this.required = required;
            this.available = available;
        }
    }
    
    static class AccountNotFoundException extends BankException {
        AccountNotFoundException(String accountId) {
            super("ACCOUNT_NOT_FOUND", "Account not found: " + accountId);
        }
    }
    
    static class TransactionLimitException extends BankException {
        TransactionLimitException(double amount, double limit) {
            super("TRANSACTION_LIMIT_EXCEEDED", 
                String.format("Amount %.2f exceeds limit %.2f", amount, limit));
        }
    }
    
    static class FrozenAccountException extends BankException {
        FrozenAccountException(String accountId) {
            super("ACCOUNT_FROZEN", "Account is frozen: " + accountId);
        }
    }
    
    // Account
    static class Account {
        private final String id;
        private double balance;
        private boolean frozen;
        private final List<String> transactions = new ArrayList<>();
        
        Account(String id, double initialBalance) {
            this.id = id;
            this.balance = initialBalance;
        }
        
        void deposit(double amount) {
            validate(amount);
            balance += amount;
            log("DEPOSIT", amount);
        }
        
        void withdraw(double amount) {
            validate(amount);
            if (frozen) throw new FrozenAccountException(id);
            if (amount > balance) throw new InsufficientFundsException(amount, balance);
            if (amount > 100_000) throw new TransactionLimitException(amount, 100_000);
            balance -= amount;
            log("WITHDRAW", amount);
        }
        
        private void validate(double amount) {
            if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        }
        
        private void log(String type, double amount) {
            transactions.add(String.format("[%s] %s: %.2f (Balance: %.2f)", 
                new Date(), type, amount, balance));
        }
        
        void freeze() { frozen = true; }
        void unfreeze() { frozen = false; }
        double getBalance() { return balance; }
        String getId() { return id; }
        List<String> getTransactions() { return Collections.unmodifiableList(transactions); }
    }
    
    // Bank Service
    static class BankService {
        private final Map<String, Account> accounts = new HashMap<>();
        
        Account createAccount(String id, double initialBalance) {
            Account account = new Account(id, initialBalance);
            accounts.put(id, account);
            return account;
        }
        
        Account getAccount(String id) {
            Account account = accounts.get(id);
            if (account == null) throw new AccountNotFoundException(id);
            return account;
        }
        
        void transfer(String fromId, String toId, double amount) {
            Account from = getAccount(fromId);
            Account to = getAccount(toId);
            
            from.withdraw(amount);
            try {
                to.deposit(amount);
            } catch (Exception e) {
                // Rollback if deposit fails
                from.deposit(amount);
                throw new BankException("TRANSFER_FAILED", 
                    "Transfer failed and was rolled back", e);
            }
            
            System.out.printf("Transferred %.2f from %s to %s%n", 
                amount, fromId, toId);
        }
    }
    
    public static void main(String[] args) {
        BankService bank = new BankService();
        
        // Setup accounts
        Account alice = bank.createAccount("ACC001", 50_000.0);
        Account bob = bank.createAccount("ACC002", 10_000.0);
        Account charlie = bank.createAccount("ACC003", 100_000.0);
        
        // Test scenarios
        Object[][] tests = {
            {"transfer", "ACC001", "ACC002", 5000.0},        // OK
            {"transfer", "ACC001", "ACC002", 200_000.0},     // InsufficientFunds
            {"transfer", "ACC001", "ACC999", 1000.0},        // AccountNotFound
            {"withdraw", "ACC003", null, 150_000.0},         // TransactionLimit
            {"freeze", "ACC001", null, null},                // Freeze
            {"transfer", "ACC001", "ACC002", 1000.0},        // Frozen
        };
        
        for (Object[] test : tests) {
            String action = (String) test[0];
            String account1 = (String) test[1];
            String account2 = (String) test[2];
            Double amount = (Double) test[3];
            
            try {
                switch (action) {
                    case "transfer" -> bank.transfer(account1, account2, amount);
                    case "withdraw" -> bank.getAccount(account1).withdraw(amount);
                    case "freeze" -> {
                        bank.getAccount(account1).freeze();
                        System.out.println("Account " + account1 + " frozen");
                    }
                }
            } catch (InsufficientFundsException e) {
                System.out.printf("[%s] Insufficient: need %.0f, have %.0f%n",
                    e.code, e.required, e.available);
            } catch (AccountNotFoundException e) {
                System.out.printf("[%s] %s%n", e.code, e.getMessage());
            } catch (TransactionLimitException e) {
                System.out.printf("[%s] %s%n", e.code, e.getMessage());
            } catch (FrozenAccountException e) {
                System.out.printf("[%s] %s%n", e.code, e.getMessage());
            } catch (BankException e) {
                System.out.printf("[%s] %s%n", e.code, e.getMessage());
            }
        }
        
        System.out.println("\n=== Final Balances ===");
        for (String id : new String[]{"ACC001", "ACC002", "ACC003"}) {
            Account acc = bank.getAccount(id);
            System.out.printf("%s: %.2f%n", id, acc.getBalance());
        }
        
        System.out.println("\n=== Transaction History (ACC001) ===");
        alice.getTransactions().forEach(t -> System.out.println("  " + t));
    }
}
```

---

## สรุป

```
Exception Handling Rules:
1. Catch specific exceptions (not Exception)
2. Never silence exceptions (empty catch)
3. Always log or re-throw exceptions
4. Preserve original cause when chaining
5. Use try-with-resources for AutoCloseable resources
6. Validate inputs before operations
7. Create custom exceptions for domain errors
8. Use checked for recoverable, unchecked for bugs
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Exception Hierarchy  
✅ try-catch-finally  
✅ Checked vs Unchecked  
✅ Custom Exceptions  
✅ try-with-resources  
✅ Exception Chaining  
✅ Multi-catch  
✅ Best Practices  
✅ Complete Bank System Example  

---

## ขั้นตอนต่อไป

**Part 011:** Generics & Type Safety  
เราจะเรียนรู้:
- Generic classes และ methods
- Bounded type parameters (extends, super)
- Wildcards (?, ? extends, ? super)
- Generic collections
- Type erasure

---

*Part 010 | Java & Spring Boot Course | สร้างโดย Claude Code*
