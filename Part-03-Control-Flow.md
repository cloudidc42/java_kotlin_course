# Part 03: Control Flow - if/else, switch, และการควบคุมโปรแกรม
## ขั้นตอนที่ 121-190: การตัดสินใจในโปรแกรม

---

## 3.1 if Statement

```java
public class IfStatement {
    
    public static void main(String[] args) {
        
        // ====== Simple if ======
        int age = 20;
        
        if (age >= 18) {
            System.out.println("เป็นผู้ใหญ่แล้ว");
        }
        
        // if แบบบรรทัดเดียว (ไม่แนะนำสำหรับโค้ดซับซ้อน)
        if (age >= 18) System.out.println("ผ่านแล้ว");
        
        // ====== if-else ======
        int score = 75;
        
        if (score >= 60) {
            System.out.println("ผ่าน ✓");
        } else {
            System.out.println("ไม่ผ่าน ✗");
        }
        
        // ====== if-else if-else ======
        int grade = 85;
        
        if (grade >= 90) {
            System.out.println("เกรด A");
        } else if (grade >= 80) {
            System.out.println("เกรด B");
        } else if (grade >= 70) {
            System.out.println("เกรด C");
        } else if (grade >= 60) {
            System.out.println("เกรด D");
        } else {
            System.out.println("เกรด F");
        }
        
        // ====== Nested if ======
        int temp = 28;
        boolean isRaining = false;
        
        if (temp >= 25) {
            System.out.println("อากาศร้อน");
            if (isRaining) {
                System.out.println("และฝนตก - อยู่บ้านดีกว่า");
            } else {
                System.out.println("แต่ไม่มีฝน - เดินเล่นได้");
            }
        } else {
            System.out.println("อากาศเย็นสบาย");
        }
    }
}
```

---

## 3.2 Condition Patterns ที่พบบ่อย

```java
public class ConditionPatterns {
    
    public static void main(String[] args) {
        
        // ====== Range Check ======
        int score = 75;
        if (score >= 0 && score <= 100) {
            System.out.println("คะแนนถูกต้อง");
        }
        
        // ====== Null Check (ป้องกัน NullPointerException) ======
        String name = null;
        
        // ❌ อาจ throw NullPointerException
        // if (name.equals("John")) { ... }
        
        // ✅ ตรวจ null ก่อนเสมอ
        if (name != null && name.equals("John")) {
            System.out.println("พบ John");
        }
        
        // ✅ วิธีที่ดีกว่า - ใช้ literal ไว้ข้างหน้า
        if ("John".equals(name)) {
            System.out.println("พบ John");
        }
        
        // ====== Empty String Check ======
        String input = "";
        
        if (input == null || input.isEmpty()) {
            System.out.println("ไม่มีข้อมูล");
        }
        
        // Java 11+
        if (input == null || input.isBlank()) {
            System.out.println("ไม่มีข้อมูล (รวม whitespace)");
        }
        
        // ====== Multiple Conditions ======
        int age = 25;
        String membership = "GOLD";
        double balance = 5000;
        
        boolean canGetDiscount = (age >= 60) || 
                                  (membership.equals("GOLD") && balance >= 1000) ||
                                  (membership.equals("PLATINUM"));
        
        if (canGetDiscount) {
            System.out.println("ได้รับส่วนลด 20%");
        }
        
        // ====== Boolean Flag Pattern ======
        boolean isLoggedIn = true;
        boolean hasPermission = false;
        boolean isAdmin = false;
        
        if (isLoggedIn) {
            if (isAdmin || hasPermission) {
                System.out.println("สามารถเข้าถึงข้อมูลลับได้");
            } else {
                System.out.println("ไม่มีสิทธิ์เข้าถึงข้อมูลลับ");
            }
        } else {
            System.out.println("กรุณาเข้าสู่ระบบก่อน");
        }
    }
}
```

---

## 3.3 switch Statement

