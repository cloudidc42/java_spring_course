# Part 008: Inheritance & Polymorphism
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Inheritance คืออะไร?](#inheritance-คืออะไร)
2. [extends Keyword](#extends-keyword)
3. [super Keyword](#super-keyword)
4. [Method Overriding](#method-overriding)
5. [Polymorphism](#polymorphism)
6. [instanceof และ Pattern Matching](#instanceof-และ-pattern-matching)
7. [Object Class](#object-class)
8. [Abstract Classes](#abstract-classes)
9. [final Keyword](#final-keyword)
10. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)

---

## Inheritance คืออะไร?

Inheritance ให้ class ลูก (subclass) สืบทอดคุณสมบัติจาก class แม่ (superclass)

```
Animal (superclass)
├── name
├── sound()
│
├── Dog (subclass)
│   ├── breed
│   └── fetch()
│
├── Cat (subclass)
│   ├── indoor
│   └── purr()
│
└── Bird (subclass)
    ├── wingspan
    └── fly()
```

### ประโยชน์ของ Inheritance
- **Code Reuse**: เขียนโค้ดครั้งเดียว ใช้ได้หลาย class
- **Extensibility**: เพิ่มฟีเจอร์ใหม่ได้ง่าย
- **Hierarchy**: จัดระเบียบ class ให้เข้าใจง่าย

---

## extends Keyword

```java
// Superclass
public class Animal {
    protected String name;
    protected int age;
    protected String sound;
    
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
        this.sound = "...";
    }
    
    public void makeSound() {
        System.out.println(name + " says: " + sound);
    }
    
    public void eat(String food) {
        System.out.println(name + " is eating " + food);
    }
    
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
    
    public String getInfo() {
        return String.format("Animal: %s, Age: %d", name, age);
    }
}

// Subclass: Dog extends Animal
public class Dog extends Animal {
    private String breed;
    private boolean trained;
    
    public Dog(String name, int age, String breed) {
        super(name, age);   // เรียก constructor ของ Animal
        this.breed = breed;
        this.trained = false;
        this.sound = "Woof!";  // override sound จาก Animal
    }
    
    // Method เพิ่มเติม
    public void fetch(String item) {
        System.out.println(name + " fetches the " + item);
    }
    
    public void train() {
        this.trained = true;
        System.out.println(name + " is trained!");
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + ", Breed: " + breed + ", Trained: " + trained;
    }
    
    public String getBreed() { return breed; }
    public boolean isTrained() { return trained; }
}

// Subclass: Cat extends Animal
public class Cat extends Animal {
    private boolean isIndoor;
    private int lives;
    
    public Cat(String name, int age, boolean isIndoor) {
        super(name, age);
        this.isIndoor = isIndoor;
        this.lives = 9;
        this.sound = "Meow!";
    }
    
    public void purr() {
        System.out.println(name + " is purring... purrr~");
    }
    
    public void useLife() {
        if (lives > 0) lives--;
        System.out.println(name + " has " + lives + " lives left");
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + ", Indoor: " + isIndoor + ", Lives: " + lives;
    }
}

// Main
public class AnimalTest {
    public static void main(String[] args) {
        // สร้าง objects
        Dog dog = new Dog("Buddy", 3, "Labrador");
        Cat cat = new Cat("Whiskers", 5, true);
        
        // Dog
        System.out.println("=== Dog ===");
        dog.makeSound();
        dog.eat("bone");
        dog.fetch("ball");
        dog.train();
        System.out.println(dog.getInfo());
        
        // Cat
        System.out.println("\n=== Cat ===");
        cat.makeSound();
        cat.eat("fish");
        cat.purr();
        cat.useLife();
        System.out.println(cat.getInfo());
        
        // Dog เป็น Animal ด้วย
        Animal animal = dog;  // upcasting
        animal.makeSound();   // เรียกได้
        // animal.fetch("stick");  // Error! Animal ไม่มี fetch
        
        // ตรวจสอบ type
        System.out.println("\ndog instanceof Dog: " + (dog instanceof Dog));
        System.out.println("dog instanceof Animal: " + (dog instanceof Animal));
        System.out.println("cat instanceof Dog: " + (cat instanceof Dog));
    }
}
```

---

## super Keyword

```java
public class Vehicle {
    protected String brand;
    protected String model;
    protected int year;
    protected double fuelLevel;
    
    public Vehicle(String brand, String model, int year) {
        this.brand = brand;
        this.model = model;
        this.year = year;
        this.fuelLevel = 100.0;
    }
    
    public void startEngine() {
        System.out.println(brand + " " + model + " engine started");
    }
    
    public void refuel(double amount) {
        fuelLevel = Math.min(100, fuelLevel + amount);
        System.out.printf("%s refueled. Fuel: %.1f%%%n", brand, fuelLevel);
    }
    
    public String getInfo() {
        return String.format("%d %s %s (Fuel: %.1f%%)", year, brand, model, fuelLevel);
    }
}

public class ElectricCar extends Vehicle {
    private double batteryLevel;
    private int range;  // km
    
    public ElectricCar(String brand, String model, int year, int range) {
        super(brand, model, year);   // เรียก Vehicle constructor
        this.batteryLevel = 100.0;
        this.range = range;
        this.fuelLevel = 0;  // ไม่ใช้น้ำมัน
    }
    
    @Override
    public void startEngine() {
        System.out.println("⚡ " + brand + " electric motor activated silently");
        super.startEngine();   // เรียก method ของ parent (optional)
    }
    
    // Override refuel to charge instead
    @Override
    public void refuel(double amount) {
        System.out.println("Charging is not called 'refueling' for EVs!");
        charge(amount);
    }
    
    public void charge(double percent) {
        batteryLevel = Math.min(100, batteryLevel + percent);
        System.out.printf("%s charged. Battery: %.1f%% (Range: ~%.0f km)%n",
            brand, batteryLevel, batteryLevel / 100 * range);
    }
    
    @Override
    public String getInfo() {
        // super.getInfo() ดึงข้อมูลจาก parent
        return super.getInfo().replace("Fuel: 0.0%", "") +
               String.format(" Battery: %.1f%% Range: %d km", batteryLevel, range);
    }
}

public class HybridCar extends Vehicle {
    private double batteryLevel;
    
    public HybridCar(String brand, String model, int year) {
        super(brand, model, year);
        this.batteryLevel = 80.0;
    }
    
    @Override
    public void startEngine() {
        System.out.println("🔋 " + brand + " starts on battery...");
        if (batteryLevel < 20) {
            System.out.println("Battery low, switching to fuel");
            super.startEngine();
        }
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + String.format(" + Battery: %.1f%%", batteryLevel);
    }
}

public class VehicleTest {
    public static void main(String[] args) {
        Vehicle vehicle = new Vehicle("Toyota", "Camry", 2023);
        ElectricCar tesla = new ElectricCar("Tesla", "Model 3", 2024, 480);
        HybridCar prius = new HybridCar("Toyota", "Prius", 2023);
        
        vehicle.startEngine();
        System.out.println(vehicle.getInfo());
        
        System.out.println();
        tesla.startEngine();
        tesla.charge(20);
        System.out.println(tesla.getInfo());
        
        System.out.println();
        prius.startEngine();
        System.out.println(prius.getInfo());
    }
}
```

---

## Method Overriding

```java
public class Shape {
    protected String color;
    
    public Shape(String color) {
        this.color = color;
    }
    
    // Method ที่ subclass จะ override
    public double area() {
        return 0;
    }
    
    public double perimeter() {
        return 0;
    }
    
    public String describe() {
        return String.format("%s %s (area=%.2f, perimeter=%.2f)",
            color, getClass().getSimpleName(), area(), perimeter());
    }
    
    @Override
    public String toString() {
        return describe();
    }
}

public class Circle extends Shape {
    private double radius;
    
    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
    
    public double getRadius() { return radius; }
}

public class Rectangle extends Shape {
    private double width, height;
    
    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double area() {
        return width * height;
    }
    
    @Override
    public double perimeter() {
        return 2 * (width + height);
    }
    
    public boolean isSquare() {
        return width == height;
    }
}

public class Triangle extends Shape {
    private double a, b, c;
    
    public Triangle(String color, double a, double b, double c) {
        super(color);
        if (a + b <= c || a + c <= b || b + c <= a) {
            throw new IllegalArgumentException("Invalid triangle sides");
        }
        this.a = a;
        this.b = b;
        this.c = c;
    }
    
    @Override
    public double area() {
        // Heron's formula
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
    
    @Override
    public double perimeter() {
        return a + b + c;
    }
}

public class ShapeTest {
    public static void main(String[] args) {
        Shape[] shapes = {
            new Circle("Red", 5),
            new Rectangle("Blue", 4, 6),
            new Triangle("Green", 3, 4, 5),
            new Circle("Yellow", 3),
            new Rectangle("Purple", 5, 5)
        };
        
        // Polymorphism: เรียก method เดียวกัน ผลต่างกัน
        System.out.println("All Shapes:");
        for (Shape s : shapes) {
            System.out.println("  " + s.describe());
        }
        
        // หา shape ที่มีพื้นที่ใหญ่สุด
        Shape largest = shapes[0];
        for (Shape s : shapes) {
            if (s.area() > largest.area()) largest = s;
        }
        System.out.println("\nLargest: " + largest.describe());
        
        // รวมพื้นที่ทั้งหมด
        double totalArea = 0;
        for (Shape s : shapes) totalArea += s.area();
        System.out.printf("Total area: %.2f%n", totalArea);
        
        // Type-specific operations
        for (Shape s : shapes) {
            if (s instanceof Rectangle r && r.isSquare()) {
                System.out.println("Square found: " + r.describe());
            }
        }
    }
}
```

---

## Polymorphism

```java
import java.util.ArrayList;
import java.util.List;

public class PolymorphismDemo {
    
    // Abstract base class
    static abstract class Employee {
        protected String name;
        protected String employeeId;
        protected double baseSalary;
        
        Employee(String name, String employeeId, double baseSalary) {
            this.name = name;
            this.employeeId = employeeId;
            this.baseSalary = baseSalary;
        }
        
        // Template method pattern
        public final double calculateMonthlyPay() {
            double base = baseSalary;
            double bonus = calculateBonus();
            double tax = calculateTax(base + bonus);
            return base + bonus - tax;
        }
        
        protected abstract double calculateBonus();
        
        protected double calculateTax(double income) {
            if (income <= 20000) return 0;
            if (income <= 30000) return income * 0.05;
            if (income <= 50000) return income * 0.10;
            return income * 0.15;
        }
        
        public void printPaySlip() {
            double bonus = calculateBonus();
            double gross = baseSalary + bonus;
            double tax = calculateTax(gross);
            
            System.out.printf("═══════════════════════════════════%n");
            System.out.printf("Payslip: %s (%s)%n", name, employeeId);
            System.out.printf("Position: %s%n", getClass().getSimpleName());
            System.out.printf("Base: %10.2f%n", baseSalary);
            System.out.printf("Bonus: %9.2f%n", bonus);
            System.out.printf("Tax: %11.2f%n", tax);
            System.out.printf("Net: %11.2f%n", calculateMonthlyPay());
            System.out.printf("═══════════════════════════════════%n");
        }
        
        public String getName() { return name; }
        public String getEmployeeId() { return employeeId; }
    }
    
    // Subclasses
    static class FullTimeEmployee extends Employee {
        private double performanceScore;  // 1-5
        
        FullTimeEmployee(String name, String id, double salary, double score) {
            super(name, id, salary);
            this.performanceScore = score;
        }
        
        @Override
        protected double calculateBonus() {
            return baseSalary * (performanceScore / 100.0);
        }
    }
    
    static class PartTimeEmployee extends Employee {
        private int hoursWorked;
        private double hourlyRate;
        
        PartTimeEmployee(String name, String id, int hours, double hourlyRate) {
            super(name, id, hours * hourlyRate);
            this.hoursWorked = hours;
            this.hourlyRate = hourlyRate;
        }
        
        @Override
        protected double calculateBonus() {
            return hoursWorked > 160 ? (hoursWorked - 160) * hourlyRate * 0.5 : 0;
        }
    }
    
    static class Manager extends FullTimeEmployee {
        private int teamSize;
        
        Manager(String name, String id, double salary, double score, int teamSize) {
            super(name, id, salary, score);
            this.teamSize = teamSize;
        }
        
        @Override
        protected double calculateBonus() {
            double performanceBonus = super.calculateBonus();
            double teamBonus = teamSize * 500;  // 500 per team member
            return performanceBonus + teamBonus;
        }
    }
    
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>();
        employees.add(new FullTimeEmployee("Alice", "E001", 45000, 4.5));
        employees.add(new PartTimeEmployee("Bob", "E002", 120, 200));
        employees.add(new Manager("Charlie", "E003", 80000, 4.8, 8));
        employees.add(new FullTimeEmployee("Diana", "E004", 38000, 3.8));
        
        // Polymorphism: เรียก calculateMonthlyPay() ต่างกันตาม type
        System.out.println("Monthly Payroll Summary:");
        System.out.printf("%-15s %10s %10s %10s%n", "Name", "Base", "Bonus", "Net");
        System.out.println("-".repeat(50));
        
        double totalPayroll = 0;
        for (Employee emp : employees) {
            double net = emp.calculateMonthlyPay();
            totalPayroll += net;
            System.out.printf("%-15s %10.2f %10.2f %10.2f%n",
                emp.getName(), emp.baseSalary, 
                emp.calculateBonus(), net);
        }
        System.out.println("-".repeat(50));
        System.out.printf("%-35s %10.2f%n", "Total Payroll:", totalPayroll);
        
        // Print individual pay slips
        System.out.println("\n");
        employees.get(2).printPaySlip();  // Manager payslip
    }
}
```

---

## instanceof และ Pattern Matching

```java
public class InstanceofPatterns {
    
    sealed interface Payment permits CreditCard, Cash, BankTransfer {}
    
    record CreditCard(String number, String holderName, int cvv) implements Payment {}
    record Cash(double amount, String currency) implements Payment {}
    record BankTransfer(String fromAccount, String toAccount, double amount) implements Payment {}
    
    static String processPayment(Payment payment) {
        // Pattern Matching (Java 21)
        return switch (payment) {
            case CreditCard cc when cc.number().startsWith("4") ->
                "Visa Card: " + maskCard(cc.number());
            case CreditCard cc when cc.number().startsWith("5") ->
                "MasterCard: " + maskCard(cc.number());
            case CreditCard cc ->
                "Card: " + maskCard(cc.number());
            case Cash c ->
                String.format("Cash payment: %.2f %s", c.amount(), c.currency());
            case BankTransfer bt ->
                String.format("Transfer: %s -> %s (%.2f)", 
                    maskAccount(bt.fromAccount()), bt.toAccount(), bt.amount());
        };
    }
    
    static String maskCard(String number) {
        return "**** **** **** " + number.substring(number.length() - 4);
    }
    
    static String maskAccount(String account) {
        return account.substring(0, 3) + "****" + account.substring(account.length() - 3);
    }
    
    // instanceof แบบเก่า
    static void oldStyleCheck(Object obj) {
        if (obj instanceof String) {
            String s = (String) obj;
            System.out.println("String length: " + s.length());
        } else if (obj instanceof Integer) {
            Integer i = (Integer) obj;
            System.out.println("Integer * 2 = " + (i * 2));
        } else if (obj instanceof double[]) {
            double[] arr = (double[]) obj;
            System.out.println("Array length: " + arr.length);
        }
    }
    
    // instanceof แบบใหม่ (Java 16+)
    static void newStyleCheck(Object obj) {
        if (obj instanceof String s) {
            System.out.println("String length: " + s.length());
        } else if (obj instanceof Integer i) {
            System.out.println("Integer * 2 = " + (i * 2));
        } else if (obj instanceof double[] arr) {
            System.out.println("Array length: " + arr.length);
        }
    }
    
    public static void main(String[] args) {
        Payment[] payments = {
            new CreditCard("4111111111111111", "John Doe", 123),
            new CreditCard("5500000000000004", "Jane Doe", 456),
            new Cash(500.00, "THB"),
            new BankTransfer("1234567890", "0987654321", 1000.00)
        };
        
        System.out.println("Processing payments:");
        for (Payment p : payments) {
            System.out.println("  " + processPayment(p));
        }
        
        System.out.println("\nObject type checks:");
        Object[] objects = {"Hello", 42, new double[]{1.0, 2.0, 3.0}, null};
        for (Object obj : objects) {
            if (obj != null) {
                newStyleCheck(obj);
            } else {
                System.out.println("null value");
            }
        }
    }
}
```

---

## Object Class

ทุก class ใน Java สืบทอดจาก `java.lang.Object`

```java
import java.util.Objects;

public class ObjectClassMethods {
    
    // ตัวอย่าง class ที่ override Object methods
    static class Point {
        private final int x;
        private final int y;
        
        Point(int x, int y) {
            this.x = x;
            this.y = y;
        }
        
        // Override toString
        @Override
        public String toString() {
            return "(" + x + ", " + y + ")";
        }
        
        // Override equals - สำคัญมาก!
        @Override
        public boolean equals(Object obj) {
            if (this == obj) return true;         // ตัวเอง
            if (obj == null) return false;         // null
            if (!(obj instanceof Point)) return false;  // ต่าง type
            Point other = (Point) obj;
            return x == other.x && y == other.y;
        }
        
        // Override hashCode - ต้อง override ด้วยเสมอถ้า override equals!
        @Override
        public int hashCode() {
            return Objects.hash(x, y);  // ใช้ Objects.hash ง่ายที่สุด
        }
        
        // Override clone (ถ้าต้องการ)
        @Override
        protected Point clone() {
            return new Point(x, y);
        }
        
        public int getX() { return x; }
        public int getY() { return y; }
    }
    
    // ตัวอย่าง Product ที่ override ทุก method
    static class Product {
        private final String id;
        private final String name;
        private double price;
        
        Product(String id, String name, double price) {
            this.id = id;
            this.name = name;
            this.price = price;
        }
        
        @Override
        public String toString() {
            return String.format("Product{id='%s', name='%s', price=%.2f}", id, name, price);
        }
        
        @Override
        public boolean equals(Object obj) {
            if (this == obj) return true;
            if (!(obj instanceof Product p)) return false;
            return Objects.equals(id, p.id);  // Products เท่ากันถ้า id เหมือนกัน
        }
        
        @Override
        public int hashCode() {
            return Objects.hash(id);
        }
        
        // Comparable สำหรับ sorting
        public int compareTo(Product other) {
            return Double.compare(this.price, other.price);
        }
    }
    
    public static void main(String[] args) {
        // toString
        Point p1 = new Point(3, 4);
        System.out.println("p1 = " + p1);        // calls toString
        System.out.println("p1 = " + p1.toString()); // explicit
        
        // equals
        Point p2 = new Point(3, 4);
        Point p3 = new Point(1, 2);
        System.out.println("\np1.equals(p2): " + p1.equals(p2));  // true
        System.out.println("p1.equals(p3): " + p1.equals(p3));   // false
        System.out.println("p1 == p2: " + (p1 == p2));            // false (different objects)
        
        // hashCode
        System.out.println("\np1.hashCode(): " + p1.hashCode());
        System.out.println("p2.hashCode(): " + p2.hashCode());
        System.out.println("Same? " + (p1.hashCode() == p2.hashCode())); // true
        
        // getClass
        System.out.println("\np1.getClass(): " + p1.getClass());
        System.out.println("p1.getClass().getName(): " + p1.getClass().getName());
        System.out.println("p1.getClass().getSimpleName(): " + p1.getClass().getSimpleName());
        
        // ใน Collections - equals/hashCode สำคัญมาก
        java.util.Set<Point> set = new java.util.HashSet<>();
        set.add(new Point(1, 2));
        set.add(new Point(1, 2));  // ซ้ำ!
        set.add(new Point(3, 4));
        System.out.println("\nSet size: " + set.size());  // 2, ไม่ใช่ 3
        System.out.println("Contains (1,2): " + set.contains(new Point(1, 2)));  // true
        
        // Objects utility class
        String s1 = null, s2 = "Hello";
        System.out.println("\nObjects.equals(null, null): " + Objects.equals(s1, s1));
        System.out.println("Objects.equals(null, str): " + Objects.equals(s1, s2));
        System.out.println("Objects.toString(null): " + Objects.toString(s1, "default"));
        System.out.println("Objects.requireNonNull: ");
        try {
            Objects.requireNonNull(s1, "s1 cannot be null");
        } catch (NullPointerException e) {
            System.out.println("  " + e.getMessage());
        }
    }
}
```

---

## Abstract Classes

```java
public abstract class AbstractVehicle {
    protected String brand;
    protected String model;
    protected int year;
    
    // Constructor
    public AbstractVehicle(String brand, String model, int year) {
        this.brand = brand;
        this.model = model;
        this.year = year;
    }
    
    // Abstract methods: subclass MUST implement
    public abstract double fuelConsumption();  // km per liter/kWh
    public abstract String fuelType();
    
    // Concrete methods: subclass CAN override
    public String startEngine() {
        return brand + " " + model + " engine started";
    }
    
    public String getInfo() {
        return String.format("%d %s %s [%s, %.1f km/%s]",
            year, brand, model, fuelType(), fuelConsumption(), fuelType().equals("Electric") ? "kWh" : "L");
    }
    
    // Template method
    public final void describe() {
        System.out.println("=".repeat(40));
        System.out.println(getInfo());
        System.out.println("Engine: " + startEngine());
        System.out.println("Eco rating: " + getEcoRating());
        System.out.println("=".repeat(40));
    }
    
    // Can be overridden
    protected String getEcoRating() {
        double fc = fuelConsumption();
        if (fc > 20) return "⭐⭐⭐⭐⭐ Excellent";
        if (fc > 15) return "⭐⭐⭐⭐ Good";
        if (fc > 10) return "⭐⭐⭐ Average";
        if (fc > 5)  return "⭐⭐ Below Average";
        return "⭐ Poor";
    }
}

class GasCar extends AbstractVehicle {
    private double engineSize;
    
    GasCar(String brand, String model, int year, double engineSize) {
        super(brand, model, year);
        this.engineSize = engineSize;
    }
    
    @Override
    public double fuelConsumption() {
        return 40 - (engineSize * 5);  // larger engine = less efficient
    }
    
    @Override
    public String fuelType() { return "Gasoline"; }
}

class ElectricVehicle extends AbstractVehicle {
    private int batteryCapacity;  // kWh
    private int range;            // km
    
    ElectricVehicle(String brand, String model, int year, int battery, int range) {
        super(brand, model, year);
        this.batteryCapacity = battery;
        this.range = range;
    }
    
    @Override
    public double fuelConsumption() {
        return (double) range / batteryCapacity;
    }
    
    @Override
    public String fuelType() { return "Electric"; }
    
    @Override
    public String startEngine() {
        return "⚡ " + brand + " " + model + " electric motor activated (silent)";
    }
    
    @Override
    protected String getEcoRating() {
        return "⭐⭐⭐⭐⭐ Zero Emission";
    }
}

class AbstractVehicleTest {
    public static void main(String[] args) {
        AbstractVehicle[] vehicles = {
            new GasCar("Toyota", "Camry 2.5", 2023, 2.5),
            new GasCar("BMW", "M3", 2023, 3.0),
            new ElectricVehicle("Tesla", "Model 3", 2024, 75, 500),
            new ElectricVehicle("BYD", "Atto 3", 2024, 60, 420)
        };
        
        for (AbstractVehicle v : vehicles) {
            v.describe();
            System.out.println();
        }
    }
}
```

---

## final Keyword

```java
public class FinalKeyword {
    
    // final class: ไม่สามารถ extend ได้
    public static final class ImmutablePoint {
        private final int x;
        private final int y;
        
        ImmutablePoint(int x, int y) {
            this.x = x;
            this.y = y;
        }
        
        public int getX() { return x; }
        public int getY() { return y; }
        
        @Override
        public String toString() { return "(" + x + ", " + y + ")"; }
    }
    
    // class ที่มี final method
    static class Base {
        // final method: ไม่สามารถ override ได้
        public final void doSomething() {
            System.out.println("Base implementation - cannot be overridden");
        }
        
        public void overrideable() {
            System.out.println("This CAN be overridden");
        }
    }
    
    static class Derived extends Base {
        // doSomething() ไม่สามารถ override ได้
        // void doSomething() {}  // Compilation Error!
        
        @Override
        public void overrideable() {
            System.out.println("Derived overrides this");
        }
    }
    
    // String เป็น final class
    // class MyString extends String {}  // Error!
    
    public static void main(String[] args) {
        // final variable
        final int MAX = 100;
        // MAX = 200;  // Error!
        
        // final reference (reference ไม่เปลี่ยน แต่ object ข้างในเปลี่ยนได้)
        final java.util.List<String> list = new java.util.ArrayList<>();
        list.add("Hello");  // ได้
        list.add("World");  // ได้
        // list = new java.util.ArrayList<>();  // Error! reference ไม่เปลี่ยน
        
        ImmutablePoint p = new ImmutablePoint(3, 4);
        System.out.println("Point: " + p);
        // p.x = 5;  // Error! field เป็น final
        
        // แต่ถ้า final class เราไม่สามารถ extend ได้
        // class DerivedPoint extends ImmutablePoint {}  // Error!
        
        Base base = new Derived();
        base.doSomething();
        base.overrideable();
    }
}
```

---

## โปรแกรมตัวอย่าง: ระบบ E-Commerce

```java
import java.util.*;

public class ECommerceSystem {
    
    // Abstract Product
    static abstract class Product {
        protected final String productId;
        protected String name;
        protected double price;
        protected int stock;
        
        Product(String productId, String name, double price, int stock) {
            this.productId = productId;
            this.name = name;
            this.price = price;
            this.stock = stock;
        }
        
        public abstract double calculateDiscount();
        
        public double getFinalPrice() {
            return price - calculateDiscount();
        }
        
        public boolean isAvailable() { return stock > 0; }
        
        public void decreaseStock(int quantity) {
            if (quantity > stock) throw new IllegalStateException("Insufficient stock");
            stock -= quantity;
        }
        
        @Override
        public String toString() {
            return String.format("[%s] %s - %.2f (%.2f off) | Stock: %d",
                productId, name, price, calculateDiscount(), stock);
        }
    }
    
    // Product Types
    static class Electronics extends Product {
        private int warrantyMonths;
        
        Electronics(String id, String name, double price, int stock, int warranty) {
            super(id, name, price, stock);
            this.warrantyMonths = warranty;
        }
        
        @Override
        public double calculateDiscount() {
            return price >= 10000 ? price * 0.10 : 0;
        }
        
        public int getWarrantyMonths() { return warrantyMonths; }
    }
    
    static class Clothing extends Product {
        private String size;
        private String color;
        private boolean onSale;
        
        Clothing(String id, String name, double price, int stock, 
                 String size, String color, boolean onSale) {
            super(id, name, price, stock);
            this.size = size;
            this.color = color;
            this.onSale = onSale;
        }
        
        @Override
        public double calculateDiscount() {
            return onSale ? price * 0.20 : 0;
        }
    }
    
    static class Food extends Product {
        private String expiryDate;
        private boolean organic;
        
        Food(String id, String name, double price, int stock, 
             String expiry, boolean organic) {
            super(id, name, price, stock);
            this.expiryDate = expiry;
            this.organic = organic;
        }
        
        @Override
        public double calculateDiscount() {
            return organic ? price * 0.05 : 0;
        }
    }
    
    // Shopping Cart
    static class Cart {
        private final String cartId;
        private final Map<Product, Integer> items = new LinkedHashMap<>();
        
        Cart(String cartId) {
            this.cartId = cartId;
        }
        
        void addItem(Product product, int quantity) {
            if (!product.isAvailable()) {
                System.out.println("Product not available: " + product.name);
                return;
            }
            items.merge(product, quantity, Integer::sum);
            System.out.printf("Added %d x %s%n", quantity, product.name);
        }
        
        void removeItem(Product product) {
            items.remove(product);
        }
        
        double getSubtotal() {
            return items.entrySet().stream()
                .mapToDouble(e -> e.getKey().getFinalPrice() * e.getValue())
                .sum();
        }
        
        void printCart() {
            System.out.println("╔═══════════════════════════════════════════════════╗");
            System.out.println("║                  Shopping Cart                    ║");
            System.out.println("╠═══════════════════════════════════════════════════╣");
            System.out.printf("║ %-20s %5s %8s %12s║%n", "Product", "Qty", "Price", "Subtotal");
            System.out.println("╠═══════════════════════════════════════════════════╣");
            
            for (Map.Entry<Product, Integer> entry : items.entrySet()) {
                Product p = entry.getKey();
                int qty = entry.getValue();
                double subtotal = p.getFinalPrice() * qty;
                System.out.printf("║ %-20s %5d %8.2f %12.2f║%n",
                    p.name.length() > 20 ? p.name.substring(0, 17) + "..." : p.name,
                    qty, p.getFinalPrice(), subtotal);
            }
            
            System.out.println("╠═══════════════════════════════════════════════════╣");
            System.out.printf("║ %-39s %12.2f║%n", "Total:", getSubtotal());
            System.out.println("╚═══════════════════════════════════════════════════╝");
        }
    }
    
    public static void main(String[] args) {
        // Products
        Electronics laptop = new Electronics("E001", "MacBook Pro 14", 59900, 10, 24);
        Electronics phone = new Electronics("E002", "iPhone 15 Pro", 42900, 20, 12);
        Clothing shirt = new Clothing("C001", "Cotton T-Shirt", 390, 100, "M", "Blue", true);
        Clothing jeans = new Clothing("C002", "Slim Fit Jeans", 1290, 50, "32", "Black", false);
        Food apple = new Food("F001", "Organic Apples 1kg", 89, 200, "2024-12-31", true);
        
        // Display products
        System.out.println("=== Product Catalog ===");
        List<Product> products = List.of(laptop, phone, shirt, jeans, apple);
        for (Product p : products) {
            System.out.println(p);
        }
        
        // Shopping
        System.out.println("\n=== Shopping Session ===");
        Cart cart = new Cart("CART001");
        cart.addItem(laptop, 1);
        cart.addItem(shirt, 2);
        cart.addItem(apple, 3);
        cart.addItem(phone, 1);
        
        System.out.println();
        cart.printCart();
        
        // Summary stats
        System.out.println("\n=== Category Summary ===");
        double electronicsTotal = products.stream()
            .filter(p -> p instanceof Electronics)
            .mapToDouble(Product::getFinalPrice)
            .sum();
        System.out.printf("Electronics: %.2f%n", electronicsTotal);
        
        long inStockCount = products.stream().filter(Product::isAvailable).count();
        System.out.println("In stock: " + inStockCount + "/" + products.size());
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Inheritance ด้วย extends  
✅ super keyword  
✅ Method Overriding  
✅ Polymorphism  
✅ instanceof และ Pattern Matching  
✅ Object class methods (toString, equals, hashCode)  
✅ Abstract Classes  
✅ final keyword  

---

## ขั้นตอนต่อไป

**Part 009:** Interfaces & Abstract Classes (Advanced)  
เราจะเรียนรู้:
- Interface
- Default Methods
- Functional Interface
- Multiple Inheritance with Interfaces
- Sealed Classes

---

*Part 008 | Java & Spring Boot Course | สร้างโดย Claude Code*
