# Part 02: Variables, Data Types และ Operators
## ขั้นตอนที่ 51-120: รากฐานของข้อมูลใน Java

---

## 2.1 Variables (ตัวแปร)

### ตัวแปรคืออะไร?

ตัวแปรคือพื้นที่ในหน่วยความจำที่ใช้เก็บข้อมูล มีชื่อเฉพาะและมีชนิดข้อมูลที่กำหนด

```java
public class VariableDemo {
    
    // Instance Variables (ตัวแปรระดับ Object)
    String name;        // ค่าเริ่มต้นเป็น null
    int age;            // ค่าเริ่มต้นเป็น 0
    boolean isActive;   // ค่าเริ่มต้นเป็น false
    
    // Class Variables (Static Variables)
    static int count = 0;
    
    public static void main(String[] args) {
        
        // Local Variables (ตัวแปรระดับ Method)
        // ต้องกำหนดค่าก่อนใช้งาน
        int number = 42;
        String greeting = "สวัสดี";
        double pi = 3.14159;
        boolean isJavaFun = true;
        
        System.out.println("ตัวเลข: " + number);
        System.out.println("คำทักทาย: " + greeting);
        System.out.printf("Pi = %.5f%n", pi);
        System.out.println("Java สนุกไหม: " + isJavaFun);
    }
}
```

### กฎการตั้งชื่อตัวแปร

```java
// ✅ ชื่อที่ถูกต้อง
int age = 25;
String firstName = "John";
double _price = 99.99;
boolean is2FA = true;
int $count = 0;

// ❌ ชื่อที่ผิด
// int 2number = 5;     // ขึ้นต้นด้วยตัวเลขไม่ได้
// int my-name = 0;     // ใช้ - ไม่ได้
// int class = 5;       // ใช้ keyword ไม่ได้
// int my name = 0;     // มีเว้นวรรคไม่ได้

// Keywords ที่ใช้เป็นชื่อตัวแปรไม่ได้
// abstract, boolean, break, byte, case, catch, char, class,
// const, continue, default, do, double, else, enum, extends,
// final, finally, float, for, goto, if, implements, import,
// instanceof, int, interface, long, native, new, package,
// private, protected, public, return, short, static, strictfp,
// super, switch, synchronized, this, throw, throws, transient,
// try, var, void, volatile, while
```

---

## 2.2 Primitive Data Types

Java มี 8 ชนิดข้อมูลพื้นฐาน (Primitive Types)

### จำนวนเต็ม (Integer Types)

```java
public class IntegerTypes {
    
    public static void main(String[] args) {
        
        // byte: 8-bit, -128 ถึง 127
        byte byteMin = Byte.MIN_VALUE;     // -128
        byte byteMax = Byte.MAX_VALUE;     //  127
        byte byteVal = 100;
        System.out.println("byte: " + byteMin + " ถึง " + byteMax);
        
        // short: 16-bit, -32,768 ถึง 32,767
        short shortMin = Short.MIN_VALUE;  // -32768
        short shortMax = Short.MAX_VALUE;  //  32767
        short shortVal = 30000;
        System.out.println("short: " + shortMin + " ถึง " + shortMax);
        
        // int: 32-bit, -2,147,483,648 ถึง 2,147,483,647
        int intMin = Integer.MIN_VALUE;    // -2147483648
        int intMax = Integer.MAX_VALUE;    //  2147483647
        int intVal = 1000000;
        System.out.println("int: " + intMin + " ถึง " + intMax);
        
        // long: 64-bit, -9.2 × 10^18 ถึง 9.2 × 10^18
        long longMin = Long.MIN_VALUE;
        long longMax = Long.MAX_VALUE;
        long longVal = 9876543210L;  // ต้องมี L หรือ l ต่อท้าย
        System.out.println("long: " + longMin + " ถึง " + longMax);
        
        // การเขียนตัวเลขแบบต่างๆ
        int decimal = 255;       // ทศนิยม
        int hex = 0xFF;          // เลขฐาน 16 (0x นำหน้า)
        int octal = 0377;        // เลขฐาน 8 (0 นำหน้า)
        int binary = 0b11111111; // เลขฐาน 2 (0b นำหน้า)
        
        System.out.println("decimal: " + decimal);
        System.out.println("hex: " + hex);
        System.out.println("octal: " + octal);
        System.out.println("binary: " + binary);
        
        // Underscore separator (Java 7+)
        int million = 1_000_000;
        long creditCard = 1234_5678_9012_3456L;
        int ip = 0xCAFE_BABE;
        System.out.println("million: " + million);
    }
}
```

### จำนวนทศนิยม (Floating-Point Types)

