# Part 08: OOP - Inheritance (การสืบทอด)
## ขั้นตอนที่ 471-540: การสืบทอดใน Java

---

## 8.1 Inheritance พื้นฐาน

```java
// Parent Class (Superclass)
public class Animal {
    
    protected String name;
    protected int age;
    protected String sound;
    protected double weight;
    
    public Animal(String name, int age, double weight) {
        this.name = name;
        this.age = age;
        this.weight = weight;
    }
    
    // Methods
    public void makeSound() {
        System.out.println(name + " says: " + sound);
    }
    
    public void eat(String food) {
        System.out.println(name + " กำลังกิน " + food);
    }
    
    public void sleep() {
        System.out.println(name + " กำลังนอน");
    }
    
    public void breathe() {
        System.out.println(name + " กำลังหายใจ");
    }
    
    public String getInfo() {
        return String.format("%s (อายุ %d ปี, %.1f กก.)", name, age, weight);
    }
    
    @Override
    public String toString() {
        return String.format("Animal{name='%s', age=%d, weight=%.1f}", name, age, weight);
    }
}

// Child Class (Subclass) - สืบทอดจาก Animal
public class Dog extends Animal {
    
    private String breed; // สายพันธุ์
    private boolean isTrained;
    
    public Dog(String name, int age, double weight, String breed) {
        super(name, age, weight); // เรียก parent constructor
        this.breed = breed;
        this.isTrained = false;
        this.sound = "Woof!";
    }
    
    // Override method จาก parent
    @Override
    public void makeSound() {
        System.out.println(name + " เห่า: " + sound + " " + sound);
    }
    
    // Method เฉพาะของ Dog
    public void fetch(String item) {
        System.out.println(name + " วิ่งไปหยิบ " + item);
    }
    
    public void train() {
        isTrained = true;
        System.out.println(name + " ได้รับการฝึกแล้ว!");
    }
    
    public void sit() {
        if (isTrained) {
            System.out.println(name + " นั่ง");
        } else {
            System.out.println(name + " ยังไม่ได้รับการฝึก");
        }
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + " [" + breed + "]" + (isTrained ? " ✓ฝึกแล้ว" : "");
    }
    
    public String getBreed() { return breed; }
    public boolean isTrained() { return isTrained; }
    
    @Override
    public String toString() {
        return String.format("Dog{name='%s', breed='%s', trained=%b}", name, breed, isTrained);
    }
}

// Another Child Class
public class Cat extends Animal {
    
    private boolean isIndoor;
    private int livesLeft = 9;
    
    public Cat(String name, int age, double weight, boolean isIndoor) {
        super(name, age, weight);
        this.isIndoor = isIndoor;
        this.sound = "Meow";
    }
    
    @Override
    public void makeSound() {
        System.out.println(name + " ร้อง: " + sound + "~");
    }
    
    public void purr() {
        System.out.println(name + " กรรกริก... ♪");
    }
    
    public void scratch(String surface) {
        System.out.println(name + " ข่วน " + surface);
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + (isIndoor ? " [แมวบ้าน]" : " [แมวนอก]");
    }
}

// Usage
public class InheritanceDemo {
    
    public static void main(String[] args) {
        
        // สร้าง Objects
        Animal generic = new Animal("Animal", 1, 5.0);
        Dog dog = new Dog("Buddy", 3, 15.5, "Golden Retriever");
        Cat cat = new Cat("Luna", 2, 4.2, true);
        
        // ====== Polymorphism ======
        Animal[] animals = {generic, dog, cat};
        
        System.out.println("=== ข้อมูลสัตว์ ===");
        for (Animal animal : animals) {
            System.out.println(animal.getInfo());
            animal.makeSound();  // เรียก method ที่ override แล้ว
            System.out.println();
        }
        
        // ====== Dog-specific methods ======
        dog.fetch("ลูกบอล");
        dog.train();
        dog.sit();
        dog.eat("กระดูก");
        
        // ====== instanceof check ======
        System.out.println("\n=== instanceof ===");
        for (Animal a : animals) {
            System.out.print(a.name + " is: ");
            if (a instanceof Dog d) {
                System.out.println("Dog (" + d.getBreed() + ")");
            } else if (a instanceof Cat c) {
                System.out.println("Cat");
            } else {
                System.out.println("Generic Animal");
            }
        }
    }
}
```

