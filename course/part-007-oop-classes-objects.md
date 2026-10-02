# Part 007: Object-Oriented Programming - Classes & Objects
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [OOP คืออะไร?](#oop-คืออะไร)
2. [Classes และ Objects](#classes-และ-objects)
3. [Constructors](#constructors)
4. [Fields และ Methods](#fields-และ-methods)
5. [Encapsulation (Getters/Setters)](#encapsulation-getterssetters)
6. [this Keyword](#this-keyword)
7. [static Members](#static-members)
8. [Enums](#enums)
9. [Records (Java 16+)](#records-java-16)
10. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)

---

## OOP คืออะไร?

Object-Oriented Programming คือแนวทางการเขียนโปรแกรมที่จัดระเบียบโค้ดเป็น Objects

### 4 หลักการหลักของ OOP

```
1. Encapsulation (การห่อหุ้ม)
   - ซ่อน data และ implementation ไว้ใน class
   - เข้าถึงผ่าน public methods เท่านั้น

2. Inheritance (การสืบทอด)  
   - class ลูกสืบทอดคุณสมบัติจาก class แม่
   (เรียนใน Part 008)

3. Polymorphism (ความหลากหลาย)
   - object เดียวกัน ทำงานต่างกันตาม context
   (เรียนใน Part 008)

4. Abstraction (การซ่อนรายละเอียด)
   - แสดงเฉพาะสิ่งที่จำเป็น ซ่อนความซับซ้อน
   (เรียนใน Part 009)
```

### Class คือ Blueprint, Object คือ Instance

```
Class (Blueprint):
┌────────────────┐
│  class Car {   │
│    String color│
│    int speed   │
│    void drive()│
│  }             │
└────────────────┘
        │ new Car()
        ▼
Objects (Instances):
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Car      │  │ Car      │  │ Car      │
│ red      │  │ blue     │  │ black    │
│ 120      │  │ 180      │  │ 200      │
└──────────┘  └──────────┘  └──────────┘
```

---

## Classes และ Objects

### สร้าง Class แรก

```java
// ไฟล์: BankAccount.java
public class BankAccount {
    // Fields (instance variables)
    private String accountNumber;
    private String owner;
    private double balance;
    private String currency;
    
    // Constructor
    public BankAccount(String accountNumber, String owner, double initialBalance) {
        this.accountNumber = accountNumber;
        this.owner = owner;
        this.balance = initialBalance;
        this.currency = "THB";
    }
    
    // Methods
    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("จำนวนเงินต้องมากกว่า 0");
        }
        balance += amount;
        System.out.printf("ฝาก %.2f %s สำเร็จ | ยอดเงิน: %.2f %s%n",
            amount, currency, balance, currency);
    }
    
    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("จำนวนเงินต้องมากกว่า 0");
        }
        if (amount > balance) {
            throw new IllegalStateException("ยอดเงินไม่เพียงพอ");
        }
        balance -= amount;
        System.out.printf("ถอน %.2f %s สำเร็จ | ยอดเงิน: %.2f %s%n",
            amount, currency, balance, currency);
    }
    
    public void displayInfo() {
        System.out.println("╔══════════════════════════════════╗");
        System.out.println("║         ข้อมูลบัญชี              ║");
        System.out.println("╠══════════════════════════════════╣");
        System.out.printf("║ เลขบัญชี: %-22s║%n", accountNumber);
        System.out.printf("║ เจ้าของ: %-23s║%n", owner);
        System.out.printf("║ ยอดเงิน: %-20.2f %s║%n", balance, currency);
        System.out.println("╚══════════════════════════════════╝");
    }
    
    // Getters
    public String getAccountNumber() { return accountNumber; }
    public String getOwner() { return owner; }
    public double getBalance() { return balance; }
}
```

```java
// ไฟล์: Main.java
public class Main {
    public static void main(String[] args) {
        // สร้าง object ด้วย new keyword
        BankAccount account1 = new BankAccount("1234567890", "สมชาย ใจดี", 10000.00);
        BankAccount account2 = new BankAccount("0987654321", "สมหญิง รักดี", 25000.00);
        
        account1.displayInfo();
        System.out.println();
        
        account1.deposit(5000);
        account1.withdraw(3000);
        
        System.out.println("\nยอดเงินล่าสุด: " + account1.getBalance());
        
        // object2 เป็น state ของตัวเอง
        account2.displayInfo();
    }
}
```

---

## Constructors

### ประเภท Constructor

```java
public class Person {
    private String name;
    private int age;
    private String email;
    
    // 1. No-arg constructor
    public Person() {
        this.name = "Unknown";
        this.age = 0;
        this.email = "";
    }
    
    // 2. Parameterized constructor
    public Person(String name, int age) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("Invalid age: " + age);
        }
        this.name = name;
        this.age = age;
        this.email = "";
    }
    
    // 3. Full constructor
    public Person(String name, int age, String email) {
        this(name, age);  // เรียก constructor อื่น (constructor chaining)
        this.email = email != null ? email : "";
    }
    
    // 4. Copy constructor
    public Person(Person other) {
        this(other.name, other.age, other.email);
    }
    
    @Override
    public String toString() {
        return String.format("Person{name='%s', age=%d, email='%s'}", 
            name, age, email);
    }
    
    public static void main(String[] args) {
        Person p1 = new Person();
        Person p2 = new Person("Alice", 30);
        Person p3 = new Person("Bob", 25, "bob@example.com");
        Person p4 = new Person(p3);  // copy
        
        System.out.println(p1);
        System.out.println(p2);
        System.out.println(p3);
        System.out.println(p4);
        System.out.println("p3 == p4: " + (p3 == p4));         // false (different objects)
        System.out.println("p3.equals(p4): " + p3.equals(p4)); // depends on equals()
    }
}
```

### Builder Pattern (แทน Constructor ที่มี parameters เยอะ)

```java
public class Pizza {
    private final String size;
    private final String crust;
    private final boolean cheese;
    private final boolean tomato;
    private final boolean pepperoni;
    private final boolean mushroom;
    private final boolean onion;
    
    // Private constructor - สร้างได้แค่ผ่าน Builder
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.crust = builder.crust;
        this.cheese = builder.cheese;
        this.tomato = builder.tomato;
        this.pepperoni = builder.pepperoni;
        this.mushroom = builder.mushroom;
        this.onion = builder.onion;
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Pizza [").append(size).append(", ").append(crust).append(" crust]");
        if (cheese) sb.append(" + Cheese");
        if (tomato) sb.append(" + Tomato");
        if (pepperoni) sb.append(" + Pepperoni");
        if (mushroom) sb.append(" + Mushroom");
        if (onion) sb.append(" + Onion");
        return sb.toString();
    }
    
    // Builder class
    public static class Builder {
        private final String size;    // required
        private String crust = "thin"; // default
        private boolean cheese = true;
        private boolean tomato = false;
        private boolean pepperoni = false;
        private boolean mushroom = false;
        private boolean onion = false;
        
        public Builder(String size) {
            this.size = size;
        }
        
        public Builder crust(String crust) { this.crust = crust; return this; }
        public Builder cheese(boolean val) { this.cheese = val; return this; }
        public Builder tomato(boolean val) { this.tomato = val; return this; }
        public Builder pepperoni(boolean val) { this.pepperoni = val; return this; }
        public Builder mushroom(boolean val) { this.mushroom = val; return this; }
        public Builder onion(boolean val) { this.onion = val; return this; }
        
        public Pizza build() { return new Pizza(this); }
    }
    
    public static void main(String[] args) {
        // สร้าง Pizza ด้วย Builder pattern
        Pizza pizza1 = new Pizza.Builder("Large")
            .crust("thick")
            .cheese(true)
            .pepperoni(true)
            .mushroom(true)
            .build();
        
        Pizza pizza2 = new Pizza.Builder("Medium")
            .tomato(true)
            .onion(true)
            .build();
        
        System.out.println(pizza1);
        System.out.println(pizza2);
    }
}
```

---

## Fields และ Methods

### Access Modifiers

```java
public class AccessModifiers {
    // public: เข้าถึงได้จากทุกที่
    public String publicField = "Anyone can access";
    
    // protected: เข้าถึงได้จาก same package และ subclasses
    protected String protectedField = "Package and subclasses";
    
    // package-private (default): เข้าถึงได้จาก same package
    String packageField = "Same package only";
    
    // private: เข้าถึงได้เฉพาะใน class นี้
    private String privateField = "This class only";
    
    // Methods
    public void publicMethod() { System.out.println("Public"); }
    protected void protectedMethod() { System.out.println("Protected"); }
    void packageMethod() { System.out.println("Package"); }
    private void privateMethod() { System.out.println("Private"); }
    
    public void demo() {
        // ใน class เดียวกัน เข้าถึงได้ทั้งหมด
        System.out.println(publicField);
        System.out.println(protectedField);
        System.out.println(packageField);
        System.out.println(privateField);
        
        publicMethod();
        protectedMethod();
        packageMethod();
        privateMethod();
    }
}
```

### Instance vs Static Methods/Fields

```java
public class Counter {
    // Static field: แชร์ระหว่างทุก instance
    private static int totalCount = 0;
    
    // Instance field: แต่ละ instance มีของตัวเอง
    private int id;
    private String name;
    private int count;
    
    public Counter(String name) {
        totalCount++;
        this.id = totalCount;
        this.name = name;
        this.count = 0;
    }
    
    // Instance method
    public void increment() {
        count++;
        System.out.printf("Counter %s (id=%d): %d%n", name, id, count);
    }
    
    // Static method
    public static int getTotalCount() {
        return totalCount;
    }
    
    // Static factory method
    public static Counter create(String name) {
        return new Counter(name);
    }
    
    public int getCount() { return count; }
    public String getName() { return name; }
    
    public static void main(String[] args) {
        System.out.println("Total: " + Counter.getTotalCount());  // 0
        
        Counter c1 = Counter.create("Alpha");
        Counter c2 = new Counter("Beta");
        Counter c3 = new Counter("Gamma");
        
        System.out.println("Total: " + Counter.getTotalCount());  // 3
        
        c1.increment();
        c1.increment();
        c2.increment();
        c3.increment();
        c3.increment();
        c3.increment();
        
        System.out.printf("\nFinal: %s=%d, %s=%d, %s=%d%n",
            c1.getName(), c1.getCount(),
            c2.getName(), c2.getCount(),
            c3.getName(), c3.getCount());
    }
}
```

---

## Encapsulation (Getters/Setters)

### Encapsulation ที่ดี

```java
public class Temperature {
    private double celsius;
    
    public Temperature(double celsius) {
        setCelsius(celsius);  // ใช้ setter เพื่อ validate
    }
    
    // Getter
    public double getCelsius() { return celsius; }
    
    // Computed properties (ไม่ต้อง store)
    public double getFahrenheit() { return celsius * 9/5 + 32; }
    public double getKelvin() { return celsius + 273.15; }
    
    // Setter with validation
    public void setCelsius(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException(
                "อุณหภูมิต่ำกว่า Absolute Zero ไม่ได้: " + celsius);
        }
        this.celsius = celsius;
    }
    
    @Override
    public String toString() {
        return String.format("%.2f°C = %.2f°F = %.2fK", 
            celsius, getFahrenheit(), getKelvin());
    }
    
    public static void main(String[] args) {
        Temperature temp = new Temperature(100.0);
        System.out.println(temp);
        
        temp.setCelsius(-40);
        System.out.println(temp);
        
        try {
            temp.setCelsius(-300);  // จะ throw exception
        } catch (IllegalArgumentException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

### ตัวอย่าง Encapsulation จริง: User Account

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

public class UserAccount {
    private final String userId;
    private String username;
    private String email;
    private String passwordHash;
    private boolean active;
    private final LocalDateTime createdAt;
    private LocalDateTime lastLoginAt;
    private int loginAttempts;
    private static final int MAX_LOGIN_ATTEMPTS = 5;
    
    // Transaction log
    private final List<String> activityLog = new ArrayList<>();
    
    public UserAccount(String userId, String username, String email, String password) {
        this.userId = Objects.requireNonNull(userId, "userId cannot be null");
        setUsername(username);
        setEmail(email);
        setPassword(password);
        this.active = true;
        this.createdAt = LocalDateTime.now();
        this.loginAttempts = 0;
        log("Account created");
    }
    
    // Username validation
    public void setUsername(String username) {
        if (username == null || username.length() < 3 || username.length() > 30) {
            throw new IllegalArgumentException("Username must be 3-30 characters");
        }
        if (!username.matches("[a-zA-Z0-9_]+")) {
            throw new IllegalArgumentException("Username can only contain letters, numbers, underscores");
        }
        this.username = username;
    }
    
    // Email validation
    public void setEmail(String email) {
        if (email == null || !email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            throw new IllegalArgumentException("Invalid email: " + email);
        }
        this.email = email;
    }
    
    // Password hashing (simplified)
    public void setPassword(String password) {
        if (password == null || password.length() < 8) {
            throw new IllegalArgumentException("Password must be at least 8 characters");
        }
        this.passwordHash = hashPassword(password);
    }
    
    // Login
    public boolean login(String password) {
        if (!active) {
            log("Login attempt on inactive account");
            return false;
        }
        
        if (loginAttempts >= MAX_LOGIN_ATTEMPTS) {
            log("Account locked - too many failed attempts");
            return false;
        }
        
        if (hashPassword(password).equals(passwordHash)) {
            loginAttempts = 0;
            lastLoginAt = LocalDateTime.now();
            log("Login successful");
            return true;
        } else {
            loginAttempts++;
            log("Login failed (attempt " + loginAttempts + ")");
            if (loginAttempts >= MAX_LOGIN_ATTEMPTS) {
                active = false;
                log("Account locked!");
            }
            return false;
        }
    }
    
    public void unlock() {
        loginAttempts = 0;
        active = true;
        log("Account unlocked");
    }
    
    private String hashPassword(String password) {
        // ใน production ใช้ BCrypt หรือ Argon2
        return Integer.toHexString(password.hashCode());
    }
    
    private void log(String action) {
        activityLog.add(LocalDateTime.now() + " - " + action);
    }
    
    // Getters (read-only access)
    public String getUserId() { return userId; }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public boolean isActive() { return active; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getLastLoginAt() { return lastLoginAt; }
    public List<String> getActivityLog() { return List.copyOf(activityLog); }
    
    public void printInfo() {
        System.out.println("User: " + username + " (" + userId + ")");
        System.out.println("Email: " + email);
        System.out.println("Active: " + active);
        System.out.println("Created: " + createdAt);
        System.out.println("Last login: " + lastLoginAt);
    }
    
    public static void main(String[] args) {
        UserAccount user = new UserAccount("U001", "johndoe", 
            "john@example.com", "password123");
        user.printInfo();
        
        System.out.println("\nLogin attempts:");
        System.out.println("Correct: " + user.login("password123"));
        System.out.println("Wrong: " + user.login("wrongpass"));
        System.out.println("Wrong: " + user.login("wrongpass"));
        System.out.println("Wrong: " + user.login("wrongpass"));
        System.out.println("Wrong: " + user.login("wrongpass"));
        System.out.println("Wrong: " + user.login("wrongpass"));  // locked!
        System.out.println("Correct (locked): " + user.login("password123"));
        
        System.out.println("\nActivity Log:");
        user.getActivityLog().forEach(log -> System.out.println("  " + log));
    }
}
```

---

## this Keyword

```java
public class Student {
    private String name;
    private int age;
    private double gpa;
    
    // this ใช้แยก field กับ parameter ที่ชื่อเดียวกัน
    public Student(String name, int age, double gpa) {
        this.name = name;  // this.name = field, name = parameter
        this.age = age;
        this.gpa = gpa;
    }
    
    // Constructor chaining ด้วย this()
    public Student(String name) {
        this(name, 18, 0.0);  // เรียก constructor ข้างบน
    }
    
    public Student() {
        this("Unknown");  // เรียก Student(String)
    }
    
    // this ส่งตัวเองเป็น argument
    public void registerForCourse(CourseRegistration reg) {
        reg.register(this);  // ส่ง Student object นี้
    }
    
    // Method chaining ด้วย return this
    public Student setName(String name) {
        this.name = name;
        return this;  // return ตัวเอง
    }
    
    public Student setAge(int age) {
        this.age = age;
        return this;
    }
    
    public Student setGpa(double gpa) {
        this.gpa = gpa;
        return this;
    }
    
    @Override
    public String toString() {
        return String.format("Student{name='%s', age=%d, gpa=%.2f}", name, age, gpa);
    }
    
    public static void main(String[] args) {
        // ใช้ method chaining
        Student s = new Student()
            .setName("Alice")
            .setAge(20)
            .setGpa(3.8);
        System.out.println(s);
        
        // Constructor ต่างๆ
        System.out.println(new Student("Bob", 22, 3.5));
        System.out.println(new Student("Charlie"));
        System.out.println(new Student());
    }
}

class CourseRegistration {
    void register(Student student) {
        System.out.println("Registered: " + student);
    }
}
```

---

## static Members

```java
public class MathConstants {
    // static final: constants
    public static final double PI = 3.141592653589793;
    public static final double E = 2.718281828459045;
    public static final double GOLDEN_RATIO = 1.618033988749895;
    
    // static variable: shared state
    private static int instanceCount = 0;
    
    // static initializer block
    static {
        System.out.println("MathConstants class loaded!");
        // ทำงานครั้งเดียวตอน class ถูก load
    }
    
    // Instance initializer block
    {
        instanceCount++;
        System.out.println("Instance #" + instanceCount + " created");
    }
    
    // static methods
    public static double circleArea(double r) {
        return PI * r * r;
    }
    
    public static boolean isPerfectSquare(int n) {
        int sqrt = (int) Math.sqrt(n);
        return sqrt * sqrt == n;
    }
    
    // static nested class
    public static class Statistics {
        public static double mean(double... data) {
            double sum = 0;
            for (double d : data) sum += d;
            return sum / data.length;
        }
    }
    
    public static void main(String[] args) {
        // เรียกใช้ static members โดยตรง
        System.out.println("PI = " + MathConstants.PI);
        System.out.println("Circle area (r=5): " + MathConstants.circleArea(5));
        System.out.println("isPerfectSquare(16): " + MathConstants.isPerfectSquare(16));
        System.out.println("Mean(1,2,3,4,5): " + MathConstants.Statistics.mean(1,2,3,4,5));
        
        // สร้าง instances
        new MathConstants();
        new MathConstants();
        System.out.println("Total instances: " + instanceCount);
    }
}
```

---

## Enums

```java
public class EnumExamples {
    
    // Basic Enum
    enum Day {
        MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
    }
    
    // Enum with fields and methods
    enum Planet {
        MERCURY(3.303e+23, 2.4397e6),
        VENUS(4.869e+24, 6.0518e6),
        EARTH(5.976e+24, 6.37814e6),
        MARS(6.421e+23, 3.3972e6);
        
        private final double mass;    // kg
        private final double radius;  // meters
        static final double G = 6.67300E-11;
        
        Planet(double mass, double radius) {
            this.mass = mass;
            this.radius = radius;
        }
        
        // Surface gravity
        double surfaceGravity() {
            return G * mass / (radius * radius);
        }
        
        // Weight on this planet
        double surfaceWeight(double otherMass) {
            return otherMass * surfaceGravity();
        }
    }
    
    // Enum with abstract method
    enum Operation {
        PLUS("+") {
            @Override
            public double apply(double x, double y) { return x + y; }
        },
        MINUS("-") {
            @Override
            public double apply(double x, double y) { return x - y; }
        },
        TIMES("*") {
            @Override
            public double apply(double x, double y) { return x * y; }
        },
        DIVIDE("/") {
            @Override
            public double apply(double x, double y) { return x / y; }
        };
        
        private final String symbol;
        
        Operation(String symbol) {
            this.symbol = symbol;
        }
        
        public abstract double apply(double x, double y);
        
        @Override
        public String toString() { return symbol; }
    }
    
    // Enum for Status
    enum OrderStatus {
        PENDING("รอดำเนินการ"),
        CONFIRMED("ยืนยันแล้ว"),
        PROCESSING("กำลังประมวลผล"),
        SHIPPED("จัดส่งแล้ว"),
        DELIVERED("ส่งถึงแล้ว"),
        CANCELLED("ยกเลิก");
        
        private final String thaiLabel;
        
        OrderStatus(String thaiLabel) {
            this.thaiLabel = thaiLabel;
        }
        
        public String getThaiLabel() { return thaiLabel; }
        
        public boolean isTerminal() {
            return this == DELIVERED || this == CANCELLED;
        }
    }
    
    public static void main(String[] args) {
        // Basic enum usage
        Day today = Day.WEDNESDAY;
        System.out.println("Today: " + today);
        System.out.println("Ordinal: " + today.ordinal());   // 2
        System.out.println("Name: " + today.name());          // WEDNESDAY
        
        // switch with enum
        String type = switch (today) {
            case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
            case SATURDAY, SUNDAY -> "Weekend";
        };
        System.out.println("Type: " + type);
        
        // Iterate enum values
        System.out.println("\nAll days:");
        for (Day day : Day.values()) {
            System.out.println("  " + day.ordinal() + ": " + day);
        }
        
        // Enum from String
        Day friday = Day.valueOf("FRIDAY");
        System.out.println("Parsed: " + friday);
        
        // Planet enum
        System.out.println("\nWeight on each planet (75 kg person):");
        double earthWeight = 75;
        double mass = earthWeight / Planet.EARTH.surfaceGravity();
        for (Planet p : Planet.values()) {
            System.out.printf("  %s: %.2f N%n", p, p.surfaceWeight(mass));
        }
        
        // Operation enum
        System.out.println("\nCalculations:");
        double x = 10, y = 3;
        for (Operation op : Operation.values()) {
            System.out.printf("  %.1f %s %.1f = %.2f%n", x, op, y, op.apply(x, y));
        }
        
        // OrderStatus
        System.out.println("\nOrder Status:");
        OrderStatus status = OrderStatus.PENDING;
        System.out.println(status.getThaiLabel());
        System.out.println("Is terminal: " + status.isTerminal());
        status = OrderStatus.DELIVERED;
        System.out.println(status.getThaiLabel());
        System.out.println("Is terminal: " + status.isTerminal());
    }
}
```

---

## Records (Java 16+)

Records เป็น immutable data classes ที่ลด boilerplate code

```java
public class RecordExamples {
    
    // Basic Record
    record Point(double x, double y) {
        // Compact constructor (validation)
        Point {
            if (Double.isNaN(x) || Double.isNaN(y)) {
                throw new IllegalArgumentException("Coordinates cannot be NaN");
            }
        }
        
        // Computed method
        double distanceTo(Point other) {
            double dx = this.x - other.x;
            double dy = this.y - other.y;
            return Math.sqrt(dx * dx + dy * dy);
        }
        
        double distanceFromOrigin() {
            return distanceTo(new Point(0, 0));
        }
        
        // Static factory method
        static Point origin() { return new Point(0, 0); }
    }
    
    // Record ที่ซับซ้อนขึ้น
    record Address(String street, String city, String country, String zipCode) {
        
        // Compact constructor with validation
        Address {
            city = city.trim();
            country = country.toUpperCase();
            if (zipCode != null && !zipCode.matches("\\d{5}")) {
                // zipCode = "00000"; // normalize
            }
        }
        
        String fullAddress() {
            return street + ", " + city + ", " + country + " " + zipCode;
        }
    }
    
    record Person(String name, int age, Address address) {
        
        // Canonical constructor with validation
        Person {
            if (name == null || name.isBlank()) throw new IllegalArgumentException("Name required");
            if (age < 0) throw new IllegalArgumentException("Age must be non-negative");
        }
        
        // Additional methods
        boolean isAdult() { return age >= 18; }
        String city() { return address.city(); }
    }
    
    // Nested records
    record LineSegment(Point start, Point end) {
        double length() { return start.distanceTo(end); }
        Point midpoint() {
            return new Point((start.x() + end.x()) / 2, 
                            (start.y() + end.y()) / 2);
        }
    }
    
    // Generic record (Java 16+)
    record Pair<A, B>(A first, B second) {
        static <X, Y> Pair<X, Y> of(X first, Y second) {
            return new Pair<>(first, second);
        }
    }
    
    public static void main(String[] args) {
        // Point
        Point p1 = new Point(3.0, 4.0);
        Point p2 = new Point(0.0, 0.0);
        System.out.println("Point: " + p1);
        System.out.println("Distance to origin: " + p1.distanceFromOrigin());
        System.out.println("Distance p1 to p2: " + p1.distanceTo(p2));
        
        // Record เป็น immutable
        // p1.x = 5;  // Error! No setter
        
        // Record มี equals, hashCode, toString อัตโนมัติ
        Point p3 = new Point(3.0, 4.0);
        System.out.println("p1.equals(p3): " + p1.equals(p3));  // true
        
        // Address
        Address addr = new Address("123 Main St", " Bangkok ", "thailand", "10100");
        System.out.println("\nAddress: " + addr.fullAddress());
        System.out.println("City: " + addr.city());  // "Bangkok" (trimmed)
        System.out.println("Country: " + addr.country());  // "THAILAND"
        
        // Person
        Person person = new Person("John", 25, addr);
        System.out.println("\nPerson: " + person);
        System.out.println("Is adult: " + person.isAdult());
        System.out.println("City: " + person.city());
        
        // Pair
        Pair<String, Integer> pair = Pair.of("score", 100);
        System.out.println("\nPair: " + pair.first() + "=" + pair.second());
        
        // LineSegment
        LineSegment line = new LineSegment(new Point(0,0), new Point(3,4));
        System.out.println("\nLine length: " + line.length());
        System.out.println("Midpoint: " + line.midpoint());
    }
}
```

---

## โปรแกรมตัวอย่าง: ระบบจัดการห้องสมุด

```java
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

public class LibrarySystem {
    
    // Book record
    record Book(String isbn, String title, String author, int year) {
        @Override
        public String toString() {
            return String.format("[%s] %s by %s (%d)", isbn, title, author, year);
        }
    }
    
    // Member class
    static class Member {
        private final String memberId;
        private String name;
        private String email;
        private final List<String> borrowedBooks = new ArrayList<>();
        private static final int MAX_BOOKS = 5;
        
        Member(String memberId, String name, String email) {
            this.memberId = memberId;
            this.name = name;
            this.email = email;
        }
        
        boolean canBorrow() { return borrowedBooks.size() < MAX_BOOKS; }
        void addBook(String isbn) { borrowedBooks.add(isbn); }
        boolean removeBook(String isbn) { return borrowedBooks.remove(isbn); }
        
        String getMemberId() { return memberId; }
        String getName() { return name; }
        List<String> getBorrowedBooks() { return List.copyOf(borrowedBooks); }
        
        @Override
        public String toString() {
            return String.format("Member{%s, %s, books=%s}", memberId, name, borrowedBooks);
        }
    }
    
    // Loan record
    record Loan(String loanId, String isbn, String memberId, 
                LocalDate borrowDate, LocalDate dueDate) {
        boolean isOverdue() { return LocalDate.now().isAfter(dueDate); }
        long daysOverdue() {
            return isOverdue() ? LocalDate.now().toEpochDay() - dueDate.toEpochDay() : 0;
        }
    }
    
    // Library
    static class Library {
        private final List<Book> books = new ArrayList<>();
        private final List<Member> members = new ArrayList<>();
        private final List<Loan> loans = new ArrayList<>();
        private int loanCounter = 0;
        
        void addBook(Book book) {
            books.add(book);
            System.out.println("Added: " + book);
        }
        
        void registerMember(Member member) {
            members.add(member);
            System.out.println("Registered: " + member.getName());
        }
        
        Optional<Book> findBook(String isbn) {
            return books.stream().filter(b -> b.isbn().equals(isbn)).findFirst();
        }
        
        Optional<Member> findMember(String memberId) {
            return members.stream().filter(m -> m.getMemberId().equals(memberId)).findFirst();
        }
        
        String borrowBook(String isbn, String memberId) {
            Optional<Book> book = findBook(isbn);
            if (book.isEmpty()) return "ไม่พบหนังสือ: " + isbn;
            
            Optional<Member> member = findMember(memberId);
            if (member.isEmpty()) return "ไม่พบสมาชิก: " + memberId;
            
            if (!member.get().canBorrow()) return "ยืมหนังสือเกินกำหนด (max 5)";
            
            boolean alreadyBorrowed = loans.stream()
                .anyMatch(l -> l.isbn().equals(isbn));
            if (alreadyBorrowed) return "หนังสือถูกยืมไปแล้ว";
            
            String loanId = "L" + String.format("%04d", ++loanCounter);
            Loan loan = new Loan(loanId, isbn, memberId, 
                LocalDate.now(), LocalDate.now().plusDays(14));
            loans.add(loan);
            member.get().addBook(isbn);
            
            return String.format("ยืมสำเร็จ! รหัส %s กำหนดคืน %s", 
                loanId, loan.dueDate());
        }
        
        String returnBook(String isbn, String memberId) {
            Loan loan = loans.stream()
                .filter(l -> l.isbn().equals(isbn) && l.memberId().equals(memberId))
                .findFirst()
                .orElse(null);
            
            if (loan == null) return "ไม่พบรายการยืม";
            
            Member member = findMember(memberId).get();
            member.removeBook(isbn);
            loans.remove(loan);
            
            if (loan.isOverdue()) {
                double fine = loan.daysOverdue() * 5.0;  // 5 บาท/วัน
                return String.format("คืนสำเร็จ (เกินกำหนด %d วัน ค่าปรับ %.2f บาท)", 
                    loan.daysOverdue(), fine);
            }
            return "คืนหนังสือสำเร็จ";
        }
        
        void printStatus() {
            System.out.println("\n=== สถานะห้องสมุด ===");
            System.out.println("หนังสือทั้งหมด: " + books.size());
            System.out.println("สมาชิกทั้งหมด: " + members.size());
            System.out.println("รายการยืมปัจจุบัน: " + loans.size());
            
            if (!loans.isEmpty()) {
                System.out.println("\nรายการยืม:");
                for (Loan loan : loans) {
                    Book book = findBook(loan.isbn()).get();
                    Member member = findMember(loan.memberId()).get();
                    System.out.printf("  %s: %s ยืมโดย %s (ครบ %s)%s%n",
                        loan.loanId(), book.title(), member.getName(),
                        loan.dueDate(), loan.isOverdue() ? " ⚠️OVERDUE" : "");
                }
            }
        }
    }
    
    public static void main(String[] args) {
        Library library = new Library();
        
        // เพิ่มหนังสือ
        library.addBook(new Book("978-001", "Java Programming", "James Gosling", 2023));
        library.addBook(new Book("978-002", "Spring Boot in Action", "Craig Walls", 2022));
        library.addBook(new Book("978-003", "Clean Code", "Robert Martin", 2008));
        
        // ลงทะเบียนสมาชิก
        library.registerMember(new Member("M001", "สมชาย", "somchai@email.com"));
        library.registerMember(new Member("M002", "สมหญิง", "somying@email.com"));
        
        System.out.println();
        
        // ยืมหนังสือ
        System.out.println(library.borrowBook("978-001", "M001"));
        System.out.println(library.borrowBook("978-002", "M001"));
        System.out.println(library.borrowBook("978-001", "M002"));  // already borrowed
        System.out.println(library.borrowBook("978-003", "M002"));
        
        library.printStatus();
        
        // คืนหนังสือ
        System.out.println("\n" + library.returnBook("978-001", "M001"));
        System.out.println(library.borrowBook("978-001", "M002"));  // now available
        
        library.printStatus();
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ หลักการ OOP 4 ข้อ  
✅ Classes และ Objects  
✅ Constructors ทุกประเภท  
✅ Builder Pattern  
✅ Fields และ Methods  
✅ Access Modifiers  
✅ Encapsulation  
✅ this Keyword  
✅ static Members  
✅ Enums พร้อม Fields และ Methods  
✅ Records (Java 16+)  

---

## ขั้นตอนต่อไป

**Part 008:** Inheritance & Polymorphism  
เราจะเรียนรู้:
- extends keyword
- super keyword
- Method Overriding
- Polymorphism
- instanceof
- Object class

---

*Part 007 | Java & Spring Boot Course | สร้างโดย Claude Code*