```java
public class FloatingPointTypes {
    
    public static void main(String[] args) {
        
        // float: 32-bit, ประมาณ 7 ตัวเลขนัยสำคัญ
        float floatVal = 3.14f;   // ต้องมี f หรือ F ต่อท้าย
        float floatMin = Float.MIN_VALUE;
        float floatMax = Float.MAX_VALUE;
        System.out.printf("float: %e ถึง %e%n", floatMin, floatMax);
        
        // double: 64-bit, ประมาณ 15-16 ตัวเลขนัยสำคัญ (แนะนำให้ใช้)
        double doubleVal = 3.141592653589793;
        double doubleMin = Double.MIN_VALUE;
        double doubleMax = Double.MAX_VALUE;
        System.out.printf("double: %e ถึง %e%n", doubleMin, doubleMax);
        
        // ตัวอย่างการคำนวณ
        double price = 1234.56;
        double tax = price * 0.07;
        double total = price + tax;
        System.out.printf("ราคา: %.2f, ภาษี: %.2f, รวม: %.2f%n", price, tax, total);
        
        // Scientific Notation
        double lightSpeed = 3e8;        // 3 × 10^8
        double electronMass = 9.11e-31; // 9.11 × 10^-31
        System.out.println("ความเร็วแสง: " + lightSpeed + " m/s");
        
        // ความแม่นยำของ float vs double
        float f1 = 0.1f + 0.2f;
        double d1 = 0.1 + 0.2;
        System.out.println("float: " + f1);     // 0.3
        System.out.println("double: " + d1);    // 0.30000000000000004
        
        // Special Values
        double infinity = Double.POSITIVE_INFINITY;
        double negInfinity = Double.NEGATIVE_INFINITY;
        double notANumber = Double.NaN;
        
        System.out.println("Infinity: " + infinity);
        System.out.println("-Infinity: " + negInfinity);
        System.out.println("NaN: " + notANumber);
        System.out.println("1/0 = " + (1.0 / 0.0));
        System.out.println("isNaN: " + Double.isNaN(0.0 / 0.0));
        
        // BigDecimal สำหรับการคำนวณที่ต้องการความแม่นยำสูง
        java.math.BigDecimal bd1 = new java.math.BigDecimal("0.1");
        java.math.BigDecimal bd2 = new java.math.BigDecimal("0.2");
        java.math.BigDecimal sum = bd1.add(bd2);
        System.out.println("BigDecimal: 0.1 + 0.2 = " + sum);  // 0.3
    }
}
```

### Boolean Type

```java
public class BooleanType {
    
    public static void main(String[] args) {
        
        // boolean: true หรือ false เท่านั้น
        boolean isJavaFun = true;
        boolean isCSharp = false;
        boolean isAdult = (18 >= 18);  // การเปรียบเทียบ
        
        System.out.println("isJavaFun: " + isJavaFun);
        System.out.println("isCSharp: " + isCSharp);
        System.out.println("isAdult: " + isAdult);
        
        // Logical Operations
        boolean a = true, b = false;
        
        System.out.println("AND (&&): " + (a && b));  // false
        System.out.println("OR  (||): " + (a || b));  // true
        System.out.println("NOT (!):  " + (!a));       // false
        System.out.println("XOR (^):  " + (a ^ b));   // true
        
        // Short-circuit evaluation
        int x = 0;
        // ถ้า a เป็น false, b จะไม่ถูกประเมิน
        boolean result1 = (a && (x++ > 0));
        System.out.println("x after &&: " + x);  // x = 1 (ถูกประเมิน)
        
        boolean result2 = (false && (x++ > 0));
        System.out.println("x after false &&: " + x);  // x = 1 (ไม่ถูกประเมิน)
    }
}
```

### Char Type

```java
public class CharType {
    
    public static void main(String[] args) {
        
        // char: 16-bit Unicode character
        char letter = 'A';
        char digit = '5';
        char symbol = '@';
        char thai = 'ก';  // Unicode รองรับภาษาไทย
        
        System.out.println("letter: " + letter);
        System.out.println("digit: " + digit);
        System.out.println("symbol: " + symbol);
        System.out.println("thai: " + thai);
        
        // Unicode escape
        char copyright = '©'; // ©
        char heart = '♥';     // ♥
        char snowman = '☃';   // ☃
        System.out.println("Copyright: " + copyright);
        System.out.println("Heart: " + heart);
        System.out.println("Snowman: " + snowman);
        
        // Escape sequences
        char newline = '\n';
        char tab = '\t';
        char singleQuote = '\'';
        char backslash = '\\';
        
        // char เป็น numeric type ด้วย
        char charA = 'A';
        int asciiA = charA;   // 65
        System.out.println("'A' ASCII = " + asciiA);
        
        char nextChar = (char)(charA + 1);  // 'B'
        System.out.println("A + 1 = " + nextChar);
        
        // วนลูปแสดงตัวอักษร
        System.out.print("ตัวอักษร: ");
        for (char c = 'A'; c <= 'Z'; c++) {
            System.out.print(c);
        }
        System.out.println();
        
        // Character methods
        System.out.println(Character.isLetter('A'));    // true
        System.out.println(Character.isDigit('5'));     // true
        System.out.println(Character.isWhitespace(' ')); // true
        System.out.println(Character.isUpperCase('A')); // true
        System.out.println(Character.toLowerCase('A')); // a
        System.out.println(Character.toUpperCase('a')); // A
    }
}
```