```java
public class SwitchStatement {
    
    public static void main(String[] args) {
        
        // ====== Basic switch (int) ======
        int day = 3;
        
        switch (day) {
            case 1:
                System.out.println("วันจันทร์");
                break;
            case 2:
                System.out.println("วันอังคาร");
                break;
            case 3:
                System.out.println("วันพุธ");
                break;
            case 4:
                System.out.println("วันพฤหัสบดี");
                break;
            case 5:
                System.out.println("วันศุกร์");
                break;
            case 6:
                System.out.println("วันเสาร์");
                break;
            case 7:
                System.out.println("วันอาทิตย์");
                break;
            default:
                System.out.println("ไม่ใช่วันที่ถูกต้อง");
        }
        
        // ====== switch กับ String ======
        String season = "summer";
        
        switch (season) {
            case "spring":
                System.out.println("ฤดูใบไม้ผลิ - อากาศสดชื่น");
                break;
            case "summer":
                System.out.println("ฤดูร้อน - อากาศร้อนมาก");
                break;
            case "autumn":
                System.out.println("ฤดูใบไม้ร่วง - ใบไม้สีแดง");
                break;
            case "winter":
                System.out.println("ฤดูหนาว - อากาศหนาว");
                break;
            default:
                System.out.println("ฤดูไม่ถูกต้อง");
        }
        
        // ====== Fall-through (ไม่มี break) ======
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
                daysInMonth = 28; // ข้ามปีอธิกสุรทิน
                break;
            default:
                daysInMonth = -1;
        }
        System.out.println("เดือน " + month + " มี " + daysInMonth + " วัน");
        
        // ====== switch กับ enum ======
        enum TrafficLight { RED, YELLOW, GREEN }
        TrafficLight light = TrafficLight.GREEN;
        
        switch (light) {
            case RED:
                System.out.println("หยุด");
                break;
            case YELLOW:
                System.out.println("เตรียมพร้อม");
                break;
            case GREEN:
                System.out.println("ไปได้");
                break;
        }
    }
}
```

### Switch Expression (Java 14+)

```java
public class SwitchExpression {
    
    public static void main(String[] args) {
        
        // ====== Switch Expression (Java 14+) ======
        // ส่งกลับค่าได้และไม่ต้องใช้ break
        
        int day = 3;
        
        // แบบ arrow (->) ง่ายและสั้น
        String dayName = switch (day) {
            case 1 -> "วันจันทร์";
            case 2 -> "วันอังคาร";
            case 3 -> "วันพุธ";
            case 4 -> "วันพฤหัสบดี";
            case 5 -> "วันศุกร์";
            case 6 -> "วันเสาร์";
            case 7 -> "วันอาทิตย์";
            default -> "ไม่ถูกต้อง";
        };
        System.out.println("วัน: " + dayName);
        
        // หลาย case ในบรรทัดเดียว
        String typeOfDay = switch (day) {
            case 1, 2, 3, 4, 5 -> "วันทำงาน";
            case 6, 7 -> "วันหยุด";
            default -> "ไม่ถูกต้อง";
        };
        System.out.println("ประเภท: " + typeOfDay);
        
        // ใช้ yield สำหรับ block
        int month = 4;
        int daysInMonth = switch (month) {
            case 1, 3, 5, 7, 8, 10, 12 -> 31;
            case 4, 6, 9, 11 -> 30;
            case 2 -> {
                int year = 2024;
                if (year % 4 == 0 && (year % 100 != 0 || year % 400 == 0)) {
                    yield 29; // ปีอธิกสุรทิน
                } else {
                    yield 28;
                }
            }
            default -> throw new IllegalArgumentException("เดือนไม่ถูกต้อง: " + month);
        };
        System.out.println("เดือน " + month + " มี " + daysInMonth + " วัน");
        
        // Pattern matching in switch (Java 21+)
        Object obj = "Hello";
        String result = switch (obj) {
            case Integer i -> "Integer: " + i;
            case String s -> "String: " + s;
            case Double d -> "Double: " + d;
            case null -> "null value";
            default -> "Unknown: " + obj;
        };
        System.out.println(result);
    }
}
```