---

## 8.2 Multi-level Inheritance

```java
// Grandparent
class Vehicle {
    protected String brand;
    protected int year;
    protected double price;
    
    public Vehicle(String brand, int year, double price) {
        this.brand = brand;
        this.year = year;
        this.price = price;
    }
    
    public void start() { System.out.println(brand + " เริ่มทำงาน"); }
    public void stop() { System.out.println(brand + " หยุดทำงาน"); }
    public String getInfo() { return brand + " (" + year + ") ฿" + String.format("%,.0f", price); }
}

// Parent (extends Vehicle)
class MotorVehicle extends Vehicle {
    protected int horsepower;
    protected String fuelType;
    protected double engineSize;
    
    public MotorVehicle(String brand, int year, double price, int horsepower, String fuelType) {
        super(brand, year, price);
        this.horsepower = horsepower;
        this.fuelType = fuelType;
    }
    
    public void refuel() { System.out.println("เติม" + fuelType); }
    
    @Override
    public String getInfo() {
        return super.getInfo() + " | " + horsepower + "HP " + fuelType;
    }
}

// Child (extends MotorVehicle)
class ElectricCar extends MotorVehicle {
    private double batteryCapacity;  // kWh
    private int chargingLevel;       // 0-100%
    
    public ElectricCar(String brand, int year, double price, int horsepower, double batteryCapacity) {
        super(brand, year, price, horsepower, "Electric");
        this.batteryCapacity = batteryCapacity;
        this.chargingLevel = 80;
    }
    
    public void charge(int targetLevel) {
        System.out.printf("กำลังชาร์จ %s จาก %d%% ถึง %d%%%n", brand, chargingLevel, targetLevel);
        chargingLevel = Math.min(targetLevel, 100);
    }
    
    public double estimatedRange() {
        return batteryCapacity * chargingLevel / 100 * 6; // 6 km/kWh
    }
    
    @Override
    public void refuel() {
        System.out.println("รถไฟฟ้าไม่ต้องเติมน้ำมัน - ใช้การชาร์จแทน");
    }
    
    @Override
    public String getInfo() {
        return super.getInfo() + " | Battery: " + batteryCapacity + "kWh | Range: " + 
               String.format("%.0f", estimatedRange()) + "km";
    }
}

class HybridCar extends MotorVehicle {
    private double batteryCapacity;
    private double fuelTankCapacity;
    
    public HybridCar(String brand, int year, double price, int horsepower) {
        super(brand, year, price, horsepower, "Hybrid");
        this.batteryCapacity = 8.8;
        this.fuelTankCapacity = 43;
    }
    
    public void switchMode(String mode) {
        System.out.println(brand + " สลับเป็นโหมด " + mode);
    }
}

public class MultiLevelInheritance {
    
    public static void main(String[] args) {
        ElectricCar tesla = new ElectricCar("Tesla Model 3", 2024, 2500000, 350, 75);
        HybridCar prius = new HybridCar("Toyota Prius", 2024, 1400000, 121);
        
        System.out.println(tesla.getInfo());
        tesla.start();
        tesla.charge(100);
        System.out.printf("ระยะทาง: %.0f km%n", tesla.estimatedRange());
        tesla.refuel();
        
        System.out.println();
        System.out.println(prius.getInfo());
        prius.refuel();
        prius.switchMode("EV");
    }
}
```

---

## 8.3 super Keyword

```java
public class SuperKeyword {
    
    static class Shape {
        protected String color;
        protected String name;
        
        public Shape(String name, String color) {
            this.name = name;
            this.color = color;
            System.out.println("Shape constructor: " + name);
        }
        
        public void draw() {
            System.out.println("Drawing " + color + " " + name);
        }
        
        public String describe() {
            return color + " " + name;
        }
    }
    
    static class Circle extends Shape {
        private double radius;
        
        public Circle(String color, double radius) {
            super("Circle", color); // เรียก parent constructor
            this.radius = radius;
            System.out.println("Circle constructor");
        }
        
        @Override
        public void draw() {
            super.draw(); // เรียก parent method
            System.out.printf("  Radius: %.1f, Area: %.2f%n", radius, Math.PI * radius * radius);
        }
        
        @Override
        public String describe() {
            return super.describe() + " with radius " + radius; // เรียก parent method
        }
    }
    
    static class ColoredCircle extends Circle {
        private String borderColor;
        
        public ColoredCircle(String fillColor, String borderColor, double radius) {
            super(fillColor, radius); // เรียก parent (Circle) constructor
            this.borderColor = borderColor;
        }
        
        @Override
        public void draw() {
            super.draw(); // เรียก Circle.draw()
            System.out.println("  Border: " + borderColor);
        }
        
        @Override
        public String describe() {
            return super.describe() + ", border: " + borderColor;
        }
    }
    
    public static void main(String[] args) {
        ColoredCircle cc = new ColoredCircle("Red", "Blue", 5.0);
        System.out.println();
        cc.draw();
        System.out.println(cc.describe());
    }
}
```

