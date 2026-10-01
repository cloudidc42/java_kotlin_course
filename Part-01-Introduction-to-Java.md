# Part 01: แนะนำ Java, ติดตั้ง JDK และ Hello World
## ขั้นตอนที่ 1-50: รากฐานของ Java Programming

---

## 1.1 Java คืออะไร?

Java เป็นภาษาโปรแกรมมิ่งที่ถูกพัฒนาโดย **Sun Microsystems** (ปัจจุบันเป็นของ Oracle) ในปี 1995 โดย **James Gosling** Java ถูกออกแบบมาด้วยหลักการ **"Write Once, Run Anywhere" (WORA)** ซึ่งหมายความว่าโค้ดที่เขียนครั้งเดียวสามารถรันได้บนทุกแพลตฟอร์ม

### คุณสมบัติหลักของ Java

```
┌─────────────────────────────────────────────────────────────┐
│                    Java Characteristics                     │
├─────────────────────────────────────────────────────────────┤
│  1. Platform Independent  - รันได้บน JVM ทุกแพลตฟอร์ม      │
│  2. Object-Oriented       - ทุกอย่างเป็น Object             │
│  3. Simple                - เรียนรู้ง่าย syntax ชัดเจน     │
│  4. Secure                - มี Security Manager             │
│  5. Robust                - Strong type checking            │
│  6. Multithreaded         - รองรับ concurrent programming   │
│  7. High Performance      - JIT Compiler                    │
│  8. Distributed           - รองรับ network programming      │
└─────────────────────────────────────────────────────────────┘
```

### ทำไมต้องเรียน Java?

1. **ใช้งานอย่างแพร่หลาย**: มากกว่า 3 พันล้านอุปกรณ์รัน Java
2. **ตลาดแรงงาน**: Developer ที่ต้องการมากที่สุดในโลก
3. **Android Development**: เป็นภาษาหลักสำหรับ Android
4. **Enterprise Applications**: ใช้ใน Banking, Finance, Healthcare
5. **เงินเดือนสูง**: Java Developer มีเงินเดือนเฉลี่ยสูง

---

## 1.2 สถาปัตยกรรมของ Java

```
┌─────────────────────────────────────────────────────────────┐
│                     Java Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Source Code (.java)                                        │
│         │                                                   │
│         ▼                                                   │
│    Java Compiler (javac)                                    │
│         │                                                   │
│         ▼                                                   │
│    Bytecode (.class)                                        │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────────────────────────┐                   │
│  │         JVM (Java Virtual Machine)   │                   │
│  │  ┌──────────────────────────────┐    │                   │
│  │  │   Class Loader               │    │                   │
│  │  │   Bytecode Verifier          │    │                   │
│  │  │   Interpreter / JIT Compiler │    │                   │
│  │  │   Garbage Collector          │    │                   │
│  │  └──────────────────────────────┘    │                   │
│  └──────────────────────────────────────┘                   │
│         │                                                   │
│         ▼                                                   │
│    Machine Code (Native OS)                                 │
│    Windows / macOS / Linux                                  │
└─────────────────────────────────────────────────────────────┘
```

### JDK vs JRE vs JVM

```
┌─────────────────────────────────────────────────────────────┐
│                          JDK                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                        JRE                            │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │                    JVM                          │  │  │
│  │  │  Class Loader, Memory, GC, Execution Engine     │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  Java Libraries (java.util, java.io, java.net...)     │  │
│  └───────────────────────────────────────────────────────┘  │
│  Development Tools (javac, javadoc, jdb, jar...)            │
└─────────────────────────────────────────────────────────────┘
```

- **JVM (Java Virtual Machine)**: รัน Bytecode
- **JRE (Java Runtime Environment)**: JVM + Libraries (สำหรับรันโปรแกรม)
- **JDK (Java Development Kit)**: JRE + Dev Tools (สำหรับพัฒนาโปรแกรม)

---

## 1.3 การติดตั้ง JDK

### Windows

**วิธีที่ 1: ดาวน์โหลดจาก Oracle**
1. ไปที่ https://www.oracle.com/java/technologies/downloads/
2. เลือก Java 21 LTS (Long Term Support)
3. เลือก Windows x64 Installer
4. รันไฟล์ .exe และทำตาม wizard

**วิธีที่ 2: ใช้ Winget (แนะนำ)**
```powershell
# ติดตั้ง JDK 21
winget install Microsoft.OpenJDK.21

# ตรวจสอบการติดตั้ง
java -version
javac -version
```

