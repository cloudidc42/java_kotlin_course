# Part 07: OOP - Classes และ Objects
## ขั้นตอนที่ 401-470: Object-Oriented Programming

---

## 7.1 OOP คืออะไร?

Object-Oriented Programming (OOP) คือแนวคิดในการเขียนโปรแกรมที่จำลองโลกแห่งความเป็นจริง โดยแบ่งโปรแกรมออกเป็น Objects ที่มีคุณสมบัติ (Properties) และพฤติกรรม (Behaviors)

```
หลักการ 4 ข้อของ OOP:
┌─────────────────────────────────────────────────────────────┐
│  1. Encapsulation  - ห่อหุ้มข้อมูลและ method ไว้ด้วยกัน    │
│  2. Inheritance    - สืบทอดคุณสมบัติจาก class แม่          │
│  3. Polymorphism   - object หนึ่งสามารถมีหลาย forms        │
│  4. Abstraction    - ซ่อนรายละเอียดที่ซับซ้อน             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7.2 Class และ Object

```java
/**
 * Class คือ blueprint (แม่พิมพ์) สำหรับสร้าง Object
 * Object คือ instance ของ Class
 */
public class Car {
    
    // ====== Fields (Instance Variables) ======
    private String brand;     // ยี่ห้อ
    private String model;     // รุ่น
    private int year;         // ปีที่ผลิต
    private double price;     // ราคา
    private String color;     // สี
    private boolean isRunning; // กำลังทำงาน
    private int speed;        // ความเร็วปัจจุบัน
    
    // Class Variable (shared by all instances)
    private static int totalCarsCreated = 0;
    
    // ====== Constructors ======
    
    // Default Constructor
    public Car() {
        this.brand = "Unknown";
        this.model = "Unknown";
        this.year = 2024;
        this.price = 0;
        this.color = "White";
        this.isRunning = false;
        this.speed = 0;
        totalCarsCreated++;
    }
    
    // Parameterized Constructor
    public Car(String brand, String model, int year, double price, String color) {
        this.brand = brand;
        this.model = model;
        this.year = year;
        this.price = price;
        this.color = color;
        this.isRunning = false;
        this.speed = 0;
        totalCarsCreated++;
    }
    
    // Copy Constructor
    public Car(Car other) {
        this(other.brand, other.model, other.year, other.price, other.color);
    }
    
    // ====== Getters ======
    public String getBrand() { return brand; }
    public String getModel() { return model; }
    public int getYear() { return year; }
    public double getPrice() { return price; }
    public String getColor() { return color; }
    public boolean isRunning() { return isRunning; }
    public int getSpeed() { return speed; }
    public static int getTotalCarsCreated() { return totalCarsCreated; }
    
    // ====== Setters ======
    public void setBrand(String brand) {
        if (brand != null && !brand.isEmpty()) {
            this.brand = brand;
        }
    }
    
    public void setColor(String color) {
        this.color = color;
    }
    
    public void setPrice(double price) {
        if (price >= 0) this.price = price;
    }
    
    // ====== Behaviors (Methods) ======
    
    public void start() {
        if (!isRunning) {
            isRunning = true;
            System.out.println(brand + " " + model + " started! 🚗");
        } else {
            System.out.println("Already running!");
        }
    }
    
    public void stop() {
        if (isRunning) {
            isRunning = false;
            speed = 0;
            System.out.println(brand + " " + model + " stopped!");
        } else {
            System.out.println("Car is not running!");
        }
    }
    
    public void accelerate(int amount) {
        if (!isRunning) {
            System.out.println("Start the car first!");
            return;
        }
        speed = Math.min(speed + amount, 200); // max 200 km/h
        System.out.printf("Speed: %d km/h%n", speed);
    }
    
    public void brake(int amount) {
        speed = Math.max(speed - amount, 0);
        System.out.printf("Speed: %d km/h%n", speed);
    }
    