---

## 3.4 Conditional Logic ขั้นสูง

```java
public class AdvancedConditional {
    
    public static void main(String[] args) {
        
        // ====== Guard Clauses (Early Return Pattern) ======
        // แทนที่จะ nested if หลายชั้น ใช้ early return
        
        // ❌ แบบเดิม (ซับซ้อน)
        String processOrderBad(String order, String customer, int quantity) {
            // ไม่ดี - nested มากเกินไป
            if (order != null) {
                if (customer != null) {
                    if (quantity > 0) {
                        return "Order processed: " + order;
                    }
                }
            }
            return "Error";
        }
        
        // ✅ แบบ Guard Clauses (อ่านง่ายกว่า)
        processOrder("ORDER001", "Customer1", 5);
    }
    
    static String processOrder(String order, String customer, int quantity) {
        // Guard clauses - เช็คเงื่อนไขผิดปกติก่อน
        if (order == null) return "Error: order ไม่มีค่า";
        if (customer == null) return "Error: customer ไม่มีค่า";
        if (quantity <= 0) return "Error: จำนวนต้องมากกว่า 0";
        if (quantity > 100) return "Error: จำนวนมากเกินไป";
        
        // Main logic
        return "Order processed: " + order + " for " + customer + " qty: " + quantity;
    }
    
    // ====== Complex Business Logic ======
    static double calculateDiscount(double price, String memberType, int age, boolean isNewUser) {
        double discount = 0;
        
        // Member type discount
        switch (memberType) {
            case "PLATINUM" -> discount += 0.20;
            case "GOLD" -> discount += 0.15;
            case "SILVER" -> discount += 0.10;
            default -> discount += 0.05;
        }
        
        // Age discount
        if (age >= 60) {
            discount += 0.10; // ผู้สูงอายุ
        } else if (age <= 12) {
            discount += 0.05; // เด็ก
        }
        
        // New user bonus
        if (isNewUser) {
            discount += 0.05;
        }
        
        // Max discount cap
        discount = Math.min(discount, 0.50); // max 50%
        
        return price * discount;
    }
}
```

---

## 3.5 Pattern Matching for instanceof (Java 16+)

```java
public class PatternMatching {
    
    public static void main(String[] args) {
        
        Object obj = "Hello World";
        
        // ❌ แบบเดิม
        if (obj instanceof String) {
            String s = (String) obj; // ต้อง cast เอง
            System.out.println("Length: " + s.length());
        }
        
        // ✅ Pattern Matching (Java 16+)
        if (obj instanceof String s) {
            // s ถูก declare และ cast อัตโนมัติ
            System.out.println("Length: " + s.length());
            System.out.println("Upper: " + s.toUpperCase());
        }
        
        // ใช้กับ condition ร่วมด้วย
        if (obj instanceof String s && s.length() > 5) {
            System.out.println("Long string: " + s);
        }
        
        // ====== Polymorphic handling ======
        Object[] objects = {42, "Hello", 3.14, true, null};
        
        for (Object o : objects) {
            String description = describe(o);
            System.out.println(description);
        }
    }
    
    static String describe(Object obj) {
        if (obj instanceof Integer i) {
            return "Integer: " + i + " (even=" + (i % 2 == 0) + ")";
        } else if (obj instanceof String s) {
            return "String: \"" + s + "\" (length=" + s.length() + ")";
        } else if (obj instanceof Double d) {
            return "Double: " + d;
        } else if (obj instanceof Boolean b) {
            return "Boolean: " + b;
        } else if (obj == null) {
            return "null";
        } else {
            return "Unknown: " + obj.getClass().getSimpleName();
        }
    }
}
```