**วิธีที่ 3: ใช้ Chocolatey**
```powershell
# ติดตั้ง Chocolatey ก่อน (ถ้ายังไม่มี)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง JDK
choco install openjdk21 -y
```

**ตั้งค่า Environment Variables (Windows)**
```
1. คลิกขวา "This PC" > Properties
2. Advanced System Settings > Environment Variables
3. System Variables > New:
   Variable name: JAVA_HOME
   Variable value: C:\Program Files\Java\jdk-21
4. แก้ไข Path > New:
   %JAVA_HOME%\bin
```

### macOS

**วิธีที่ 1: ใช้ Homebrew (แนะนำ)**
```bash
# ติดตั้ง Homebrew ก่อน
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง JDK 21
brew install openjdk@21

# เพิ่ม symlink
sudo ln -sfn /opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-21.jdk

# ตั้งค่า JAVA_HOME ใน ~/.zshrc หรือ ~/.bash_profile
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 21)' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.zshrc
source ~/.zshrc
```

**วิธีที่ 2: ใช้ SDKMAN (แนะนำสำหรับจัดการหลาย version)**
```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ดูรายการ JDK ที่มี
sdk list java

# ติดตั้ง JDK 21
sdk install java 21.0.2-oracle

# สลับ version
sdk use java 21.0.2-oracle

# ดู version ปัจจุบัน
sdk current java
```

### Linux (Ubuntu/Debian)

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง JDK 21
sudo apt install openjdk-21-jdk -y

# ตรวจสอบ
java -version
javac -version

# ถ้ามีหลาย Java version
sudo update-alternatives --config java

# ตั้งค่า JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Java version
java -version
# ผลลัพธ์ที่ควรได้:
# openjdk version "21.0.2" 2024-01-16
# OpenJDK Runtime Environment (build 21.0.2+13-58)
# OpenJDK 64-Bit Server VM (build 21.0.2+13-58, mixed mode, sharing)

# ตรวจสอบ Compiler version
javac -version
# ผลลัพธ์: javac 21.0.2

# ตรวจสอบ JAVA_HOME
echo $JAVA_HOME
# ผลลัพธ์: /path/to/jdk
```

---

## 1.4 IDE (Integrated Development Environment)

### IntelliJ IDEA (แนะนำมากที่สุด)

```
ดาวน์โหลด: https://www.jetbrains.com/idea/
- Community Edition: ฟรี (เหมาะสำหรับ Java/Kotlin)
- Ultimate Edition: จ่ายเงิน (เหมาะสำหรับ Enterprise, Spring Boot)
```

**การตั้งค่า IntelliJ IDEA:**
1. ดาวน์โหลดและติดตั้ง
2. เปิดโปรแกรม > "New Project"
3. เลือก "Java" หรือ "Kotlin"
4. เลือก JDK ที่ติดตั้ง
5. ตั้งชื่อ Project

### Eclipse

```bash
# ดาวน์โหลด: https://www.eclipse.org/downloads/
# Eclipse IDE for Java Developers
```

### VS Code (Visual Studio Code)

```bash
# ดาวน์โหลด: https://code.visualstudio.com/

# ติดตั้ง Extensions:
# 1. Extension Pack for Java (Microsoft)
# 2. Kotlin (fwcd)
# 3. Debugger for Java

# สร้าง Java Project
# Ctrl+Shift+P > Java: Create Java Project
```

---

## 1.5 Hello World - โปรแกรมแรกของคุณ

### สร้างไฟล์ HelloWorld.java

```java
/**
 * โปรแกรมแรก: Hello World
 * 
 * คำอธิบาย: โปรแกรมที่แสดงข้อความ "Hello, World!" บน Console
 * 
 * @author Your Name
 * @version 1.0
 * @since 2024-01-01
 */
public class HelloWorld {
    
    /**
     * Main method - จุดเริ่มต้นของโปรแกรม Java ทุกโปรแกรม
     * 
     * @param args command-line arguments
     */
    public static void main(String[] args) {
        // แสดงข้อความบน Console
        System.out.println("Hello, World!");
        
        // แสดงข้อความแบบไม่ขึ้นบรรทัดใหม่
        System.out.print("Hello");
        System.out.print(", ");
        System.out.println("Java!");
        
        // แสดงข้อความแบบ formatted
        System.out.printf("สวัสดี %s! คุณอายุ %d ปี%n", "สมชาย", 25);
        
        // String.format สำหรับสร้าง string
        String message = String.format("Java Version: %s", System.getProperty("java.version"));
        System.out.println(message);
    }
}
```

### วิธี Compile และ Run

```bash
# วิธีที่ 1: Compile แล้ว Run แยก
# Compile (สร้าง .class file)
javac HelloWorld.java