    public String getInfo() {
        return String.format("%d %s %s (%s) - ฿%,.0f", year, brand, model, color, price);
    }
    
    // ====== toString ======
    @Override
    public String toString() {
        return String.format("Car{brand='%s', model='%s', year=%d, price=%.0f, color='%s', running=%b}",
            brand, model, year, price, color, isRunning);
    }
    
    // ====== equals ======
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Car)) return false;
        Car other = (Car) obj;
        return brand.equals(other.brand) && model.equals(other.model) && year == other.year;
    }
    
    // ====== hashCode ======
    @Override
    public int hashCode() {
        return java.util.Objects.hash(brand, model, year);
    }
}
```

```java
public class CarDemo {
    
    public static void main(String[] args) {
        
        // ====== สร้าง Objects ======
        Car car1 = new Car();  // Default constructor
        Car car2 = new Car("Toyota", "Camry", 2024, 1500000, "Silver");
        Car car3 = new Car("Honda", "Civic", 2023, 900000, "Red");
        Car car4 = new Car(car2);  // Copy constructor
        
        // ====== ใช้ Methods ======
        System.out.println(car2.getInfo());
        
        car2.start();
        car2.accelerate(50);
        car2.accelerate(80);
        car2.brake(30);
        car2.stop();
        
        // ====== Static Method ======
        System.out.println("Total cars created: " + Car.getTotalCarsCreated());
        
        // ====== toString ======
        System.out.println(car3);
        
        // ====== equals ======
        Car car5 = new Car("Toyota", "Camry", 2024, 1500000, "Blue");
        System.out.println("car2.equals(car4): " + car2.equals(car4));  // true (same brand/model/year)
        System.out.println("car2.equals(car3): " + car2.equals(car3));  // false
        
        // ====== Array of Objects ======
        Car[] fleet = {car1, car2, car3, car4};
        System.out.println("\n=== Fleet ===");
        for (Car car : fleet) {
            System.out.println(car.getInfo());
        }
        
        // ====== Null Object ======
        Car nullCar = null;
        // nullCar.start(); // NullPointerException!
        if (nullCar != null) {
            nullCar.start();
        }
    }
}
```

---

## 7.3 Constructors ขั้นสูง

```java
public class Person {
    
    private String firstName;
    private String lastName;
    private int age;
    private String email;
    private String phone;
    
    // ====== Constructor Chaining ======
    
    public Person(String firstName, String lastName) {
        this(firstName, lastName, 0); // เรียก constructor อื่น
    }
    
    public Person(String firstName, String lastName, int age) {
        this(firstName, lastName, age, null); // เรียก constructor อื่น
    }
    
    public Person(String firstName, String lastName, int age, String email) {
        this(firstName, lastName, age, email, null);
    }
    
    // Main constructor
    public Person(String firstName, String lastName, int age, String email, String phone) {
        if (firstName == null || firstName.isEmpty())
            throw new IllegalArgumentException("First name cannot be empty");
        if (lastName == null || lastName.isEmpty())
            throw new IllegalArgumentException("Last name cannot be empty");
        if (age < 0 || age > 150)
            throw new IllegalArgumentException("Invalid age: " + age);
        
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
        this.email = email;
        this.phone = phone;
    }
    
    // ====== Builder Pattern ======
    // (แนะนำสำหรับ class ที่มี parameter เยอะ)
    
    public static class Builder {
        // Required
        private final String firstName;
        private final String lastName;
        
        // Optional
        private int age = 0;
        private String email = null;
        private String phone = null;
        
        public Builder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Person build() {
            return new Person(firstName, lastName, age, email, phone);
        }
    }
    