---

## 3.6 Sealed Classes and Pattern Matching (Java 17+)

```java
// Shape.java
sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}

record Triangle(double base, double height) implements Shape {
    public double area() { return 0.5 * base * height; }
}

public class SealedClasses {
    
    public static void main(String[] args) {
        Shape[] shapes = {
            new Circle(5),
            new Rectangle(4, 6),
            new Triangle(3, 8)
        };
        
        for (Shape shape : shapes) {
            // Switch expression กับ Pattern Matching (Java 21+)
            String description = switch (shape) {
                case Circle c -> String.format("วงกลม รัศมี=%.1f พื้นที่=%.2f", c.radius(), c.area());
                case Rectangle r -> String.format("สี่เหลี่ยม %.1f×%.1f พื้นที่=%.2f", r.width(), r.height(), r.area());
                case Triangle t -> String.format("สามเหลี่ยม ฐาน=%.1f สูง=%.1f พื้นที่=%.2f", t.base(), t.height(), t.area());
            };
            System.out.println(description);
        }
    }
}
```

---

## 3.7 โปรแกรมตัวอย่าง: Simple ATM Machine

```java
import java.util.Scanner;

public class ATMMachine {
    
    private static double balance = 10000.0;
    private static final int PIN = 1234;
    private static final int MAX_WITHDRAWAL = 50000;
    private static final int MAX_ATTEMPTS = 3;
    
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("╔══════════════════════════╗");
        System.out.println("║      ตู้ ATM              ║");
        System.out.println("╚══════════════════════════╝");
        
        // PIN Verification
        if (!verifyPIN(scanner)) {
            System.out.println("บัญชีถูกล็อก กรุณาติดต่อธนาคาร");
            scanner.close();
            return;
        }
        
        System.out.println("\nเข้าสู่ระบบสำเร็จ!");
        
        boolean running = true;
        while (running) {
            System.out.println("\n========= เมนู =========");
            System.out.println("1. ตรวจสอบยอดเงิน");
            System.out.println("2. ฝากเงิน");
            System.out.println("3. ถอนเงิน");
            System.out.println("4. ออกจากระบบ");
            System.out.print("เลือก: ");
            
            int choice = scanner.nextInt();
            
            switch (choice) {
                case 1:
                    checkBalance();
                    break;
                case 2:
                    deposit(scanner);
                    break;
                case 3:
                    withdraw(scanner);
                    break;
                case 4:
                    System.out.println("ขอบคุณที่ใช้บริการ!");
                    running = false;
                    break;
                default:
                    System.out.println("กรุณาเลือก 1-4");
            }
        }
        
        scanner.close();
    }
    
    private static boolean verifyPIN(Scanner scanner) {
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            System.out.printf("ใส่ PIN (%d/%d): ", attempt, MAX_ATTEMPTS);
            int enteredPIN = scanner.nextInt();
            
            if (enteredPIN == PIN) {
                return true;
            }
            
            if (attempt < MAX_ATTEMPTS) {
                System.out.println("PIN ไม่ถูกต้อง ลองใหม่อีกครั้ง");
            }
        }
        return false;
    }
    
    private static void checkBalance() {
        System.out.printf("ยอดเงินคงเหลือ: ฿%,.2f%n", balance);
    }
    
    private static void deposit(Scanner scanner) {
        System.out.print("จำนวนเงินที่ต้องการฝาก: ฿");
        double amount = scanner.nextDouble();
        
        if (amount <= 0) {
            System.out.println("จำนวนเงินต้องมากกว่า 0");
            return;
        }
        
        balance += amount;
        System.out.printf("ฝากเงินสำเร็จ ยอดเงินใหม่: ฿%,.2f%n", balance);
    }
    
    private static void withdraw(Scanner scanner) {
        System.out.print("จำนวนเงินที่ต้องการถอน: ฿");
        double amount = scanner.nextDouble();
        
        if (amount <= 0) {
            System.out.println("จำนวนเงินต้องมากกว่า 0");
        } else if (amount > MAX_WITHDRAWAL) {
            System.out.printf("ไม่สามารถถอนเกิน ฿%,d ต่อครั้ง%n", MAX_WITHDRAWAL);
        } else if (amount > balance) {
            System.out.println("ยอดเงินไม่เพียงพอ");
        } else {
            balance -= amount;
            System.out.printf("ถอนเงินสำเร็จ ยอดเงินคงเหลือ: ฿%,.2f%n", balance);
        }
    }
}
```