---

## 2.3 ตารางสรุป Primitive Types

```
┌──────────┬──────────┬──────────────────────────────────────────────┬────────────┐
│  Type    │   Size   │             Range                            │  Default   │
├──────────┼──────────┼──────────────────────────────────────────────┼────────────┤
│ byte     │  8 bit   │ -128 to 127                                  │ 0          │
│ short    │ 16 bit   │ -32,768 to 32,767                            │ 0          │
│ int      │ 32 bit   │ -2,147,483,648 to 2,147,483,647             │ 0          │
│ long     │ 64 bit   │ -9.2×10^18 to 9.2×10^18                    │ 0L         │
│ float    │ 32 bit   │ ±1.4×10^-45 to ±3.4×10^38 (7 digits)       │ 0.0f       │
│ double   │ 64 bit   │ ±4.9×10^-324 to ±1.8×10^308 (15-16 digits)│ 0.0d       │
│ boolean  │  1 bit   │ true or false                               │ false      │
│ char     │ 16 bit   │ '\u0000' to '￿' (0 to 65,535)         │ '\u0000'   │
└──────────┴──────────┴──────────────────────────────────────────────┴────────────┘
```

---

## 2.4 Reference Types (Non-Primitive Types)

```java
import java.util.ArrayList;

public class ReferenceTypes {
    
    public static void main(String[] args) {
        
        // String
        String name = "สมชาย";
        String nullStr = null;  // Reference types สามารถเป็น null ได้
        
        // Array
        int[] numbers = {1, 2, 3, 4, 5};
        String[] names = {"Alice", "Bob", "Charlie"};
        
        // Object (Class instances)
        ArrayList<String> list = new ArrayList<>();
        list.add("Java");
        list.add("Kotlin");
        
        // Wrapper Classes (Primitive -> Object)
        Integer intObj = 42;        // auto-boxing
        Double doubleObj = 3.14;    // auto-boxing
        Boolean boolObj = true;     // auto-boxing
        
        // Unboxing (Object -> Primitive)
        int intVal = intObj;        // auto-unboxing
        double doubleVal = doubleObj;
        
        System.out.println("Integer max: " + Integer.MAX_VALUE);
        System.out.println("Integer min: " + Integer.MIN_VALUE);
        System.out.println("Integer binary: " + Integer.toBinaryString(255));
        System.out.println("Integer hex: " + Integer.toHexString(255));
        System.out.println("Integer octal: " + Integer.toOctalString(255));
        
        // Parse String to Number
        int parsed = Integer.parseInt("42");
        double parsedDouble = Double.parseDouble("3.14");
        System.out.println("Parsed int: " + parsed);
        System.out.println("Parsed double: " + parsedDouble);
        
        // Number to String
        String intStr = Integer.toString(42);
        String doubleStr = Double.toString(3.14);
        String str = String.valueOf(100);
        System.out.println("int to String: " + intStr);
    }
}
```

---

## 2.5 String (ชนิดข้อมูลสำคัญ)