    // Getters
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public String getFullName() { return firstName + " " + lastName; }
    public int getAge() { return age; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
    
    @Override
    public String toString() {
        return String.format("Person{name='%s', age=%d, email='%s'}", 
            getFullName(), age, email);
    }
    
    public static void main(String[] args) {
        
        // Traditional Constructor
        Person p1 = new Person("John", "Doe");
        Person p2 = new Person("Jane", "Doe", 25);
        Person p3 = new Person("Bob", "Smith", 30, "bob@example.com");
        
        System.out.println(p1);
        System.out.println(p2);
        System.out.println(p3);
        
        // Builder Pattern (readable)
        Person p4 = new Person.Builder("Alice", "Johnson")
            .age(28)
            .email("alice@example.com")
            .phone("0812345678")
            .build();
        
        System.out.println(p4);
        System.out.println("Phone: " + p4.getPhone());
    }
}
```

---

## 7.4 Encapsulation

```java
public class BankAccount {
    
    private final String accountNumber;
    private final String ownerName;
    private double balance;
    private boolean isActive;
    private java.util.List<String> transactions;
    
    private static final double MIN_BALANCE = 100.0;
    private static final double MAX_WITHDRAWAL_PER_DAY = 50000.0;
    private double withdrawnToday = 0;
    
    public BankAccount(String accountNumber, String ownerName, double initialBalance) {
        if (initialBalance < MIN_BALANCE) {
            throw new IllegalArgumentException(
                String.format("Initial balance must be at least ฿%.2f", MIN_BALANCE));
        }
        this.accountNumber = accountNumber;
        this.ownerName = ownerName;
        this.balance = initialBalance;
        this.isActive = true;
        this.transactions = new java.util.ArrayList<>();
        addTransaction("เปิดบัญชี", initialBalance);
    }
    
    // ====== Getters (Read-only) ======
    public String getAccountNumber() { return accountNumber; }
    public String getOwnerName() { return ownerName; }
    public double getBalance() { return balance; }
    public boolean isActive() { return isActive; }
    
    // ====== Operations ======
    
    public void deposit(double amount) {
        validateActive();
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
        addTransaction("ฝากเงิน", amount);
        System.out.printf("ฝากเงิน ฿%,.2f สำเร็จ ยอดคงเหลือ: ฿%,.2f%n", amount, balance);
    }
    
    public void withdraw(double amount) {
        validateActive();
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance - MIN_BALANCE) {
            throw new IllegalStateException("ยอดเงินไม่เพียงพอ (ต้องเก็บขั้นต่ำ ฿" + MIN_BALANCE + ")");
        }
        if (withdrawnToday + amount > MAX_WITHDRAWAL_PER_DAY) {
            throw new IllegalStateException("เกินวงเงินถอนต่อวัน");
        }
        balance -= amount;
        withdrawnToday += amount;
        addTransaction("ถอนเงิน", -amount);
        System.out.printf("ถอนเงิน ฿%,.2f สำเร็จ ยอดคงเหลือ: ฿%,.2f%n", amount, balance);
    }
    
    public void transfer(BankAccount target, double amount) {
        this.withdraw(amount);
        target.deposit(amount);
        System.out.printf("โอน ฿%,.2f ไปยังบัญชี %s สำเร็จ%n", amount, target.getAccountNumber());
    }
    
    public void printStatement() {
        System.out.println("\n═══════════════════════════════════════");
        System.out.printf("บัญชี: %s  ชื่อ: %s%n", accountNumber, ownerName);
        System.out.println("═══════════════════════════════════════");
        for (String tx : transactions) {
            System.out.println(tx);
        }
        System.out.printf("ยอดคงเหลือ: ฿%,.2f%n", balance);
        System.out.println("═══════════════════════════════════════");
    }
    
    private void validateActive() {
        if (!isActive) throw new IllegalStateException("บัญชีถูกปิดแล้ว");
    }
    
    private void addTransaction(String type, double amount) {
        String tx = String.format("[%s] %s: ฿%,.2f", 
            java.time.LocalDateTime.now().toString().substring(0, 16), type, Math.abs(amount));
        transactions.add(tx);
    }
    
