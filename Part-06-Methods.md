# Part 06: Methods และ Functions
## ขั้นตอนที่ 331-400: การสร้างและใช้งาน Methods

---

## 6.1 Methods พื้นฐาน

```java
public class MethodBasics {
    
    // ====== Method Syntax ======
    // [modifier] [return-type] methodName([parameters]) {
    //     // method body
    //     [return value;]
    // }
    
    // Method ไม่มีค่าส่งกลับ (void)
    public static void greet() {
        System.out.println("สวัสดี!");
    }
    
    // Method มีค่าส่งกลับ
    public static int add(int a, int b) {
        return a + b;
    }
    
    // Method มี parameter และ return value
    public static double calculateArea(double radius) {
        return Math.PI * radius * radius;
    }
    
    // Method กับ String
    public static String formatName(String first, String last) {
        return last + " " + first;
    }
    
    public static void main(String[] args) {
        // เรียกใช้ methods
        greet();
        
        int sum = add(5, 3);
        System.out.println("5 + 3 = " + sum);
        
        double area = calculateArea(7.5);
        System.out.printf("Area: %.2f%n", area);
        
        String name = formatName("สมชาย", "นามสกุล");
        System.out.println("Name: " + name);
        
        // Method ใน expression
        System.out.println("10 + 20 = " + add(10, 20));
        
        // Nested method calls
        System.out.println("max of 3 numbers: " + Math.max(Math.max(1, 5), 3));
    }
}
```

---

## 6.2 Parameters และ Return Types

```java
public class ParametersAndReturns {
    
    // ====== Multiple Parameters ======
    public static double calculateBMI(double weight, double height) {
        return weight / (height * height);
    }
    
    // ====== Varargs (Variable Arguments) ======
    // รับ parameters ไม่จำกัดจำนวน
    public static int sum(int... numbers) {
        int total = 0;
        for (int num : numbers) {
            total += num;
        }
        return total;
    }
    
    // Varargs กับ parameters อื่น (varargs ต้องเป็นตัวสุดท้าย)
    public static String format(String separator, String... words) {
        return String.join(separator, words);
    }
    
    // ====== Array Parameters ======
    public static double average(double[] values) {
        if (values.length == 0) return 0;
        double sum = 0;
        for (double v : values) sum += v;
        return sum / values.length;
    }
    
    // ====== Multiple Return Values (ใช้ Array หรือ Object) ======
    public static int[] minMax(int[] arr) {
        int min = arr[0], max = arr[0];
        for (int val : arr) {
            if (val < min) min = val;
            if (val > max) max = val;
        }
        return new int[]{min, max};
    }
    
    // ====== Return Boolean ======
    public static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i <= Math.sqrt(n); i++) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    public static void main(String[] args) {
        // BMI
        System.out.printf("BMI: %.2f%n", calculateBMI(70, 1.75));
        
        // Varargs
        System.out.println("sum(1,2,3): " + sum(1, 2, 3));
        System.out.println("sum(1,2,3,4,5): " + sum(1, 2, 3, 4, 5));
        System.out.println("sum(): " + sum());
        
        // Format
        System.out.println(format(", ", "Java", "Kotlin", "Python"));
        System.out.println(format("-", "2024", "01", "15"));
        
        // Average
        double[] scores = {85.5, 92.0, 78.5, 96.0, 88.0};
        System.out.printf("Average: %.2f%n", average(scores));
        
        // Min/Max
        int[] nums = {3, 1, 4, 1, 5, 9, 2, 6};
        int[] result = minMax(nums);
        System.out.printf("Min: %d, Max: %d%n", result[0], result[1]);
        
        // isPrime
        for (int i = 2; i <= 20; i++) {
            if (isPrime(i)) System.out.print(i + " ");
        }
        System.out.println();
    }
}
```

---

## 6.3 Method Overloading

