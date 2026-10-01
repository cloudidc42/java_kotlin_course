# Part 04: Loops - for, while, do-while และการวนซ้ำ
## ขั้นตอนที่ 191-260: การทำงานซ้ำในโปรแกรม

---

## 4.1 for Loop

```java
public class ForLoop {
    
    public static void main(String[] args) {
        
        // ====== Basic for Loop ======
        // for (initialization; condition; update)
        
        System.out.println("=== นับ 1-10 ===");
        for (int i = 1; i <= 10; i++) {
            System.out.print(i + " ");
        }
        System.out.println();
        
        // นับถอยหลัง
        System.out.println("=== นับถอยหลัง ===");
        for (int i = 10; i >= 1; i--) {
            System.out.print(i + " ");
        }
        System.out.println();
        
        // เพิ่มทีละ 2
        System.out.println("=== เลขคู่ 2-20 ===");
        for (int i = 2; i <= 20; i += 2) {
            System.out.print(i + " ");
        }
        System.out.println();
        
        // เลขคี่
        System.out.println("=== เลขคี่ 1-19 ===");
        for (int i = 1; i <= 20; i += 2) {
            System.out.print(i + " ");
        }
        System.out.println();
        
        // ====== for Loop กับ String ======
        String text = "Hello, Java!";
        System.out.println("=== ตัวอักษรทีละตัว ===");
        for (int i = 0; i < text.length(); i++) {
            System.out.print(text.charAt(i) + " ");
        }
        System.out.println();
        
        // ====== Multiple Variables ======
        System.out.println("=== Multiple Variables ===");
        for (int i = 0, j = 10; i <= 10 && j >= 0; i++, j--) {
            System.out.printf("i=%d, j=%d%n", i, j);
        }
    }
}
```

---

## 4.2 Enhanced for Loop (for-each)

```java
import java.util.ArrayList;
import java.util.List;

public class EnhancedForLoop {
    
    public static void main(String[] args) {
        
        // ====== Array ======
        int[] numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
        
        System.out.print("Numbers: ");
        for (int num : numbers) {
            System.out.print(num + " ");
        }
        System.out.println();
        
        // คำนวณผลรวม
        int sum = 0;
        for (int num : numbers) {
            sum += num;
        }
        System.out.println("Sum: " + sum);
        
        // ====== String Array ======
        String[] fruits = {"แอปเปิ้ล", "กล้วย", "ส้ม", "องุ่น", "มะม่วง"};
        
        System.out.println("=== ผลไม้ ===");
        for (String fruit : fruits) {
            System.out.println("- " + fruit);
        }
        
        // ====== 2D Array ======
        int[][] matrix = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };
        
        System.out.println("=== Matrix ===");
        for (int[] row : matrix) {
            for (int value : row) {
                System.out.printf("%3d", value);
            }
            System.out.println();
        }
        
        // ====== Collection (ArrayList) ======
        List<String> cities = new ArrayList<>();
        cities.add("กรุงเทพ");
        cities.add("เชียงใหม่");
        cities.add("ภูเก็ต");
        cities.add("ขอนแก่น");
        
        System.out.println("=== เมืองไทย ===");
        for (String city : cities) {
            System.out.println("🏙️ " + city);
        }
    }
}
```

---

## 4.3 while Loop

```java
import java.util.Scanner;

public class WhileLoop {
    
    public static void main(String[] args) {
        
        // ====== Basic while Loop ======
        // while (condition) { ... }
        
        int count = 1;
        System.out.print("นับ: ");
        while (count <= 10) {
            System.out.print(count + " ");
            count++;
        }
        System.out.println();
        
        // ====== Input Validation ======
        Scanner scanner = new Scanner(System.in);
        int age = -1;
        
        while (age < 0 || age > 150) {
            System.out.print("ใส่อายุ (0-150): ");
            age = scanner.nextInt();
            
            if (age < 0 || age > 150) {
                System.out.println("อายุไม่ถูกต้อง ลองใหม่");
            }
        }
        System.out.println("อายุ: " + age);
        
        // ====== Menu Loop ======
        boolean running = true;
        while (running) {
            System.out.println("\n=== เมนู ===");
            System.out.println("1. ทำอะไรบางอย่าง");
            System.out.println("2. ออก");
            System.out.print("เลือก: ");
            
            int choice = scanner.nextInt();
            
            switch (choice) {
                case 1:
                    System.out.println("ทำงาน...");
                    break;
                case 2:
                    running = false;
                    System.out.println("ลาก่อน!");
                    break;
                default:
                    System.out.println("เลือก 1 หรือ 2");
            }
        }
        
        // ====== Number Guessing Game ======
        int secret = (int)(Math.random() * 100) + 1;
        int guess = 0;
        int attempts = 0;
        
        System.out.println("\n=== เกมทายตัวเลข (1-100) ===");
        
        while (guess != secret) {
            System.out.print("ทาย: ");
            guess = scanner.nextInt();
            attempts++;
            
            if (guess < secret) {
                System.out.println("น้อยเกินไป!");
            } else if (guess > secret) {
                System.out.println("มากเกินไป!");
            } else {
                System.out.printf("ถูกต้อง! ทายถูกใน %d ครั้ง%n", attempts);
            }
        }
        
        scanner.close();
    }
}
```