    public void closeAccount() {
        validateActive();
        isActive = false;
        System.out.println("บัญชี " + accountNumber + " ถูกปิดแล้ว");
    }
    
    @Override
    public String toString() {
        return String.format("BankAccount{acc='%s', owner='%s', balance=%.2f, active=%b}",
            accountNumber, ownerName, balance, isActive);
    }
    
    public static void main(String[] args) {
        BankAccount acc1 = new BankAccount("ACC001", "สมชาย ใจดี", 5000);
        BankAccount acc2 = new BankAccount("ACC002", "สมหญิง รักสวย", 3000);
        
        acc1.deposit(2000);
        acc1.withdraw(1500);
        acc1.transfer(acc2, 1000);
        
        acc1.printStatement();
        acc2.printStatement();
        
        // Error handling
        try {
            acc1.withdraw(100000); // เกิน max
        } catch (IllegalStateException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

---

## 7.5 Records (Java 16+)

```java
// Record - class แบบง่ายสำหรับข้อมูล (immutable)
public record Point(double x, double y) {
    
    // Compact constructor สำหรับ validation
    public Point {
        if (Double.isNaN(x) || Double.isNaN(y)) {
            throw new IllegalArgumentException("Coordinates cannot be NaN");
        }
    }
    
    // Additional methods
    public double distanceTo(Point other) {
        double dx = this.x - other.x;
        double dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    public Point translate(double dx, double dy) {
        return new Point(x + dx, y + dy);
    }
    
    public static Point origin() {
        return new Point(0, 0);
    }
    
    public static void main(String[] args) {
        Point p1 = new Point(3.0, 4.0);
        Point p2 = new Point(0.0, 0.0);
        
        System.out.println("p1: " + p1);          // Point[x=3.0, y=4.0]
        System.out.println("x: " + p1.x());       // auto-generated getter
        System.out.println("y: " + p1.y());
        
        System.out.printf("Distance: %.2f%n", p1.distanceTo(p2));
        
        Point p3 = p1.translate(1, 1);
        System.out.println("Translated: " + p3);
        
        // equals, hashCode, toString auto-generated
        System.out.println("Equal: " + p1.equals(new Point(3.0, 4.0)));
    }
}
```

---

## 7.6 โปรแกรมตัวอย่าง: Library Management System

```java
import java.util.*;

// Book class
class Book {
    private final String isbn;
    private String title;
    private String author;
    private int year;
    private boolean isAvailable;
    
    public Book(String isbn, String title, String author, int year) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.year = year;
        this.isAvailable = true;
    }
    
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public int getYear() { return year; }
    public boolean isAvailable() { return isAvailable; }
    
    public void borrow() {
        if (!isAvailable) throw new IllegalStateException("หนังสือถูกยืมแล้ว");
        isAvailable = false;
    }
    
    public void returnBook() {
        isAvailable = true;
    }
    
    @Override
    public String toString() {
        return String.format("[%s] %s โดย %s (%d) %s",
            isbn, title, author, year, isAvailable ? "✓ ว่าง" : "✗ ยืมอยู่");
    }
}

// Member class
class Member {
    private final String memberId;
    private String name;
    private List<Book> borrowedBooks;
    private static final int MAX_BOOKS = 3;
    
    public Member(String memberId, String name) {
        this.memberId = memberId;
        this.name = name;
        this.borrowedBooks = new ArrayList<>();
    }
    
    public String getMemberId() { return memberId; }
    public String getName() { return name; }
    public List<Book> getBorrowedBooks() { return Collections.unmodifiableList(borrowedBooks); }
    
    public void borrowBook(Book book) {
        if (borrowedBooks.size() >= MAX_BOOKS) {
            throw new IllegalStateException("ยืมได้สูงสุด " + MAX_BOOKS + " เล่ม");
        }
        book.borrow();
        borrowedBooks.add(book);
        System.out.printf("%s ยืม '%s' สำเร็จ%n", name, book.getTitle());
    }
    
    public void returnBook(Book book) {
        if (!borrowedBooks.remove(book)) {
            throw new IllegalArgumentException("ไม่พบหนังสือในรายการยืม");
        }
        book.returnBook();
        System.out.printf("%s คืน '%s' สำเร็จ%n", name, book.getTitle());
    }
    
    @Override
    public String toString() {
        return String.format("Member{id='%s', name='%s', books=%d/%d}",
            memberId, name, borrowedBooks.size(), MAX_BOOKS);
    }
}

// Library class
public class Library {
    private List<Book> books = new ArrayList<>();
    private List<Member> members = new ArrayList<>();
    
    public void addBook(Book book) {
        books.add(book);
        System.out.println("เพิ่มหนังสือ: " + book.getTitle());
    }
    
    public void addMember(Member member) {
        members.add(member);
        System.out.println("เพิ่มสมาชิก: " + member.getName());
    }
    
    public Book findBook(String isbn) {
        return books.stream()
                   .filter(b -> b.getIsbn().equals(isbn))
                   .findFirst()
                   .orElse(null);
    }
    
    public Member findMember(String memberId) {
        return members.stream()
                     .filter(m -> m.getMemberId().equals(memberId))
                     .findFirst()
                     .orElse(null);
    }
    
    public List<Book> searchByTitle(String keyword) {
        List<Book> result = new ArrayList<>();
        for (Book b : books) {
            if (b.getTitle().toLowerCase().contains(keyword.toLowerCase())) {
                result.add(b);
            }
        }
        return result;
    }
    
    public void printAvailableBooks() {
        System.out.println("\n=== หนังสือที่ว่าง ===");
        books.stream()
             .filter(Book::isAvailable)
             .forEach(System.out::println);
    }
    
    public void printAllBooks() {
        System.out.println("\n=== หนังสือทั้งหมด ===");
        books.forEach(System.out::println);
    }
    
    public static void main(String[] args) {
        Library library = new Library();
        
        // เพิ่มหนังสือ
        library.addBook(new Book("001", "Clean Code", "Robert Martin", 2008));
        library.addBook(new Book("002", "Effective Java", "Joshua Bloch", 2018));
        library.addBook(new Book("003", "Design Patterns", "Gang of Four", 1994));
        library.addBook(new Book("004", "The Pragmatic Programmer", "Hunt & Thomas", 2019));
        library.addBook(new Book("005", "Java: The Complete Reference", "Herbert Schildt", 2022));
        
        // เพิ่มสมาชิก
        library.addMember(new Member("M001", "สมชาย"));
        library.addMember(new Member("M002", "สมหญิง"));
        
        // ยืมหนังสือ
        Member member1 = library.findMember("M001");
        Book book1 = library.findBook("001");
        Book book2 = library.findBook("002");
        
        member1.borrowBook(book1);
        member1.borrowBook(book2);
        
        library.printAllBooks();
        library.printAvailableBooks();
        
        // คืนหนังสือ
        member1.returnBook(book1);
        library.printAvailableBooks();
        
        // ค้นหา
        System.out.println("\n=== ค้นหา 'Java' ===");
        library.searchByTitle("Java").forEach(System.out::println);
        
        System.out.println("\n" + member1);
    }
}
```

---

## สรุป Part 07

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Class & Object | blueprint, instances, fields, methods |
| Constructors | default, parameterized, copy, chaining |
| Encapsulation | private fields, getters/setters |
| Builder Pattern | สำหรับ class ที่มี parameter เยอะ |
| Records | immutable data class (Java 16+) |
| toString, equals, hashCode | override method สำคัญ |

---

## ขั้นตอนต่อไป

➡️ [Part 08: OOP - Inheritance (การสืบทอด)](./Part-08-OOP-Inheritance.md)