```java
public class MethodOverloading {
    
    // ====== Overloading: ชื่อเหมือนกัน แต่ parameter ต่างกัน ======
    
    // add สำหรับ int
    public static int add(int a, int b) {
        System.out.println("add(int, int)");
        return a + b;
    }
    
    // add สำหรับ double
    public static double add(double a, double b) {
        System.out.println("add(double, double)");
        return a + b;
    }
    
    // add สำหรับ 3 int
    public static int add(int a, int b, int c) {
        System.out.println("add(int, int, int)");
        return a + b + c;
    }
    
    // add สำหรับ String
    public static String add(String a, String b) {
        System.out.println("add(String, String)");
        return a + b;
    }
    
    // ====== print overloading ======
    public static void print(int value) {
        System.out.println("Int: " + value);
    }
    
    public static void print(double value) {
        System.out.println("Double: " + value);
    }
    
    public static void print(String value) {
        System.out.println("String: " + value);
    }
    
    public static void print(boolean value) {
        System.out.println("Boolean: " + value);
    }
    
    public static void main(String[] args) {
        System.out.println("=== Overloading ===");
        
        System.out.println(add(5, 3));
        System.out.println(add(5.5, 3.2));
        System.out.println(add(1, 2, 3));
        System.out.println(add("Hello", " World"));
        
        System.out.println("\n=== Print Overloading ===");
        print(42);
        print(3.14);
        print("Java");
        print(true);
        
        // Java เลือก method ที่เหมาะสมที่สุด
        print((int)10);    // ใช้ print(int)
        print(10.0);       // ใช้ print(double)
    }
}
```

---

## 6.4 Pass by Value vs Pass by Reference

```java
import java.util.Arrays;

public class PassByValue {
    
    public static void main(String[] args) {
        
        // ====== Primitive: Pass by Value ======
        // ค่า copy ไปให้ method ไม่กระทบ original
        
        int x = 10;
        System.out.println("Before: x = " + x);
        changePrimitive(x);
        System.out.println("After: x = " + x);  // ยังคง 10
        
        // ====== Object: Pass by Reference (ค่า reference copy ไป) ======
        // สามารถเปลี่ยน state ของ object ได้ แต่ไม่สามารถเปลี่ยน reference ได้
        
        int[] arr = {1, 2, 3};
        System.out.println("\nBefore: " + Arrays.toString(arr));
        modifyArray(arr);
        System.out.println("After: " + Arrays.toString(arr));  // เปลี่ยนแล้ว!
        
        // String: Immutable (ไม่เปลี่ยน)
        StringBuilder sb = new StringBuilder("Hello");
        System.out.println("\nBefore: " + sb);
        modifyString(sb);
        System.out.println("After: " + sb);  // เปลี่ยนแล้ว!
        
        String str = "Hello";
        System.out.println("\nBefore String: " + str);
        tryChangeString(str);
        System.out.println("After String: " + str);  // ไม่เปลี่ยน (immutable)
    }
    
    static void changePrimitive(int val) {
        val = 999;  // เปลี่ยนแค่ local copy
        System.out.println("Inside method: val = " + val);
    }
    
    static void modifyArray(int[] arr) {
        arr[0] = 999;  // เปลี่ยน element ของ array จริงๆ
        System.out.println("Inside method: " + Arrays.toString(arr));
    }
    
    static void modifyString(StringBuilder sb) {
        sb.append(" World");  // เปลี่ยน object จริงๆ
        System.out.println("Inside method: " + sb);
    }
    
    static void tryChangeString(String str) {
        str = "Changed";  // เปลี่ยนแค่ local reference
        System.out.println("Inside method: " + str);
    }
}
```

---

## 6.5 Recursive Methods

```java
public class RecursiveMethods {
    
    public static void main(String[] args) {
        
        // ====== Factorial ======
        System.out.println("Factorial:");
        for (int i = 0; i <= 10; i++) {
            System.out.printf("%2d! = %,d%n", i, factorial(i));
        }
        
        // ====== Fibonacci ======
        System.out.println("\nFibonacci:");
        System.out.print("Fib: ");
        for (int i = 0; i <= 15; i++) {
            System.out.print(fibMemo(i) + " ");
        }
        System.out.println();
        
        // ====== Tower of Hanoi ======
        System.out.println("\nTower of Hanoi (3 disks):");
        hanoi(3, 'A', 'C', 'B');
        
        // ====== GCD (Greatest Common Divisor) ======
        System.out.println("\nGCD:");
        System.out.println("GCD(48, 18) = " + gcd(48, 18));
        System.out.println("GCD(100, 75) = " + gcd(100, 75));
        
        // ====== Binary to Decimal ======
        System.out.println("\nBinary to Decimal:");
        System.out.println("1010 = " + binaryToDecimal(1010));
        System.out.println("11111111 = " + binaryToDecimal(11111111));
        
        // ====== Power ======
        System.out.println("\nPower:");
        System.out.println("2^10 = " + power(2, 10));
        System.out.println("3^5 = " + power(3, 5));
    }
    
    static long factorial(int n) {
        if (n <= 1) return 1;
        return n * factorial(n - 1);
    }
    
    // Fibonacci with memoization
    static long[] memo = new long[100];
    static long fibMemo(int n) {
        if (n <= 1) return n;
        if (memo[n] != 0) return memo[n];
        return memo[n] = fibMemo(n - 1) + fibMemo(n - 2);
    }
    
    // Tower of Hanoi
    static int hanoiMoves = 0;
    static void hanoi(int n, char from, char to, char aux) {
        if (n == 1) {
            System.out.printf("Move disk 1 from %c to %c%n", from, to);
            hanoiMoves++;
            return;
        }
        hanoi(n - 1, from, aux, to);
        System.out.printf("Move disk %d from %c to %c%n", n, from, to);
        hanoiMoves++;
        hanoi(n - 1, aux, to, from);
    }
    
    // Euclidean GCD
    static int gcd(int a, int b) {
        if (b == 0) return a;
        return gcd(b, a % b);
    }
    
    // Binary to Decimal
    static int binaryToDecimal(int n) {
        if (n == 0) return 0;
        return (n % 10) + 2 * binaryToDecimal(n / 10);
    }
    
    // Fast Power (O(log n))
    static long power(long base, int exp) {
        if (exp == 0) return 1;
        if (exp % 2 == 0) {
            long half = power(base, exp / 2);
            return half * half;
        }
        return base * power(base, exp - 1);
    }
}
```

