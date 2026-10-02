# Part 003: Control Flow - if/else, switch
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [if Statement](#if-statement)
2. [if-else Statement](#if-else-statement)
3. [if-else-if Ladder](#if-else-if-ladder)
4. [Nested if](#nested-if)
5. [switch Statement](#switch-statement)
6. [switch Expression (Java 14+)](#switch-expression-java-14)
7. [Pattern Matching switch (Java 21)](#pattern-matching-switch-java-21)
8. [Guard Clauses](#guard-clauses)
9. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## if Statement

### รูปแบบพื้นฐาน

```java
// รูปแบบ: if (condition) { statements; }
public class IfBasic {
    public static void main(String[] args) {
        int temperature = 35;
        
        if (temperature > 30) {
            System.out.println("อากาศร้อนมาก!");
            System.out.println("ควรดื่มน้ำมากๆ");
        }
        
        // Single line (ไม่แนะนำ แต่ทำได้)
        if (temperature > 30) System.out.println("ร้อน!");
        
        // ตัวอย่างการตรวจสอบ
        int score = 85;
        if (score >= 80) {
            System.out.println("ผ่านการสอบ! คะแนน: " + score);
        }
        
        // Boolean variable
        boolean isLoggedIn = true;
        if (isLoggedIn) {
            System.out.println("ยินดีต้อนรับ!");
        }
        
        // Null check
        String username = "admin";
        if (username != null && !username.isEmpty()) {
            System.out.println("Username: " + username);
        }
    }
}
```

---

## if-else Statement

```java
public class IfElse {
    public static void main(String[] args) {
        // พื้นฐาน
        int number = 7;
        
        if (number % 2 == 0) {
            System.out.println(number + " เป็นจำนวนคู่");
        } else {
            System.out.println(number + " เป็นจำนวนคี่");
        }
        
        // ตรวจสอบอายุ
        int age = 17;
        String access;
        if (age >= 18) {
            access = "อนุญาต";
        } else {
            access = "ไม่อนุญาต (ต้องอายุ 18 ปีขึ้นไป)";
        }
        System.out.println("การเข้าถึง: " + access);
        
        // ตรวจสอบเกรด
        double gpa = 3.5;
        if (gpa >= 3.5) {
            System.out.println("เกียรตินิยมอันดับหนึ่ง");
        } else {
            System.out.println("ไม่ได้รับเกียรตินิยม");
        }
    }
}
```

---

## if-else-if Ladder

```java
public class IfElseIfLadder {
    public static void main(String[] args) {
        // ระบบเกรด
        int score = 78;
        
        String grade;
        String description;
        
        if (score >= 90) {
            grade = "A";
            description = "ยอดเยี่ยม";
        } else if (score >= 80) {
            grade = "B";
            description = "ดีมาก";
        } else if (score >= 70) {
            grade = "C";
            description = "ดี";
        } else if (score >= 60) {
            grade = "D";
            description = "พอใช้";
        } else {
            grade = "F";
            description = "ไม่ผ่าน";
        }
        
        System.out.println("คะแนน: " + score);
        System.out.println("เกรด: " + grade + " (" + description + ")");
        
        // ตัวอย่าง: คำนวณค่าจัดส่ง
        double orderAmount = 350.0;
        double shippingFee;
        String shippingMessage;
        
        if (orderAmount >= 1000) {
            shippingFee = 0;
            shippingMessage = "ฟรีค่าจัดส่ง!";
        } else if (orderAmount >= 500) {
            shippingFee = 20;
            shippingMessage = "ค่าจัดส่ง 20 บาท";
        } else if (orderAmount >= 200) {
            shippingFee = 40;
            shippingMessage = "ค่าจัดส่ง 40 บาท";
        } else {
            shippingFee = 60;
            shippingMessage = "ค่าจัดส่ง 60 บาท";
        }
        
        System.out.printf("\nยอดสั่งซื้อ: %.2f บาท%n", orderAmount);
        System.out.println(shippingMessage);
        System.out.printf("รวม: %.2f บาท%n", orderAmount + shippingFee);
        
        // ตัวอย่าง: ประเมิน BMI
        double bmi = 22.5;
        String bmiCategory;
        
        if (bmi < 18.5) {
            bmiCategory = "น้ำหนักน้อยเกินไป";
        } else if (bmi < 25.0) {
            bmiCategory = "น้ำหนักปกติ ✓";
        } else if (bmi < 30.0) {
            bmiCategory = "น้ำหนักเกิน";
        } else if (bmi < 35.0) {
            bmiCategory = "อ้วนระดับ 1";
        } else if (bmi < 40.0) {
            bmiCategory = "อ้วนระดับ 2";
        } else {
            bmiCategory = "อ้วนมาก ระดับ 3";
        }
        
        System.out.printf("%nBMI: %.1f - %s%n", bmi, bmiCategory);
    }
}
```

---

## Nested if

```java
public class NestedIf {
    public static void main(String[] args) {
        // ตัวอย่าง: ระบบ Login
        String username = "admin";
        String password = "secret123";
        boolean isActive = true;
        
        if (username != null && !username.isEmpty()) {
            if (password != null && password.length() >= 6) {
                if (isActive) {
                    System.out.println("เข้าสู่ระบบสำเร็จ!");
                    System.out.println("ยินดีต้อนรับ: " + username);
                } else {
                    System.out.println("บัญชีถูกระงับ กรุณาติดต่อผู้ดูแลระบบ");
                }
            } else {
                System.out.println("รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร");
            }
        } else {
            System.out.println("กรุณาใส่ชื่อผู้ใช้");
        }
        
        // แต่ควรใช้ Guard Clauses แทน Nested if (ดูหัวข้อถัดไป)
        
        // ตัวอย่าง: ตรวจสอบสามเหลี่ยม
        int a = 5, b = 5, c = 5;
        
        if (a + b > c && a + c > b && b + c > a) {
            if (a == b && b == c) {
                System.out.println("\nสามเหลี่ยมด้านเท่า");
            } else if (a == b || b == c || a == c) {
                System.out.println("\nสามเหลี่ยมหน้าจั่ว");
            } else {
                System.out.println("\nสามเหลี่ยมด้านไม่เท่า");
            }
        } else {
            System.out.println("\nไม่ใช่สามเหลี่ยม");
        }
    }
}
```

---

## switch Statement

### switch แบบดั้งเดิม

```java
public class SwitchStatement {
    public static void main(String[] args) {
        // switch กับ int
        int dayOfWeek = 3;
        String dayName;
        
        switch (dayOfWeek) {
            case 1:
                dayName = "วันจันทร์";
                break;
            case 2:
                dayName = "วันอังคาร";
                break;
            case 3:
                dayName = "วันพุธ";
                break;
            case 4:
                dayName = "วันพฤหัสบดี";
                break;
            case 5:
                dayName = "วันศุกร์";
                break;
            case 6:
                dayName = "วันเสาร์";
                break;
            case 7:
                dayName = "วันอาทิตย์";
                break;
            default:
                dayName = "ไม่ถูกต้อง";
        }
        System.out.println("วันที่ " + dayOfWeek + " คือ " + dayName);
        
        // switch กับ String
        String season = "summer";
        String weather;
        
        switch (season.toLowerCase()) {
            case "spring":
                weather = "อากาศอบอุ่น ดอกไม้บาน";
                break;
            case "summer":
                weather = "อากาศร้อน ฝนตก";
                break;
            case "autumn":
            case "fall":
                weather = "อากาศเย็น ใบไม้ร่วง";
                break;
            case "winter":
                weather = "อากาศหนาว หิมะตก";
                break;
            default:
                weather = "ไม่รู้จักฤดูกาลนี้";
        }
        System.out.println("ฤดู " + season + ": " + weather);
        
        // Fall-through behavior (ไม่มี break)
        int month = 4;
        int daysInMonth;
        
        switch (month) {
            case 1: case 3: case 5: case 7:
            case 8: case 10: case 12:
                daysInMonth = 31;
                break;
            case 4: case 6: case 9: case 11:
                daysInMonth = 30;
                break;
            case 2:
                daysInMonth = 28;  // ไม่นับปีอธิกสุรทิน
                break;
            default:
                daysInMonth = -1;
        }
        System.out.println("เดือน " + month + " มี " + daysInMonth + " วัน");
    }
}
```

---

## switch Expression (Java 14+)

### Arrow switch (แนะนำ)

```java
public class SwitchExpression {
    public static void main(String[] args) {
        // switch expression แบบ arrow
        int dayOfWeek = 3;
        
        String dayName = switch (dayOfWeek) {
            case 1 -> "วันจันทร์";
            case 2 -> "วันอังคาร";
            case 3 -> "วันพุธ";
            case 4 -> "วันพฤหัสบดี";
            case 5 -> "วันศุกร์";
            case 6 -> "วันเสาร์";
            case 7 -> "วันอาทิตย์";
            default -> throw new IllegalArgumentException("Invalid day: " + dayOfWeek);
        };
        System.out.println(dayName);
        
        // Multiple labels
        boolean isWeekend = switch (dayOfWeek) {
            case 1, 2, 3, 4, 5 -> false;
            case 6, 7 -> true;
            default -> throw new IllegalArgumentException("Invalid day");
        };
        System.out.println("เป็นวันหยุด: " + isWeekend);
        
        // switch expression กับ block (yield)
        String season = "summer";
        String description = switch (season) {
            case "spring" -> "ดอกไม้บาน";
            case "summer" -> {
                String base = "ร้อน";
                String extra = " และมีฝน";
                yield base + extra;  // ใช้ yield ใน block
            }
            case "autumn", "fall" -> "ใบไม้ร่วง";
            case "winter" -> "หนาว";
            default -> "ไม่รู้จัก";
        };
        System.out.println("ฤดู: " + description);
        
        // ใช้ใน method call
        printDayType(dayOfWeek);
        
        // switch กับ enum (จะเรียนเรื่อง enum ใน Part 007)
        // Day day = Day.WEDNESDAY;
        // String type = switch (day) {
        //     case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
        //     case SATURDAY, SUNDAY -> "Weekend";
        // };
    }
    
    static void printDayType(int day) {
        String type = switch (day) {
            case 1, 2, 3, 4, 5 -> "วันทำงาน";
            case 6, 7 -> "วันหยุด";
            default -> "ไม่ถูกต้อง";
        };
        System.out.println("ประเภทวัน: " + type);
    }
}
```

---

## Pattern Matching switch (Java 21)

```java
public class PatternMatchingSwitch {
    // Sealed interface สำหรับ example
    sealed interface Shape permits Circle, Rectangle, Triangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double width, double height) implements Shape {}
    record Triangle(double base, double height) implements Shape {}
    
    public static void main(String[] args) {
        // Pattern matching กับ Object types
        Object obj = "Hello, World!";
        
        String result = switch (obj) {
            case Integer i -> "Integer: " + i;
            case Long l -> "Long: " + l;
            case Double d -> "Double: " + d;
            case String s -> "String ความยาว " + s.length() + ": " + s;
            case int[] arr -> "int array ความยาว " + arr.length;
            case null -> "null value";
            default -> "Unknown type: " + obj.getClass().getName();
        };
        System.out.println(result);
        
        // Pattern matching กับ Guarded patterns
        Object value = 42;
        String category = switch (value) {
            case Integer i when i < 0 -> "จำนวนลบ";
            case Integer i when i == 0 -> "ศูนย์";
            case Integer i when i > 0 && i <= 100 -> "1 ถึง 100";
            case Integer i -> "มากกว่า 100: " + i;
            default -> "ไม่ใช่ integer";
        };
        System.out.println(category);
        
        // คำนวณพื้นที่ด้วย Pattern matching
        Shape shape = new Circle(5.0);
        double area = calculateArea(shape);
        System.out.printf("พื้นที่: %.2f%n", area);
        
        // ทดสอบกับ Shape อื่น
        Shape rect = new Rectangle(4.0, 6.0);
        System.out.printf("พื้นที่สี่เหลี่ยม: %.2f%n", calculateArea(rect));
        
        Shape tri = new Triangle(3.0, 4.0);
        System.out.printf("พื้นที่สามเหลี่ยม: %.2f%n", calculateArea(tri));
    }
    
    static double calculateArea(Shape shape) {
        return switch (shape) {
            case Circle c -> Math.PI * c.radius() * c.radius();
            case Rectangle r -> r.width() * r.height();
            case Triangle t -> 0.5 * t.base() * t.height();
        };
    }
}
```

---

## Guard Clauses (Best Practice)

Guard Clauses ช่วยลด Nested if ให้โค้ดอ่านง่ายขึ้น

```java
public class GuardClauses {
    
    // ❌ แบบที่ควรหลีกเลี่ยง (Nested if สูงมาก)
    static String processOrderBad(String userId, String productId, int quantity, double balance) {
        if (userId != null) {
            if (productId != null) {
                if (quantity > 0) {
                    if (balance > 0) {
                        double price = getPrice(productId) * quantity;
                        if (balance >= price) {
                            return "สั่งซื้อสำเร็จ";
                        } else {
                            return "ยอดเงินไม่พียงพอ";
                        }
                    } else {
                        return "ยอดเงินไม่ถูกต้อง";
                    }
                } else {
                    return "จำนวนต้องมากกว่า 0";
                }
            } else {
                return "กรุณาระบุสินค้า";
            }
        } else {
            return "กรุณาเข้าสู่ระบบ";
        }
    }
    
    // ✅ แบบที่แนะนำ (Guard Clauses)
    static String processOrderGood(String userId, String productId, int quantity, double balance) {
        // Guard clauses - ตรวจสอบ error cases ก่อน return ทันที
        if (userId == null || userId.isEmpty()) {
            return "กรุณาเข้าสู่ระบบ";
        }
        
        if (productId == null || productId.isEmpty()) {
            return "กรุณาระบุสินค้า";
        }
        
        if (quantity <= 0) {
            return "จำนวนต้องมากกว่า 0";
        }
        
        if (balance <= 0) {
            return "ยอดเงินไม่ถูกต้อง";
        }
        
        double price = getPrice(productId) * quantity;
        if (balance < price) {
            return String.format("ยอดเงินไม่เพียงพอ (ต้องการ %.2f บาท)", price);
        }
        
        // Happy path - โค้ดหลักอยู่ท้ายสุด
        return "สั่งซื้อสำเร็จ ราคา: " + price;
    }
    
    static double getPrice(String productId) {
        return switch (productId) {
            case "P001" -> 99.0;
            case "P002" -> 199.0;
            case "P003" -> 299.0;
            default -> 0.0;
        };
    }
    
    public static void main(String[] args) {
        // ทดสอบ
        System.out.println(processOrderGood(null, "P001", 2, 500.0));
        System.out.println(processOrderGood("user1", null, 2, 500.0));
        System.out.println(processOrderGood("user1", "P001", -1, 500.0));
        System.out.println(processOrderGood("user1", "P001", 2, 50.0));
        System.out.println(processOrderGood("user1", "P001", 2, 500.0));
    }
}
```

---

## โปรแกรมตัวอย่าง

### ตัวอย่าง 1: ระบบคำนวณภาษี

```java
import java.util.Scanner;

public class TaxCalculator {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.println("╔══════════════════════════════════════╗");
        System.out.println("║    ระบบคำนวณภาษีเงินได้บุคคลธรรมดา  ║");
        System.out.println("╚══════════════════════════════════════╝");
        
        System.out.print("รายได้ต่อปี (บาท): ");
        double income = sc.nextDouble();
        
        System.out.print("ค่าใช้จ่าย (50% ของรายได้ แต่ไม่เกิน 100,000): ");
        double expenses = Math.min(income * 0.5, 100000);
        
        System.out.print("ค่าลดหย่อนส่วนตัว (60,000): ");
        double personalDeduction = 60000;
        
        double netIncome = income - expenses - personalDeduction;
        double tax = 0;
        String taxBracket;
        
        if (netIncome <= 0) {
            tax = 0;
            taxBracket = "ไม่ต้องเสียภาษี";
        } else if (netIncome <= 150000) {
            tax = 0;
            taxBracket = "0% (รายได้ไม่เกิน 150,000)";
        } else if (netIncome <= 300000) {
            tax = (netIncome - 150000) * 0.05;
            taxBracket = "5% (150,001 - 300,000)";
        } else if (netIncome <= 500000) {
            tax = (150000 * 0.05) + (netIncome - 300000) * 0.10;
            taxBracket = "10% (300,001 - 500,000)";
        } else if (netIncome <= 750000) {
            tax = (150000 * 0.05) + (200000 * 0.10) + (netIncome - 500000) * 0.15;
            taxBracket = "15% (500,001 - 750,000)";
        } else if (netIncome <= 1000000) {
            tax = (150000 * 0.05) + (200000 * 0.10) + (250000 * 0.15) + (netIncome - 750000) * 0.20;
            taxBracket = "20% (750,001 - 1,000,000)";
        } else if (netIncome <= 2000000) {
            tax = (150000 * 0.05) + (200000 * 0.10) + (250000 * 0.15) + (250000 * 0.20) + (netIncome - 1000000) * 0.25;
            taxBracket = "25% (1,000,001 - 2,000,000)";
        } else {
            tax = (150000 * 0.05) + (200000 * 0.10) + (250000 * 0.15) + (250000 * 0.20) + (1000000 * 0.25) + (netIncome - 2000000) * 0.30;
            taxBracket = "30% (มากกว่า 2,000,000)";
        }
        
        System.out.println("\n════════════════════════════════════════");
        System.out.printf("รายได้รวม:          %,15.2f บาท%n", income);
        System.out.printf("หักค่าใช้จ่าย:       %,15.2f บาท%n", expenses);
        System.out.printf("หักค่าลดหย่อนส่วนตัว: %,15.2f บาท%n", personalDeduction);
        System.out.printf("รายได้สุทธิ:         %,15.2f บาท%n", netIncome);
        System.out.println("ขั้นภาษี: " + taxBracket);
        System.out.printf("ภาษีที่ต้องชำระ:      %,15.2f บาท%n", tax);
        System.out.printf("Effective rate:     %15.2f%%%n", 
            income > 0 ? (tax / income) * 100 : 0);
        System.out.println("════════════════════════════════════════");
        
        sc.close();
    }
}
```

### ตัวอย่าง 2: เกม Rock-Paper-Scissors

```java
import java.util.Scanner;
import java.util.Random;

public class RockPaperScissors {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        Random random = new Random();
        
        int playerWins = 0, computerWins = 0, draws = 0;
        
        System.out.println("╔══════════════════════════════╗");
        System.out.println("║   เกม Rock-Paper-Scissors   ║");
        System.out.println("╚══════════════════════════════╝");
        
        boolean playing = true;
        while (playing) {
            System.out.println("\n1 = Rock (กำปั้น)");
            System.out.println("2 = Paper (กระดาษ)");
            System.out.println("3 = Scissors (กรรไกร)");
            System.out.println("0 = ออกจากเกม");
            System.out.print("เลือก: ");
            
            int playerChoice = sc.nextInt();
            
            if (playerChoice == 0) {
                playing = false;
                continue;
            }
            
            if (playerChoice < 1 || playerChoice > 3) {
                System.out.println("กรุณาเลือก 1-3");
                continue;
            }
            
            int computerChoice = random.nextInt(3) + 1;
            
            String playerName = switch (playerChoice) {
                case 1 -> "Rock 🪨";
                case 2 -> "Paper 📄";
                case 3 -> "Scissors ✂️";
                default -> "Unknown";
            };
            
            String computerName = switch (computerChoice) {
                case 1 -> "Rock 🪨";
                case 2 -> "Paper 📄";
                case 3 -> "Scissors ✂️";
                default -> "Unknown";
            };
            
            System.out.println("\nคุณเลือก: " + playerName);
            System.out.println("คอมพิวเตอร์เลือก: " + computerName);
            
            // ตรวจสอบผล
            String result;
            if (playerChoice == computerChoice) {
                result = "เสมอ!";
                draws++;
            } else if ((playerChoice == 1 && computerChoice == 3) ||
                       (playerChoice == 2 && computerChoice == 1) ||
                       (playerChoice == 3 && computerChoice == 2)) {
                result = "คุณชนะ! 🎉";
                playerWins++;
            } else {
                result = "คอมพิวเตอร์ชนะ! 😢";
                computerWins++;
            }
            
            System.out.println("ผล: " + result);
            System.out.printf("สกอร์ - คุณ: %d | คอมพิวเตอร์: %d | เสมอ: %d%n",
                playerWins, computerWins, draws);
        }
        
        // สรุปผล
        System.out.println("\n╔══════════════════════════════╗");
        System.out.println("║         สรุปผลการเล่น        ║");
        System.out.println("╚══════════════════════════════╝");
        System.out.printf("คุณชนะ: %d ครั้ง%n", playerWins);
        System.out.printf("คอมพิวเตอร์ชนะ: %d ครั้ง%n", computerWins);
        System.out.printf("เสมอ: %d ครั้ง%n", draws);
        
        int totalGames = playerWins + computerWins + draws;
        if (totalGames > 0) {
            if (playerWins > computerWins) {
                System.out.println("ผู้ชนะโดยรวม: คุณ! 🏆");
            } else if (computerWins > playerWins) {
                System.out.println("ผู้ชนะโดยรวม: คอมพิวเตอร์ 🤖");
            } else {
                System.out.println("ผลโดยรวม: เสมอ!");
            }
        }
        
        sc.close();
    }
}
```

### ตัวอย่าง 3: ระบบ ATM จำลอง

```java
import java.util.Scanner;

public class ATMSimulation {
    static double balance = 15000.00;
    static String pin = "1234";
    static int wrongAttempts = 0;
    static final int MAX_WRONG_ATTEMPTS = 3;
    
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.println("╔═══════════════════════════╗");
        System.out.println("║       ตู้ ATM จำลอง       ║");
        System.out.println("╚═══════════════════════════╝");
        
        // Authentication
        if (!authenticate(sc)) {
            System.out.println("บัตรถูกระงับเนื่องจากกรอก PIN ผิดเกินกำหนด");
            sc.close();
            return;
        }
        
        // Main menu
        boolean running = true;
        while (running) {
            System.out.println("\n═══════════════════════════");
            System.out.println("1. ตรวจสอบยอดเงิน");
            System.out.println("2. ถอนเงิน");
            System.out.println("3. ฝากเงิน");
            System.out.println("4. โอนเงิน");
            System.out.println("5. ออกจากระบบ");
            System.out.print("เลือกรายการ: ");
            
            int choice = sc.nextInt();
            
            switch (choice) {
                case 1 -> checkBalance();
                case 2 -> withdraw(sc);
                case 3 -> deposit(sc);
                case 4 -> transfer(sc);
                case 5 -> {
                    System.out.println("\nขอบคุณที่ใช้บริการ กรุณานำบัตรออก");
                    running = false;
                }
                default -> System.out.println("กรุณาเลือก 1-5");
            }
        }
        
        sc.close();
    }
    
    static boolean authenticate(Scanner sc) {
        while (wrongAttempts < MAX_WRONG_ATTEMPTS) {
            System.out.print("กรุณาใส่ PIN (4 หลัก): ");
            String inputPin = sc.next();
            
            if (inputPin.equals(pin)) {
                System.out.println("ยืนยันตัวตนสำเร็จ");
                return true;
            } else {
                wrongAttempts++;
                int remaining = MAX_WRONG_ATTEMPTS - wrongAttempts;
                if (remaining > 0) {
                    System.out.println("PIN ไม่ถูกต้อง เหลืออีก " + remaining + " ครั้ง");
                }
            }
        }
        return false;
    }
    
    static void checkBalance() {
        System.out.println("\n── ยอดเงินคงเหลือ ──");
        System.out.printf("%.2f บาท%n", balance);
    }
    
    static void withdraw(Scanner sc) {
        System.out.print("จำนวนเงินที่ต้องการถอน: ");
        double amount = sc.nextDouble();
        
        if (amount <= 0) {
            System.out.println("จำนวนเงินไม่ถูกต้อง");
            return;
        }
        
        if (amount % 100 != 0) {
            System.out.println("กรุณาถอนเงินเป็นจำนวนเท่าของ 100 บาท");
            return;
        }
        
        if (amount > 50000) {
            System.out.println("ถอนได้สูงสุด 50,000 บาทต่อครั้ง");
            return;
        }
        
        if (balance - amount < 1000) {
            System.out.println("ยอดเงินไม่เพียงพอ (ต้องคงเหลือ 1,000 บาท)");
            return;
        }
        
        balance -= amount;
        System.out.printf("ถอนเงิน %.2f บาท สำเร็จ%n", amount);
        System.out.printf("ยอดเงินคงเหลือ: %.2f บาท%n", balance);
    }
    
    static void deposit(Scanner sc) {
        System.out.print("จำนวนเงินที่ต้องการฝาก: ");
        double amount = sc.nextDouble();
        
        if (amount <= 0) {
            System.out.println("จำนวนเงินไม่ถูกต้อง");
            return;
        }
        
        if (amount > 500000) {
            System.out.println("ฝากได้สูงสุด 500,000 บาทต่อครั้ง");
            return;
        }
        
        balance += amount;
        System.out.printf("ฝากเงิน %.2f บาท สำเร็จ%n", amount);
        System.out.printf("ยอดเงินคงเหลือ: %.2f บาท%n", balance);
    }
    
    static void transfer(Scanner sc) {
        System.out.print("บัญชีปลายทาง: ");
        String targetAccount = sc.next();
        
        System.out.print("จำนวนเงินที่ต้องการโอน: ");
        double amount = sc.nextDouble();
        
        if (amount <= 0) {
            System.out.println("จำนวนเงินไม่ถูกต้อง");
            return;
        }
        
        if (balance - amount < 0) {
            System.out.println("ยอดเงินไม่เพียงพอ");
            return;
        }
        
        balance -= amount;
        System.out.printf("โอนเงิน %.2f บาท ไปยัง %s สำเร็จ%n", amount, targetAccount);
        System.out.printf("ยอดเงินคงเหลือ: %.2f บาท%n", balance);
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ระบบเกรดนักเรียน
สร้างโปรแกรมรับคะแนน 3 วิชา แล้วคำนวณคะแนนเฉลี่ยและแสดงเกรด

```java
// เฉลย
import java.util.Scanner;

public class StudentGradeSystem {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.println("=== ระบบคำนวณเกรด ===");
        System.out.print("ชื่อนักเรียน: ");
        String name = sc.nextLine();
        
        System.out.print("คะแนนวิชาคณิตศาสตร์ (0-100): ");
        double math = sc.nextDouble();
        System.out.print("คะแนนวิชาภาษาอังกฤษ (0-100): ");
        double english = sc.nextDouble();
        System.out.print("คะแนนวิชาวิทยาศาสตร์ (0-100): ");
        double science = sc.nextDouble();
        
        double average = (math + english + science) / 3;
        
        String grade;
        String status;
        
        if (average >= 80) {
            grade = "A";
            status = "ผ่านด้วยคะแนนดีเยี่ยม";
        } else if (average >= 70) {
            grade = "B";
            status = "ผ่านด้วยคะแนนดี";
        } else if (average >= 60) {
            grade = "C";
            status = "ผ่าน";
        } else if (average >= 50) {
            grade = "D";
            status = "ผ่านขั้นต่ำ";
        } else {
            grade = "F";
            status = "ไม่ผ่าน";
        }
        
        System.out.println("\n=== ผลการเรียน ===");
        System.out.println("ชื่อ: " + name);
        System.out.printf("คณิตศาสตร์: %.1f%n", math);
        System.out.printf("ภาษาอังกฤษ: %.1f%n", english);
        System.out.printf("วิทยาศาสตร์: %.1f%n", science);
        System.out.printf("คะแนนเฉลี่ย: %.2f%n", average);
        System.out.println("เกรด: " + grade);
        System.out.println("สถานะ: " + status);
        
        sc.close();
    }
}
```

### แบบฝึกหัดที่ 2: โปรแกรมแปลงเดือน
สร้างโปรแกรมรับเลขเดือน 1-12 แสดงชื่อเดือนทั้งไทยและอังกฤษ พร้อมจำนวนวัน

```java
// เฉลย
import java.util.Scanner;

public class MonthConverter {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.print("ใส่เลขเดือน (1-12): ");
        int month = sc.nextInt();
        
        if (month < 1 || month > 12) {
            System.out.println("เดือนไม่ถูกต้อง");
            sc.close();
            return;
        }
        
        String thaiName = switch (month) {
            case 1 -> "มกราคม";
            case 2 -> "กุมภาพันธ์";
            case 3 -> "มีนาคม";
            case 4 -> "เมษายน";
            case 5 -> "พฤษภาคม";
            case 6 -> "มิถุนายน";
            case 7 -> "กรกฎาคม";
            case 8 -> "สิงหาคม";
            case 9 -> "กันยายน";
            case 10 -> "ตุลาคม";
            case 11 -> "พฤศจิกายน";
            case 12 -> "ธันวาคม";
            default -> "";
        };
        
        String engName = switch (month) {
            case 1 -> "January";
            case 2 -> "February";
            case 3 -> "March";
            case 4 -> "April";
            case 5 -> "May";
            case 6 -> "June";
            case 7 -> "July";
            case 8 -> "August";
            case 9 -> "September";
            case 10 -> "October";
            case 11 -> "November";
            case 12 -> "December";
            default -> "";
        };
        
        int days = switch (month) {
            case 1, 3, 5, 7, 8, 10, 12 -> 31;
            case 4, 6, 9, 11 -> 30;
            case 2 -> 28;
            default -> 0;
        };
        
        System.out.println("\nเดือนที่ " + month + ":");
        System.out.println("ภาษาไทย: " + thaiName);
        System.out.println("ภาษาอังกฤษ: " + engName);
        System.out.println("จำนวนวัน: " + days + " วัน");
        
        sc.close();
    }
}
```

### แบบฝึกหัดที่ 3: ระบบลดราคา
สร้างโปรแกรมคำนวณราคาสินค้าหลังลดราคาตามเงื่อนไข:
- สมาชิกทอง: ลด 20%
- สมาชิกเงิน: ลด 15%
- สมาชิกทองแดง: ลด 10%
- ซื้อครบ 1000: ลดเพิ่ม 5%
- ใส่ coupon "SAVE50": ลดเพิ่ม 50 บาท

```java
// เฉลย
import java.util.Scanner;

public class DiscountSystem {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        
        System.out.println("=== ระบบคำนวณราคา ===");
        System.out.print("ราคาสินค้า: ");
        double price = sc.nextDouble();
        
        System.out.println("ระดับสมาชิก (1=ทอง, 2=เงิน, 3=ทองแดง, 4=ทั่วไป): ");
        int memberLevel = sc.nextInt();
        
        System.out.print("Coupon code (ไม่มีใส่ -): ");
        String coupon = sc.next();
        
        double memberDiscount = switch (memberLevel) {
            case 1 -> 0.20;
            case 2 -> 0.15;
            case 3 -> 0.10;
            default -> 0.0;
        };
        
        double discountAmount = price * memberDiscount;
        double priceAfterMember = price - discountAmount;
        
        // ลดเพิ่มถ้าซื้อครบ 1000
        double bulkDiscount = priceAfterMember >= 1000 ? priceAfterMember * 0.05 : 0;
        double priceAfterBulk = priceAfterMember - bulkDiscount;
        
        // Coupon
        double couponDiscount = coupon.equalsIgnoreCase("SAVE50") ? 50 : 0;
        double finalPrice = Math.max(0, priceAfterBulk - couponDiscount);
        
        System.out.println("\n=== สรุปการคำนวณ ===");
        System.out.printf("ราคาเดิม:           %8.2f บาท%n", price);
        System.out.printf("ส่วนลดสมาชิก (%.0f%%): %8.2f บาท%n", memberDiscount * 100, discountAmount);
        System.out.printf("ส่วนลดซื้อครบ (5%%): %8.2f บาท%n", bulkDiscount);
        System.out.printf("ส่วนลด Coupon:      %8.2f บาท%n", couponDiscount);
        System.out.println("─".repeat(35));
        System.out.printf("ราคาสุทธิ:          %8.2f บาท%n", finalPrice);
        System.out.printf("ประหยัดทั้งหมด:      %8.2f บาท%n", price - finalPrice);
        
        sc.close();
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ if Statement พื้นฐาน  
✅ if-else Statement  
✅ if-else-if Ladder  
✅ Nested if  
✅ switch Statement แบบดั้งเดิม  
✅ switch Expression (Java 14+) แบบ Arrow  
✅ Pattern Matching ใน switch (Java 21)  
✅ Guard Clauses สำหรับโค้ดสะอาด  
✅ ตัวอย่างโปรแกรมจริง  

---

## ขั้นตอนต่อไป

**Part 004:** Loops - for, while, do-while  
เราจะเรียนรู้:
- for loop
- while loop
- do-while loop
- for-each loop
- break และ continue
- Nested loops

---

*Part 003 | Java & Spring Boot Course | สร้างโดย Claude Code*