---

## 4.4 do-while Loop

```java
import java.util.Scanner;

public class DoWhileLoop {
    
    public static void main(String[] args) {
        
        // ====== Basic do-while ======
        // ทำงานอย่างน้อย 1 ครั้งก่อนเช็คเงื่อนไข
        
        int i = 1;
        System.out.print("do-while: ");
        do {
            System.out.print(i + " ");
            i++;
        } while (i <= 10);
        System.out.println();
        
        // เปรียบเทียบ while vs do-while
        int x = 100; // เงื่อนไขไม่เป็นจริงตั้งแต่แรก
        
        // while - ไม่ทำงาน
        while (x < 10) {
            System.out.println("while: " + x);
            x++;
        }
        System.out.println("while ทำงาน 0 ครั้ง");
        
        // do-while - ทำงาน 1 ครั้ง
        do {
            System.out.println("do-while: " + x);
            x++;
        } while (x < 10);
        System.out.println("do-while ทำงาน 1 ครั้ง");
        
        // ====== Menu ที่แสดงก่อนตรวจ ======
        Scanner scanner = new Scanner(System.in);
        int choice;
        
        do {
            System.out.println("\n=== ร้านอาหาร ===");
            System.out.println("1. ข้าวผัด - 60฿");
            System.out.println("2. ผัดไทย - 70฿");
            System.out.println("3. ต้มยำ - 80฿");
            System.out.println("0. ออก");
            System.out.print("เลือก: ");
            choice = scanner.nextInt();
            
            switch (choice) {
                case 1: System.out.println("สั่งข้าวผัด"); break;
                case 2: System.out.println("สั่งผัดไทย"); break;
                case 3: System.out.println("สั่งต้มยำ"); break;
                case 0: System.out.println("ขอบคุณ!"); break;
                default: System.out.println("ไม่มีในเมนู");
            }
        } while (choice != 0);
        
        scanner.close();
    }
}
```

---

## 4.5 Nested Loops

```java
public class NestedLoops {
    
    public static void main(String[] args) {
        
        // ====== Multiplication Table ======
        System.out.println("=== ตารางสูตรคูณ ===");
        System.out.print("    ");
        for (int j = 1; j <= 10; j++) {
            System.out.printf("%4d", j);
        }
        System.out.println();
        System.out.println("    " + "─".repeat(40));
        
        for (int i = 1; i <= 10; i++) {
            System.out.printf("%3d│", i);
            for (int j = 1; j <= 10; j++) {
                System.out.printf("%4d", i * j);
            }
            System.out.println();
        }
        
        // ====== Star Patterns ======
        
        // รูปสี่เหลี่ยม
        System.out.println("\n=== สี่เหลี่ยม ===");
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 8; j++) {
                System.out.print("* ");
            }
            System.out.println();
        }
        
        // รูปสามเหลี่ยมขวา
        System.out.println("\n=== สามเหลี่ยมขวา ===");
        for (int i = 1; i <= 5; i++) {
            for (int j = 1; j <= i; j++) {
                System.out.print("* ");
            }
            System.out.println();
        }
        
        // รูปสามเหลี่ยมกลับหัว
        System.out.println("\n=== สามเหลี่ยมกลับหัว ===");
        for (int i = 5; i >= 1; i--) {
            for (int j = 1; j <= i; j++) {
                System.out.print("* ");
            }
            System.out.println();
        }
        
        // รูปสามเหลี่ยมกลาง
        System.out.println("\n=== สามเหลี่ยมกลาง ===");
        int rows = 7;
        for (int i = 1; i <= rows; i++) {
            // ช่องว่าง
            for (int j = rows - i; j > 0; j--) {
                System.out.print(" ");
            }
            // ดาว
            for (int j = 1; j <= (2 * i - 1); j++) {
                System.out.print("*");
            }
            System.out.println();
        }
        
        // รูปเพชร
        System.out.println("\n=== รูปเพชร ===");
        int n = 5;
        // ครึ่งบน
        for (int i = 1; i <= n; i++) {
            for (int j = n - i; j > 0; j--) System.out.print(" ");
            for (int j = 1; j <= (2 * i - 1); j++) System.out.print("*");
            System.out.println();
        }
        // ครึ่งล่าง
        for (int i = n - 1; i >= 1; i--) {
            for (int j = n - i; j > 0; j--) System.out.print(" ");
            for (int j = 1; j <= (2 * i - 1); j++) System.out.print("*");
            System.out.println();
        }
    }
}
```