---

## 6.6 Lambda Expressions (Java 8+) พื้นฐาน

```java
import java.util.*;
import java.util.function.*;

public class LambdaBasics {
    
    public static void main(String[] args) {
        
        // ====== Functional Interface ======
        // Interface ที่มี abstract method แค่ตัวเดียว
        
        // Runnable (ไม่มี parameter ไม่มี return)
        Runnable r = () -> System.out.println("Running!");
        r.run();
        
        // Consumer<T> (มี parameter ไม่มี return)
        Consumer<String> printer = s -> System.out.println("Hello, " + s);
        printer.accept("Java");
        
        // Supplier<T> (ไม่มี parameter มี return)
        Supplier<String> greeting = () -> "สวัสดี!";
        System.out.println(greeting.get());
        
        // Function<T, R> (มี parameter มี return)
        Function<Integer, Integer> square = n -> n * n;
        System.out.println("5² = " + square.apply(5));
        
        // BiFunction<T, U, R> (2 parameters)
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        System.out.println("3 + 4 = " + add.apply(3, 4));
        
        // Predicate<T> (return boolean)
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<Integer> isPositive = n -> n > 0;
        
        System.out.println("10 is even: " + isEven.test(10));
        System.out.println("10 is positive: " + isPositive.test(10));
        
        // Predicate composition
        Predicate<Integer> isEvenAndPositive = isEven.and(isPositive);
        System.out.println("10 is even and positive: " + isEvenAndPositive.test(10));
        
        // ====== Lambda กับ Collections ======
        List<String> names = Arrays.asList("Charlie", "Alice", "Bob", "David");
        
        // Sort with lambda
        names.sort((a, b) -> a.compareTo(b));
        System.out.println("Sorted: " + names);
        
        // Sort by length
        names.sort((a, b) -> a.length() - b.length());
        System.out.println("By length: " + names);
        
        // forEach
        names.forEach(name -> System.out.println("Hello, " + name));
        
        // removeIf
        List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
        numbers.removeIf(n -> n % 2 == 0);
        System.out.println("Odd numbers: " + numbers);
        
        // ====== Method Reference ======
        // Class::staticMethod
        Function<String, Integer> parser = Integer::parseInt;
        System.out.println("Parsed: " + parser.apply("42"));
        
        // instance::method
        String prefix = "Hello, ";
        Function<String, String> greeter = prefix::concat;
        System.out.println(greeter.apply("World"));
        
        // Class::instanceMethod
        Function<String, String> toUpper = String::toUpperCase;
        System.out.println(toUpper.apply("java"));
        
        // Constructor reference
        Supplier<ArrayList<String>> listMaker = ArrayList::new;
        ArrayList<String> list = listMaker.get();
        list.add("Created from constructor reference");
        System.out.println(list);
    }
}
```

---

## 6.7 Higher-Order Functions