```java
public class StringDemo {
    
    public static void main(String[] args) {
        
        // การสร้าง String
        String s1 = "Hello";                     // String Literal
        String s2 = new String("Hello");          // String Object
        String s3 = String.valueOf(42);           // จากตัวเลข
        String s4 = new String(new char[]{'H','i'}); // จาก char array
        
        // String Methods ที่ใช้บ่อย
        String text = "  Hello, Java World!  ";
        
        // ความยาว
        System.out.println("Length: " + text.length());
        
        // ตัดช่องว่าง
        System.out.println("Trim: [" + text.trim() + "]");
        System.out.println("Strip: [" + text.strip() + "]");  // Java 11+
        
        // แปลงตัวอักษร
        System.out.println("Upper: " + text.toUpperCase());
        System.out.println("Lower: " + text.toLowerCase());
        
        // ค้นหา
        System.out.println("Contains 'Java': " + text.contains("Java"));
        System.out.println("Starts with '  H': " + text.startsWith("  H"));
        System.out.println("Ends with '  ': " + text.endsWith("  "));
        System.out.println("IndexOf 'Java': " + text.indexOf("Java"));
        System.out.println("LastIndexOf 'l': " + text.lastIndexOf('l'));
        
        // ตัด/แทนที่
        String trimmed = text.trim();
        System.out.println("Substring(7): " + trimmed.substring(7));
        System.out.println("Substring(7,11): " + trimmed.substring(7, 11));
        System.out.println("Replace: " + trimmed.replace("Java", "Kotlin"));
        System.out.println("ReplaceAll: " + trimmed.replaceAll("[aeiou]", "*"));
        
        // แยก
        String csv = "apple,banana,cherry,date";
        String[] fruits = csv.split(",");
        for (String fruit : fruits) {
            System.out.println(" - " + fruit);
        }
        
        // รวม
        String joined = String.join(", ", fruits);
        System.out.println("Joined: " + joined);
        
        // เปรียบเทียบ
        String a = "Hello";
        String b = "Hello";
        String c = new String("Hello");
        
        System.out.println("a == b: " + (a == b));           // true (same literal)
        System.out.println("a == c: " + (a == c));           // false (different object)
        System.out.println("a.equals(c): " + a.equals(c));  // true (same content)
        System.out.println("a.equalsIgnoreCase('HELLO'): " + a.equalsIgnoreCase("HELLO"));
        
        // charAt
        String hello = "Hello";
        System.out.println("charAt(0): " + hello.charAt(0));  // H
        
        // toCharArray
        char[] chars = hello.toCharArray();
        for (char ch : chars) {
            System.out.print(ch + " ");
        }
        System.out.println();
        
        // isEmpty, isBlank (Java 11+)
        System.out.println("''.isEmpty(): " + "".isEmpty());
        System.out.println("'  '.isEmpty(): " + "  ".isEmpty());  // false
        System.out.println("'  '.isBlank(): " + "  ".isBlank());  // true (Java 11+)
        
        // repeat (Java 11+)
        System.out.println("Ha".repeat(3));  // HaHaHa
        
        // formatted (Java 15+)
        // String info = "Name: %s, Age: %d".formatted("John", 25);
        
        // String comparison
        System.out.println("compare: " + "apple".compareTo("banana"));  // negative
        System.out.println("compare: " + "banana".compareTo("apple"));  // positive
    }
}
```

---

## 2.6 Type Conversion (การแปลงชนิดข้อมูล)

```java
public class TypeConversion {
    
    public static void main(String[] args) {
        
        // ====== Widening Conversion (Implicit) ======
        // แปลงอัตโนมัติจากขนาดเล็ก -> ขนาดใหญ่
        // byte -> short -> int -> long -> float -> double
        
        byte byteVal = 100;
        short shortVal = byteVal;   // byte -> short
        int intVal = shortVal;      // short -> int
        long longVal = intVal;      // int -> long
        float floatVal = longVal;   // long -> float
        double doubleVal = floatVal; // float -> double
        
        System.out.println("byte -> double: " + doubleVal);
        
        // char -> int
        char charVal = 'A';
        int charToInt = charVal;  // 65
        System.out.println("char 'A' -> int: " + charToInt);
        
        // ====== Narrowing Conversion (Explicit Cast) ======
        // ต้อง cast เองจากขนาดใหญ่ -> ขนาดเล็ก
        
        double d = 9.99;
        int i = (int) d;     // ตัดทศนิยมออก = 9 (ไม่ปัดเศษ)
        System.out.println("double 9.99 -> int: " + i);
        
        long l = 1234567890123L;
        int narrowed = (int) l;  // อาจเสียข้อมูล!
        System.out.println("long -> int (may lose data): " + narrowed);
        
        double price = 99.95;
        float f = (float) price;
        System.out.println("double -> float: " + f);
        
        int num = 65;
        char c = (char) num;  // 'A'
        System.out.println("int 65 -> char: " + c);
        
        // ====== String Conversion ======
        
        // Primitive -> String
        int x = 42;
        String s1 = "" + x;              // concatenation
        String s2 = String.valueOf(x);   // valueOf
        String s3 = Integer.toString(x); // wrapper class
        
        double pi = 3.14159;
        String piStr = String.format("%.2f", pi);  // formatted
        
        // String -> Primitive
        String numStr = "42";
        int parsed = Integer.parseInt(numStr);
        double parsedD = Double.parseDouble("3.14");
        boolean parsedB = Boolean.parseBoolean("true");
        
        System.out.println("Parsed: " + parsed + ", " + parsedD + ", " + parsedB);
        
        // ====== Math Methods ======
        System.out.println("round(9.5): " + Math.round(9.5));      // 10
        System.out.println("ceil(9.1): " + Math.ceil(9.1));         // 10.0
        System.out.println("floor(9.9): " + Math.floor(9.9));       // 9.0
        System.out.println("abs(-5): " + Math.abs(-5));              // 5
        System.out.println("max(3,7): " + Math.max(3, 7));           // 7
        System.out.println("min(3,7): " + Math.min(3, 7));           // 3
        System.out.println("pow(2,10): " + (int)Math.pow(2, 10));   // 1024
        System.out.println("sqrt(16): " + Math.sqrt(16));            // 4.0
        System.out.println("random(): " + Math.random());            // 0.0 - 1.0
    }
}
```

---