# Run
java HelloWorld

# วิธีที่ 2: Run ตรงๆ (Java 11+)
java HelloWorld.java

# ผลลัพธ์:
# Hello, World!
# Hello, Java!
# สวัสดี สมชาย! คุณอายุ 25 ปี
# Java Version: 21.0.2
```

### อธิบาย Syntax ของ HelloWorld

```java
// 1. การประกาศ Class
public class HelloWorld {
//     ↑        ↑
//  modifier  ชื่อ class (ต้องตรงกับชื่อไฟล์)

    // 2. Main Method
    public static void main(String[] args) {
    //     ↑       ↑    ↑     ↑         ↑
    //  modifier static return  ชื่อ method  parameters
    
        // 3. Statement
        System.out.println("Hello, World!");
        //  ↑       ↑       ↑
        // class   field   method
    }
}
```

---

## 1.6 โครงสร้างโปรแกรม Java

### ส่วนประกอบหลัก

```java
// 1. Package Declaration (ถ้ามี)
package com.example.myapp;

// 2. Import Statements
import java.util.Scanner;
import java.util.ArrayList;

// 3. Class Declaration
public class MyFirstProgram {
    
    // 4. Class Variables (Fields)
    private static int programCount = 0;
    
    // 5. Constructor
    public MyFirstProgram() {
        programCount++;
    }
    
    // 6. Main Method
    public static void main(String[] args) {
        System.out.println("โปรแกรมที่ " + (programCount + 1));
        
        // 7. Local Variables
        String name = "Java Developer";
        int year = 2024;
        
        // 8. Statements
        System.out.println("ยินดีต้อนรับ " + name + " ปี " + year);
    }
    
    // 9. Methods
    public static void printInfo() {
        System.out.println("Java is awesome!");
    }
}
```

---

## 1.7 Comments ใน Java

```java
public class CommentsDemo {
    
    public static void main(String[] args) {
        
        // 1. Single-line Comment
        // นี่คือ comment บรรทัดเดียว
        
        /* 2. Multi-line Comment
           สามารถเขียนได้หลายบรรทัด
           ใช้สำหรับอธิบายโค้ดยาวๆ
        */
        
        /**
         * 3. Javadoc Comment
         * ใช้สำหรับสร้าง documentation
         * @param name ชื่อผู้ใช้
         * @return ข้อความต้อนรับ
         */
        
        System.out.println("Hello!"); // inline comment
        
        // TODO: เพิ่มฟีเจอร์ใหม่ในอนาคต
        // FIXME: แก้ไข bug ตรงนี้
        // NOTE: ระวังการใช้งาน method นี้
        
        // ตัวอย่างโค้ดที่ comment ออก
        // System.out.println("โค้ดนี้ถูก comment ออก");
    }
}
```

---

## 1.8 การรับ Input จากผู้ใช้

```java
import java.util.Scanner;

public class UserInput {
    
    public static void main(String[] args) {
        // สร้าง Scanner object สำหรับรับ input
        Scanner scanner = new Scanner(System.in);
        
        // รับ String
        System.out.print("กรุณาใส่ชื่อของคุณ: ");
        String name = scanner.nextLine();
        
        // รับ Integer
        System.out.print("กรุณาใส่อายุ: ");
        int age = scanner.nextInt();
        
        // รับ Double
        System.out.print("กรุณาใส่ความสูง (เมตร): ");
        double height = scanner.nextDouble();
        
        // รับ Boolean
        System.out.print("คุณเป็นนักศึกษาหรือไม่? (true/false): ");
        boolean isStudent = scanner.nextBoolean();
        
        // แสดงผลลัพธ์
        System.out.println("\n========== ข้อมูลของคุณ ==========");
        System.out.println("ชื่อ: " + name);
        System.out.println("อายุ: " + age + " ปี");
        System.out.printf("ความสูง: %.2f เมตร%n", height);
        System.out.println("นักศึกษา: " + (isStudent ? "ใช่" : "ไม่ใช่"));
        
        // ปิด scanner เมื่อใช้งานเสร็จ
        scanner.close();
    }
}
```

**ผลลัพธ์:**
```
กรุณาใส่ชื่อของคุณ: สมชาย
กรุณาใส่อายุ: 25
กรุณาใส่ความสูง (เมตร): 1.75
คุณเป็นนักศึกษาหรือไม่? (true/false): true

