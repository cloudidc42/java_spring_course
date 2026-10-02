# Part 005: Methods & Functions
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Methods คืออะไร?](#methods-คืออะไร)
2. [การสร้าง Method](#การสร้าง-method)
3. [Parameters และ Arguments](#parameters-และ-arguments)
4. [Return Types](#return-types)
5. [Method Overloading](#method-overloading)
6. [Varargs](#varargs)
7. [Recursion](#recursion)
8. [Scope ของ Variables](#scope-ของ-variables)
9. [Static vs Instance Methods](#static-vs-instance-methods)
10. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)

---

## Methods คืออะไร?

Method คือ block of code ที่ทำงานเฉพาะอย่าง สามารถเรียกใช้ซ้ำได้

### ทำไมต้องใช้ Methods?
```
โค้ดที่ไม่มี Method:
├── ซ้ำซ้อน (DRY violation)
├── แก้ไขยาก
├── อ่านยาก
└── ทดสอบยาก

โค้ดที่มี Method:
├── DRY (Don't Repeat Yourself)
├── แก้ไขที่เดียว
├── อ่านง่าย
└── ทดสอบได้ทีละ unit
```

---

## การสร้าง Method

### รูปแบบพื้นฐาน

```java
// accessModifier returnType methodName(parameters) { body }
public class MethodBasics {
    
    // Method ที่ไม่มี parameter ไม่ return ค่า
    static void greet() {
        System.out.println("สวัสดี!");
    }
    
    // Method ที่มี parameter
    static void greetUser(String name) {
        System.out.println("สวัสดี, " + name + "!");
    }
    
    // Method ที่ return ค่า
    static int add(int a, int b) {
        return a + b;
    }
    
    // Method ที่ซับซ้อนขึ้น
    static String getGrade(double score) {
        if (score >= 90) return "A";
        if (score >= 80) return "B";
        if (score >= 70) return "C";
        if (score >= 60) return "D";
        return "F";
    }
    
    public static void main(String[] args) {
        greet();
        greetUser("สมชาย");
        
        int sum = add(10, 20);
        System.out.println("10 + 20 = " + sum);
        
        double score = 85.5;
        String grade = getGrade(score);
        System.out.println("คะแนน " + score + " ได้เกรด " + grade);
    }
}
```

### Method Design Principles

```java
public class MethodDesign {
    
    // ❌ Method ที่ทำหลายอย่างเกินไป
    static void processEverything(int[] data) {
        // Sort
        // Filter
        // Calculate stats
        // Print report
        // Save to file
        // ... too much!
    }
    
    // ✅ แบ่ง Method ให้ทำงานเดียว (Single Responsibility)
    static int[] sortData(int[] data) {
        int[] sorted = data.clone();
        java.util.Arrays.sort(sorted);
        return sorted;
    }
    
    static double calculateAverage(int[] data) {
        int sum = 0;
        for (int val : data) sum += val;
        return (double) sum / data.length;
    }
    
    static void printReport(int[] data, double average) {
        System.out.println("Average: " + average);
        System.out.println("Count: " + data.length);
    }
    
    public static void main(String[] args) {
        int[] data = {5, 3, 8, 1, 9, 2, 7};
        
        int[] sorted = sortData(data);
        double avg = calculateAverage(data);
        printReport(sorted, avg);
    }
}
```

---

## Parameters และ Arguments

### Pass by Value

```java
public class PassByValue {
    
    // Java เป็น Pass by Value เสมอ!
    static void tryToChange(int x) {
        x = 100;  // แก้ local copy เท่านั้น
        System.out.println("ใน method: x = " + x);
    }
    
    static void tryToChangeArray(int[] arr) {
        arr[0] = 100;  // แก้ array จริง! (เพราะ pass reference by value)
    }
    
    static void tryToReassignArray(int[] arr) {
        arr = new int[]{100, 200};  // ไม่ impact ของเดิม
    }
    
    public static void main(String[] args) {
        int num = 5;
        tryToChange(num);
        System.out.println("หลัง method: num = " + num);  // ยังเป็น 5
        
        System.out.println();
        
        int[] arr = {1, 2, 3};
        System.out.println("ก่อน: " + arr[0]);
        tryToChangeArray(arr);
        System.out.println("หลัง tryToChangeArray: " + arr[0]);  // 100
        
        int[] arr2 = {1, 2, 3};
        tryToReassignArray(arr2);
        System.out.println("หลัง tryToReassignArray: " + arr2[0]);  // ยังเป็น 1
    }
}
```

### Multiple Parameters

```java
public class MultipleParameters {
    
    // หลาย parameters
    static double calculateBMI(double weight, double height) {
        return weight / (height * height);
    }
    
    // Parameter กับ default behavior (Java ไม่มี default params, ใช้ overloading)
    static String formatName(String firstName, String lastName) {
        return firstName + " " + lastName;
    }
    
    static String formatName(String firstName, String lastName, String title) {
        return title + " " + firstName + " " + lastName;
    }
    
    // Return multiple values (ใช้ array หรือ record)
    static double[] getMinMax(double[] data) {
        double min = data[0], max = data[0];
        for (double val : data) {
            if (val < min) min = val;
            if (val > max) max = val;
        }
        return new double[]{min, max};  // return array
    }
    
    // Java 16+: ใช้ record แทน
    record MinMaxResult(double min, double max) {}
    
    static MinMaxResult getMinMaxRecord(double[] data) {
        double min = data[0], max = data[0];
        for (double val : data) {
            if (val < min) min = val;
            if (val > max) max = val;
        }
        return new MinMaxResult(min, max);
    }
    
    public static void main(String[] args) {
        double bmi = calculateBMI(70.0, 1.75);
        System.out.printf("BMI: %.2f%n", bmi);
        
        System.out.println(formatName("John", "Doe"));
        System.out.println(formatName("Jane", "Doe", "Dr."));
        
        double[] data = {3.5, 1.2, 8.9, 4.1, 7.3};
        double[] result = getMinMax(data);
        System.out.printf("Min: %.1f, Max: %.1f%n", result[0], result[1]);
        
        MinMaxResult mmResult = getMinMaxRecord(data);
        System.out.printf("Min: %.1f, Max: %.1f%n", mmResult.min(), mmResult.max());
    }
}
```

---

## Return Types

```java
public class ReturnTypes {
    
    // void - ไม่ return ค่า
    static void printLine(int count) {
        System.out.println("-".repeat(count));
    }
    
    // ทุก primitive type
    static int getAge() { return 25; }
    static double getPi() { return Math.PI; }
    static boolean isPositive(int n) { return n > 0; }
    static char getGrade(int score) {
        return score >= 60 ? 'P' : 'F';
    }
    
    // Return String
    static String greet(String name) {
        return "สวัสดี, " + name;
    }
    
    // Return Array
    static int[] createRange(int start, int end) {
        int[] range = new int[end - start + 1];
        for (int i = 0; i < range.length; i++) {
            range[i] = start + i;
        }
        return range;
    }
    
    // Early return
    static String classifyAge(int age) {
        if (age < 0) return "Invalid";
        if (age < 13) return "เด็ก";
        if (age < 18) return "วัยรุ่น";
        if (age < 60) return "ผู้ใหญ่";
        return "ผู้สูงอายุ";
    }
    
    public static void main(String[] args) {
        printLine(30);
        System.out.println("Pi = " + getPi());
        System.out.println("Is 5 positive? " + isPositive(5));
        System.out.println(greet("สมชาย"));
        
        int[] range = createRange(1, 10);
        System.out.print("Range: ");
        for (int n : range) System.out.print(n + " ");
        System.out.println();
        
        System.out.println(classifyAge(5));
        System.out.println(classifyAge(16));
        System.out.println(classifyAge(35));
        System.out.println(classifyAge(70));
    }
}
```

---

## Method Overloading

```java
public class MethodOverloading {
    
    // ชื่อเดียวกัน แต่ parameter ต่างกัน
    
    // คำนวณพื้นที่
    static double area(double radius) {
        return Math.PI * radius * radius;
    }
    
    static double area(double width, double height) {
        return width * height;
    }
    
    static double area(double base, double height, boolean isTriangle) {
        return isTriangle ? 0.5 * base * height : base * height;
    }
    
    // print แบบต่างๆ
    static void print(int value) {
        System.out.println("int: " + value);
    }
    
    static void print(double value) {
        System.out.println("double: " + value);
    }
    
    static void print(String value) {
        System.out.println("String: " + value);
    }
    
    static void print(int a, int b) {
        System.out.println("int pair: " + a + ", " + b);
    }
    
    // ระวัง! ต้องระวัง ambiguity
    // static void print(long value) {...}  // อาจทำให้ compiler สับสน
    
    // คำนวณค่าสูงสุด
    static int max(int a, int b) {
        return a > b ? a : b;
    }
    
    static int max(int a, int b, int c) {
        return max(max(a, b), c);  // เรียก overloaded version
    }
    
    static double max(double a, double b) {
        return a > b ? a : b;
    }
    
    public static void main(String[] args) {
        System.out.println("วงกลม: " + area(5.0));
        System.out.println("สี่เหลี่ยม: " + area(4.0, 6.0));
        System.out.println("สามเหลี่ยม: " + area(3.0, 4.0, true));
        
        print(42);
        print(3.14);
        print("Hello");
        print(10, 20);
        
        System.out.println("\nMax(3,7): " + max(3, 7));
        System.out.println("Max(3,7,5): " + max(3, 7, 5));
        System.out.println("Max(3.5,2.8): " + max(3.5, 2.8));
    }
}
```

---

## Varargs

```java
public class Varargs {
    
    // Varargs: รับ parameter กี่ตัวก็ได้
    // ต้องเป็น parameter สุดท้าย
    static int sum(int... numbers) {
        int total = 0;
        for (int n : numbers) {
            total += n;
        }
        return total;
    }
    
    static double average(double... numbers) {
        if (numbers.length == 0) return 0;
        double total = 0;
        for (double n : numbers) total += n;
        return total / numbers.length;
    }
    
    // Varargs กับ parameter อื่น
    static String format(String label, Object... values) {
        StringBuilder sb = new StringBuilder(label + ": ");
        for (int i = 0; i < values.length; i++) {
            if (i > 0) sb.append(", ");
            sb.append(values[i]);
        }
        return sb.toString();
    }
    
    // ส่ง array ได้ด้วย
    static void printAll(String... items) {
        for (String item : items) {
            System.out.println("  - " + item);
        }
    }
    
    public static void main(String[] args) {
        System.out.println(sum());           // 0
        System.out.println(sum(1));          // 1
        System.out.println(sum(1, 2, 3));    // 6
        System.out.println(sum(1, 2, 3, 4, 5));  // 15
        
        System.out.println(average(1, 2, 3, 4, 5));  // 3.0
        
        System.out.println(format("ผลไม้", "apple", "banana", "cherry"));
        System.out.println(format("ตัวเลข", 1, 2, 3, 4));
        
        printAll("Java", "Spring Boot", "Docker", "Kubernetes");
        
        // ส่ง array
        int[] arr = {10, 20, 30};
        // sum(arr);  // Error! ต้องแปลง
        // ต้องใช้วิธีอื่น หรือ accept int[]
        
        String[] names = {"Alice", "Bob", "Charlie"};
        printAll(names);  // String varargs รับ String array ได้
    }
}
```

---

## Recursion

```java
public class Recursion {
    
    // Factorial: n! = n × (n-1)!
    static long factorial(int n) {
        // Base case
        if (n <= 1) return 1;
        // Recursive case
        return n * factorial(n - 1);
    }
    
    // Fibonacci
    static long fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    // Fibonacci แบบ Memoization (เร็วกว่ามาก)
    static long[] memo = new long[100];
    static long fibMemo(int n) {
        if (n <= 1) return n;
        if (memo[n] != 0) return memo[n];
        memo[n] = fibMemo(n - 1) + fibMemo(n - 2);
        return memo[n];
    }
    
    // Power: x^n
    static double power(double base, int exp) {
        if (exp == 0) return 1;
        if (exp < 0) return 1 / power(base, -exp);
        return base * power(base, exp - 1);
    }
    
    // กลับ String
    static String reverse(String s) {
        if (s.isEmpty()) return s;
        return reverse(s.substring(1)) + s.charAt(0);
    }
    
    // Binary Search แบบ Recursive
    static int binarySearch(int[] arr, int target, int left, int right) {
        if (left > right) return -1;
        
        int mid = (left + right) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) return binarySearch(arr, target, mid + 1, right);
        return binarySearch(arr, target, left, mid - 1);
    }
    
    // Hanoi Tower
    static void hanoi(int n, char from, char to, char aux) {
        if (n == 1) {
            System.out.printf("ย้าย disk 1 จาก %c ไป %c%n", from, to);
            return;
        }
        hanoi(n - 1, from, aux, to);
        System.out.printf("ย้าย disk %d จาก %c ไป %c%n", n, from, to);
        hanoi(n - 1, aux, to, from);
    }
    
    public static void main(String[] args) {
        // Factorial
        for (int i = 0; i <= 10; i++) {
            System.out.printf("%2d! = %d%n", i, factorial(i));
        }
        
        // Fibonacci (slow for large n)
        System.out.println("\nFibonacci (recursive):");
        for (int i = 0; i <= 15; i++) {
            System.out.print(fibonacci(i) + " ");
        }
        System.out.println();
        
        // Fibonacci (fast with memo)
        System.out.println("\nFibonacci (memoized):");
        System.out.println("fib(50) = " + fibMemo(50));
        
        // Power
        System.out.println("\nPower:");
        System.out.println("2^10 = " + (long)power(2, 10));
        System.out.println("3^5 = " + (long)power(3, 5));
        
        // Reverse
        System.out.println("\nReverse: " + reverse("Hello World"));
        
        // Binary Search
        int[] sortedArr = {1, 3, 5, 7, 9, 11, 13, 15};
        System.out.println("\nBinary Search:");
        System.out.println("ค้นหา 7: index " + binarySearch(sortedArr, 7, 0, sortedArr.length - 1));
        System.out.println("ค้นหา 6: index " + binarySearch(sortedArr, 6, 0, sortedArr.length - 1));
        
        // Hanoi Tower (3 disk)
        System.out.println("\nHanoi Tower (3 disks):");
        hanoi(3, 'A', 'C', 'B');
    }
}
```

---

## Scope ของ Variables

```java
public class VariableScope {
    // Class-level (static field)
    static int classVar = 100;
    
    // Instance field (จะเรียนใน OOP)
    int instanceVar = 200;
    
    static void demonstrateScope() {
        // Method-level (local variable)
        int localVar = 1;
        System.out.println("classVar: " + classVar);  // เข้าถึงได้
        System.out.println("localVar: " + localVar);
        
        // Block scope
        {
            int blockVar = 2;
            System.out.println("blockVar: " + blockVar);  // เข้าถึงได้ใน block
        }
        // System.out.println(blockVar);  // Error! อยู่นอก block
        
        // Loop scope
        for (int i = 0; i < 3; i++) {
            int loopLocal = i * 2;
            System.out.println("loopLocal: " + loopLocal);
        }
        // System.out.println(loopLocal);  // Error!
        // System.out.println(i);          // Error!
        
        // Variable shadowing (ระวัง!)
        int classVar = 999;  // shadow class variable
        System.out.println("local classVar: " + classVar);    // 999
        System.out.println("class classVar: " + VariableScope.classVar);  // 100
    }
    
    public static void main(String[] args) {
        demonstrateScope();
        
        // Scope ใน if block
        int x = 10;
        if (x > 5) {
            int y = 20;  // y อยู่ใน if block เท่านั้น
            System.out.println("y = " + y);
        }
        // System.out.println(y);  // Error!
    }
}
```

---

## Static vs Instance Methods

```java
public class StaticVsInstance {
    
    // Static: เรียกได้โดยตรงจาก class
    // ไม่ต้องสร้าง object
    static double calculateCircleArea(double radius) {
        return Math.PI * radius * radius;
    }
    
    // Instance: ต้องสร้าง object ก่อน
    // มีการเข้าถึง instance state
    private String name;
    private int age;
    
    StaticVsInstance(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Instance method ใช้ this.name, this.age
    String getInfo() {
        return name + " (อายุ " + age + " ปี)";
    }
    
    // Static utility methods (เหมือน Math class)
    static int clamp(int value, int min, int max) {
        return Math.max(min, Math.min(max, value));
    }
    
    static boolean isPalindrome(String s) {
        String clean = s.toLowerCase().replaceAll("[^a-z0-9]", "");
        String reversed = new StringBuilder(clean).reverse().toString();
        return clean.equals(reversed);
    }
    
    public static void main(String[] args) {
        // เรียก static method โดยตรง
        System.out.println("Area: " + calculateCircleArea(5));
        System.out.println("Clamp: " + clamp(150, 0, 100));
        System.out.println("Palindrome 'racecar': " + isPalindrome("racecar"));
        System.out.println("Palindrome 'hello': " + isPalindrome("hello"));
        
        // เรียก instance method ต้องสร้าง object ก่อน
        StaticVsInstance person = new StaticVsInstance("สมชาย", 25);
        System.out.println(person.getInfo());
        
        StaticVsInstance person2 = new StaticVsInstance("สมหญิง", 30);
        System.out.println(person2.getInfo());
    }
}
```

---

## โปรแกรมตัวอย่าง

### โปรแกรม Math Library

```java
public class MathLibrary {
    
    // Basic Math
    static boolean isEven(int n) { return n % 2 == 0; }
    static boolean isOdd(int n) { return n % 2 != 0; }
    static boolean isPrime(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (int i = 3; i * i <= n; i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    static long gcd(long a, long b) {
        return b == 0 ? a : gcd(b, a % b);
    }
    
    static long lcm(long a, long b) {
        return a / gcd(a, b) * b;
    }
    
    // Statistics
    static double mean(double[] data) {
        double sum = 0;
        for (double d : data) sum += d;
        return sum / data.length;
    }
    
    static double variance(double[] data) {
        double mean = mean(data);
        double sumSq = 0;
        for (double d : data) sumSq += Math.pow(d - mean, 2);
        return sumSq / data.length;
    }
    
    static double stdDev(double[] data) {
        return Math.sqrt(variance(data));
    }
    
    // Number conversion
    static String toBinary(int n) { return Integer.toBinaryString(n); }
    static String toHex(int n) { return Integer.toHexString(n).toUpperCase(); }
    static String toOctal(int n) { return Integer.toOctalString(n); }
    
    // Geometric
    static double circleArea(double r) { return Math.PI * r * r; }
    static double circlePerimeter(double r) { return 2 * Math.PI * r; }
    static double rectangleArea(double w, double h) { return w * h; }
    static double triangleArea(double b, double h) { return 0.5 * b * h; }
    static double pythagorean(double a, double b) { return Math.sqrt(a*a + b*b); }
    
    // Finance
    static double simpleInterest(double principal, double rate, double time) {
        return principal * rate * time / 100;
    }
    
    static double compoundInterest(double principal, double rate, int n, double time) {
        return principal * Math.pow(1 + rate / (100 * n), n * time) - principal;
    }
    
    public static void main(String[] args) {
        System.out.println("=== Math Library Demo ===");
        
        // Basic
        System.out.println("\n--- Basic ---");
        System.out.println("isPrime(17): " + isPrime(17));
        System.out.println("GCD(48, 18): " + gcd(48, 18));
        System.out.println("LCM(4, 6): " + lcm(4, 6));
        
        // Stats
        System.out.println("\n--- Statistics ---");
        double[] data = {2, 4, 4, 4, 5, 5, 7, 9};
        System.out.printf("Mean: %.2f%n", mean(data));
        System.out.printf("Variance: %.2f%n", variance(data));
        System.out.printf("Std Dev: %.2f%n", stdDev(data));
        
        // Conversion
        System.out.println("\n--- Conversion ---");
        System.out.println("255 in binary: " + toBinary(255));
        System.out.println("255 in hex: " + toHex(255));
        System.out.println("255 in octal: " + toOctal(255));
        
        // Geometry
        System.out.println("\n--- Geometry ---");
        System.out.printf("Circle area (r=5): %.2f%n", circleArea(5));
        System.out.printf("Triangle area (b=3, h=4): %.2f%n", triangleArea(3, 4));
        System.out.printf("Hypotenuse (3,4): %.2f%n", pythagorean(3, 4));
        
        // Finance
        System.out.println("\n--- Finance ---");
        System.out.printf("Simple interest: %.2f%n", simpleInterest(10000, 5, 3));
        System.out.printf("Compound interest (monthly): %.2f%n", 
            compoundInterest(10000, 5, 12, 3));
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: String Utility Methods

```java
// เฉลย
public class StringUtils {
    
    static boolean isPalindrome(String s) {
        String clean = s.toLowerCase().replaceAll("[^a-zA-Z0-9ก-ฮ]", "");
        String reversed = new StringBuilder(clean).reverse().toString();
        return clean.equals(reversed);
    }
    
    static int countVowels(String s) {
        int count = 0;
        for (char c : s.toLowerCase().toCharArray()) {
            if ("aeiou".indexOf(c) >= 0) count++;
        }
        return count;
    }
    
    static String capitalizeWords(String s) {
        String[] words = s.split(" ");
        StringBuilder result = new StringBuilder();
        for (String word : words) {
            if (!word.isEmpty()) {
                result.append(Character.toUpperCase(word.charAt(0)))
                      .append(word.substring(1).toLowerCase())
                      .append(" ");
            }
        }
        return result.toString().trim();
    }
    
    static String reverseWords(String s) {
        String[] words = s.split(" ");
        StringBuilder result = new StringBuilder();
        for (int i = words.length - 1; i >= 0; i--) {
            result.append(words[i]);
            if (i > 0) result.append(" ");
        }
        return result.toString();
    }
    
    public static void main(String[] args) {
        System.out.println("palindrome 'racecar': " + isPalindrome("racecar"));
        System.out.println("palindrome 'hello': " + isPalindrome("hello"));
        System.out.println("vowels in 'Hello World': " + countVowels("Hello World"));
        System.out.println("capitalize: " + capitalizeWords("hello world java"));
        System.out.println("reverse words: " + reverseWords("Hello World Java"));
    }
}
```

### แบบฝึกหัดที่ 2: Recursive Problems

```java
// เฉลย
public class RecursiveProblems {
    
    // ผลรวม digits: 123 -> 1+2+3 = 6
    static int sumDigits(int n) {
        if (n < 0) n = -n;
        if (n < 10) return n;
        return n % 10 + sumDigits(n / 10);
    }
    
    // นับจำนวน digits
    static int countDigits(int n) {
        if (n < 0) n = -n;
        if (n < 10) return 1;
        return 1 + countDigits(n / 10);
    }
    
    // กลับตัวเลข: 123 -> 321
    static int reverseNumber(int n) {
        boolean negative = n < 0;
        n = Math.abs(n);
        int result = reverseHelper(n, 0);
        return negative ? -result : result;
    }
    
    static int reverseHelper(int n, int reversed) {
        if (n == 0) return reversed;
        return reverseHelper(n / 10, reversed * 10 + n % 10);
    }
    
    public static void main(String[] args) {
        System.out.println("sumDigits(12345) = " + sumDigits(12345));
        System.out.println("countDigits(12345) = " + countDigits(12345));
        System.out.println("reverseNumber(12345) = " + reverseNumber(12345));
        System.out.println("reverseNumber(-12345) = " + reverseNumber(-12345));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ การสร้าง Method พื้นฐาน  
✅ Pass by Value  
✅ Parameters และ Return Types  
✅ Method Overloading  
✅ Varargs  
✅ Recursion  
✅ Variable Scope  
✅ Static vs Instance Methods  

---

## ขั้นตอนต่อไป

**Part 006:** Arrays & Collections Basics  
เราจะเรียนรู้:
- Single & Multi-dimensional Arrays
- Array Methods
- ArrayList
- LinkedList
- Stack, Queue

---

*Part 005 | Java & Spring Boot Course | สร้างโดย Claude Code*
