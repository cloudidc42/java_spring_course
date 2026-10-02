# Part 004: Loops - for, while, do-while
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [for Loop](#for-loop)
2. [while Loop](#while-loop)
3. [do-while Loop](#do-while-loop)
4. [for-each Loop](#for-each-loop)
5. [break และ continue](#break-และ-continue)
6. [Nested Loops](#nested-loops)
7. [Labeled Loops](#labeled-loops)
8. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## for Loop

### รูปแบบพื้นฐาน

```java
// for (initialization; condition; update) { body; }
public class ForLoopBasic {
    public static void main(String[] args) {
        // นับ 1 ถึง 5
        for (int i = 1; i <= 5; i++) {
            System.out.println("i = " + i);
        }
        
        // นับถอยหลัง
        for (int i = 10; i >= 1; i--) {
            System.out.print(i + " ");
        }
        System.out.println();  // newline
        
        // นับทีละ 2
        for (int i = 0; i <= 20; i += 2) {
            System.out.print(i + " ");
        }
        System.out.println();
        
        // Multiple variables
        for (int i = 0, j = 10; i <= 10; i++, j--) {
            System.out.printf("i=%d, j=%d%n", i, j);
        }
    }
}
```

### for Loop ตัวอย่างการใช้งาน

```java
public class ForLoopExamples {
    public static void main(String[] args) {
        // หาผลรวม 1 ถึง 100
        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            sum += i;
        }
        System.out.println("ผลรวม 1-100: " + sum);  // 5050
        
        // หาค่าเฉลี่ย
        int[] scores = {85, 92, 78, 90, 88, 76, 95};
        int total = 0;
        for (int score : scores) {
            total += score;
        }
        double average = (double) total / scores.length;
        System.out.printf("คะแนนเฉลี่ย: %.2f%n", average);
        
        // หาค่าสูงสุดและต่ำสุด
        int max = scores[0], min = scores[0];
        for (int i = 1; i < scores.length; i++) {
            if (scores[i] > max) max = scores[i];
            if (scores[i] < min) min = scores[i];
        }
        System.out.println("สูงสุด: " + max + ", ต่ำสุด: " + min);
        
        // ตารางสูตรคูณ
        System.out.println("\nตารางสูตรคูณแม่ 5:");
        for (int i = 1; i <= 12; i++) {
            System.out.printf("5 × %2d = %3d%n", i, 5 * i);
        }
        
        // Fibonacci
        System.out.println("\nFibonacci 15 ตัว:");
        int a = 0, b = 1;
        System.out.print(a + " " + b + " ");
        for (int i = 2; i < 15; i++) {
            int next = a + b;
            System.out.print(next + " ");
            a = b;
            b = next;
        }
        System.out.println();
    }
}
```

---

## while Loop

```java
public class WhileLoop {
    public static void main(String[] args) {
        // รูปแบบ: while (condition) { body; }
        // ใช้เมื่อไม่รู้จำนวนรอบล่วงหน้า
        
        // นับ 1-5
        int i = 1;
        while (i <= 5) {
            System.out.println("i = " + i);
            i++;  // อย่าลืม! จะเกิด infinite loop
        }
        
        // อ่านจนกว่าจะได้ค่าที่ถูกต้อง
        java.util.Scanner sc = new java.util.Scanner(System.in);
        int number = 0;
        boolean validInput = false;
        
        while (!validInput) {
            System.out.print("ใส่จำนวนบวก: ");
            if (sc.hasNextInt()) {
                number = sc.nextInt();
                if (number > 0) {
                    validInput = true;
                } else {
                    System.out.println("ต้องเป็นจำนวนบวกเท่านั้น");
                }
            } else {
                System.out.println("กรุณาใส่ตัวเลข");
                sc.next();  // consume invalid input
            }
        }
        System.out.println("ได้รับ: " + number);
        
        // หาร จนกว่าจะหาร 2 ไม่ได้
        int n = 256;
        int steps = 0;
        System.out.print("\n" + n + " -> ");
        while (n > 1 && n % 2 == 0) {
            n /= 2;
            steps++;
            System.out.print(n + " -> ");
        }
        System.out.println("(หาร 2 ได้ " + steps + " ครั้ง)");
        
        sc.close();
    }
}
```

### Infinite Loop (ระวัง!)

```java
public class InfiniteLoopExample {
    public static void main(String[] args) {
        // Infinite loop ที่ออกด้วย break
        int count = 0;
        while (true) {  // เงื่อนไขเป็น true เสมอ
            count++;
            System.out.println("Count: " + count);
            
            if (count >= 5) {
                break;  // ออกจาก loop
            }
        }
        
        // Pattern ที่ใช้บ่อยใน server applications
        // while (serverIsRunning) {
        //     Request request = waitForRequest();
        //     processRequest(request);
        // }
        
        // Loop ที่รอ event
        boolean serverRunning = true;
        int requests = 0;
        while (serverRunning) {
            // simulate handling requests
            requests++;
            System.out.println("Processing request #" + requests);
            
            if (requests >= 3) {
                System.out.println("Server shutting down...");
                serverRunning = false;
            }
        }
    }
}
```

---

## do-while Loop

```java
public class DoWhileLoop {
    public static void main(String[] args) {
        // do { body; } while (condition);
        // ทำงานอย่างน้อย 1 ครั้ง แม้เงื่อนไขเป็น false
        
        // ตัวอย่างพื้นฐาน
        int i = 1;
        do {
            System.out.println("i = " + i);
            i++;
        } while (i <= 5);
        
        // เหมาะกับ menu-driven program
        java.util.Scanner sc = new java.util.Scanner(System.in);
        int choice;
        
        do {
            System.out.println("\n=== MENU ===");
            System.out.println("1. ตัวเลือก A");
            System.out.println("2. ตัวเลือก B");
            System.out.println("3. ตัวเลือก C");
            System.out.println("0. ออก");
            System.out.print("เลือก: ");
            choice = sc.nextInt();
            
            switch (choice) {
                case 1 -> System.out.println("เลือก A");
                case 2 -> System.out.println("เลือก B");
                case 3 -> System.out.println("เลือก C");
                case 0 -> System.out.println("ลาก่อน!");
                default -> System.out.println("กรุณาเลือก 0-3");
            }
        } while (choice != 0);
        
        // Validate input
        int pin;
        do {
            System.out.print("\nใส่ PIN (4 หลัก): ");
            pin = sc.nextInt();
            
            if (pin < 1000 || pin > 9999) {
                System.out.println("PIN ต้องมี 4 หลัก");
            }
        } while (pin < 1000 || pin > 9999);
        
        System.out.println("PIN ที่ได้รับ: " + pin);
        
        sc.close();
    }
}
```

---

## for-each Loop

```java
import java.util.List;
import java.util.ArrayList;

public class ForEachLoop {
    public static void main(String[] args) {
        // for (ElementType element : collection) { body; }
        
        // กับ Array
        int[] numbers = {1, 2, 3, 4, 5};
        System.out.print("Numbers: ");
        for (int num : numbers) {
            System.out.print(num + " ");
        }
        System.out.println();
        
        // กับ String array
        String[] fruits = {"apple", "banana", "cherry", "date"};
        System.out.println("Fruits:");
        for (String fruit : fruits) {
            System.out.println("  - " + fruit);
        }
        
        // กับ List
        List<String> cities = new ArrayList<>();
        cities.add("กรุงเทพ");
        cities.add("เชียงใหม่");
        cities.add("ภูเก็ต");
        cities.add("ขอนแก่น");
        
        System.out.println("\nเมืองในไทย:");
        for (String city : cities) {
            System.out.println("  • " + city);
        }
        
        // กับ 2D array
        int[][] matrix = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };
        
        System.out.println("\nMatrix:");
        for (int[] row : matrix) {
            for (int val : row) {
                System.out.printf("%3d", val);
            }
            System.out.println();
        }
        
        // หาผลรวม
        double[] prices = {99.99, 149.50, 79.99, 250.00};
        double total = 0;
        for (double price : prices) {
            total += price;
        }
        System.out.printf("%nยอดรวม: %.2f%n", total);
        
        // Note: for-each ไม่สามารถแก้ไข element ของ primitive array ได้
        int[] arr = {1, 2, 3};
        for (int x : arr) {
            x *= 2;  // ไม่เปลี่ยนแปลง array จริง!
        }
        System.out.print("\nArr ยังเหมือนเดิม: ");
        for (int x : arr) System.out.print(x + " ");
        System.out.println();
        
        // ถ้าต้องการแก้ไขต้องใช้ for ธรรมดา
        for (int i = 0; i < arr.length; i++) {
            arr[i] *= 2;
        }
        System.out.print("Arr หลังแก้ไข: ");
        for (int x : arr) System.out.print(x + " ");
    }
}
```

---

## break และ continue

```java
public class BreakContinue {
    public static void main(String[] args) {
        // break: ออกจาก loop ทันที
        System.out.println("=== break ===");
        for (int i = 1; i <= 10; i++) {
            if (i == 6) {
                System.out.println("พบ 6! หยุด.");
                break;
            }
            System.out.print(i + " ");
        }
        System.out.println();
        
        // continue: ข้ามไปรอบถัดไป
        System.out.println("\n=== continue ===");
        System.out.print("เลขคี่: ");
        for (int i = 1; i <= 10; i++) {
            if (i % 2 == 0) {
                continue;  // ข้ามเลขคู่
            }
            System.out.print(i + " ");
        }
        System.out.println();
        
        // หาค่า prime number
        System.out.println("\nจำนวนเฉพาะ 1-50:");
        for (int n = 2; n <= 50; n++) {
            boolean isPrime = true;
            for (int i = 2; i <= Math.sqrt(n); i++) {
                if (n % i == 0) {
                    isPrime = false;
                    break;  // ออกจาก inner loop
                }
            }
            if (isPrime) {
                System.out.print(n + " ");
            }
        }
        System.out.println();
        
        // Linear search
        int[] arr = {3, 7, 1, 9, 4, 6, 8, 2, 5};
        int target = 6;
        int foundIndex = -1;
        
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) {
                foundIndex = i;
                break;  // พบแล้วหยุด
            }
        }
        
        if (foundIndex != -1) {
            System.out.println("\nพบ " + target + " ที่ index " + foundIndex);
        } else {
            System.out.println("ไม่พบ " + target);
        }
        
        // Skip negative numbers
        double[] data = {1.5, -2.3, 4.1, -0.5, 3.7, -1.0, 2.8};
        double positiveSum = 0;
        
        System.out.print("\nบวกเฉพาะค่าบวก: ");
        for (double val : data) {
            if (val <= 0) {
                continue;  // ข้ามค่าที่ไม่ใช่บวก
            }
            System.out.print(val + " ");
            positiveSum += val;
        }
        System.out.printf("\nผลรวม: %.1f%n", positiveSum);
    }
}
```

---

## Nested Loops

```java
public class NestedLoops {
    public static void main(String[] args) {
        // ตารางสูตรคูณ
        System.out.println("ตารางสูตรคูณ 1-5:");
        System.out.printf("%5s", "×");
        for (int i = 1; i <= 5; i++) {
            System.out.printf("%5d", i);
        }
        System.out.println();
        System.out.println("-".repeat(30));
        
        for (int i = 1; i <= 5; i++) {
            System.out.printf("%5d", i);
            for (int j = 1; j <= 5; j++) {
                System.out.printf("%5d", i * j);
            }
            System.out.println();
        }
        
        // Pattern: สามเหลี่ยม
        System.out.println("\nสามเหลี่ยมดาว:");
        int rows = 6;
        for (int i = 1; i <= rows; i++) {
            for (int j = 1; j <= i; j++) {
                System.out.print("★ ");
            }
            System.out.println();
        }
        
        // Pattern: สามเหลี่ยมกลับหัว
        System.out.println("\nสามเหลี่ยมกลับหัว:");
        for (int i = rows; i >= 1; i--) {
            for (int j = 1; j <= i; j++) {
                System.out.print("★ ");
            }
            System.out.println();
        }
        
        // Pattern: เพชร
        System.out.println("\nรูปเพชร:");
        int n = 5;
        for (int i = 1; i <= n; i++) {
            // spaces
            for (int j = 1; j <= n - i; j++) System.out.print(" ");
            // stars
            for (int j = 1; j <= 2 * i - 1; j++) System.out.print("*");
            System.out.println();
        }
        for (int i = n - 1; i >= 1; i--) {
            for (int j = 1; j <= n - i; j++) System.out.print(" ");
            for (int j = 1; j <= 2 * i - 1; j++) System.out.print("*");
            System.out.println();
        }
        
        // Matrix operations
        System.out.println("\nคูณ Matrix 2x2:");
        int[][] A = {{1, 2}, {3, 4}};
        int[][] B = {{5, 6}, {7, 8}};
        int[][] C = new int[2][2];
        
        for (int i = 0; i < 2; i++) {
            for (int j = 0; j < 2; j++) {
                for (int k = 0; k < 2; k++) {
                    C[i][j] += A[i][k] * B[k][j];
                }
            }
        }
        
        System.out.println("A × B = ");
        for (int[] row : C) {
            for (int val : row) {
                System.out.printf("%5d", val);
            }
            System.out.println();
        }
    }
}
```

---

## Labeled Loops

```java
public class LabeledLoops {
    public static void main(String[] args) {
        // Label ใช้กับ break/continue ใน nested loops
        
        // ออกจาก outer loop
        outer:
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                if (i == 2 && j == 2) {
                    System.out.println("พบที่ (" + i + "," + j + ") ออกจาก outer loop");
                    break outer;
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        
        // continue outer loop
        System.out.println("\ncontinue outer:");
        search:
        for (int i = 0; i < 4; i++) {
            for (int j = 0; j < 4; j++) {
                if (j == 2) {
                    System.out.println("j=2, ข้ามไป outer iteration ถัดไป");
                    continue search;
                }
                System.out.println("(" + i + "," + j + ")");
            }
        }
        
        // ค้นหาใน 2D array
        int[][] matrix = {
            {1, 5, 3},
            {7, 2, 8},
            {4, 9, 6}
        };
        int target = 8;
        boolean found = false;
        
        findTarget:
        for (int i = 0; i < matrix.length; i++) {
            for (int j = 0; j < matrix[i].length; j++) {
                if (matrix[i][j] == target) {
                    System.out.printf("%nพบ %d ที่ [%d][%d]%n", target, i, j);
                    found = true;
                    break findTarget;
                }
            }
        }
        
        if (!found) {
            System.out.println("ไม่พบ " + target);
        }
    }
}
```

---

## โปรแกรมตัวอย่าง

### ตัวอย่าง 1: เกมทายตัวเลข

```java
import java.util.Scanner;
import java.util.Random;

public class NumberGuessingGame {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        Random random = new Random();
        
        int totalGames = 0, totalGuesses = 0;
        boolean playAgain = true;
        
        System.out.println("╔════════════════════════════╗");
        System.out.println("║     เกมทายตัวเลข          ║");
        System.out.println("╚════════════════════════════╝");
        
        while (playAgain) {
            int secretNumber = random.nextInt(100) + 1;  // 1-100
            int attempts = 0;
            int maxAttempts = 7;
            boolean won = false;
            
            System.out.println("\nฉันคิดเลข 1-100 ไว้ คุณมี " + maxAttempts + " ครั้ง");
            
            while (attempts < maxAttempts) {
                attempts++;
                System.out.printf("ครั้งที่ %d/%d - ทาย: ", attempts, maxAttempts);
                int guess = sc.nextInt();
                
                if (guess == secretNumber) {
                    System.out.println("🎉 ถูกต้อง! ใช้ " + attempts + " ครั้ง");
                    won = true;
                    break;
                } else if (guess < secretNumber) {
                    int remaining = maxAttempts - attempts;
                    System.out.println("น้อยเกินไป! เหลือ " + remaining + " ครั้ง");
                } else {
                    int remaining = maxAttempts - attempts;
                    System.out.println("มากเกินไป! เหลือ " + remaining + " ครั้ง");
                }
            }
            
            if (!won) {
                System.out.println("😢 หมดสิทธิ์! เลขที่คิดคือ " + secretNumber);
            }
            
            totalGames++;
            totalGuesses += attempts;
            
            System.out.print("\nเล่นอีกครั้ง? (y/n): ");
            String answer = sc.next();
            playAgain = answer.equalsIgnoreCase("y");
        }
        
        System.out.println("\n=== สถิติ ===");
        System.out.println("จำนวนเกม: " + totalGames);
        System.out.printf("เฉลี่ยครั้งที่ทาย: %.1f%n", 
            totalGames > 0 ? (double) totalGuesses / totalGames : 0);
        
        sc.close();
    }
}
```

### ตัวอย่าง 2: Prime Number Generator

```java
public class PrimeGenerator {
    public static void main(String[] args) {
        int limit = 100;
        
        // Sieve of Eratosthenes
        boolean[] isPrime = new boolean[limit + 1];
        
        // ตั้งค่าเริ่มต้นทุกตัวเป็น true
        for (int i = 2; i <= limit; i++) {
            isPrime[i] = true;
        }
        
        // ตัดตัวที่ไม่ใช่จำนวนเฉพาะออก
        for (int i = 2; i * i <= limit; i++) {
            if (isPrime[i]) {
                for (int j = i * i; j <= limit; j += i) {
                    isPrime[j] = false;
                }
            }
        }
        
        // แสดงผล
        System.out.println("จำนวนเฉพาะ 1 ถึง " + limit + ":");
        int count = 0;
        for (int i = 2; i <= limit; i++) {
            if (isPrime[i]) {
                System.out.printf("%4d", i);
                count++;
                if (count % 10 == 0) System.out.println();
            }
        }
        System.out.println("\n\nพบทั้งหมด " + count + " ตัว");
        
        // ตรวจสอบว่าเป็นจำนวนเฉพาะ
        System.out.println("\nตรวจสอบ:");
        int[] testNumbers = {7, 15, 97, 100, 101};
        for (int num : testNumbers) {
            boolean prime = num <= limit ? isPrime[num] : checkPrime(num);
            System.out.println(num + " เป็นจำนวนเฉพาะ: " + prime);
        }
    }
    
    static boolean checkPrime(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (int i = 3; i <= Math.sqrt(n); i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
}
```

### ตัวอย่าง 3: Simple Statistics

```java
import java.util.Scanner;
import java.util.Arrays;

public class SimpleStatistics {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.print("ใส่จำนวนข้อมูล: ");
        int n = sc.nextInt();
        
        double[] data = new double[n];
        System.out.println("ใส่ข้อมูล:");
        for (int i = 0; i < n; i++) {
            System.out.print("  ข้อมูลที่ " + (i + 1) + ": ");
            data[i] = sc.nextDouble();
        }
        
        // คำนวณสถิติ
        double sum = 0;
        double min = data[0], max = data[0];
        
        for (double val : data) {
            sum += val;
            if (val < min) min = val;
            if (val > max) max = val;
        }
        
        double mean = sum / n;
        
        // Variance และ Standard Deviation
        double variance = 0;
        for (double val : data) {
            variance += Math.pow(val - mean, 2);
        }
        variance /= n;
        double stdDev = Math.sqrt(variance);
        
        // Median
        double[] sorted = Arrays.copyOf(data, n);
        Arrays.sort(sorted);
        double median;
        if (n % 2 == 0) {
            median = (sorted[n/2 - 1] + sorted[n/2]) / 2.0;
        } else {
            median = sorted[n/2];
        }
        
        // Mode (ค่าที่พบบ่อยที่สุด)
        double mode = sorted[0];
        int maxCount = 1, currentCount = 1;
        for (int i = 1; i < n; i++) {
            if (sorted[i] == sorted[i-1]) {
                currentCount++;
                if (currentCount > maxCount) {
                    maxCount = currentCount;
                    mode = sorted[i];
                }
            } else {
                currentCount = 1;
            }
        }
        
        System.out.println("\n═══════════════════════════");
        System.out.println("  สถิติ");
        System.out.println("═══════════════════════════");
        System.out.printf("จำนวนข้อมูล:    %10.0f%n", (double)n);
        System.out.printf("ผลรวม:          %10.2f%n", sum);
        System.out.printf("ค่าเฉลี่ย:      %10.2f%n", mean);
        System.out.printf("มัธยฐาน:        %10.2f%n", median);
        System.out.printf("ฐานนิยม:        %10.2f%n", mode);
        System.out.printf("ค่าสูงสุด:      %10.2f%n", max);
        System.out.printf("ค่าต่ำสุด:      %10.2f%n", min);
        System.out.printf("พิสัย:          %10.2f%n", max - min);
        System.out.printf("ความแปรปรวน:    %10.2f%n", variance);
        System.out.printf("ส่วนเบี่ยงเบน:  %10.2f%n", stdDev);
        System.out.println("═══════════════════════════");
        
        sc.close();
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้างตัวเลข
พิมพ์ตัวเลข 1-100 โดย:
- ถ้าหาร 3 ลงตัว พิมพ์ "Fizz"
- ถ้าหาร 5 ลงตัว พิมพ์ "Buzz"  
- ถ้าหาร 15 ลงตัว พิมพ์ "FizzBuzz"
- นอกนั้นพิมพ์ตัวเลข

```java
// เฉลย
public class FizzBuzz {
    public static void main(String[] args) {
        for (int i = 1; i <= 100; i++) {
            String output;
            if (i % 15 == 0) {
                output = "FizzBuzz";
            } else if (i % 3 == 0) {
                output = "Fizz";
            } else if (i % 5 == 0) {
                output = "Buzz";
            } else {
                output = String.valueOf(i);
            }
            System.out.printf("%-10s", output);
            if (i % 10 == 0) System.out.println();
        }
    }
}
```

### แบบฝึกหัดที่ 2: Pattern Printing
พิมพ์รูปแบบต่อไปนี้:
```
1
12
123
1234
12345
```

```java
// เฉลย
public class NumberPattern {
    public static void main(String[] args) {
        int rows = 5;
        for (int i = 1; i <= rows; i++) {
            for (int j = 1; j <= i; j++) {
                System.out.print(j);
            }
            System.out.println();
        }
        
        // แบบที่ 2
        System.out.println("\n");
        for (int i = 1; i <= rows; i++) {
            for (int j = rows; j >= i; j--) {
                System.out.printf("%2d", j);
            }
            System.out.println();
        }
    }
}
```

### แบบฝึกหัดที่ 3: Bubble Sort
เขียน Bubble Sort เรียงตัวเลขจากน้อยไปมาก

```java
// เฉลย
import java.util.Arrays;

public class BubbleSort {
    public static void main(String[] args) {
        int[] arr = {64, 34, 25, 12, 22, 11, 90};
        
        System.out.println("ก่อนเรียง: " + Arrays.toString(arr));
        
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    // swap
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                }
            }
        }
        
        System.out.println("หลังเรียง:  " + Arrays.toString(arr));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ for Loop ทุกรูปแบบ  
✅ while Loop  
✅ do-while Loop  
✅ for-each Loop  
✅ break และ continue  
✅ Nested Loops  
✅ Labeled Loops  

---

## ขั้นตอนต่อไป

**Part 005:** Methods & Functions  
เราจะเรียนรู้:
- การสร้าง Method
- Parameters และ Return Types
- Method Overloading
- Recursion
- Variable Scope

---

*Part 004 | Java & Spring Boot Course | สร้างโดย Claude Code*