========== ข้อมูลของคุณ ==========
ชื่อ: สมชาย
อายุ: 25 ปี
ความสูง: 1.75 เมตร
นักศึกษา: ใช่
```

---

## 1.9 การแสดงผลแบบต่างๆ

```java
public class OutputFormats {
    
    public static void main(String[] args) {
        
        // ====== System.out.println ======
        // พิมพ์แล้วขึ้นบรรทัดใหม่
        System.out.println("บรรทัดที่ 1");
        System.out.println("บรรทัดที่ 2");
        
        // ====== System.out.print ======
        // พิมพ์โดยไม่ขึ้นบรรทัดใหม่
        System.out.print("A");
        System.out.print("B");
        System.out.print("C");
        System.out.println(); // ขึ้นบรรทัดใหม่
        
        // ====== System.out.printf ======
        // พิมพ์แบบ formatted
        
        // %s - String
        System.out.printf("ชื่อ: %s%n", "สมชาย");
        
        // %d - Integer
        System.out.printf("อายุ: %d ปี%n", 25);
        
        // %f - Float/Double (%.2f = 2 ทศนิยม)
        System.out.printf("เงินเดือน: %.2f บาท%n", 50000.75);
        
        // %b - Boolean
        System.out.printf("แต่งงานแล้ว: %b%n", false);
        
        // %c - Character
        System.out.printf("เกรด: %c%n", 'A');
        
        // %10s - จัดชิดขวา ความกว้าง 10
        System.out.printf("%10s: %d%n", "อายุ", 25);
        
        // %-10s - จัดชิดซ้าย ความกว้าง 10
        System.out.printf("%-10s: %d%n", "อายุ", 25);
        
        // %05d - เติม 0 ข้างหน้า
        System.out.printf("รหัส: %05d%n", 42);
        
        // Escape Characters
        System.out.println("Tab:\tEnd");           // \t = Tab
        System.out.println("NewLine:\nEnd");        // \n = New Line
        System.out.println("Quote: \"Hello\"");     // \" = Double Quote
        System.out.println("Backslash: \\");        // \\ = Backslash
        System.out.println("Apostrophe: \'");       // \' = Single Quote
        
        // String concatenation
        String firstName = "สม";
        String lastName = "ชาย";
        int year = 2024;
        
        // วิธีที่ 1: ใช้ +
        System.out.println("ชื่อ: " + firstName + " " + lastName);
        
        // วิธีที่ 2: StringBuilder (มีประสิทธิภาพสูงกว่า)
        StringBuilder sb = new StringBuilder();
        sb.append("ชื่อ: ")
          .append(firstName)
          .append(" ")
          .append(lastName)
          .append(" (")
          .append(year)
          .append(")");
        System.out.println(sb.toString());
        
        // วิธีที่ 3: String.format
        String formatted = String.format("ชื่อ: %s %s (%d)", firstName, lastName, year);
        System.out.println(formatted);
    }
}
```

---

## 1.10 โปรแกรมตัวอย่างแรก: Calculator อย่างง่าย

```java
import java.util.Scanner;

public class SimpleCalculator {
    
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("=============================");
        System.out.println("     Simple Calculator       ");
        System.out.println("=============================");
        
        System.out.print("ใส่ตัวเลขที่ 1: ");
        double num1 = scanner.nextDouble();
        
        System.out.print("ใส่ตัวเลขที่ 2: ");
        double num2 = scanner.nextDouble();
        
        System.out.println("\n----- ผลลัพธ์ -----");
        System.out.printf("%.2f + %.2f = %.2f%n", num1, num2, num1 + num2);
        System.out.printf("%.2f - %.2f = %.2f%n", num1, num2, num1 - num2);
        System.out.printf("%.2f × %.2f = %.2f%n", num1, num2, num1 * num2);
        
        if (num2 != 0) {
            System.out.printf("%.2f ÷ %.2f = %.2f%n", num1, num2, num1 / num2);
        } else {
            System.out.println("ไม่สามารถหารด้วย 0 ได้!");
        }
        
        System.out.printf("%.2f %% %.2f = %.2f%n", num1, num2, num1 % num2);
        