---

## 4.6 break, continue, และ return

```java
public class LoopControl {
    
    public static void main(String[] args) {
        
        // ====== break - หยุดลูปทันที ======
        System.out.println("=== break ===");
        for (int i = 1; i <= 10; i++) {
            if (i == 6) {
                System.out.println("หยุดที่ " + i);
                break; // ออกจากลูปทันที
            }
            System.out.print(i + " ");
        }
        System.out.println();
        
        // ====== continue - ข้ามรอบนี้ ======
        System.out.println("\n=== continue ===");
        System.out.print("เลขคี่: ");
        for (int i = 1; i <= 10; i++) {
            if (i % 2 == 0) {
                continue; // ข้ามเลขคู่
            }
            System.out.print(i + " ");
        }
        System.out.println();
        
        // ====== break ใน nested loop ======
        System.out.println("\n=== break ใน nested loop ===");
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (j == 2) break; // ออกแค่ลูปใน
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        
        // ====== Labeled break ======
        System.out.println("\n=== Labeled break ===");
        outer:
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (i == 2 && j == 2) {
                    System.out.println("Break outer loop at i=" + i + " j=" + j);
                    break outer; // ออกจาก outer loop
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        
        // ====== Labeled continue ======
        System.out.println("\n=== Labeled continue ===");
        outer:
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (j == 2) {
                    continue outer; // ข้ามไปรอบถัดไปของ outer loop
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        
        // ====== Real-world example: Find prime numbers ======
        System.out.println("\n=== เลขเฉพาะ 1-50 ===");
        System.out.print("Primes: ");
        for (int num = 2; num <= 50; num++) {
            boolean isPrime = true;
            for (int divisor = 2; divisor <= Math.sqrt(num); divisor++) {
                if (num % divisor == 0) {
                    isPrime = false;
                    break; // หยุดหาตัวหาร
                }
            }
            if (isPrime) {
                System.out.print(num + " ");
            }
        }
        System.out.println();
    }
}
```

---

## 4.7 Infinite Loop และการหลีกเลี่ยง

```java
public class InfiniteLoops {
    
    public static void main(String[] args) {
        
        // ====== Infinite Loop Patterns ======
        // (ต้องระวัง!)
        
        // for loop ไม่มีสิ้นสุด
        // for (;;) {
        //     System.out.println("ไม่มีวันหยุด!");
        // }
        
        // while loop ไม่มีสิ้นสุด
        // while (true) {
        //     System.out.println("วิ่งตลอดไป!");
        // }
        
        // ====== Server Loop Pattern ======
        // (ตัวอย่างการใช้ infinite loop อย่างถูกต้อง)
        boolean serverRunning = true;
        int requestCount = 0;
        int maxRequests = 5; // จำลองการหยุดเซิร์ฟเวอร์
        
        System.out.println("=== Server Started ===");
        
        while (serverRunning) {
            requestCount++;
            System.out.println("Processing request #" + requestCount);
            
            // จำลองการหยุด server
            if (requestCount >= maxRequests) {
                serverRunning = false;
                System.out.println("Server stopped after " + requestCount + " requests");
            }
        }
        
        // ====== Timeout Pattern ======
        long startTime = System.currentTimeMillis();
        long timeout = 1000; // 1 second
        int iterations = 0;
        
        while (System.currentTimeMillis() - startTime < timeout) {
            iterations++;
            // ทำงาน...
        }
        System.out.println("ทำงาน " + iterations + " ครั้งใน 1 วินาที");
    }
}
```