## 2.7 Operators (ตัวดำเนินการ)

### Arithmetic Operators

```java
public class ArithmeticOperators {
    
    public static void main(String[] args) {
        int a = 10, b = 3;
        
        // Basic Operations
        System.out.println("a + b = " + (a + b));   // 13
        System.out.println("a - b = " + (a - b));   // 7
        System.out.println("a * b = " + (a * b));   // 30
        System.out.println("a / b = " + (a / b));   // 3 (integer division)
        System.out.println("a % b = " + (a % b));   // 1 (remainder)
        
        // Integer vs Double Division
        System.out.println("10 / 3 = " + (10 / 3));        // 3
        System.out.println("10.0 / 3 = " + (10.0 / 3));    // 3.333...
        System.out.println("10 / 3.0 = " + (10 / 3.0));    // 3.333...
        System.out.println("(double)10 / 3 = " + ((double)10 / 3)); // 3.333...
        
        // Increment/Decrement
        int x = 5;
        System.out.println("x: " + x);    // 5
        System.out.println("x++: " + x++); // 5 (post-increment: ใช้แล้วเพิ่ม)
        System.out.println("x: " + x);    // 6
        System.out.println("++x: " + ++x); // 7 (pre-increment: เพิ่มแล้วใช้)
        System.out.println("x: " + x);    // 7
        System.out.println("x--: " + x--); // 7 (post-decrement)
        System.out.println("x: " + x);    // 6
        System.out.println("--x: " + --x); // 5 (pre-decrement)
        
        // Assignment Operators
        int n = 10;
        n += 5;  System.out.println("n += 5: " + n);  // 15
        n -= 3;  System.out.println("n -= 3: " + n);  // 12
        n *= 2;  System.out.println("n *= 2: " + n);  // 24
        n /= 4;  System.out.println("n /= 4: " + n);  // 6
        n %= 4;  System.out.println("n %= 4: " + n);  // 2
        
        // Compound examples
        double total = 0;
        total += 100.50;  // เพิ่มราคาสินค้า
        total += 200.75;  // เพิ่มราคาสินค้า
        total *= 1.07;    // บวก VAT 7%
        System.out.printf("ยอดรวมหลัง VAT: %.2f%n", total);
    }
}
```

### Comparison Operators

```java
public class ComparisonOperators {
    
    public static void main(String[] args) {
        int a = 5, b = 10;
        
        System.out.println("a == b: " + (a == b));  // false
        System.out.println("a != b: " + (a != b));  // true
        System.out.println("a < b: "  + (a < b));   // true
        System.out.println("a > b: "  + (a > b));   // false
        System.out.println("a <= b: " + (a <= b));  // true
        System.out.println("a >= b: " + (a >= b));  // false
        
        // String comparison (ต้องใช้ .equals() ไม่ใช่ ==)
        String s1 = "Hello";
        String s2 = new String("Hello");
        
        System.out.println("s1 == s2: " + (s1 == s2));          // false
        System.out.println("s1.equals(s2): " + s1.equals(s2));  // true
        
        // null check
        String nullStr = null;
        System.out.println("nullStr == null: " + (nullStr == null));
        
        // instanceof
        Object obj = "Hello";
        System.out.println("obj instanceof String: " + (obj instanceof String)); // true
        System.out.println("obj instanceof Integer: " + (obj instanceof Integer)); // false
    }
}
```

### Logical Operators

```java
public class LogicalOperators {
    
    public static void main(String[] args) {
        boolean a = true, b = false;
        
        // && (AND) - ทั้งคู่ต้องเป็น true
        System.out.println("true && true  = " + (true && true));   // true
        System.out.println("true && false = " + (true && false));  // false
        System.out.println("false && true = " + (false && true));  // false
        System.out.println("false && false= " + (false && false)); // false
        
        // || (OR) - อย่างน้อยหนึ่งต้องเป็น true
        System.out.println("true || true  = " + (true || true));   // true
        System.out.println("true || false = " + (true || false));  // true
        System.out.println("false || true = " + (false || true));  // true
        System.out.println("false || false= " + (false || false)); // false
        
        // ! (NOT)
        System.out.println("!true  = " + (!true));   // false
        System.out.println("!false = " + (!false));  // true
        
        // ^ (XOR) - ต่างกันจึงเป็น true
        System.out.println("true ^ true  = " + (true ^ true));    // false
        System.out.println("true ^ false = " + (true ^ false));   // true
        
        // Short-circuit evaluation
        int x = 0;
        boolean r1 = (false && (++x > 0)); // ++x ไม่ถูก execute
        System.out.println("x after false &&: " + x); // 0
        
        boolean r2 = (true || (++x > 0));  // ++x ไม่ถูก execute
        System.out.println("x after true ||: " + x);  // 0
        
        // Practical example
        String name = "John";
        int age = 20;
        boolean hasID = true;
        
        boolean canBuyAlcohol = (age >= 18 && hasID);
        boolean isFreeShipping = (age >= 60 || name.equals("VIP"));
        
        System.out.println("สามารถซื้อแอลกอฮอล์: " + canBuyAlcohol);
        System.out.println("ส่งฟรี: " + isFreeShipping);
    }
}
```