```java
import java.util.*;
import java.util.function.*;

public class HigherOrderFunctions {
    
    // ====== Methods ที่รับ Function เป็น parameter ======
    
    static int applyOperation(int x, int y, BiFunction<Integer, Integer, Integer> op) {
        return op.apply(x, y);
    }
    
    static List<Integer> transform(List<Integer> list, Function<Integer, Integer> f) {
        List<Integer> result = new ArrayList<>();
        for (int item : list) {
            result.add(f.apply(item));
        }
        return result;
    }
    
    static List<Integer> filter(List<Integer> list, Predicate<Integer> pred) {
        List<Integer> result = new ArrayList<>();
        for (int item : list) {
            if (pred.test(item)) result.add(item);
        }
        return result;
    }
    
    static <T> T reduce(List<T> list, T identity, BinaryOperator<T> op) {
        T result = identity;
        for (T item : list) {
            result = op.apply(result, item);
        }
        return result;
    }
    
    // ====== Method ที่ return Function ======
    static Function<Integer, Integer> multiplier(int factor) {
        return n -> n * factor;
    }
    
    static Predicate<Integer> greaterThan(int threshold) {
        return n -> n > threshold;
    }
    
    public static void main(String[] args) {
        
        // applyOperation
        System.out.println("5 + 3 = " + applyOperation(5, 3, (a, b) -> a + b));
        System.out.println("5 * 3 = " + applyOperation(5, 3, (a, b) -> a * b));
        System.out.println("5 ^ 3 = " + applyOperation(5, 3, (a, b) -> (int)Math.pow(a, b)));
        
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // transform
        List<Integer> doubled = transform(numbers, n -> n * 2);
        System.out.println("Doubled: " + doubled);
        
        List<Integer> squared = transform(numbers, n -> n * n);
        System.out.println("Squared: " + squared);
        
        // filter
        List<Integer> evens = filter(numbers, n -> n % 2 == 0);
        System.out.println("Evens: " + evens);
        
        // reduce
        Integer sum = reduce(numbers, 0, Integer::sum);
        Integer product = reduce(numbers, 1, (a, b) -> a * b);
        System.out.println("Sum: " + sum);
        System.out.println("Product: " + product);
        
        // Higher-order functions returning functions
        Function<Integer, Integer> triple = multiplier(3);
        Function<Integer, Integer> times10 = multiplier(10);
        
        System.out.println("Triple 5: " + triple.apply(5));
        System.out.println("Times10 7: " + times10.apply(7));
        
        Predicate<Integer> isGt5 = greaterThan(5);
        List<Integer> bigNums = filter(numbers, isGt5);
        System.out.println("Greater than 5: " + bigNums);
        
        // Function composition
        Function<Integer, Integer> doubleIt = n -> n * 2;
        Function<Integer, Integer> addOne = n -> n + 1;
        
        Function<Integer, Integer> doubleTheAddOne = doubleIt.andThen(addOne);
        Function<Integer, Integer> addOneThenDouble = doubleIt.compose(addOne);
        
        System.out.println("doubleThenAddOne(5): " + doubleTheAddOne.apply(5));   // 11
        System.out.println("addOneThenDouble(5): " + addOneThenDouble.apply(5));  // 12
    }
}
```

---

## 6.8 Static vs Instance Methods

```java
public class StaticVsInstance {
    
    // Instance variables
    private String name;
    private int value;
    private static int instanceCount = 0;
    
    // Constructor
    public StaticVsInstance(String name, int value) {
        this.name = name;
        this.value = value;
        instanceCount++;
    }
    
    // ====== Instance Methods ======
    // ต้องเรียกผ่าน object instance
    
    public void display() {
        System.out.println("Name: " + name + ", Value: " + value);
    }
    
    public int getValue() {
        return value;
    }
    
    public void setValue(int value) {
        this.value = value;
    }
    
    // ====== Static Methods ======
    // เรียกได้โดยตรงผ่าน class name
    
    public static int getInstanceCount() {
        return instanceCount;
    }
    
    public static double calculateAverage(int... numbers) {
        int sum = 0;
        for (int n : numbers) sum += n;
        return (double) sum / numbers.length;
    }
    
    public static boolean isValidName(String name) {
        return name != null && !name.trim().isEmpty() && name.length() >= 2;
    }
    
    // ====== Factory Methods (Static) ======
    public static StaticVsInstance createDefault() {
        return new StaticVsInstance("Default", 0);
    }
    
    public static StaticVsInstance createWithName(String name) {
        return new StaticVsInstance(name, 100);
    }
    
    public static void main(String[] args) {
        
        // Static methods - ไม่ต้องสร้าง instance
        System.out.println("Instance count: " + StaticVsInstance.getInstanceCount());
        System.out.println("Average: " + StaticVsInstance.calculateAverage(1, 2, 3, 4, 5));
        System.out.println("Valid name 'John': " + StaticVsInstance.isValidName("John"));
        System.out.println("Valid name '': " + StaticVsInstance.isValidName(""));
        
        // Instance methods - ต้องสร้าง instance ก่อน
        StaticVsInstance obj1 = new StaticVsInstance("Alice", 42);
        StaticVsInstance obj2 = new StaticVsInstance("Bob", 100);
        StaticVsInstance obj3 = StaticVsInstance.createDefault();
        
        obj1.display();
        obj2.display();
        obj3.display();
        
        System.out.println("Instance count: " + StaticVsInstance.getInstanceCount());
        
        // Modify via instance method
        obj1.setValue(99);
        obj1.display();
    }
}
```