---

## 4.8 Iterator Pattern

```java
import java.util.*;

public class IteratorPattern {
    
    public static void main(String[] args) {
        
        List<String> languages = new ArrayList<>(Arrays.asList(
            "Java", "Kotlin", "Python", "JavaScript", "Go"
        ));
        
        // ====== for-each (แนะนำสำหรับอ่านอย่างเดียว) ======
        System.out.println("=== for-each ===");
        for (String lang : languages) {
            System.out.println(lang);
        }
        
        // ====== Iterator (เหมาะเมื่อต้องลบ element) ======
        System.out.println("\n=== Iterator ===");
        Iterator<String> it = languages.iterator();
        while (it.hasNext()) {
            String lang = it.next();
            if (lang.startsWith("J")) {
                System.out.println("ลบ: " + lang);
                it.remove(); // ลบอย่างปลอดภัย
            }
        }
        System.out.println("เหลือ: " + languages);
        
        // ====== ListIterator (bidirectional) ======
        List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        ListIterator<Integer> lit = numbers.listIterator(numbers.size());
        
        System.out.print("\n=== Reverse with ListIterator: ");
        while (lit.hasPrevious()) {
            System.out.print(lit.previous() + " ");
        }
        System.out.println();
        
        // ====== Map iteration ======
        Map<String, Integer> scores = new LinkedHashMap<>();
        scores.put("Alice", 95);
        scores.put("Bob", 87);
        scores.put("Charlie", 92);
        scores.put("Diana", 88);
        
        System.out.println("\n=== Map iteration ===");
        
        // Entry Set
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.printf("%-10s: %d%n", entry.getKey(), entry.getValue());
        }
        
        // Key Set
        System.out.print("Keys: ");
        for (String key : scores.keySet()) {
            System.out.print(key + " ");
        }
        System.out.println();
        
        // Values
        System.out.print("Values: ");
        for (int score : scores.values()) {
            System.out.print(score + " ");
        }
        System.out.println();
    }
}
```

---

## 4.9 Functional Loops (Java 8+)

```java
import java.util.*;
import java.util.stream.*;

public class FunctionalLoops {
    
    public static void main(String[] args) {
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // ====== forEach ======
        System.out.print("forEach: ");
        numbers.forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // ====== Stream operations ======
        
        // filter - กรองข้อมูล
        System.out.print("เลขคู่: ");
        numbers.stream()
               .filter(n -> n % 2 == 0)
               .forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // map - แปลงข้อมูล
        System.out.print("ยกกำลัง 2: ");
        numbers.stream()
               .map(n -> n * n)
               .forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // reduce - รวมข้อมูล
        int sum = numbers.stream()
                         .reduce(0, Integer::sum);
        System.out.println("Sum: " + sum);
        
        // Chaining
        int result = numbers.stream()
                            .filter(n -> n % 2 == 0)   // กรองเลขคู่
                            .map(n -> n * n)            // ยกกำลัง 2
                            .reduce(0, Integer::sum);   // รวมกัน
        System.out.println("Sum of squares of even numbers: " + result);
        
        // collect to List
        List<Integer> evenSquares = numbers.stream()
                                          .filter(n -> n % 2 == 0)
                                          .map(n -> n * n)
                                          .collect(Collectors.toList());
        System.out.println("Even squares: " + evenSquares);
        
        // IntStream.range (แทน for loop)
        System.out.print("IntStream.range: ");
        IntStream.range(1, 11).forEach(i -> System.out.print(i + " "));
        System.out.println();
        
        IntStream.rangeClosed(1, 10).forEach(i -> System.out.print(i + " "));
        System.out.println();
        
        // ====== String processing ======
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eve");
        
        String result2 = names.stream()
                              .filter(name -> name.length() > 3)
                              .map(String::toUpperCase)
                              .sorted()
                              .collect(Collectors.joining(", "));
        System.out.println("Long names: " + result2);
    }
}
```

---

## 4.10 โปรแกรมตัวอย่าง: เกมทายตัวเลข (Complete Version)