### Bitwise Operators

```java
public class BitwiseOperators {
    
    public static void main(String[] args) {
        int a = 0b1010; // 10
        int b = 0b1100; // 12
        
        System.out.println("a = " + Integer.toBinaryString(a) + " (" + a + ")");
        System.out.println("b = " + Integer.toBinaryString(b) + " (" + b + ")");
        
        // Bitwise AND
        int and = a & b;  // 1000 = 8
        System.out.println("a & b = " + Integer.toBinaryString(and) + " (" + and + ")");
        
        // Bitwise OR
        int or = a | b;   // 1110 = 14
        System.out.println("a | b = " + Integer.toBinaryString(or) + " (" + or + ")");
        
        // Bitwise XOR
        int xor = a ^ b;  // 0110 = 6
        System.out.println("a ^ b = " + Integer.toBinaryString(xor) + " (" + xor + ")");
        
        // Bitwise NOT
        int not = ~a;     // ...11110101 = -11
        System.out.println("~a = " + not);
        
        // Left Shift (x << n = x * 2^n)
        int left = a << 2;  // 1010 << 2 = 101000 = 40
        System.out.println("a << 2 = " + left); // 40
        
        // Right Shift (x >> n = x / 2^n)
        int right = a >> 1; // 1010 >> 1 = 101 = 5
        System.out.println("a >> 1 = " + right); // 5
        
        // Unsigned Right Shift
        int negNum = -8;
        System.out.println("-8 >> 1 = " + (negNum >> 1));   // -4 (sign preserved)
        System.out.println("-8 >>> 1 = " + (negNum >>> 1));  // 2147483644
        
        // Practical use: Check if number is even/odd
        for (int i = 0; i <= 5; i++) {
            String evenOdd = (i & 1) == 0 ? "คู่" : "คี่";
            System.out.println(i + " เป็นเลข" + evenOdd);
        }
        
        // Swap without temp variable
        int x = 5, y = 10;
        System.out.println("Before: x=" + x + ", y=" + y);
        x ^= y;
        y ^= x;
        x ^= y;
        System.out.println("After: x=" + x + ", y=" + y);
    }
}
```

### Ternary Operator

```java
public class TernaryOperator {
    
    public static void main(String[] args) {
        
        // Syntax: condition ? valueIfTrue : valueIfFalse
        
        int age = 20;
        String category = (age >= 18) ? "ผู้ใหญ่" : "เด็ก";
        System.out.println("หมวดหมู่: " + category);
        
        int x = 15;
        String evenOdd = (x % 2 == 0) ? "เลขคู่" : "เลขคี่";
        System.out.println(x + " เป็น" + evenOdd);
        
        // Nested ternary
        int score = 75;
        String grade = (score >= 90) ? "A" :
                       (score >= 80) ? "B" :
                       (score >= 70) ? "C" :
                       (score >= 60) ? "D" : "F";
        System.out.println("เกรด: " + grade);
        
        // ใช้ใน System.out.println
        int balance = 500;
        System.out.println("ยอดเงิน " + (balance > 0 ? "บวก" : "ลบ") + ": " + balance);
        
        // ใช้คำนวณ absolute value
        int num = -42;
        int abs = (num >= 0) ? num : -num;
        System.out.println("|" + num + "| = " + abs);
        
        // ใช้กับ method call
        double temperature = 36.5;
        System.out.println(temperature > 37.5 ? "มีไข้" : "ปกติ");
    }
}
```

### Operator Precedence

```java
public class OperatorPrecedence {
    
    public static void main(String[] args) {
        
        // ลำดับการทำงานของ Operators (สูง -> ต่ำ)
        // 1. () [] .
        // 2. ++ -- ~ !
        // 3. * / %
        // 4. + -
        // 5. << >> >>>
        // 6. < <= > >= instanceof
        // 7. == !=
        // 8. &
        // 9. ^
        // 10. |
        // 11. &&
        // 12. ||
        // 13. ?:
        // 14. = += -= *= /= %= etc.
        
        // ตัวอย่าง
        int result1 = 2 + 3 * 4;      // 14 (คูณก่อน)
        int result2 = (2 + 3) * 4;    // 20 (วงเล็บก่อน)
        
        System.out.println("2 + 3 * 4 = " + result1);
        System.out.println("(2 + 3) * 4 = " + result2);
        
        // ตัวอย่างที่ซับซ้อน
        int a = 5, b = 3, c = 2;
        int complex = a + b * c - a / c + b % c;
        // = 5 + (3*2) - (5/2) + (3%2)
        // = 5 + 6 - 2 + 1
        // = 10
        System.out.println("a + b * c - a / c + b % c = " + complex);
        
        // ตัวอย่างที่ผิดพลาดได้ง่าย
        boolean r1 = false || true && false;  // false || (true && false) = false
        boolean r2 = (false || true) && false; // true && false = false
        System.out.println("false || true && false = " + r1);
        System.out.println("(false || true) && false = " + r2);
        
        // แนะนำให้ใส่วงเล็บเพื่อความชัดเจน
        int value = ((5 + 3) * (10 - 4)) / 2;
        System.out.println("((5+3)*(10-4))/2 = " + value);
    }
}
```

