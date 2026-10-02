# Part 001: Introduction to Java & Environment Setup
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Java คืออะไร?](#java-คืออะไร)
2. [ทำไมต้องเรียน Java?](#ทำไมต้องเรียน-java)
3. [Java Ecosystem](#java-ecosystem)
4. [การติดตั้ง JDK](#การติดตั้ง-jdk)
5. [การติดตั้ง IntelliJ IDEA](#การติดตั้ง-intellij-idea)
6. [โปรแกรมแรก: Hello World](#โปรแกรมแรก-hello-world)
7. [การ Compile และ Run](#การ-compile-และ-run)
8. [โครงสร้างโปรเจกต์ Java](#โครงสร้างโปรเจกต์-java)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Java คืออะไร?

Java เป็นภาษาโปรแกรมเชิงวัตถุ (Object-Oriented Programming) ที่พัฒนาโดย Sun Microsystems
(ปัจจุบัน Oracle) ในปี 1995 โดย James Gosling

### คุณสมบัติสำคัญของ Java

```
Write Once, Run Anywhere (WORA)
```

หมายความว่า โค้ด Java ที่เขียนครั้งเดียว สามารถรันได้บนทุก Platform ที่มี JVM (Java Virtual Machine)

### Java Platform Components

```
┌─────────────────────────────────────────┐
│           Java Source Code (.java)       │
└─────────────────┬───────────────────────┘
                  │ javac (compiler)
                  ▼
┌─────────────────────────────────────────┐
│           Bytecode (.class)              │
└─────────────────┬───────────────────────┘
                  │
        ┌─────────▼─────────┐
        │   JVM (Windows)   │
        │   JVM (Linux)     │
        │   JVM (macOS)     │
        └───────────────────┘
```

---

## ทำไมต้องเรียน Java?

### 1. ความนิยมสูงมาก
- อยู่ใน Top 3 ภาษาโปรแกรมยอดนิยมมาตลอด 20+ ปี
- ใช้ในองค์กรชั้นนำทั่วโลก

### 2. งานมาก รายได้ดี
- Java Developer มีความต้องการสูงในตลาดแรงงาน
- เงินเดือนเฉลี่ยสูงกว่าภาษาอื่น

### 3. Spring Boot = Enterprise Standard
- Spring Boot คือ Framework ที่ใช้งานมากที่สุดสำหรับ Enterprise Java
- ใช้ใน Banking, Finance, E-commerce ชั้นนำ

### 4. Ecosystem ที่แข็งแกร่ง
```
Java Ecosystem:
├── Spring Framework (Web, Security, Data)
├── Hibernate (ORM)
├── Apache Kafka (Message Streaming)
├── Apache Maven / Gradle (Build Tools)
├── JUnit / Mockito (Testing)
├── Docker / Kubernetes (Container)
└── Microservices Architecture
```

---

## Java Ecosystem

### Java SE (Standard Edition)
Java SE คือ Java พื้นฐาน ประกอบด้วย:
- Core Libraries (java.lang, java.util, java.io)
- Collections Framework
- Concurrency
- JDBC

### Java EE / Jakarta EE (Enterprise Edition)
สำหรับการพัฒนา Enterprise Application:
- Servlets & JSP
- EJB (Enterprise JavaBeans)
- JPA (Java Persistence API)
- JAX-RS (REST API)

### Spring Framework
Framework ยอดนิยมที่สร้างบน Java SE:
- Spring Core (IoC Container)
- Spring MVC (Web Framework)
- Spring Boot (Auto-configuration)
- Spring Data (Database Access)
- Spring Security (Authentication & Authorization)
- Spring Cloud (Microservices)

---

## การติดตั้ง JDK

### JDK คืออะไร?
JDK (Java Development Kit) ประกอบด้วย:
- **JRE** (Java Runtime Environment) - สำหรับรัน Java
- **javac** - Java Compiler
- **java** - Java Launcher
- **javadoc** - Documentation Generator
- **jar** - Archive Tool

### เลือก JDK Version ไหน?

แนะนำ **JDK 21** (LTS - Long Term Support) หรือ **JDK 17** (LTS)

```
Java LTS Versions:
- Java 8  (2014) - ยังใช้งานอยู่ในหลายองค์กร
- Java 11 (2018) - LTS
- Java 17 (2021) - LTS ★ แนะนำขั้นต่ำ
- Java 21 (2023) - LTS ★★ แนะนำสุด
```

### การติดตั้งบน Windows

**วิธีที่ 1: ดาวน์โหลดจาก Oracle**
1. ไปที่ https://www.oracle.com/java/technologies/downloads/
2. เลือก Java 21 > Windows > x64 Installer
3. รันไฟล์ .exe และทำตามขั้นตอน

**วิธีที่ 2: ใช้ SDKMAN (แนะนำสำหรับ Developer)**
```bash
# ติดตั้ง SDKMAN (บน Git Bash หรือ WSL)
curl -s "https://get.sdkman.io" | bash

# เปิด terminal ใหม่ แล้ว
sdk install java 21.0.1-oracle

# ตรวจสอบ
java -version
```

**วิธีที่ 3: ใช้ Winget**
```bash
winget install Oracle.JDK.21
```

### การติดตั้งบน macOS

**วิธีที่ 1: ใช้ Homebrew (แนะนำ)**
```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Java
brew install --cask temurin@21

# ตรวจสอบ
java -version
```

**วิธีที่ 2: ใช้ SDKMAN**
```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 21.0.1-tem
```

### การติดตั้งบน Ubuntu/Debian Linux

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง OpenJDK 21
sudo apt install openjdk-21-jdk

# ตรวจสอบ
java -version
javac -version
```

### การติดตั้งบน CentOS/RHEL/Fedora

```bash
# ติดตั้ง OpenJDK 21
sudo dnf install java-21-openjdk-devel

# ตรวจสอบ
java -version
```

### ตั้งค่า JAVA_HOME (สำคัญ!)

**Windows:**
```
1. เปิด System Properties > Advanced > Environment Variables
2. เพิ่ม System Variable ใหม่:
   Variable name: JAVA_HOME
   Variable value: C:\Program Files\Java\jdk-21 (ตามที่ติดตั้ง)
3. แก้ไข Path variable เพิ่ม: %JAVA_HOME%\bin
```

**macOS/Linux:**
```bash
# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

# Reload
source ~/.bashrc
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Java version
java -version
# ผลลัพธ์ที่คาดหวัง:
# openjdk version "21.0.1" 2023-10-17
# OpenJDK Runtime Environment (build 21.0.1+12-29)
# OpenJDK 64-Bit Server VM (build 21.0.1+12-29, mixed mode, sharing)

# ตรวจสอบ Compiler
javac -version
# ผลลัพธ์: javac 21.0.1

# ตรวจสอบ JAVA_HOME
echo $JAVA_HOME  # Linux/Mac
echo %JAVA_HOME% # Windows
```

---

## การติดตั้ง IntelliJ IDEA

IntelliJ IDEA เป็น IDE ที่ดีที่สุดสำหรับ Java Development

### ดาวน์โหลดและติดตั้ง

1. ไปที่ https://www.jetbrains.com/idea/download/
2. เลือก **Community Edition** (ฟรี) หรือ **Ultimate** (เสียเงิน แต่มี Spring Boot support ดีกว่า)
   - สำหรับนักเรียน: ขอ Free License ได้ที่ https://www.jetbrains.com/student/
3. ดาวน์โหลดและติดตั้ง

### การตั้งค่าเบื้องต้น

**ตั้งค่า JDK ใน IntelliJ:**
1. File > Project Structure (Ctrl+Alt+Shift+S)
2. Platform Settings > SDKs
3. กด + > Add JDK
4. เลือก folder ที่ติดตั้ง JDK

**Plugins ที่แนะนำ:**
- **Lombok** - ลด boilerplate code
- **Spring Assistant** - สำหรับ Spring Boot
- **Docker** - สำหรับ Docker
- **GitToolBox** - สำหรับ Git
- **SonarLint** - Code Quality

### Keyboard Shortcuts สำคัญ

| Shortcut | การทำงาน |
|----------|----------|
| `Ctrl+Shift+A` | Find Action |
| `Ctrl+N` | Go to Class |
| `Ctrl+Shift+N` | Go to File |
| `Shift+Shift` | Search Everywhere |
| `Alt+Enter` | Show Intentions |
| `Ctrl+B` | Go to Declaration |
| `Ctrl+Alt+L` | Reformat Code |
| `Ctrl+/` | Line Comment |
| `Ctrl+Shift+F` | Find in Files |
| `Shift+F10` | Run |
| `Shift+F9` | Debug |

---

## โปรแกรมแรก: Hello World

### สร้างโปรเจกต์ใหม่

**ใน IntelliJ IDEA:**
1. File > New > Project
2. เลือก Java
3. ตั้งชื่อ Project: `hello-world`
4. เลือก JDK ที่ติดตั้ง
5. กด Create

### สร้างไฟล์ HelloWorld.java

```java
// ไฟล์: HelloWorld.java
// โปรแกรม Java แรกของเรา

public class HelloWorld {
    
    // main method คือจุดเริ่มต้นของโปรแกรม Java
    public static void main(String[] args) {
        // พิมพ์ข้อความออกทาง console
        System.out.println("Hello, World!");
        System.out.println("ยินดีต้อนรับสู่โลกของ Java!");
    }
}
```

### อธิบายโค้ดทีละบรรทัด

```java
public class HelloWorld {
```
- `public` = Access Modifier (เข้าถึงได้จากทุกที่)
- `class` = คำสำคัญสำหรับประกาศ Class
- `HelloWorld` = ชื่อ Class (ต้องตรงกับชื่อไฟล์)
- `{` = เปิด block ของ class

```java
    public static void main(String[] args) {
```
- `public` = เข้าถึงได้จากทุกที่
- `static` = เรียกใช้ได้โดยไม่ต้องสร้าง object
- `void` = ไม่มีค่า return
- `main` = ชื่อ method (JVM จะหา method นี้เป็นจุดเริ่มต้น)
- `String[] args` = Parameter รับ command-line arguments

```java
        System.out.println("Hello, World!");
```
- `System` = Built-in class
- `out` = Static field ของ PrintStream
- `println` = Method สำหรับพิมพ์และขึ้นบรรทัดใหม่

---

## การ Compile และ Run

### วิธีที่ 1: ใช้ Command Line

```bash
# สร้างไฟล์ HelloWorld.java
# Compile
javac HelloWorld.java

# ผลลัพธ์: จะได้ไฟล์ HelloWorld.class

# Run
java HelloWorld

# ผลลัพธ์:
# Hello, World!
# ยินดีต้อนรับสู่โลกของ Java!
```

### วิธีที่ 2: ใช้ IntelliJ IDEA
1. คลิกขวาที่ไฟล์ > Run 'HelloWorld.main()'
2. หรือกด Shift+F10

### วิธีที่ 3: Java 11+ Single File Execution
```bash
# ไม่ต้อง compile แยก
java HelloWorld.java
```

---

## โครงสร้างโปรเจกต์ Java

### โครงสร้างพื้นฐาน
```
my-project/
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── example/
│                   └── HelloWorld.java
└── out/  (compiled files)
```

### โครงสร้าง Maven Project (จะเรียนใน Part 018)
```
my-maven-project/
├── src/
│   ├── main/
│   │   ├── java/           # Source code
│   │   └── resources/      # Configuration files
│   └── test/
│       ├── java/           # Test code
│       └── resources/      # Test configs
├── target/                 # Compiled output
└── pom.xml                 # Maven configuration
```

---

## ตัวอย่างโปรแกรมเพิ่มเติม

### ตัวอย่าง 1: รับ Input จากผู้ใช้

```java
import java.util.Scanner;

public class UserInput {
    public static void main(String[] args) {
        // สร้าง Scanner object สำหรับรับ input
        Scanner scanner = new Scanner(System.in);
        
        // แสดงข้อความ prompt
        System.out.print("กรุณาใส่ชื่อของคุณ: ");
        
        // รับ input
        String name = scanner.nextLine();
        
        // แสดงผล
        System.out.println("สวัสดี, " + name + "!");
        System.out.println("ยินดีต้อนรับสู่ Java Programming!");
        
        // ปิด Scanner
        scanner.close();
    }
}
```

**ผลลัพธ์:**
```
กรุณาใส่ชื่อของคุณ: สมชาย
สวัสดี, สมชาย!
ยินดีต้อนรับสู่ Java Programming!
```

### ตัวอย่าง 2: การคำนวณพื้นฐาน

```java
public class BasicCalculation {
    public static void main(String[] args) {
        // ประกาศตัวแปร
        int a = 10;
        int b = 3;
        
        // การคำนวณพื้นฐาน
        System.out.println("a = " + a + ", b = " + b);
        System.out.println("บวก: a + b = " + (a + b));
        System.out.println("ลบ: a - b = " + (a - b));
        System.out.println("คูณ: a * b = " + (a * b));
        System.out.println("หาร: a / b = " + (a / b));           // Integer division
        System.out.println("เศษ: a % b = " + (a % b));           // Modulo
        
        // Floating point division
        double result = (double) a / b;
        System.out.println("หารแบบทศนิยม: " + result);
        System.out.printf("หารแบบทศนิยม (2 ตำแหน่ง): %.2f%n", result);
    }
}
```

**ผลลัพธ์:**
```
a = 10, b = 3
บวก: a + b = 13
ลบ: a - b = 7
คูณ: a * b = 30
หาร: a / b = 3
เศษ: a % b = 1
หารแบบทศนิยม: 3.3333333333333335
หารแบบทศนิยม (2 ตำแหน่ง): 3.33
```

### ตัวอย่าง 3: โปรแกรม BMI Calculator

```java
import java.util.Scanner;

public class BMICalculator {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("=== โปรแกรมคำนวณ BMI ===");
        
        System.out.print("น้ำหนัก (กิโลกรัม): ");
        double weight = scanner.nextDouble();
        
        System.out.print("ส่วนสูง (เมตร): ");
        double height = scanner.nextDouble();
        
        // คำนวณ BMI
        double bmi = weight / (height * height);
        
        System.out.printf("%nBMI ของคุณคือ: %.2f%n", bmi);
        
        // แสดงผลการประเมิน
        String category;
        if (bmi < 18.5) {
            category = "น้ำหนักน้อยเกินไป (Underweight)";
        } else if (bmi < 25.0) {
            category = "น้ำหนักปกติ (Normal)";
        } else if (bmi < 30.0) {
            category = "น้ำหนักเกิน (Overweight)";
        } else {
            category = "อ้วน (Obese)";
        }
        
        System.out.println("ผลการประเมิน: " + category);
        
        scanner.close();
    }
}
```

**ผลลัพธ์:**
```
=== โปรแกรมคำนวณ BMI ===
น้ำหนัก (กิโลกรัม): 70
ส่วนสูง (เมตร): 1.75

BMI ของคุณคือ: 22.86
ผลการประเมิน: น้ำหนักปกติ (Normal)
```

---

## Java Versions และ Features

### Java 8 (2014) - Major Features
```java
// Lambda Expression
Runnable r = () -> System.out.println("Hello Lambda!");

// Stream API
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
int sum = numbers.stream()
    .filter(n -> n % 2 == 0)
    .mapToInt(Integer::intValue)
    .sum();

// Optional
Optional<String> optional = Optional.of("Hello");
optional.ifPresent(System.out::println);
```

### Java 11 (2018) - LTS Features
```java
// String methods
String text = "  Hello World  ";
text.strip();           // ลบ whitespace
text.isBlank();         // ตรวจสอบว่าว่าง
text.lines();           // แยกเป็น Stream ของบรรทัด

// Single file programs
// รัน java HelloWorld.java ได้เลย

// var keyword (Java 10)
var list = new ArrayList<String>();
```

### Java 17 (2021) - LTS Features
```java
// Sealed Classes
public sealed class Shape permits Circle, Rectangle, Triangle {
}

// Pattern Matching for instanceof
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Records
public record Point(int x, int y) {}

// Text Blocks
String html = """
    <html>
        <body>
            <p>Hello, World!</p>
        </body>
    </html>
    """;
```

### Java 21 (2023) - LTS Features
```java
// Virtual Threads (Project Loom)
Thread.ofVirtual().start(() -> {
    System.out.println("Virtual Thread!");
});

// Pattern Matching for switch
String result = switch (obj) {
    case Integer i -> "Integer: " + i;
    case String s -> "String: " + s;
    case null -> "null";
    default -> "Unknown";
};

// Sequenced Collections
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst(); // "a"
list.getLast();  // "c"
```

---

## การตั้งค่า VS Code สำหรับ Java (ทางเลือก)

หากต้องการใช้ VS Code แทน IntelliJ:

### ติดตั้ง Extensions
1. **Extension Pack for Java** by Microsoft
   - Language Support for Java
   - Debugger for Java
   - Test Runner for Java
   - Maven for Java
   - Project Manager for Java
   - IntelliCode

### การใช้งาน
```bash
# เปิด VS Code
code .

# สร้างไฟล์ Java ใหม่
# VS Code จะ suggest Java project structure อัตโนมัติ
```

---

## Git Setup สำหรับ Java Project

### ติดตั้ง Git
```bash
# Ubuntu
sudo apt install git

# macOS
brew install git

# Windows: ดาวน์โหลดจาก https://git-scm.com
```

### ตั้งค่าเบื้องต้น
```bash
git config --global user.name "ชื่อของคุณ"
git config --global user.email "email@example.com"
```

### สร้าง .gitignore สำหรับ Java
```gitignore
# Compiled class file
*.class

# Log file
*.log

# BlueJ files
*.ctxt

# Mobile Tools for Java
.mtj.tmp/

# Package Files
*.jar
*.war
*.nar
*.ear
*.zip
*.tar.gz
*.rar

# virtual machine crash logs
hs_err_pid*

# IntelliJ IDEA
.idea/
*.iml
*.iws
*.ipr
out/

# Eclipse
.classpath
.project
.settings/
bin/

# NetBeans
nbproject/private/
build/
nbbuild/
dist/
nbdist/
.nb-gradle/

# Maven
target/

# Gradle
.gradle/
build/

# Spring Boot
application-local.properties
application-local.yml
```

---

## สรุป JVM Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Java Program                          │
└──────────────────────────┬───────────────────────────────┘
                           │ java HelloWorld
                           ▼
┌──────────────────────────────────────────────────────────┐
│                  JVM (Java Virtual Machine)               │
│                                                          │
│  ┌─────────────────┐    ┌──────────────────────────────┐ │
│  │   Class Loader  │    │         Memory                │ │
│  │                 │    │  ┌──────────────────────────┐ │ │
│  │ 1. Bootstrap    │    │  │    Method Area (Meta)    │ │ │
│  │ 2. Extension    │    │  ├──────────────────────────┤ │ │
│  │ 3. Application  │    │  │         Heap             │ │ │
│  └─────────────────┘    │  ├──────────────────────────┤ │ │
│                         │  │    Stack (per thread)    │ │ │
│  ┌─────────────────┐    │  ├──────────────────────────┤ │ │
│  │ Execution Engine│    │  │    PC Register           │ │ │
│  │                 │    │  └──────────────────────────┘ │ │
│  │ - Interpreter   │    └──────────────────────────────┘ │
│  │ - JIT Compiler  │                                     │
│  │ - GC            │                                     │
│  └─────────────────┘                                     │
└──────────────────────────────────────────────────────────┘
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Hello World แบบหลายภาษา
สร้างโปรแกรมที่แสดงข้อความ "Hello, World!" ทั้งภาษาไทยและอังกฤษ

```java
// เฉลย
public class HelloMultiLanguage {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
        System.out.println("สวัสดีโลก!");
        System.out.println("こんにちは世界!");
        System.out.println("你好世界!");
        System.out.println("Hola Mundo!");
    }
}
```

### แบบฝึกหัดที่ 2: โปรแกรมแนะนำตัว
สร้างโปรแกรมที่แสดงข้อมูลส่วนตัว เช่น ชื่อ อายุ อาชีพ

```java
// เฉลย
public class AboutMe {
    public static void main(String[] args) {
        String name = "สมชาย ใจดี";
        int age = 25;
        String job = "Software Developer";
        String hobby = "เขียนโปรแกรม";
        
        System.out.println("=".repeat(30));
        System.out.println("ประวัติส่วนตัว");
        System.out.println("=".repeat(30));
        System.out.println("ชื่อ: " + name);
        System.out.println("อายุ: " + age + " ปี");
        System.out.println("อาชีพ: " + job);
        System.out.println("งานอดิเรก: " + hobby);
        System.out.println("=".repeat(30));
    }
}
```

### แบบฝึกหัดที่ 3: เครื่องแปลงอุณหภูมิ
สร้างโปรแกรมแปลงอุณหภูมิจาก Celsius เป็น Fahrenheit และ Kelvin

```java
// เฉลย
import java.util.Scanner;

public class TemperatureConverter {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("=== เครื่องแปลงอุณหภูมิ ===");
        System.out.print("ใส่อุณหภูมิ (Celsius): ");
        double celsius = scanner.nextDouble();
        
        // แปลงสูตร
        double fahrenheit = (celsius * 9 / 5) + 32;
        double kelvin = celsius + 273.15;
        
        System.out.println("\nผลการแปลง:");
        System.out.printf("%.2f°C = %.2f°F%n", celsius, fahrenheit);
        System.out.printf("%.2f°C = %.2fK%n", celsius, kelvin);
        
        scanner.close();
    }
}
```

**ผลลัพธ์:**
```
=== เครื่องแปลงอุณหภูมิ ===
ใส่อุณหภูมิ (Celsius): 100

ผลการแปลง:
100.00°C = 212.00°F
100.00°C = 373.15K
```

### แบบฝึกหัดที่ 4: โปรแกรมเช็คจำนวนคู่/คี่
```java
// เฉลย
import java.util.Scanner;

public class EvenOddChecker {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("ใส่จำนวนเต็ม: ");
        int number = scanner.nextInt();
        
        if (number % 2 == 0) {
            System.out.println(number + " เป็นจำนวนคู่ (Even)");
        } else {
            System.out.println(number + " เป็นจำนวนคี่ (Odd)");
        }
        
        // ตรวจสอบเพิ่มเติม
        if (number > 0) {
            System.out.println("และเป็นจำนวนบวก (Positive)");
        } else if (number < 0) {
            System.out.println("และเป็นจำนวนลบ (Negative)");
        } else {
            System.out.println("และเป็นศูนย์ (Zero)");
        }
        
        scanner.close();
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Java คืออะไรและทำไมต้องเรียน  
✅ การติดตั้ง JDK บนทุก OS  
✅ การติดตั้งและตั้งค่า IntelliJ IDEA  
✅ สร้างโปรแกรม Hello World แรก  
✅ เข้าใจโครงสร้างโปรแกรม Java  
✅ Java Versions และ Features สำคัญ  

---

## ขั้นตอนต่อไป

**Part 002:** Variables, Data Types & Operators  
เราจะเรียนรู้เกี่ยวกับ:
- Primitive Data Types (int, double, boolean, char)
- Reference Types (String, Arrays, Objects)
- Operators ทุกประเภท
- Type Casting

---

*Part 001 | Java & Spring Boot Course | สร้างโดย Claude Code*