```java
import java.util.Scanner;
import java.util.Random;

public class NumberGuessingGame {
    
    private static final int MIN = 1;
    private static final int MAX = 100;
    private static final int MAX_ATTEMPTS = 10;
    
    private static Scanner scanner = new Scanner(System.in);
    private static int totalGames = 0;
    private static int wins = 0;
    private static int totalAttempts = 0;
    
    public static void main(String[] args) {
        System.out.println("╔══════════════════════════════╗");
        System.out.println("║    เกมทายตัวเลข 1-100       ║");
        System.out.println("╚══════════════════════════════╝");
        System.out.println("คุณมี " + MAX_ATTEMPTS + " ครั้งในการทาย");
        
        boolean playAgain = true;
        
        while (playAgain) {
            playGame();
            
            System.out.print("\nเล่นอีกรอบ? (y/n): ");
            String answer = scanner.next();
            playAgain = answer.equalsIgnoreCase("y");
        }
        
        showStats();
        scanner.close();
    }
    
    private static void playGame() {
        totalGames++;
        Random random = new Random();
        int secret = random.nextInt(MAX - MIN + 1) + MIN;
        int attempts = 0;
        boolean guessed = false;
        
        System.out.println("\n=== รอบที่ " + totalGames + " ===");
        System.out.println("ฉันคิดเลข " + MIN + "-" + MAX + " ทายสิ!");
        
        while (attempts < MAX_ATTEMPTS && !guessed) {
            attempts++;
            System.out.printf("[%d/%d] ทาย: ", attempts, MAX_ATTEMPTS);
            
            int guess;
            try {
                guess = scanner.nextInt();
            } catch (Exception e) {
                System.out.println("กรุณาใส่ตัวเลข!");
                scanner.nextLine();
                attempts--;
                continue;
            }
            
            if (guess < MIN || guess > MAX) {
                System.out.printf("ต้องเป็นเลข %d-%d%n", MIN, MAX);
                attempts--;
                continue;
            }
            
            int remaining = MAX_ATTEMPTS - attempts;
            
            if (guess == secret) {
                guessed = true;
                wins++;
                totalAttempts += attempts;
                System.out.println("🎉 ถูกต้อง! ตอบถูกใน " + attempts + " ครั้ง!");
                
                // Rating
                String rating;
                if (attempts <= 3) rating = "⭐⭐⭐ ยอดเยี่ยม!";
                else if (attempts <= 6) rating = "⭐⭐ ดีมาก!";
                else rating = "⭐ ดี!";
                System.out.println("Rating: " + rating);
                
            } else if (guess < secret) {
                System.out.printf("น้อยเกินไป! (เหลือ %d ครั้ง)%n", remaining);
                
                // Give hints near end
                if (remaining <= 3 && remaining > 0) {
                    int diff = secret - guess;
                    if (diff <= 5) System.out.println("💡 ใกล้มากแล้ว!");
                    else if (diff <= 15) System.out.println("💡 ใกล้แล้ว");
                }
            } else {
                System.out.printf("มากเกินไป! (เหลือ %d ครั้ง)%n", remaining);
                
                if (remaining <= 3 && remaining > 0) {
                    int diff = guess - secret;
                    if (diff <= 5) System.out.println("💡 ใกล้มากแล้ว!");
                    else if (diff <= 15) System.out.println("💡 ใกล้แล้ว");
                }
            }
        }
        
        if (!guessed) {
            System.out.println("\n❌ หมดครั้งแล้ว! ตัวเลขที่ถูกต้องคือ " + secret);
        }
    }
    
    private static void showStats() {
        System.out.println("\n╔══════════════════════════════╗");
        System.out.println("║         สถิติการเล่น          ║");
        System.out.println("╠══════════════════════════════╣");
        System.out.printf( "║ เล่นทั้งหมด: %17d รอบ║%n", totalGames);
        System.out.printf( "║ ชนะ:         %17d รอบ║%n", wins);
        System.out.printf( "║ แพ้:         %17d รอบ║%n", totalGames - wins);
        
        if (wins > 0) {
            double avgAttempts = (double) totalAttempts / wins;
            System.out.printf("║ ครั้งเฉลี่ย: %17.1f ครั้ง║%n", avgAttempts);
        }
        
        double winRate = totalGames > 0 ? (double) wins / totalGames * 100 : 0;
        System.out.printf("║ อัตราชนะ:   %17.1f %%   ║%n", winRate);
        System.out.println("╚══════════════════════════════╝");
    }
}
```

---

## 4.11 Recursion (การเรียกตัวเอง)