---

## 6.9 Method Best Practices

```java
public class MethodBestPractices {
    
    // ====== 1. ชื่อ method ควรเป็น verb ======
    public static double calculateTax(double income) {
        return income * 0.07;
    }
    
    public static boolean isValidEmail(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
    
    // ====== 2. Method ควรทำสิ่งเดียว (Single Responsibility) ======
    
    // ❌ ไม่ดี - ทำหลายอย่าง
    public static void processAndPrintUser(String name, int age) {
        // ประมวลผล
        String processedName = name.trim().toUpperCase();
        String ageGroup = age >= 18 ? "Adult" : "Minor";
        // พิมพ์
        System.out.println(processedName + " - " + ageGroup);
    }
    
    // ✅ ดีกว่า - แยกหน้าที่
    public static String processName(String name) {
        return name.trim().toUpperCase();
    }
    
    public static String getAgeGroup(int age) {
        return age >= 18 ? "Adult" : "Minor";
    }
    
    public static void printUserInfo(String name, int age) {
        System.out.println(processName(name) + " - " + getAgeGroup(age));
    }
    
    // ====== 3. ใช้ Guard Clauses ======
    public static double divide(double a, double b) {
        if (b == 0) throw new ArithmeticException("Division by zero");
        return a / b;
    }
    
    // ====== 4. Meaningful Parameter Names ======
    
    // ❌ ไม่ดี
    public static double calc(double p, double r, int n) {
        return p * Math.pow(1 + r, n);
    }
    
    // ✅ ดีกว่า
    public static double calculateCompoundInterest(double principal, double rate, int years) {
        return principal * Math.pow(1 + rate, years);
    }
    
    // ====== 5. ค่า Default Parameters (ใช้ Overloading) ======
    public static String connect(String host, int port, String protocol) {
        return protocol + "://" + host + ":" + port;
    }
    
    public static String connect(String host, int port) {
        return connect(host, port, "https");
    }
    
    public static String connect(String host) {
        return connect(host, 443);
    }
    
    public static void main(String[] args) {
        // Test
        System.out.println("Tax: " + calculateTax(50000));
        System.out.println("Valid: " + isValidEmail("test@example.com"));
        
        printUserInfo("  john doe  ", 25);
        
        System.out.println("Interest: " + calculateCompoundInterest(10000, 0.05, 10));
        
        System.out.println(connect("example.com"));
        System.out.println(connect("example.com", 8080));
        System.out.println(connect("example.com", 8080, "http"));
    }
}
```

---

## 6.10 โปรแกรมตัวอย่าง: Math Utilities Library