---

## 8.4 Abstract Classes

```java
// Abstract class - ไม่สามารถสร้าง instance ได้โดยตรง
abstract class Employee {
    
    protected String name;
    protected String id;
    protected String department;
    
    public Employee(String name, String id, String department) {
        this.name = name;
        this.id = id;
        this.department = department;
    }
    
    // Abstract method - ต้อง override ใน subclass
    public abstract double calculateSalary();
    public abstract String getJobTitle();
    
    // Concrete methods - มี implementation
    public void displayInfo() {
        System.out.println("═══════════════════════════");
        System.out.println("ชื่อ: " + name);
        System.out.println("รหัส: " + id);
        System.out.println("ตำแหน่ง: " + getJobTitle());
        System.out.println("แผนก: " + department);
        System.out.printf("เงินเดือน: ฿%,.2f%n", calculateSalary());
        System.out.println("═══════════════════════════");
    }
    
    public String getName() { return name; }
    public String getId() { return id; }
}

// Full-time Employee
class FullTimeEmployee extends Employee {
    private double baseSalary;
    private double bonusPercent;
    
    public FullTimeEmployee(String name, String id, String department, 
                           double baseSalary, double bonusPercent) {
        super(name, id, department);
        this.baseSalary = baseSalary;
        this.bonusPercent = bonusPercent;
    }
    
    @Override
    public double calculateSalary() {
        return baseSalary * (1 + bonusPercent / 100);
    }
    
    @Override
    public String getJobTitle() { return "พนักงานประจำ"; }
}

// Part-time Employee
class PartTimeEmployee extends Employee {
    private double hourlyRate;
    private int hoursWorked;
    
    public PartTimeEmployee(String name, String id, String department, 
                           double hourlyRate, int hoursWorked) {
        super(name, id, department);
        this.hourlyRate = hourlyRate;
        this.hoursWorked = hoursWorked;
    }
    
    @Override
    public double calculateSalary() {
        return hourlyRate * hoursWorked;
    }
    
    @Override
    public String getJobTitle() { return "พนักงานพาร์ทไทม์"; }
}

// Contractor
class Contractor extends Employee {
    private double dailyRate;
    private int daysWorked;
    private double vatRate = 0.07;
    
    public Contractor(String name, String id, String department, 
                     double dailyRate, int daysWorked) {
        super(name, id, department);
        this.dailyRate = dailyRate;
        this.daysWorked = daysWorked;
    }
    
    @Override
    public double calculateSalary() {
        double gross = dailyRate * daysWorked;
        return gross * (1 + vatRate);
    }
    
    @Override
    public String getJobTitle() { return "ผู้รับเหมา"; }
}

public class AbstractClassDemo {
    
    public static void main(String[] args) {
        
        // Employee[] employees = {new Employee(...)}; // Error! Cannot instantiate abstract class
        
        Employee[] employees = {
            new FullTimeEmployee("สมชาย ใจดี", "E001", "IT", 45000, 15),
            new PartTimeEmployee("สมหญิง รักงาน", "E002", "HR", 200, 80),
            new Contractor("บริษัท ABC", "C001", "ก่อสร้าง", 2000, 20)
        };
        
        double totalPayroll = 0;
        
        for (Employee emp : employees) {
            emp.displayInfo();
            totalPayroll += emp.calculateSalary();
        }
        
        System.out.printf("\nรวมเงินเดือนทั้งหมด: ฿%,.2f%n", totalPayroll);
        
        // instanceof check
        for (Employee emp : employees) {
            if (emp instanceof FullTimeEmployee fte) {
                System.out.println(fte.getName() + " เป็นพนักงานประจำ");
            }
        }
    }
}
```