        scanner.close();
    }
}
```

**ผลลัพธ์:**
```
=============================
     Simple Calculator       
=============================
ใส่ตัวเลขที่ 1: 10
ใส่ตัวเลขที่ 2: 3

----- ผลลัพธ์ -----
10.00 + 3.00 = 13.00
10.00 - 3.00 = 7.00
10.00 × 3.00 = 30.00
10.00 ÷ 3.00 = 3.33
10.00 % 3.00 = 1.00
```

---

## 1.11 การจัดการ Project Structure

### โครงสร้าง Project ที่แนะนำ

```
my-java-project/
├── src/
│   └── main/
│       └── java/
│           └── com/
│               └── example/
│                   ├── Main.java
│                   ├── model/
│                   │   └── User.java
│                   ├── service/
│                   │   └── UserService.java
│                   └── util/
│                       └── Helper.java
├── test/
│   └── main/
│       └── java/
│           └── com/
│               └── example/
│                   └── UserServiceTest.java
├── lib/
│   └── external-library.jar
├── build.gradle (หรือ pom.xml)
└── README.md
```

### Package Declaration

```java
// ไฟล์: src/main/java/com/example/model/User.java

package com.example.model;  // ประกาศ package

public class User {
    private String name;
    private int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + "}";
    }
}
```

```java
// ไฟล์: src/main/java/com/example/Main.java

package com.example;

import com.example.model.User;  // import class จาก package อื่น

public class Main {
    public static void main(String[] args) {
        User user = new User("สมชาย", 25);
        System.out.println(user);
    }
}
```

---

## 1.12 การ Compile หลายไฟล์

```bash
# Compile ไฟล์เดียว
javac HelloWorld.java

# Compile ทุกไฟล์ใน directory ปัจจุบัน
javac *.java

# Compile พร้อมกำหนด output directory
javac -d bin src/main/java/com/example/*.java

# Compile พร้อม classpath
javac -cp lib/external.jar src/Main.java

# Run โปรแกรมที่มี package
java -cp bin com.example.Main

# สร้าง JAR file
jar cf myapp.jar -C bin .

# Run จาก JAR
java -jar myapp.jar

# ดู manifest ของ JAR
jar tf myapp.jar
```

---

## 1.13 Java Versions History

```
Java Versions ที่สำคัญ:
┌──────────┬──────────┬─────────────────────────────────────────┐
│ Version  │   ปี     │ Feature ที่สำคัญ                        │
├──────────┼──────────┼─────────────────────────────────────────┤
│ Java 8   │ 2014     │ Lambda, Stream API, Optional, DateTime  │
│ Java 11  │ 2018     │ LTS, HTTP Client, String methods        │
│ Java 17  │ 2021     │ LTS, Sealed Classes, Pattern Matching   │
│ Java 21  │ 2023     │ LTS, Virtual Threads, Record Patterns   │
│ Java 22  │ 2024     │ Unnamed Variables, String Templates     │
└──────────┴──────────┴─────────────────────────────────────────┘

LTS (Long-Term Support) versions: 8, 11, 17, 21
แนะนำให้ใช้ Java 21 สำหรับโปรเจคใหม่
```

---

## 1.14 Best Practices เบื้องต้น

```java
/**
 * ตัวอย่าง Best Practices ใน Java
 */
public class BestPractices {
    
    // 1. ตั้งชื่อ Class แบบ PascalCase
    // ✅ ถูก: HelloWorld, UserProfile, DatabaseConnection
    // ❌ ผิด: helloworld, user_profile, database-connection
    
    // 2. ตั้งชื่อ Method และ Variable แบบ camelCase
    // ✅ ถูก: getUserName(), firstName, totalAmount
    // ❌ ผิด: GetUserName(), first_name, TotalAmount
    
    // 3. ตั้งชื่อ Constants แบบ UPPER_SNAKE_CASE
    static final int MAX_SIZE = 100;          // ✅
    static final String DATABASE_URL = "..."; // ✅
    
    // 4. ใช้ meaningful names
    // ✅ ถูก
    int numberOfStudents = 30;
    String customerFirstName = "John";
    
    // ❌ ผิด
    int n = 30;
    String s = "John";
    
    // 5. หนึ่ง method ทำหน้าที่เดียว (Single Responsibility)
    public static void printUserInfo(String name, int age) {
        System.out.println("Name: " + name);
        System.out.println("Age: " + age);
    }
    