---

## 3.8 Error-Prone Patterns และการหลีกเลี่ยง

```java
public class CommonMistakes {
    
    public static void main(String[] args) {
        
        // ====== Mistake 1: Comparing String with == ======
        String s1 = new String("Hello");
        String s2 = new String("Hello");
        
        // ❌ ผิด
        if (s1 == s2) {
            System.out.println("เหมือนกัน (ผิด!)");
        }
        
        // ✅ ถูก
        if (s1.equals(s2)) {
            System.out.println("เหมือนกัน (ถูก!)");
        }
        
        // ====== Mistake 2: Assignment ใน Condition ======
        int x = 5;
        
        // ❌ อาจพิมพ์ผิด (= แทน ==) - Java compiler จะเตือน
        // if (x = 5) { ... } // Error in Java
        
        // ✅ ถูก
        if (x == 5) {
            System.out.println("x เท่ากับ 5");
        }
        
        // Yoda Conditions (ป้องกันการพิมพ์ผิด)
        if (5 == x) { // ถ้าพิมพ์ 5 = x จะ error ทันที
            System.out.println("x เท่ากับ 5");
        }
        
        // ====== Mistake 3: Floating Point Comparison ======
        double a = 0.1 + 0.2;
        double b = 0.3;
        
        // ❌ ผิด
        if (a == b) {
            System.out.println("เท่ากัน (ผิด!)");
        }
        
        // ✅ ถูก - ใช้ epsilon
        double epsilon = 1e-9;
        if (Math.abs(a - b) < epsilon) {
            System.out.println("เท่ากัน (ถูก!)");
        }
        
        // ====== Mistake 4: Integer Overflow ======
        int maxInt = Integer.MAX_VALUE;
        int overflow = maxInt + 1;  // overflow!
        System.out.println("MAX_INT + 1 = " + overflow);  // -2147483648
        
        // ✅ ถูก - ใช้ long
        long safeValue = (long)maxInt + 1;
        System.out.println("Safe: " + safeValue);
        
        // ====== Mistake 5: Null Check ======
        String name = null;
        
        // ❌ อาจ NullPointerException
        // if (name.length() > 0) { ... }
        
        // ✅ ถูก
        if (name != null && name.length() > 0) {
            System.out.println("Name: " + name);
        }
        
        // ✅ หรือใช้ Optional (Java 8+)
        java.util.Optional<String> optName = java.util.Optional.ofNullable(name);
        optName.ifPresent(n -> System.out.println("Name: " + n));
    }
}
```

---

## 3.9 Conditional Expressions ขั้นสูง