---

## 8.5 final Keyword

```java
// final class - ไม่สามารถ extend ได้
final class ImmutablePoint {
    private final double x;  // final field - ไม่สามารถเปลี่ยนค่าได้
    private final double y;
    
    public ImmutablePoint(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    // final method - ไม่สามารถ override ได้
    public final double distanceTo(ImmutablePoint other) {
        double dx = this.x - other.x;
        double dy = this.y - other.y;
        return Math.sqrt(dx * dx + dy * dy);
    }
    
    @Override
    public String toString() {
        return String.format("(%.2f, %.2f)", x, y);
    }
}

// class ExtendedPoint extends ImmutablePoint {} // Error! Cannot inherit from final class

class ExampleClass {
    
    // final variable
    final int MAX_SIZE = 100;
    
    // final method - subclass ไม่สามารถ override ได้
    public final void criticalOperation() {
        System.out.println("This cannot be overridden");
    }
    
    // Non-final method - สามารถ override ได้
    public void normalOperation() {
        System.out.println("This can be overridden");
    }
}

class SubClass extends ExampleClass {
    // criticalOperation() cannot be overridden
    
    @Override
    public void normalOperation() {
        System.out.println("Overridden!");
    }
}

public class FinalDemo {
    
    public static void main(String[] args) {
        ImmutablePoint p1 = new ImmutablePoint(3, 4);
        ImmutablePoint p2 = new ImmutablePoint(0, 0);
        
        // p1.x = 5; // Error! Cannot assign to final field
        
        System.out.println("p1: " + p1);
        System.out.println("Distance: " + p1.distanceTo(p2));
        
        // final local variable
        final int MAX = 100;
        // MAX = 200; // Error!
        
        // final in lambda
        final String prefix = "Hello";
        Runnable r = () -> System.out.println(prefix + " World");
        r.run();
    }
}
```

---

## 8.6 Inheritance Best Practices

```java
/**
 * Design Principle: "Favor Composition over Inheritance"
 * ใช้ Composition แทน Inheritance เมื่อเป็นไปได้
 */
public class InheritanceBestPractices {
    
    // ====== IS-A vs HAS-A ======
    
    // ✅ IS-A (Inheritance): Dog IS-A Animal
    // class Dog extends Animal
    
    // ✅ HAS-A (Composition): Car HAS-A Engine
    // class Car { Engine engine; }
    
    // ❌ อย่าใช้ Inheritance เพียงแค่ต้องการ code reuse
    // ถ้าไม่มีความสัมพันธ์แบบ IS-A ให้ใช้ Composition แทน
    
    // Composition Example
    static class Engine {
        private int horsepower;
        private String type;
        
        public Engine(int hp, String type) {
            this.horsepower = hp;
            this.type = type;
        }
        
        public void start() { System.out.println(type + " engine started (" + horsepower + "HP)"); }
        public void stop() { System.out.println("Engine stopped"); }
        public int getHorsepower() { return horsepower; }
    }
    
    static class Car {
        private String brand;
        private Engine engine; // HAS-A relationship
        
        public Car(String brand, int hp, String engineType) {
            this.brand = brand;
            this.engine = new Engine(hp, engineType); // composition
        }
        
        public void start() {
            System.out.println(brand + " กำลังสตาร์ท...");
            engine.start();
        }
        
        public int getHorsepower() {
            return engine.getHorsepower();
        }
    }
    
    // ====== Liskov Substitution Principle ======
    // Subclass ต้องสามารถแทน Superclass ได้โดยไม่เสียความถูกต้อง
    
    static abstract class Shape {
        public abstract double area();
        
        public void printArea() {
            System.out.printf("Area = %.2f%n", area());
        }
    }
    
    static class Rectangle extends Shape {
        protected double width, height;
        
        public Rectangle(double width, double height) {
            this.width = width;
            this.height = height;
        }
        
        @Override
        public double area() { return width * height; }
    }
    
    // ✅ Square IS-A Rectangle (แต่ต้องระวัง)
    static class Square extends Rectangle {
        public Square(double side) {
            super(side, side);
        }
        
        // Override เพื่อรักษา invariant
        public void setWidth(double side) { this.width = this.height = side; }
        public void setHeight(double side) { this.width = this.height = side; }
    }
    
    public static void main(String[] args) {
        
        // Composition example
        Car car = new Car("BMW", 300, "Turbo");
        car.start();
        System.out.println("Power: " + car.getHorsepower() + "HP");
        
        // LSP example
        Shape[] shapes = {
            new Rectangle(4, 5),
            new Square(6)
        };
        
        for (Shape s : shapes) {
            s.printArea();
        }
    }
}
```

