# Part 002: Variables, Data Types & Operators
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Variables คืออะไร?](#variables-คืออะไร)
2. [Primitive Data Types](#primitive-data-types)
3. [Reference Types](#reference-types)
4. [String และ String Methods](#string-และ-string-methods)
5. [Type Casting](#type-casting)
6. [Operators](#operators)
7. [String Formatting](#string-formatting)
8. [Constants (final)](#constants-final)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Variables คืออะไร?

Variable คือ "ชื่อ" ที่ใช้อ้างอิงข้อมูลในหน่วยความจำ

### การประกาศ Variable

```java
// รูปแบบ: DataType variableName = value;
int age = 25;
String name = "สมชาย";
double salary = 50000.50;
boolean isStudent = false;
```

### กฎการตั้งชื่อ Variable

```java
// ✅ ถูกต้อง
int myAge = 25;
String firstName = "Java";
double totalPrice = 100.50;
boolean isActive = true;
int _count = 0;
int $price = 100;

// ❌ ผิด
int 1age = 25;        // ห้ามขึ้นต้นด้วยตัวเลข
int my-name = 10;     // ห้ามใช้ -
int class = 5;        // ห้ามใช้ keyword
```

### Naming Conventions (มาตรฐาน)

```java
// Variables & Methods: camelCase
int studentAge = 20;
String firstName = "John";
void calculateTax() {}

// Constants: UPPER_SNAKE_CASE
final double PI = 3.14159;
final int MAX_SIZE = 100;

// Classes: PascalCase
class StudentRecord {}
class BankAccount {}

// Packages: lowercase
package com.example.myapp;
```

### var Keyword (Java 10+)

```java
// Type Inference - compiler จะ infer type เอง
var name = "Hello";           // String
var age = 25;                 // int
var pi = 3.14;                // double
var list = new ArrayList<>(); // ArrayList

// ใช้ได้เฉพาะ local variable เท่านั้น
// ห้ามใช้เป็น parameter หรือ field
```

---

## Primitive Data Types

Java มี Primitive Types 8 ชนิด:

### 1. Integer Types (จำนวนเต็ม)

```java
public class IntegerTypes {
    public static void main(String[] args) {
        // byte: 8-bit, -128 ถึง 127
        byte byteVal = 127;
        System.out.println("byte: " + byteVal);
        System.out.println("byte max: " + Byte.MAX_VALUE);
        System.out.println("byte min: " + Byte.MIN_VALUE);
        
        // short: 16-bit, -32,768 ถึง 32,767
        short shortVal = 32767;
        System.out.println("\nshort: " + shortVal);
        System.out.println("short max: " + Short.MAX_VALUE);
        
        // int: 32-bit, -2,147,483,648 ถึง 2,147,483,647
        int intVal = 2147483647;
        System.out.println("\nint: " + intVal);
        System.out.println("int max: " + Integer.MAX_VALUE);
        
        // long: 64-bit, ใส่ L ต่อท้าย
        long longVal = 9_223_372_036_854_775_807L;
        System.out.println("\nlong: " + longVal);
        System.out.println("long max: " + Long.MAX_VALUE);
        
        // Underscore ในตัวเลข (Java 7+) ช่วยให้อ่านง่าย
        int million = 1_000_000;
        long creditCard = 1234_5678_9012_3456L;
        System.out.println("\n1 million: " + million);
    }
}
```

### 2. Floating Point Types (จำนวนทศนิยม)

```java
public class FloatingPointTypes {
    public static void main(String[] args) {
        // float: 32-bit, ความแม่นยำ 6-7 digits, ใส่ f ต่อท้าย
        float floatVal = 3.14f;
        System.out.println("float: " + floatVal);
        System.out.println("float max: " + Float.MAX_VALUE);
        
        // double: 64-bit, ความแม่นยำ 15-16 digits (แนะนำใช้)
        double doubleVal = 3.141592653589793;
        System.out.println("\ndouble: " + doubleVal);
        System.out.println("double max: " + Double.MAX_VALUE);
        
        // Scientific notation
        double scientific = 1.5e10;  // 1.5 × 10^10
        System.out.println("\nScientific: " + scientific);
        
        // ปัญหา floating point precision
        double a = 0.1 + 0.2;
        System.out.println("\n0.1 + 0.2 = " + a);  // ได้ 0.30000000000000004
        
        // แก้โดยใช้ BigDecimal สำหรับการเงิน
        java.math.BigDecimal bd1 = new java.math.BigDecimal("0.1");
        java.math.BigDecimal bd2 = new java.math.BigDecimal("0.2");
        System.out.println("BigDecimal: " + bd1.add(bd2));  // ได้ 0.3
    }
}
```

### 3. Character Type

```java
public class CharType {
    public static void main(String[] args) {
        // char: 16-bit Unicode character
        char grade = 'A';
        char thaiChar = 'ก';
        char unicodeChar = 'A';  // 'A' ใน Unicode
        
        System.out.println("char: " + grade);
        System.out.println("Thai char: " + thaiChar);
        System.out.println("Unicode char: " + unicodeChar);
        
        // char เป็นตัวเลขได้
        char c = 'A';
        System.out.println("'A' as int: " + (int) c);  // 65
        System.out.println("'A' + 1 = " + (char)(c + 1));  // 'B'
        
        // Escape sequences
        System.out.println("Tab:\tHello");
        System.out.println("Newline:\nWorld");
        System.out.println("Quote: \"Java\"");
        System.out.println("Backslash: \\");
        
        // char range
        System.out.println("A-Z:");
        for (char ch = 'A'; ch <= 'Z'; ch++) {
            System.out.print(ch + " ");
        }
    }
}
```

### 4. Boolean Type

```java
public class BooleanType {
    public static void main(String[] args) {
        // boolean: true หรือ false เท่านั้น
        boolean isJavaFun = true;
        boolean isHard = false;
        
        System.out.println("Is Java fun? " + isJavaFun);
        System.out.println("Is Java hard? " + isHard);
        
        // Boolean expressions
        int age = 20;
        boolean isAdult = age >= 18;
        boolean canVote = isAdult && age < 100;
        
        System.out.println("Is adult: " + isAdult);
        System.out.println("Can vote: " + canVote);
        
        // ห้ามทำแบบนี้ใน Java (ต่างจาก C/C++)
        // if (1) {} // Error ใน Java
        // if ("hello") {} // Error ใน Java
        
        // ต้องใช้ boolean expression เท่านั้น
        if (isJavaFun) {
            System.out.println("Java is fun!");
        }
    }
}
```

### ตาราง Primitive Types สรุป

```
┌──────────┬────────┬────────────────────────────────────────┬─────────────┐
│ Type     │ Size   │ Range                                  │ Default     │
├──────────┼────────┼────────────────────────────────────────┼─────────────┤
│ byte     │ 8-bit  │ -128 to 127                            │ 0           │
│ short    │ 16-bit │ -32,768 to 32,767                      │ 0           │
│ int      │ 32-bit │ -2,147,483,648 to 2,147,483,647        │ 0           │
│ long     │ 64-bit │ -9.2×10¹⁸ to 9.2×10¹⁸                │ 0L          │
│ float    │ 32-bit │ ~±3.4×10³⁸ (6-7 digits)               │ 0.0f        │
│ double   │ 64-bit │ ~±1.7×10³⁰⁸ (15-16 digits)           │ 0.0d        │
│ char     │ 16-bit │ '\u0000' to '￿' (0-65535)        │ '\u0000'    │
│ boolean  │ 1-bit  │ true or false                          │ false       │
└──────────┴────────┴────────────────────────────────────────┴─────────────┘
```

---

## Reference Types

### Wrapper Classes

ทุก Primitive Type มี Wrapper Class คู่กัน:

```java
public class WrapperClasses {
    public static void main(String[] args) {
        // Primitive
        int primitiveInt = 42;
        
        // Wrapper
        Integer wrapperInt = 42;
        Integer fromString = Integer.parseInt("100");
        Integer fromValueOf = Integer.valueOf(42);
        
        // Auto-boxing: primitive -> wrapper อัตโนมัติ
        Integer autoBoxed = primitiveInt;  // int -> Integer
        
        // Auto-unboxing: wrapper -> primitive อัตโนมัติ  
        int unboxed = wrapperInt;  // Integer -> int
        
        // Utility methods
        System.out.println("Max int: " + Integer.MAX_VALUE);
        System.out.println("Parse: " + Integer.parseInt("255"));
        System.out.println("Binary: " + Integer.toBinaryString(255));
        System.out.println("Hex: " + Integer.toHexString(255));
        System.out.println("Compare: " + Integer.compare(10, 20));
        
        // Double utilities
        String doubleStr = "3.14";
        double d = Double.parseDouble(doubleStr);
        System.out.println("\nDouble parsed: " + d);
        System.out.println("Is NaN: " + Double.isNaN(d));
        System.out.println("Is Infinite: " + Double.isInfinite(1.0/0));
        
        // Null handling - wrapper สามารถเป็น null ได้
        Integer nullableInt = null;
        System.out.println("\nNullable: " + nullableInt);
        
        // NullPointerException!
        // int cantBeNull = nullableInt;  // NullPointerException
    }
}
```

---

## String และ String Methods

String เป็น Reference Type ที่ใช้บ่อยมากที่สุด

### String Basics

```java
public class StringBasics {
    public static void main(String[] args) {
        // การสร้าง String
        String s1 = "Hello, World!";           // String literal
        String s2 = new String("Hello");       // String object (ไม่แนะนำ)
        String s3 = String.valueOf(42);        // แปลงจากตัวเลข
        
        // String เป็น Immutable (เปลี่ยนแปลงไม่ได้)
        String original = "Hello";
        String modified = original.toUpperCase();
        System.out.println("Original: " + original);   // Hello (ไม่เปลี่ยน)
        System.out.println("Modified: " + modified);   // HELLO
        
        // String comparison - ต้องใช้ .equals() ไม่ใช่ ==
        String a = "hello";
        String b = "hello";
        String c = new String("hello");
        
        System.out.println("\na == b: " + (a == b));       // true (String Pool)
        System.out.println("a == c: " + (a == c));         // false (different object)
        System.out.println("a.equals(c): " + a.equals(c)); // true (same content)
        System.out.println("equalsIgnoreCase: " + a.equalsIgnoreCase("HELLO")); // true
    }
}
```

### String Methods ที่ใช้บ่อย

```java
public class StringMethods {
    public static void main(String[] args) {
        String text = "  Hello, Java World!  ";
        
        // ความยาว
        System.out.println("Length: " + text.length());
        
        // Trim
        System.out.println("Trim: '" + text.trim() + "'");
        System.out.println("Strip: '" + text.strip() + "'");  // Java 11+
        
        // Case
        System.out.println("Upper: " + text.toUpperCase());
        System.out.println("Lower: " + text.toLowerCase());
        
        // Substring
        String str = "Hello, World!";
        System.out.println("\nSubstring(7): " + str.substring(7));          // World!
        System.out.println("Substring(7,12): " + str.substring(7, 12));    // World
        
        // Search
        System.out.println("indexOf 'o': " + str.indexOf('o'));              // 4
        System.out.println("lastIndexOf 'o': " + str.lastIndexOf('o'));      // 8
        System.out.println("contains 'World': " + str.contains("World"));   // true
        System.out.println("startsWith 'Hello': " + str.startsWith("Hello")); // true
        System.out.println("endsWith '!': " + str.endsWith("!"));           // true
        
        // Replace
        String replaced = str.replace("World", "Java");
        System.out.println("\nReplace: " + replaced);
        String replaceAll = "aababab".replaceAll("ab", "X");
        System.out.println("ReplaceAll: " + replaceAll);
        
        // Split
        String csv = "apple,banana,cherry,date";
        String[] fruits = csv.split(",");
        System.out.println("\nSplit result:");
        for (String fruit : fruits) {
            System.out.println("  - " + fruit);
        }
        
        // Join (Java 8+)
        String joined = String.join(" | ", "A", "B", "C");
        System.out.println("\nJoined: " + joined);
        
        // char at position
        System.out.println("\nChar at 0: " + str.charAt(0));     // 'H'
        
        // Convert to char array
        char[] chars = str.toCharArray();
        System.out.println("First char: " + chars[0]);
        
        // isEmpty vs isBlank (Java 11+)
        System.out.println("\n\"\" isEmpty: " + "".isEmpty());         // true
        System.out.println("\"  \" isEmpty: " + "  ".isEmpty());       // false
        System.out.println("\"  \" isBlank: " + "  ".isBlank());       // true Java 11+
        
        // repeat (Java 11+)
        String repeated = "Ha".repeat(3);
        System.out.println("\nRepeat: " + repeated);  // HaHaHa
    }
}
```

### StringBuilder และ StringBuffer

```java
public class StringBuilderExample {
    public static void main(String[] args) {
        // StringBuilder: mutable, ไม่ thread-safe แต่เร็วกว่า
        StringBuilder sb = new StringBuilder();
        sb.append("Hello");
        sb.append(", ");
        sb.append("World");
        sb.append("!");
        System.out.println(sb.toString());
        
        // Method chaining
        StringBuilder sb2 = new StringBuilder()
            .append("Java ")
            .append("is ")
            .append("awesome!")
            .append(" Version: ")
            .append(21);
        System.out.println(sb2);
        
        // Insert, Delete, Replace
        StringBuilder sb3 = new StringBuilder("Hello World");
        sb3.insert(5, ",");         // Hello, World
        sb3.delete(0, 6);           // World
        sb3.replace(0, 5, "Java");  // Java
        System.out.println("After operations: " + sb3);
        
        // Reverse
        StringBuilder sb4 = new StringBuilder("Hello");
        System.out.println("Reversed: " + sb4.reverse());
        
        // Performance test
        long start, end;
        
        // String concatenation (slow)
        start = System.currentTimeMillis();
        String str = "";
        for (int i = 0; i < 10000; i++) {
            str += "a";  // สร้าง String object ใหม่ทุกครั้ง!
        }
        end = System.currentTimeMillis();
        System.out.println("\nString concat time: " + (end - start) + "ms");
        
        // StringBuilder (fast)
        start = System.currentTimeMillis();
        StringBuilder sbPerf = new StringBuilder();
        for (int i = 0; i < 10000; i++) {
            sbPerf.append("a");
        }
        end = System.currentTimeMillis();
        System.out.println("StringBuilder time: " + (end - start) + "ms");
    }
}
```

### Text Blocks (Java 15+)

```java
public class TextBlockExample {
    public static void main(String[] args) {
        // แบบเก่า
        String oldJson = "{\n" +
            "    \"name\": \"John\",\n" +
            "    \"age\": 30\n" +
            "}";
        
        // Text Block
        String json = """
                {
                    "name": "John",
                    "age": 30
                }
                """;
        
        String html = """
                <html>
                    <body>
                        <h1>Hello, World!</h1>
                    </body>
                </html>
                """;
        
        String sql = """
                SELECT u.id, u.name, u.email
                FROM users u
                INNER JOIN orders o ON u.id = o.user_id
                WHERE u.active = true
                ORDER BY u.name
                """;
        
        System.out.println(json);
        System.out.println(html);
    }
}
```

---

## Type Casting

### Implicit Casting (Widening)

```java
public class ImplicitCasting {
    public static void main(String[] args) {
        // Widening: ขนาดเล็ก -> ขนาดใหญ่ (อัตโนมัติ)
        byte byteVal = 100;
        short shortVal = byteVal;   // byte -> short
        int intVal = shortVal;      // short -> int
        long longVal = intVal;      // int -> long
        float floatVal = longVal;   // long -> float
        double doubleVal = floatVal; // float -> double
        
        System.out.println("byte: " + byteVal);
        System.out.println("short: " + shortVal);
        System.out.println("int: " + intVal);
        System.out.println("long: " + longVal);
        System.out.println("float: " + floatVal);
        System.out.println("double: " + doubleVal);
        
        // Widening: byte -> short -> int -> long -> float -> double
        //                                              ↑ char สามารถ widen ไป int ได้ด้วย
    }
}
```

### Explicit Casting (Narrowing)

```java
public class ExplicitCasting {
    public static void main(String[] args) {
        // Narrowing: ขนาดใหญ่ -> ขนาดเล็ก (ต้อง cast เอง)
        double doubleVal = 3.99;
        int intVal = (int) doubleVal;  // ตัดทศนิยมทิ้ง!
        System.out.println("double to int: " + intVal);  // 3 (ไม่ใช่ 4)
        
        int bigInt = 300;
        byte byteVal = (byte) bigInt;  // Overflow!
        System.out.println("int to byte: " + byteVal);  // 44 (300 - 256)
        
        // int to char
        int code = 65;
        char ch = (char) code;
        System.out.println("int to char: " + ch);  // 'A'
        
        // char to int
        char grade = 'B';
        int asciiCode = (int) grade;
        System.out.println("char to int: " + asciiCode);  // 66
        
        // String to number
        String numStr = "42";
        int parsed = Integer.parseInt(numStr);
        double parsedDouble = Double.parseDouble("3.14");
        System.out.println("\nParsed int: " + parsed);
        System.out.println("Parsed double: " + parsedDouble);
        
        // Number to String
        int num = 100;
        String str1 = String.valueOf(num);
        String str2 = Integer.toString(num);
        String str3 = "" + num;  // implicit conversion
        System.out.println("\nNumber to String: " + str1 + ", " + str2 + ", " + str3);
    }
}
```

---

## Operators

### Arithmetic Operators

```java
public class ArithmeticOperators {
    public static void main(String[] args) {
        int a = 17, b = 5;
        
        System.out.println("a = " + a + ", b = " + b);
        System.out.println("a + b = " + (a + b));   // 22
        System.out.println("a - b = " + (a - b));   // 12
        System.out.println("a * b = " + (a * b));   // 85
        System.out.println("a / b = " + (a / b));   // 3 (integer division)
        System.out.println("a % b = " + (a % b));   // 2 (remainder)
        
        // Increment / Decrement
        int x = 10;
        System.out.println("\nPost-increment: " + x++);  // 10 (แล้วเพิ่ม)
        System.out.println("After: " + x);                // 11
        
        int y = 10;
        System.out.println("Pre-increment: " + ++y);  // 11 (เพิ่มก่อนแล้วใช้)
        System.out.println("After: " + y);             // 11
        
        // Compound assignment
        int n = 10;
        n += 5;   // n = n + 5 = 15
        n -= 3;   // n = n - 3 = 12
        n *= 2;   // n = n * 2 = 24
        n /= 4;   // n = n / 4 = 6
        n %= 4;   // n = n % 4 = 2
        System.out.println("\nCompound result: " + n);  // 2
        
        // Math class
        System.out.println("\nMath operations:");
        System.out.println("pow(2, 10) = " + Math.pow(2, 10));     // 1024
        System.out.println("sqrt(144) = " + Math.sqrt(144));       // 12
        System.out.println("abs(-5) = " + Math.abs(-5));           // 5
        System.out.println("max(10, 20) = " + Math.max(10, 20));   // 20
        System.out.println("min(10, 20) = " + Math.min(10, 20));   // 10
        System.out.println("ceil(3.2) = " + Math.ceil(3.2));       // 4
        System.out.println("floor(3.9) = " + Math.floor(3.9));     // 3
        System.out.println("round(3.5) = " + Math.round(3.5));     // 4
        System.out.println("random = " + Math.random());           // 0.0 to 1.0
    }
}
```

### Comparison Operators

```java
public class ComparisonOperators {
    public static void main(String[] args) {
        int a = 10, b = 20;
        
        System.out.println("a == b: " + (a == b));   // false
        System.out.println("a != b: " + (a != b));   // true
        System.out.println("a > b: " + (a > b));     // false
        System.out.println("a < b: " + (a < b));     // true
        System.out.println("a >= b: " + (a >= b));   // false
        System.out.println("a <= b: " + (a <= b));   // true
        
        // String comparison (ต้องใช้ .equals())
        String s1 = "Hello";
        String s2 = "Hello";
        String s3 = new String("Hello");
        
        System.out.println("\nString == (reference): " + (s1 == s2));       // true (pool)
        System.out.println("String == (new): " + (s1 == s3));               // false
        System.out.println("String equals: " + s1.equals(s3));              // true
        System.out.println("String compareTo: " + s1.compareTo("hello"));   // negative
    }
}
```

### Logical Operators

```java
public class LogicalOperators {
    public static void main(String[] args) {
        boolean a = true, b = false;
        
        // AND: ทั้งคู่ต้องเป็น true
        System.out.println("a && b: " + (a && b));   // false
        System.out.println("a && a: " + (a && a));   // true
        
        // OR: อย่างน้อยหนึ่งต้องเป็น true
        System.out.println("a || b: " + (a || b));   // true
        System.out.println("b || b: " + (b || b));   // false
        
        // NOT: กลับค่า
        System.out.println("!a: " + (!a));            // false
        System.out.println("!b: " + (!b));            // true
        
        // Short-circuit evaluation
        int x = 10;
        // ถ้า false && ... = false (ไม่ evaluate ฝั่งขวา)
        boolean result1 = (x > 20) && (++x > 0);
        System.out.println("\nx after &&: " + x);  // 10 (ไม่เพิ่ม)
        
        // ถ้า true || ... = true (ไม่ evaluate ฝั่งขวา)
        boolean result2 = (x > 0) || (++x > 0);
        System.out.println("x after ||: " + x);   // 10 (ไม่เพิ่ม)
        
        // ตัวอย่างใช้งานจริง
        String name = null;
        // Safe null check
        boolean isValid = name != null && !name.isEmpty();
        System.out.println("\nnull check: " + isValid);  // false (ปลอดภัย)
        
        // ถ้าทำแบบนี้จะ NullPointerException
        // boolean bad = !name.isEmpty() && name != null;
    }
}
```

### Bitwise Operators

```java
public class BitwiseOperators {
    public static void main(String[] args) {
        int a = 0b1010;  // 10 in decimal
        int b = 0b1100;  // 12 in decimal
        
        System.out.println("a = " + Integer.toBinaryString(a) + " (" + a + ")");
        System.out.println("b = " + Integer.toBinaryString(b) + " (" + b + ")");
        
        // AND: ทั้งคู่เป็น 1 ถึงได้ 1
        System.out.println("a & b = " + Integer.toBinaryString(a & b) + " (" + (a & b) + ")");  // 1000 = 8
        
        // OR: อย่างน้อยหนึ่งเป็น 1
        System.out.println("a | b = " + Integer.toBinaryString(a | b) + " (" + (a | b) + ")");  // 1110 = 14
        
        // XOR: ต่างกันถึงได้ 1
        System.out.println("a ^ b = " + Integer.toBinaryString(a ^ b) + " (" + (a ^ b) + ")");  // 0110 = 6
        
        // NOT: กลับทุก bit
        System.out.println("~a = " + (~a));  // -11
        
        // Shift left: คูณ 2^n
        System.out.println("a << 2 = " + (a << 2) + " (= " + a + " * 4)");  // 40
        
        // Shift right: หาร 2^n
        System.out.println("a >> 1 = " + (a >> 1) + " (= " + a + " / 2)");  // 5
        
        // ใช้งานจริง: ตรวจสอบ flag
        int permissions = 0b0111;  // read=1, write=2, execute=4
        int READ = 1, WRITE = 2, EXECUTE = 4;
        
        System.out.println("\nPermissions: " + Integer.toBinaryString(permissions));
        System.out.println("Can read: " + ((permissions & READ) != 0));    // true
        System.out.println("Can write: " + ((permissions & WRITE) != 0));  // true
        System.out.println("Can execute: " + ((permissions & EXECUTE) != 0)); // true
    }
}
```

### Ternary Operator

```java
public class TernaryOperator {
    public static void main(String[] args) {
        // รูปแบบ: condition ? valueIfTrue : valueIfFalse
        int age = 20;
        String status = age >= 18 ? "ผู้ใหญ่" : "เยาวชน";
        System.out.println("Status: " + status);
        
        // Nested ternary (ไม่แนะนำ อ่านยาก)
        int score = 75;
        String grade = score >= 90 ? "A" : 
                       score >= 80 ? "B" : 
                       score >= 70 ? "C" : 
                       score >= 60 ? "D" : "F";
        System.out.println("Grade: " + grade);
        
        // ควรใช้ if-else แทน nested ternary
        String gradeClean;
        if (score >= 90) {
            gradeClean = "A";
        } else if (score >= 80) {
            gradeClean = "B";
        } else if (score >= 70) {
            gradeClean = "C";
        } else if (score >= 60) {
            gradeClean = "D";
        } else {
            gradeClean = "F";
        }
        System.out.println("Grade (clean): " + gradeClean);
        
        // ใช้ใน String concatenation
        int items = 1;
        System.out.println("You have " + items + " " + (items == 1 ? "item" : "items"));
    }
}
```

### instanceof Operator

```java
public class InstanceofExample {
    public static void main(String[] args) {
        Object obj = "Hello, World!";
        
        // Old style
        if (obj instanceof String) {
            String s = (String) obj;
            System.out.println("Length: " + s.length());
        }
        
        // Pattern matching (Java 16+)
        if (obj instanceof String s) {
            System.out.println("Pattern match length: " + s.length());
        }
        
        // ตัวอย่างกับ hierarchy
        Number num = Integer.valueOf(42);
        System.out.println("Is Number: " + (num instanceof Number));   // true
        System.out.println("Is Integer: " + (num instanceof Integer)); // true
        System.out.println("Is Double: " + (num instanceof Double));   // false
        
        // Null check
        String nullStr = null;
        System.out.println("null instanceof String: " + (nullStr instanceof String));  // false
    }
}
```

---

## String Formatting

### printf และ String.format

```java
public class StringFormatting {
    public static void main(String[] args) {
        String name = "สมชาย";
        int age = 25;
        double salary = 55000.75;
        
        // printf - print with format
        System.out.printf("ชื่อ: %s, อายุ: %d ปี, เงินเดือน: %.2f บาท%n", 
                          name, age, salary);
        
        // String.format - return formatted string
        String formatted = String.format("%-15s | %3d | %10.2f", name, age, salary);
        System.out.println(formatted);
        
        // Format specifiers:
        // %s  = String
        // %d  = decimal integer
        // %f  = floating point
        // %e  = scientific notation
        // %c  = character
        // %b  = boolean
        // %n  = newline
        // %t  = date/time
        
        // Width and alignment
        System.out.printf("|%10s|%n", "right");  // right-aligned
        System.out.printf("|%-10s|%n", "left");  // left-aligned
        System.out.printf("|%010d|%n", 42);      // zero-padded
        
        // Numbers
        System.out.printf("%,d%n", 1000000);     // 1,000,000
        System.out.printf("%+d%n", 42);          // +42
        System.out.printf("%.5f%n", Math.PI);   // 3.14159
        System.out.printf("%e%n", 123456.789);  // 1.234568e+05
        
        // Table example
        System.out.println("\n" + "=".repeat(45));
        System.out.printf("%-15s | %-5s | %-12s%n", "ชื่อ", "อายุ", "เงินเดือน");
        System.out.println("-".repeat(45));
        
        Object[][] data = {
            {"สมชาย", 25, 50000.00},
            {"สมหญิง", 30, 75000.50},
            {"สมศรี", 28, 62500.25}
        };
        
        for (Object[] row : data) {
            System.out.printf("%-15s | %-5d | %12.2f%n", 
                              row[0], row[1], row[2]);
        }
        System.out.println("=".repeat(45));
        
        // Formatted Strings (Java 15+)
        String result = "Hello %s, you are %d years old".formatted(name, age);
        System.out.println("\n" + result);
    }
}
```

---

## Constants (final)

```java
public class Constants {
    // Class-level constants
    static final double PI = 3.14159265358979;
    static final int MAX_ATTEMPTS = 3;
    static final String APP_NAME = "MyJavaApp";
    
    public static void main(String[] args) {
        // Local constant
        final int MAX_SIZE = 100;
        final String GREETING = "Hello";
        
        // ไม่สามารถเปลี่ยนค่าได้
        // MAX_SIZE = 200;  // Compilation Error!
        
        System.out.println("PI: " + PI);
        System.out.println("Max attempts: " + MAX_ATTEMPTS);
        System.out.println("App: " + APP_NAME);
        
        // ใช้งานใน calculation
        double radius = 5.0;
        double area = PI * radius * radius;
        System.out.printf("Circle area: %.2f%n", area);
        
        // Enum (วิธีที่ดีกว่าสำหรับ related constants)
        // จะเรียนใน Part 007 เกี่ยวกับ OOP
    }
}
```

---

## โปรแกรมตัวอย่างครบถ้วน: Simple Bank Account

```java
import java.util.Scanner;

public class SimpleBankAccount {
    // Constants
    static final double MIN_BALANCE = 500.0;
    static final double MAX_WITHDRAWAL = 50000.0;
    
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        // Initial setup
        String accountHolder = "สมชาย ใจดี";
        String accountNumber = "1234567890";
        double balance = 10000.0;
        
        System.out.println("╔══════════════════════════════════╗");
        System.out.println("║     ระบบธนาคารจำลอง              ║");
        System.out.println("╚══════════════════════════════════╝");
        System.out.printf("เจ้าของบัญชี: %s%n", accountHolder);
        System.out.printf("เลขที่บัญชี: %s%n", accountNumber);
        System.out.printf("ยอดเงินปัจจุบัน: %,.2f บาท%n", balance);
        
        boolean running = true;
        while (running) {
            System.out.println("\n─────────────────────────────────");
            System.out.println("1. ฝากเงิน");
            System.out.println("2. ถอนเงิน");
            System.out.println("3. ดูยอดเงิน");
            System.out.println("4. ออกจากระบบ");
            System.out.print("เลือกรายการ: ");
            
            int choice = scanner.nextInt();
            
            switch (choice) {
                case 1 -> {
                    System.out.print("จำนวนเงินที่ต้องการฝาก: ");
                    double amount = scanner.nextDouble();
                    if (amount > 0) {
                        balance += amount;
                        System.out.printf("ฝากเงิน %.2f บาท สำเร็จ%n", amount);
                        System.out.printf("ยอดเงินคงเหลือ: %,.2f บาท%n", balance);
                    } else {
                        System.out.println("จำนวนเงินไม่ถูกต้อง");
                    }
                }
                case 2 -> {
                    System.out.print("จำนวนเงินที่ต้องการถอน: ");
                    double amount = scanner.nextDouble();
                    if (amount <= 0) {
                        System.out.println("จำนวนเงินไม่ถูกต้อง");
                    } else if (amount > MAX_WITHDRAWAL) {
                        System.out.printf("ถอนได้สูงสุด %.2f บาทต่อครั้ง%n", MAX_WITHDRAWAL);
                    } else if ((balance - amount) < MIN_BALANCE) {
                        System.out.printf("ยอดเงินไม่เพียงพอ (ต้องคงเหลือ %.2f บาท)%n", MIN_BALANCE);
                    } else {
                        balance -= amount;
                        System.out.printf("ถอนเงิน %.2f บาท สำเร็จ%n", amount);
                        System.out.printf("ยอดเงินคงเหลือ: %,.2f บาท%n", balance);
                    }
                }
                case 3 -> {
                    System.out.println("\n── ยอดเงินคงเหลือ ──");
                    System.out.printf("บัญชี: %s%n", accountNumber);
                    System.out.printf("ยอดเงิน: %,.2f บาท%n", balance);
                }
                case 4 -> {
                    System.out.println("ขอบคุณที่ใช้บริการ!");
                    running = false;
                }
                default -> System.out.println("กรุณาเลือก 1-4");
            }
        }
        
        scanner.close();
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ตรวจสอบ Data Types
สร้างโปรแกรมที่แสดงค่า max/min ของทุก primitive type

```java
// เฉลย
public class DataTypeLimits {
    public static void main(String[] args) {
        System.out.println("Data Type Limits:");
        System.out.printf("byte:   %d to %d%n", Byte.MIN_VALUE, Byte.MAX_VALUE);
        System.out.printf("short:  %d to %d%n", Short.MIN_VALUE, Short.MAX_VALUE);
        System.out.printf("int:    %d to %d%n", Integer.MIN_VALUE, Integer.MAX_VALUE);
        System.out.printf("long:   %d to %d%n", Long.MIN_VALUE, Long.MAX_VALUE);
        System.out.printf("float:  %e to %e%n", Float.MIN_VALUE, Float.MAX_VALUE);
        System.out.printf("double: %e to %e%n", Double.MIN_VALUE, Double.MAX_VALUE);
        System.out.printf("char:   %d to %d%n", (int)Character.MIN_VALUE, (int)Character.MAX_VALUE);
    }
}
```

### แบบฝึกหัดที่ 2: String Manipulation
เขียนโปรแกรมที่รับชื่อ-นามสกุล แล้วแสดงผลหลายรูปแบบ

```java
// เฉลย
import java.util.Scanner;

public class StringManipulation {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.print("ชื่อ: ");
        String firstName = sc.nextLine().trim();
        System.out.print("นามสกุล: ");
        String lastName = sc.nextLine().trim();
        
        String fullName = firstName + " " + lastName;
        System.out.println("\nผลลัพธ์:");
        System.out.println("ชื่อเต็ม: " + fullName);
        System.out.println("ตัวพิมพ์ใหญ่: " + fullName.toUpperCase());
        System.out.println("ตัวพิมพ์เล็ก: " + fullName.toLowerCase());
        System.out.println("ความยาว: " + fullName.length() + " ตัวอักษร");
        System.out.println("กลับหน้าหลัง: " + new StringBuilder(fullName).reverse());
        System.out.println("ย่อ: " + firstName.charAt(0) + "." + lastName.charAt(0) + ".");
        sc.close();
    }
}
```

### แบบฝึกหัดที่ 3: Calculator ครบครัน
สร้าง Calculator ที่รองรับ +, -, *, /, % และ power

```java
// เฉลย
import java.util.Scanner;

public class FullCalculator {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.println("=== เครื่องคิดเลข ===");
        System.out.print("ตัวเลขที่ 1: ");
        double a = sc.nextDouble();
        System.out.print("ตัวดำเนินการ (+,-,*,/,%,^): ");
        String op = sc.next();
        System.out.print("ตัวเลขที่ 2: ");
        double b = sc.nextDouble();
        
        double result = switch (op) {
            case "+" -> a + b;
            case "-" -> a - b;
            case "*" -> a * b;
            case "/" -> b != 0 ? a / b : Double.NaN;
            case "%" -> b != 0 ? a % b : Double.NaN;
            case "^" -> Math.pow(a, b);
            default -> {
                System.out.println("ตัวดำเนินการไม่ถูกต้อง");
                yield 0;
            }
        };
        
        if (Double.isNaN(result)) {
            System.out.println("ไม่สามารถหารด้วยศูนย์ได้");
        } else {
            System.out.printf("%.2f %s %.2f = %.6f%n", a, op, b, result);
        }
        
        sc.close();
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Variables และการตั้งชื่อตามมาตรฐาน  
✅ Primitive Data Types ทั้ง 8 ชนิด  
✅ Wrapper Classes และ Auto-boxing  
✅ String และ Methods ที่ใช้บ่อย  
✅ StringBuilder สำหรับ String ที่มีการเปลี่ยนแปลง  
✅ Type Casting ทั้ง Implicit และ Explicit  
✅ Operators ทุกประเภท  
✅ String Formatting  
✅ Constants ด้วย final  

---

## ขั้นตอนต่อไป

**Part 003:** Control Flow: if/else, switch  
เราจะเรียนรู้:
- if, if-else, if-else-if
- switch statement
- switch expression (Java 14+)
- Pattern matching ใน switch (Java 21)

---

*Part 002 | Java & Spring Boot Course | สร้างโดย Claude Code*