```java
public class AdvancedExpressions {
    
    public static void main(String[] args) {
        
        // ====== Method Chaining ======
        String result = "  Hello, World!  "
            .trim()
            .toLowerCase()
            .replace("hello", "hi")
            .replace("world", "java");
        System.out.println(result); // "hi, java!"
        
        // ====== Conditional Assignment Patterns ======
        
        // Null Coalescing Pattern
        String username = null;
        String displayName = (username != null) ? username : "Guest";
        System.out.println("Display name: " + displayName);
        
        // Java 9+: Objects.requireNonNullElse
        String safeName = java.util.Objects.requireNonNullElse(username, "Guest");
        System.out.println("Safe name: " + safeName);
        
        // ====== Complex Business Rules ======
        int[] prices = {100, 250, 500, 1000, 2500};
        int totalPurchase = 0;
        
        for (int price : prices) {
            totalPurchase += price;
        }
        
        // Tiered discount
        double discountRate;
        if (totalPurchase >= 5000) {
            discountRate = 0.20;
        } else if (totalPurchase >= 3000) {
            discountRate = 0.15;
        } else if (totalPurchase >= 1000) {
            discountRate = 0.10;
        } else {
            discountRate = 0.05;
        }
        
        double discount = totalPurchase * discountRate;
        double finalPrice = totalPurchase - discount;
        
        System.out.printf("ยอดซื้อ: ฿%,d%n", totalPurchase);
        System.out.printf("ส่วนลด (%.0f%%): ฿%.2f%n", discountRate * 100, discount);
        System.out.printf("ราคาสุทธิ: ฿%.2f%n", finalPrice);
    }
}
```

---

## 3.10 แบบฝึกหัด Part 03

### แบบฝึกหัดที่ 1: เกรดนักศึกษา
```java
import java.util.Scanner;

public class GradeSystem {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("ใส่คะแนน (0-100): ");
        int score = scanner.nextInt();
        
        if (score < 0 || score > 100) {
            System.out.println("คะแนนไม่ถูกต้อง!");
        } else {
            String grade = (score >= 80) ? "A" :
                           (score >= 70) ? "B" :
                           (score >= 60) ? "C" :
                           (score >= 50) ? "D" : "F";
            
            String status = score >= 50 ? "ผ่าน ✓" : "ไม่ผ่าน ✗";
            
            System.out.println("คะแนน: " + score);
            System.out.println("เกรด: " + grade);
            System.out.println("สถานะ: " + status);
        }
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 2: Simple Menu System
```java
import java.util.Scanner;

public class MenuSystem {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("=== ร้านอาหาร ===");
        System.out.println("1. ข้าวผัด - 60 บาท");
        System.out.println("2. ผัดไทย - 70 บาท");
        System.out.println("3. ต้มยำ - 80 บาท");
        System.out.println("4. แกงเขียวหวาน - 75 บาท");
        System.out.print("เลือกเมนู: ");
        
        int choice = scanner.nextInt();
        
        String menuName;
        int price;
        
        switch (choice) {
            case 1:
                menuName = "ข้าวผัด";
                price = 60;
                break;
            case 2:
                menuName = "ผัดไทย";
                price = 70;
                break;
            case 3:
                menuName = "ต้มยำ";
                price = 80;
                break;
            case 4:
                menuName = "แกงเขียวหวาน";
                price = 75;
                break;
            default:
                System.out.println("ไม่มีเมนูนี้");
                scanner.close();
                return;
        }
        
        System.out.print("จำนวน: ");
        int qty = scanner.nextInt();
        
        int total = price * qty;
        System.out.printf("สั่ง: %s x%d = %d บาท%n", menuName, qty, total);
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 3: Fizz Buzz Classic
```java
public class FizzBuzz {
    public static void main(String[] args) {
        for (int i = 1; i <= 100; i++) {
            if (i % 15 == 0) {
                System.out.println("FizzBuzz");
            } else if (i % 3 == 0) {
                System.out.println("Fizz");
            } else if (i % 5 == 0) {
                System.out.println("Buzz");
            } else {
                System.out.println(i);
            }
        }
    }
}
```

---

## สรุป Part 03

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| if statement | Simple if, if-else, if-else if-else, nested if |
| switch statement | Traditional switch, Switch Expression (Java 14+) |
| Pattern Matching | instanceof patterns (Java 16+) |
| Sealed Classes | Exhaustive switch (Java 17+) |
| Best Practices | Guard clauses, null checks, float comparison |

---

## ขั้นตอนต่อไป

➡️ [Part 04: Loops - for, while, do-while](./Part-04-Loops.md)