---

## 8.7 แบบฝึกหัด Part 08

### แบบฝึกหัด: School System

```java
// แบบฝึกหัด: สร้างระบบโรงเรียน

abstract class SchoolPerson {
    protected String name;
    protected int age;
    protected String id;
    
    public SchoolPerson(String name, int age, String id) {
        this.name = name;
        this.age = age;
        this.id = id;
    }
    
    public abstract String getRole();
    
    public void introduce() {
        System.out.printf("สวัสดี ฉันชื่อ %s เป็น%s (รหัส: %s)%n", name, getRole(), id);
    }
    
    public String getName() { return name; }
}

class Student extends SchoolPerson {
    private double gpa;
    private String major;
    
    public Student(String name, int age, String id, String major) {
        super(name, age, id);
        this.major = major;
        this.gpa = 0.0;
    }
    
    @Override
    public String getRole() { return "นักศึกษา"; }
    
    public void study(String subject) {
        System.out.println(name + " กำลังเรียน " + subject);
    }
    
    public void setGpa(double gpa) { this.gpa = gpa; }
    public double getGpa() { return gpa; }
    
    @Override
    public String toString() {
        return String.format("%s | %s | GPA: %.2f", name, major, gpa);
    }
}

class Teacher extends SchoolPerson {
    private String subject;
    private double salary;
    
    public Teacher(String name, int age, String id, String subject, double salary) {
        super(name, age, id);
        this.subject = subject;
        this.salary = salary;
    }
    
    @Override
    public String getRole() { return "อาจารย์"; }
    
    public void teach(String topic) {
        System.out.println(name + " กำลังสอนเรื่อง " + topic + " (วิชา: " + subject + ")");
    }
    
    public void gradeStudent(Student student, double score) {
        student.setGpa(score);
        System.out.printf("%s ให้คะแนน %s = %.2f%n", name, student.getName(), score);
    }
}

public class SchoolSystem {
    
    public static void main(String[] args) {
        Teacher t1 = new Teacher("อ.สมชาย", 45, "T001", "Java Programming", 50000);
        Teacher t2 = new Teacher("อ.สมหญิง", 38, "T002", "Data Structures", 48000);
        
        Student[] students = {
            new Student("นาย A", 20, "S001", "Computer Science"),
            new Student("นาย B", 21, "S002", "Computer Science"),
            new Student("นางสาว C", 20, "S003", "Software Engineering")
        };
        
        System.out.println("=== รายชื่อ ===");
        t1.introduce();
        for (Student s : students) s.introduce();
        
        System.out.println("\n=== การเรียนการสอน ===");
        t1.teach("OOP in Java");
        students[0].study("Java");
        
        System.out.println("\n=== การให้คะแนน ===");
        t1.gradeStudent(students[0], 3.75);
        t1.gradeStudent(students[1], 3.50);
        t2.gradeStudent(students[2], 3.90);
        
        System.out.println("\n=== ผลการเรียน ===");
        for (Student s : students) System.out.println(s);
        
        // Polymorphism
        System.out.println("\n=== บุคลากรทั้งหมด ===");
        SchoolPerson[] people = new SchoolPerson[students.length + 2];
        people[0] = t1;
        people[1] = t2;
        System.arraycopy(students, 0, people, 2, students.length);
        
        for (SchoolPerson p : people) {
            p.introduce();
        }
    }
}
```

---

## สรุป Part 08

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Inheritance | extends, super, override |
| Multi-level | 3+ levels of inheritance |
| Abstract Classes | abstract methods, concrete methods |
| final Keyword | final class, method, variable |
| super Keyword | calling parent constructor/method |
| Best Practices | IS-A, HAS-A, LSP |

---

## ขั้นตอนต่อไป

➡️ [Part 09: OOP - Polymorphism และ Abstraction](./Part-09-OOP-Polymorphism-Abstraction.md)