```java
public class MathUtils {
    
    // ====== Basic Math ======
    public static int abs(int n) { return n < 0 ? -n : n; }
    public static double abs(double n) { return n < 0 ? -n : n; }
    
    public static int max(int a, int b) { return a > b ? a : b; }
    public static int min(int a, int b) { return a < b ? a : b; }
    
    public static int max(int... nums) {
        int max = nums[0];
        for (int n : nums) if (n > max) max = n;
        return max;
    }
    
    // ====== Number Theory ======
    public static boolean isPrime(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (int i = 3; i <= Math.sqrt(n); i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    public static int gcd(int a, int b) {
        return b == 0 ? a : gcd(b, a % b);
    }
    
    public static int lcm(int a, int b) {
        return a / gcd(a, b) * b;
    }
    
    public static long factorial(int n) {
        if (n < 0) throw new IllegalArgumentException("n must be non-negative");
        if (n <= 1) return 1;
        return n * factorial(n - 1);
    }
    
    public static long fibonacci(int n) {
        if (n < 0) throw new IllegalArgumentException("n must be non-negative");
        if (n <= 1) return n;
        long a = 0, b = 1;
        for (int i = 2; i <= n; i++) {
            long temp = a + b;
            a = b;
            b = temp;
        }
        return b;
    }
    
    // ====== Statistics ======
    public static double mean(double[] data) {
        if (data.length == 0) throw new IllegalArgumentException("Empty array");
        double sum = 0;
        for (double d : data) sum += d;
        return sum / data.length;
    }
    
    public static double median(double[] data) {
        if (data.length == 0) throw new IllegalArgumentException("Empty array");
        double[] sorted = data.clone();
        java.util.Arrays.sort(sorted);
        int n = sorted.length;
        if (n % 2 == 0) return (sorted[n/2 - 1] + sorted[n/2]) / 2.0;
        return sorted[n / 2];
    }
    
    public static double standardDeviation(double[] data) {
        double mean = mean(data);
        double sum = 0;
        for (double d : data) sum += (d - mean) * (d - mean);
        return Math.sqrt(sum / data.length);
    }
    
    public static double[] normalize(double[] data) {
        double min = data[0], max = data[0];
        for (double d : data) {
            if (d < min) min = d;
            if (d > max) max = d;
        }
        double range = max - min;
        double[] normalized = new double[data.length];
        for (int i = 0; i < data.length; i++) {
            normalized[i] = range == 0 ? 0 : (data[i] - min) / range;
        }
        return normalized;
    }
    
    // ====== Geometry ======
    public static double circleArea(double radius) {
        return Math.PI * radius * radius;
    }
    
    public static double circlePerimeter(double radius) {
        return 2 * Math.PI * radius;
    }
    
    public static double rectangleArea(double width, double height) {
        return width * height;
    }
    
    public static double triangleArea(double base, double height) {
        return 0.5 * base * height;
    }
    
    // Pythagorean theorem
    public static double hypotenuse(double a, double b) {
        return Math.sqrt(a * a + b * b);
    }
    
    // ====== Conversion ======
    public static double celsiusToFahrenheit(double c) { return c * 9.0/5.0 + 32; }
    public static double fahrenheitToCelsius(double f) { return (f - 32) * 5.0/9.0; }
    public static double kmToMiles(double km) { return km * 0.621371; }
    public static double milesToKm(double miles) { return miles * 1.60934; }
    
    // ====== Testing ======
    public static void main(String[] args) {
        System.out.println("=== Basic Math ===");
        System.out.println("abs(-5) = " + abs(-5));
        System.out.println("max(3,7,2,9,5) = " + max(3,7,2,9,5));
        
        System.out.println("\n=== Number Theory ===");
        System.out.print("Primes: ");
        for (int i = 2; i <= 30; i++) if (isPrime(i)) System.out.print(i + " ");
        System.out.println();
        System.out.println("GCD(48,18) = " + gcd(48, 18));
        System.out.println("LCM(4,6) = " + lcm(4, 6));
        System.out.println("10! = " + factorial(10));
        
        System.out.println("\n=== Statistics ===");
        double[] scores = {85, 92, 78, 96, 88, 74, 95, 83};
        System.out.printf("Mean: %.2f%n", mean(scores));
        System.out.printf("Median: %.2f%n", median(scores));
        System.out.printf("StdDev: %.2f%n", standardDeviation(scores));
        
        System.out.println("\n=== Geometry ===");
        System.out.printf("Circle area (r=5): %.2f%n", circleArea(5));
        System.out.printf("Triangle area (b=3, h=4): %.2f%n", triangleArea(3, 4));
        System.out.printf("Hypotenuse (3,4): %.2f%n", hypotenuse(3, 4));
        
        System.out.println("\n=== Conversion ===");
        System.out.printf("37°C = %.1f°F%n", celsiusToFahrenheit(37));
        System.out.printf("100 km = %.2f miles%n", kmToMiles(100));
    }
}
```

---

## สรุป Part 06

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Method Basics | syntax, parameters, return types |
| Varargs | variable arguments |
| Overloading | same name, different parameters |
| Pass by Value | primitive vs reference types |
| Recursion | factorial, fibonacci, GCD |
| Lambda Basics | functional interfaces |
| Higher-Order Functions | functions as parameters/return |
| Best Practices | naming, single responsibility |

---

## ขั้นตอนต่อไป

➡️ [Part 07: OOP - Classes และ Objects](./Part-07-OOP-Classes-Objects.md)