    // 6. จัดการ resources ด้วย try-with-resources
    public static void readFile(String path) {
        try (java.io.BufferedReader reader = new java.io.BufferedReader(
                new java.io.FileReader(path))) {
            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println(line);
            }
        } catch (java.io.IOException e) {
            System.err.println("Error reading file: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        // 7. ใช้ String.format แทนการ concatenate หลายๆ ครั้ง
        String name = "สมชาย";
        int age = 25;
        double gpa = 3.75;
        
        // ❌ ไม่แนะนำ
        System.out.println("ชื่อ: " + name + ", อายุ: " + age + ", GPA: " + gpa);
        
        // ✅ แนะนำ
        System.out.printf("ชื่อ: %s, อายุ: %d, GPA: %.2f%n", name, age, gpa);
        
        // หรือ
        String info = String.format("ชื่อ: %s, อายุ: %d, GPA: %.2f", name, age, gpa);
        System.out.println(info);
    }
}
```

---

## 1.15 แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 1: Hello, Thailand!
สร้างโปรแกรมที่แสดง:
```
สวัสดี ประเทศไทย!
Welcome to Java Programming!
ปี: 2024
```

**เฉลย:**
```java
public class HelloThailand {
    public static void main(String[] args) {
        System.out.println("สวัสดี ประเทศไทย!");
        System.out.println("Welcome to Java Programming!");
        System.out.println("ปี: " + java.time.Year.now().getValue());
    }
}
```

### แบบฝึกหัดที่ 2: Personal Info
สร้างโปรแกรมรับข้อมูลส่วนตัวและแสดงผลในรูปแบบสวยงาม

**เฉลย:**
```java
import java.util.Scanner;

public class PersonalInfo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("╔══════════════════════════╗");
        System.out.println("║    Personal Information  ║");
        System.out.println("╚══════════════════════════╝");
        
        System.out.print("ชื่อ: ");
        String name = scanner.nextLine();
        
        System.out.print("อายุ: ");
        int age = scanner.nextInt();
        scanner.nextLine(); // clear buffer
        
        System.out.print("อาชีพ: ");
        String occupation = scanner.nextLine();
        
        System.out.print("เงินเดือน: ");
        double salary = scanner.nextDouble();
        
        System.out.println("\n╔══════════════════════════╗");
        System.out.printf("║ ชื่อ: %-20s║%n", name);
        System.out.printf("║ อายุ: %-20d║%n", age);
        System.out.printf("║ อาชีพ: %-19s║%n", occupation);
        System.out.printf("║ เงินเดือน: %-15.2f║%n", salary);
        System.out.println("╚══════════════════════════╝");
        
        scanner.close();
    }
}
```

### แบบฝึกหัดที่ 3: ASCII Art
สร้างโปรแกรมแสดง ASCII Art ของดาว

**เฉลย:**
```java
public class StarPattern {
    public static void main(String[] args) {
        System.out.println("★ ★ ★ ★ ★");
        System.out.println("  ★ ★ ★");
        System.out.println("    ★");
        System.out.println();
        
        // ดาวที่มีชื่อ Java
        System.out.println("  ╔═══╗");
        System.out.println("  ║   ║");
        System.out.println("  ║ J ║");
        System.out.println("  ║ A ║");
        System.out.println("  ║ V ║");
        System.out.println("  ║ A ║");
        System.out.println("  ╚═══╝");
    }
}
```

---

## สรุป Part 01

ใน Part 01 นี้ เราได้เรียนรู้:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Java คืออะไร | ประวัติ, คุณสมบัติ, ทำไมต้องเรียน |
| สถาปัตยกรรม | JVM, JRE, JDK, Bytecode |
| การติดตั้ง | JDK บน Windows, macOS, Linux |
| IDE | IntelliJ IDEA, Eclipse, VS Code |
| Hello World | โปรแกรมแรก, Syntax พื้นฐาน |
| การแสดงผล | println, print, printf |
| การรับ Input | Scanner class |
| Best Practices | การตั้งชื่อ, การจัดโค้ด |

---

## ขั้นตอนต่อไป

➡️ [Part 02: Variables, Data Types และ Operators](./Part-02-Variables-DataTypes-Operators.md)

---

*หมายเหตุ: ตรวจสอบให้แน่ใจว่าสามารถ compile และ run โปรแกรม Hello World ได้สำเร็จก่อนไปยัง Part ถัดไป*