---

## 2.8 var Keyword (Java 10+)

```java
public class VarKeyword {
    
    public static void main(String[] args) {
        
        // var - type inference สำหรับ local variables
        var number = 42;           // inferred as int
        var text = "Hello";        // inferred as String
        var pi = 3.14;             // inferred as double
        var list = new java.util.ArrayList<String>(); // inferred as ArrayList<String>
        
        System.out.println("number type: " + ((Object)number).getClass().getSimpleName());
        System.out.println("text type: " + text.getClass().getSimpleName());
        
        // var ไม่ได้เปลี่ยนชนิดข้อมูล - ยังคง strongly typed
        // number = "string"; // Error! ไม่สามารถเปลี่ยนชนิดได้
        
        // ใช้ใน enhanced for loop
        int[] numbers = {1, 2, 3, 4, 5};
        for (var n : numbers) {
            System.out.print(n + " ");
        }
        System.out.println();
        
        // ข้อจำกัด:
        // var ใช้ได้เฉพาะ local variables เท่านั้น
        // ต้องกำหนดค่าเริ่มต้นทันที
        // ไม่สามารถใช้กับ null ได้โดยตรง
        // var nullVal = null; // Error!
        
        // แต่ทำแบบนี้ได้
        var str = (String) null;
        System.out.println("str: " + str);
    }
}
```

---

## 2.9 Constants (ค่าคงที่)

```java
public class Constants {
    
    // Class Constants (ใช้ static final)
    static final int MAX_USERS = 1000;
    static final String APP_NAME = "MyApp";
    static final double PI = 3.14159265358979;
    static final String DATABASE_URL = "jdbc:mysql://localhost:3306/mydb";
    
    // Enum Constants
    enum Color { RED, GREEN, BLUE }
    enum Direction { NORTH, SOUTH, EAST, WEST }
    enum DayOfWeek { MON, TUE, WED, THU, FRI, SAT, SUN }
    
    public static void main(String[] args) {
        
        System.out.println("App: " + APP_NAME);
        System.out.println("Max Users: " + MAX_USERS);
        System.out.println("PI: " + PI);
        
        // Java Math Constants
        System.out.println("Math.PI: " + Math.PI);
        System.out.println("Math.E: " + Math.E);
        
        // Enum usage
        Color favoriteColor = Color.BLUE;
        System.out.println("Favorite color: " + favoriteColor);
        
        Direction dir = Direction.NORTH;
        System.out.println("Direction: " + dir);
        
        // Enum in switch
        DayOfWeek today = DayOfWeek.MON;
        switch (today) {
            case MON:
            case TUE:
            case WED:
            case THU:
            case FRI:
                System.out.println("วันทำงาน");
                break;
            case SAT:
            case SUN:
                System.out.println("วันหยุด");
                break;
        }
        
        // Cannot modify constants
        // MAX_USERS = 2000; // Error: cannot assign to final variable
    }
}
```

---

## 2.10 โปรแกรมตัวอย่าง: BMI Calculator