```java
public class Recursion {
    
    public static void main(String[] args) {
        
        // ====== Factorial ======
        System.out.println("=== Factorial ===");
        for (int i = 0; i <= 10; i++) {
            System.out.printf("%2d! = %,d%n", i, factorial(i));
        }
        
        // ====== Fibonacci ======
        System.out.println("\n=== Fibonacci ===");
        System.out.print("Fibonacci: ");
        for (int i = 0; i <= 10; i++) {
            System.out.print(fibonacci(i) + " ");
        }
        System.out.println();
        
        // ====== Power ======
        System.out.println("\n=== Power ===");
        System.out.println("2^10 = " + power(2, 10));
        System.out.println("3^5 = " + power(3, 5));
        
        // ====== Palindrome ======
        String[] words = {"racecar", "hello", "level", "java", "madam"};
        System.out.println("\n=== Palindrome Check ===");
        for (String word : words) {
            System.out.println(word + ": " + (isPalindrome(word) ? "palindrome" : "not palindrome"));
        }
        
        // ====== Sum of digits ======
        System.out.println("\n=== Sum of Digits ===");
        System.out.println("sumDigits(12345) = " + sumDigits(12345));
        System.out.println("sumDigits(9999) = " + sumDigits(9999));
    }
    
    // n! = n × (n-1)!
    static long factorial(int n) {
        if (n <= 1) return 1; // Base case
        return n * factorial(n - 1); // Recursive case
    }
    
    // F(n) = F(n-1) + F(n-2)
    static int fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2);
    }
    
    // Fibonacci with memoization (much faster)
    static long[] memo = new long[100];
    static long fibMemo(int n) {
        if (n <= 1) return n;
        if (memo[n] != 0) return memo[n];
        memo[n] = fibMemo(n - 1) + fibMemo(n - 2);
        return memo[n];
    }
    
    // a^b
    static long power(long base, int exp) {
        if (exp == 0) return 1;
        if (exp % 2 == 0) {
            long half = power(base, exp / 2);
            return half * half;
        }
        return base * power(base, exp - 1);
    }
    
    // Check palindrome
    static boolean isPalindrome(String s) {
        if (s.length() <= 1) return true;
        if (s.charAt(0) != s.charAt(s.length() - 1)) return false;
        return isPalindrome(s.substring(1, s.length() - 1));
    }
    
    // Sum of digits
    static int sumDigits(int n) {
        if (n == 0) return 0;
        return (n % 10) + sumDigits(n / 10);
    }
}
```

---

## 4.12 แบบฝึกหัด Part 04

### แบบฝึกหัดที่ 1: สูตรคูณแม่ที่เลือก
```java
import java.util.Scanner;

public class MultiplicationTable {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("แสดงสูตรคูณแม่ที่: ");
        int n = scanner.nextInt();
        
        System.out.println("=== สูตรคูณแม่ " + n + " ===");
        for (int i = 1; i <= 12; i++) {
            System.out.printf("%d × %2d = %3d%n", n, i, n * i);
        }
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 2: หาเลขเฉพาะ
```java
public class PrimeNumbers {
    public static void main(String[] args) {
        System.out.print("เลขเฉพาะ 1-100: ");
        int count = 0;
        
        for (int num = 2; num <= 100; num++) {
            if (isPrime(num)) {
                System.out.print(num + " ");
                count++;
            }
        }
        System.out.println("\nพบ " + count + " จำนวน");
    }
    
    static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i <= Math.sqrt(n); i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
}
```

### แบบฝึกหัดที่ 3: Pascal's Triangle
```java
public class PascalTriangle {
    public static void main(String[] args) {
        int rows = 8;
        int[][] triangle = new int[rows][];
        
        for (int i = 0; i < rows; i++) {
            triangle[i] = new int[i + 1];
            triangle[i][0] = triangle[i][i] = 1;
            
            for (int j = 1; j < i; j++) {
                triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j];
            }
        }
        
        System.out.println("=== Pascal's Triangle ===");
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < rows - i - 1; j++) {
                System.out.print("  ");
            }
            for (int val : triangle[i]) {
                System.out.printf("%4d", val);
            }
            System.out.println();
        }
    }
}
```

---

## สรุป Part 04

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| for loop | basic, enhanced for-each, multiple variables |
| while loop | condition-first, input validation |
| do-while loop | action-first, menu systems |
| Nested loops | patterns, tables |
| break/continue | loop control, labeled breaks |
| Recursion | factorial, fibonacci, palindrome |
| Functional loops | Stream API, forEach |

---

## ขั้นตอนต่อไป

➡️ [Part 05: Arrays และ Multidimensional Arrays](./Part-05-Arrays.md)