```java
import java.util.Scanner;

/**
 * BMI Calculator
 * คำนวณ Body Mass Index (ดัชนีมวลกาย)
 */
public class BMICalculator {
    
    // Constants
    static final double BMI_UNDERWEIGHT = 18.5;
    static final double BMI_NORMAL_MAX = 24.9;
    static final double BMI_OVERWEIGHT_MAX = 29.9;
    
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("╔════════════════════════════╗");
        System.out.println("║      BMI Calculator        ║");
        System.out.println("╚════════════════════════════╝");
        
        System.out.print("น้ำหนัก (กิโลกรัม): ");
        double weight = scanner.nextDouble();
        
        System.out.print("ส่วนสูง (เมตร): ");
        double height = scanner.nextDouble();
        
        // คำนวณ BMI
        double bmi = weight / (height * height);
        
        // กำหนดหมวดหมู่
        String category;
        String advice;
        
        if (bmi < BMI_UNDERWEIGHT) {
            category = "น้ำหนักน้อยเกินไป";
            advice = "ควรเพิ่มน้ำหนักและรับประทานอาหารให้ครบ 5 หมู่";
        } else if (bmi <= BMI_NORMAL_MAX) {
            category = "น้ำหนักปกติ";
            advice = "ดีมาก! รักษาน้ำหนักนี้ไว้";
        } else if (bmi <= BMI_OVERWEIGHT_MAX) {
            category = "น้ำหนักเกิน";
            advice = "ควรออกกำลังกายและควบคุมอาหาร";
        } else {
            category = "อ้วน";
            advice = "ควรปรึกษาแพทย์และปรับพฤติกรรมการกิน";
        }
        
        System.out.println("\n════════════ ผลลัพธ์ ════════════");
        System.out.printf("น้ำหนัก:   %.1f กิโลกรัม%n", weight);
        System.out.printf("ส่วนสูง:   %.2f เมตร%n", height);
        System.out.printf("BMI:       %.2f%n", bmi);
        System.out.println("หมวดหมู่: " + category);
        System.out.println("คำแนะนำ:  " + advice);
        
        // แสดง BMI Scale
        System.out.println("\n════════ ตาราง BMI ════════");
        System.out.println("< 18.5    น้ำหนักน้อย");
        System.out.println("18.5-24.9 ปกติ");
        System.out.println("25.0-29.9 น้ำหนักเกิน");
        System.out.println(">= 30.0   อ้วน");
        
        scanner.close();
    }
}
```

---

## 2.11 แบบฝึกหัด Part 02

### แบบฝึกหัดที่ 1: Temperature Converter
แปลงอุณหภูมิระหว่าง Celsius, Fahrenheit, และ Kelvin

```java
import java.util.Scanner;

public class TemperatureConverter {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("ใส่อุณหภูมิ Celsius: ");
        double celsius = scanner.nextDouble();
        
        double fahrenheit = (celsius * 9.0 / 5.0) + 32;
        double kelvin = celsius + 273.15;
        
        System.out.printf("%.2f°C = %.2f°F = %.2fK%n", celsius, fahrenheit, kelvin);
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 2: Tip Calculator
คำนวณทิปจากราคาอาหาร

```java
import java.util.Scanner;

public class TipCalculator {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.print("ราคาอาหาร (บาท): ");
        double billAmount = scanner.nextDouble();
        
        System.out.print("เปอร์เซ็นต์ทิป (เช่น 15): ");
        double tipPercent = scanner.nextDouble();
        
        double tipAmount = billAmount * tipPercent / 100;
        double totalBill = billAmount + tipAmount;
        
        System.out.println("\n====== ใบเสร็จ ======");
        System.out.printf("ราคาอาหาร: %8.2f บาท%n", billAmount);
        System.out.printf("ทิป (%.0f%%): %8.2f บาท%n", tipPercent, tipAmount);
        System.out.printf("รวมทั้งหมด: %7.2f บาท%n", totalBill);
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 3: Bitwise Flag System
ใช้ Bitwise operators สร้างระบบ permissions

```java
public class PermissionSystem {
    // Permission flags
    static final int READ    = 0b001; // 1
    static final int WRITE   = 0b010; // 2
    static final int EXECUTE = 0b100; // 4
    
    public static void main(String[] args) {
        // กำหนด permissions
        int userPermissions = READ | WRITE;           // 3 (011)
        int adminPermissions = READ | WRITE | EXECUTE; // 7 (111)
        int guestPermissions = READ;                  // 1 (001)
        
        // ตรวจสอบ permissions
        System.out.println("=== User Permissions ===");
        System.out.println("Read: " + ((userPermissions & READ) != 0));
        System.out.println("Write: " + ((userPermissions & WRITE) != 0));
        System.out.println("Execute: " + ((userPermissions & EXECUTE) != 0));
        
        System.out.println("\n=== Admin Permissions ===");
        System.out.println("Read: " + ((adminPermissions & READ) != 0));
        System.out.println("Write: " + ((adminPermissions & WRITE) != 0));
        System.out.println("Execute: " + ((adminPermissions & EXECUTE) != 0));
        
        // เพิ่ม permission
        guestPermissions |= WRITE;
        System.out.println("\nAfter adding Write to Guest:");
        System.out.println("Write: " + ((guestPermissions & WRITE) != 0));
        
        // ลบ permission
        guestPermissions &= ~WRITE;
        System.out.println("\nAfter removing Write from Guest:");
        System.out.println("Write: " + ((guestPermissions & WRITE) != 0));
    }
}
```

---

## สรุป Part 02

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Variables | การประกาศ, ตั้งชื่อ, ขอบเขต |
| Primitive Types | byte, short, int, long, float, double, boolean, char |
| Reference Types | String, Array, Object, Wrapper Classes |
| Type Conversion | Widening, Narrowing, String conversion |
| Operators | Arithmetic, Comparison, Logical, Bitwise, Ternary |
| Constants | final, static final, enum |

---

## ขั้นตอนต่อไป

➡️ [Part 03: Control Flow - if/else, switch](./Part-03-Control-Flow.md)
