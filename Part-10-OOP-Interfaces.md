# Part 10: OOP - Interfaces ขั้นสูงและ Design Patterns
## ขั้นตอนที่ 611-680: Interface ในระดับ Professional

---

## 10.1 Interface ขั้นสูง (Java 8-21+)

```java
import java.util.*;
import java.util.function.*;

// ====== Default Methods ======
interface Collection<E> {
    void add(E element);
    void remove(E element);
    int size();
    boolean contains(E element);
    
    // Default methods
    default boolean isEmpty() {
        return size() == 0;
    }
    
    default void addAll(java.util.Collection<E> elements) {
        elements.forEach(this::add);
    }
    
    default void printAll() {
        System.out.println("Size: " + size());
    }
}

// ====== Interface with Generics ======
interface Transformer<I, O> {
    O transform(I input);
    
    default <R> Transformer<I, R> andThen(Transformer<O, R> after) {
        return input -> after.transform(this.transform(input));
    }
    
    static <T> Transformer<T, T> identity() {
        return t -> t;
    }
}

// ====== Sealed Interface (Java 17+) ======
sealed interface Result<T> permits Result.Success, Result.Failure {
    
    record Success<T>(T value) implements Result<T> {
        public boolean isSuccess() { return true; }
    }
    
    record Failure<T>(String error, Exception cause) implements Result<T> {
        public Failure(String error) { this(error, null); }
        public boolean isSuccess() { return false; }
    }
    
    default boolean isSuccess() { return false; }
    
    default T getOrElse(T defaultValue) {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> defaultValue;
        };
    }
    
    default <R> Result<R> map(Function<T, R> mapper) {
        return switch (this) {
            case Success<T> s -> {
                try {
                    yield new Success<>(mapper.apply(s.value()));
                } catch (Exception e) {
                    yield new Failure<>(e.getMessage(), e);
                }
            }
            case Failure<T> f -> new Failure<>(f.error(), f.cause());
        };
    }
    
    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }
    
    static <T> Result<T> failure(String error) {
        return new Failure<>(error);
    }
    
    static <T> Result<T> attempt(Supplier<T> supplier) {
        try {
            return success(supplier.get());
        } catch (Exception e) {
            return failure(e.getMessage());
        }
    }
}

public class AdvancedInterfaceDemo {
    
    public static void main(String[] args) {
        
        // ====== Transformer chaining ======
        Transformer<String, Integer> length = String::length;
        Transformer<Integer, Boolean> isLong = n -> n > 5;
        Transformer<String, Boolean> isLongString = length.andThen(isLong);
        
        String[] words = {"Hi", "Hello", "Java", "Programming"};
        for (String w : words) {
            System.out.printf("'%s' is long: %b%n", w, isLongString.transform(w));
        }
        
        // ====== Result type ======
        System.out.println("\n=== Result Type ===");
        
        Result<Integer> success = Result.attempt(() -> Integer.parseInt("42"));
        Result<Integer> failure = Result.attempt(() -> Integer.parseInt("not a number"));
        
        System.out.println("Success: " + success.isSuccess());
        System.out.println("Value: " + success.getOrElse(-1));
        System.out.println("Failure: " + failure.isSuccess());
        System.out.println("Default: " + failure.getOrElse(-1));
        
        // map
        Result<String> mapped = success.map(n -> "Number: " + n);
        System.out.println("Mapped: " + mapped.getOrElse("Error"));
        
        // Pattern matching with sealed interface
        for (Result<Integer> result : List.of(success, failure)) {
            String message = switch (result) {
                case Result.Success<Integer> s -> "Got: " + s.value();
                case Result.Failure<Integer> f -> "Error: " + f.error();
            };
            System.out.println(message);
        }
    }
}
```

---

## 10.2 Design Patterns ด้วย Interface

### Factory Pattern

```java
// ====== Abstract Factory Pattern ======

interface Button {
    void render();
    void onClick();
}

interface TextField {
    void render();
    String getValue();
    void setValue(String value);
}

interface UIFactory {
    Button createButton(String label);
    TextField createTextField(String placeholder);
}

// Windows UI
class WindowsButton implements Button {
    private String label;
    public WindowsButton(String label) { this.label = label; }
    @Override public void render() { System.out.println("[Windows Button: " + label + "]"); }
    @Override public void onClick() { System.out.println("Windows button clicked: " + label); }
}

class WindowsTextField implements TextField {
    private String placeholder;
    private String value = "";
    
    public WindowsTextField(String placeholder) { this.placeholder = placeholder; }
    
    @Override public void render() {
        System.out.println("[Windows TextField: " + (value.isEmpty() ? placeholder : value) + "]");
    }
    @Override public String getValue() { return value; }
    @Override public void setValue(String value) { this.value = value; }
}

class WindowsUIFactory implements UIFactory {
    @Override public Button createButton(String label) { return new WindowsButton(label); }
    @Override public TextField createTextField(String placeholder) { return new WindowsTextField(placeholder); }
}

// macOS UI
class MacButton implements Button {
    private String label;
    public MacButton(String label) { this.label = label; }
    @Override public void render() { System.out.println("(Mac Button: " + label + ")"); }
    @Override public void onClick() { System.out.println("Mac button clicked: " + label); }
}

class MacTextField implements TextField {
    private String placeholder;
    private String value = "";
    
    public MacTextField(String placeholder) { this.placeholder = placeholder; }
    
    @Override public void render() {
        System.out.println("(Mac TextField: " + (value.isEmpty() ? placeholder : value) + ")");
    }
    @Override public String getValue() { return value; }
    @Override public void setValue(String value) { this.value = value; }
}

class MacUIFactory implements UIFactory {
    @Override public Button createButton(String label) { return new MacButton(label); }
    @Override public TextField createTextField(String placeholder) { return new MacTextField(placeholder); }
}

class LoginForm {
    private Button loginButton;
    private Button cancelButton;
    private TextField usernameField;
    private TextField passwordField;
    
    public LoginForm(UIFactory factory) {
        loginButton = factory.createButton("เข้าสู่ระบบ");
        cancelButton = factory.createButton("ยกเลิก");
        usernameField = factory.createTextField("ชื่อผู้ใช้");
        passwordField = factory.createTextField("รหัสผ่าน");
    }
    
    public void render() {
        System.out.println("=== Login Form ===");
        usernameField.render();
        passwordField.render();
        loginButton.render();
        cancelButton.render();
    }
    
    public void fillAndSubmit(String username, String password) {
        usernameField.setValue(username);
        passwordField.setValue("****");
        System.out.println("Submitting with user: " + username);
        loginButton.onClick();
    }
}

public class AbstractFactoryDemo {
    
    public static void main(String[] args) {
        
        // Detect OS and use appropriate factory
        String os = System.getProperty("os.name", "Windows").toLowerCase();
        UIFactory factory = os.contains("mac") ? new MacUIFactory() : new WindowsUIFactory();
        
        System.out.println("Using factory: " + factory.getClass().getSimpleName());
        
        LoginForm form = new LoginForm(factory);
        form.render();
        System.out.println();
        form.fillAndSubmit("admin", "secret");
        
        // Force different factory
        System.out.println("\n--- Mac version ---");
        LoginForm macForm = new LoginForm(new MacUIFactory());
        macForm.render();
    }
}
```

### Decorator Pattern

```java
// ====== Decorator Pattern ======

interface Coffee {
    double getCost();
    String getDescription();
}

class SimpleCoffee implements Coffee {
    @Override public double getCost() { return 50; }
    @Override public String getDescription() { return "กาแฟ"; }
}

abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription();
    }
    
    @Override
    public double getCost() {
        return coffee.getCost();
    }
}

class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    
    @Override
    public double getCost() { return coffee.getCost() + 15; }
    
    @Override
    public String getDescription() { return coffee.getDescription() + " + นม"; }
}

class SugarDecorator extends CoffeeDecorator {
    private int teaspoons;
    public SugarDecorator(Coffee coffee, int teaspoons) {
        super(coffee);
        this.teaspoons = teaspoons;
    }
    
    @Override
    public double getCost() { return coffee.getCost() + (teaspoons * 5); }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + " + น้ำตาล×" + teaspoons;
    }
}

class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) { super(coffee); }
    
    @Override
    public double getCost() { return coffee.getCost() + 20; }
    
    @Override
    public String getDescription() { return coffee.getDescription() + " + วิปครีม"; }
}

class SizeDecorator extends CoffeeDecorator {
    private String size;
    private double multiplier;
    
    public SizeDecorator(Coffee coffee, String size) {
        super(coffee);
        this.size = size;
        this.multiplier = switch (size) {
            case "S" -> 0.8;
            case "M" -> 1.0;
            case "L" -> 1.3;
            case "XL" -> 1.5;
            default -> 1.0;
        };
    }
    
    @Override
    public double getCost() { return coffee.getCost() * multiplier; }
    
    @Override
    public String getDescription() { return coffee.getDescription() + " [" + size + "]"; }
}

public class DecoratorPatternDemo {
    
    public static void main(String[] args) {
        
        // สร้างกาแฟหลายสูตร
        Coffee coffee1 = new SimpleCoffee();
        System.out.printf("%-30s ฿%.2f%n", coffee1.getDescription(), coffee1.getCost());
        
        Coffee latte = new MilkDecorator(
                           new SugarDecorator(
                               new SimpleCoffee(), 2));
        System.out.printf("%-30s ฿%.2f%n", latte.getDescription(), latte.getCost());
        
        Coffee cappuccino = new WhipDecorator(
                               new MilkDecorator(
                                   new SugarDecorator(
                                       new SimpleCoffee(), 1)));
        System.out.printf("%-30s ฿%.2f%n", cappuccino.getDescription(), cappuccino.getCost());
        
        // Large version
        Coffee largeCappuccino = new SizeDecorator(cappuccino, "L");
        System.out.printf("%-30s ฿%.2f%n", largeCappuccino.getDescription(), largeCappuccino.getCost());
        
        // Menu system
        System.out.println("\n=== Coffee Menu ===");
        Coffee[] menu = {
            new SimpleCoffee(),
            new SizeDecorator(new MilkDecorator(new SimpleCoffee()), "S"),
            new SizeDecorator(new MilkDecorator(new SimpleCoffee()), "M"),
            new SizeDecorator(new MilkDecorator(new SimpleCoffee()), "L"),
            cappuccino,
            largeCappuccino
        };
        
        for (Coffee c : menu) {
            System.out.printf("%-40s ฿%6.2f%n", c.getDescription(), c.getCost());
        }
    }
}
```

---

## 10.3 Comparable และ Comparator

```java
import java.util.*;

// ====== Comparable ======
class Student implements Comparable<Student> {
    private String name;
    private double gpa;
    private int age;
    
    public Student(String name, double gpa, int age) {
        this.name = name;
        this.gpa = gpa;
        this.age = age;
    }
    
    // Natural ordering: by GPA (desc), then name (asc)
    @Override
    public int compareTo(Student other) {
        int gpaCompare = Double.compare(other.gpa, this.gpa); // desc
        if (gpaCompare != 0) return gpaCompare;
        return this.name.compareTo(other.name); // asc
    }
    
    @Override
    public String toString() {
        return String.format("%-10s GPA:%.2f Age:%d", name, gpa, age);
    }
    
    // Getters
    public String getName() { return name; }
    public double getGpa() { return gpa; }
    public int getAge() { return age; }
}

public class ComparableComparatorDemo {
    
    public static void main(String[] args) {
        
        List<Student> students = Arrays.asList(
            new Student("Charlie", 3.5, 20),
            new Student("Alice", 3.8, 22),
            new Student("Bob", 3.8, 21),
            new Student("David", 3.2, 19),
            new Student("Eve", 3.9, 23)
        );
        
        // Natural ordering (Comparable)
        System.out.println("=== Natural Order (GPA desc, Name asc) ===");
        Collections.sort(students);
        students.forEach(System.out::println);
        
        // Custom Comparator
        System.out.println("\n=== By Age ===");
        students.sort(Comparator.comparingInt(Student::getAge));
        students.forEach(System.out::println);
        
        System.out.println("\n=== By Name ===");
        students.sort(Comparator.comparing(Student::getName));
        students.forEach(System.out::println);
        
        // Chained comparators
        System.out.println("\n=== By GPA desc, then Age asc ===");
        students.sort(
            Comparator.comparingDouble(Student::getGpa).reversed()
                      .thenComparingInt(Student::getAge)
        );
        students.forEach(System.out::println);
        
        // Finding
        Optional<Student> topStudent = students.stream()
            .max(Comparator.comparingDouble(Student::getGpa));
        System.out.println("\nTop student: " + topStudent.orElse(null));
        
        // TreeSet (sorted automatically)
        TreeSet<Student> sortedSet = new TreeSet<>(students);
        System.out.println("\nTreeSet (natural order):");
        sortedSet.forEach(System.out::println);
    }
}
```

---

## 10.4 Iterable Interface

```java
import java.util.Iterator;
import java.util.NoSuchElementException;

// Custom Iterable
class Range implements Iterable<Integer> {
    private final int start;
    private final int end;
    private final int step;
    
    public Range(int start, int end, int step) {
        if (step == 0) throw new IllegalArgumentException("Step cannot be zero");
        this.start = start;
        this.end = end;
        this.step = step;
    }
    
    public Range(int start, int end) { this(start, end, 1); }
    
    public static Range of(int end) { return new Range(0, end); }
    
    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            private int current = start;
            
            @Override
            public boolean hasNext() {
                return step > 0 ? current < end : current > end;
            }
            
            @Override
            public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                int value = current;
                current += step;
                return value;
            }
        };
    }
    
    public static void main(String[] args) {
        
        // Forward
        System.out.print("Range(1, 11): ");
        for (int n : new Range(1, 11)) {
            System.out.print(n + " ");
        }
        System.out.println();
        
        // With step
        System.out.print("Range(0, 20, 3): ");
        for (int n : new Range(0, 20, 3)) {
            System.out.print(n + " ");
        }
        System.out.println();
        
        // Backward
        System.out.print("Range(10, 0, -1): ");
        for (int n : new Range(10, 0, -1)) {
            System.out.print(n + " ");
        }
        System.out.println();
        
        // Using in stream
        Range.of(10).forEach(n -> System.out.print(n * n + " "));
        System.out.println();
    }
}
```

---

## 10.5 Cloneable Interface

```java
// Cloneable สำหรับ deep copy
class Address implements Cloneable {
    String street;
    String city;
    
    public Address(String street, String city) {
        this.street = street;
        this.city = city;
    }
    
    @Override
    protected Address clone() {
        try {
            return (Address) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    public String toString() { return street + ", " + city; }
}

class Employee implements Cloneable {
    String name;
    int age;
    Address address; // reference type
    
    public Employee(String name, int age, Address address) {
        this.name = name;
        this.age = age;
        this.address = address;
    }
    
    // Shallow Clone
    public Employee shallowClone() {
        try {
            return (Employee) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    // Deep Clone
    @Override
    protected Employee clone() {
        try {
            Employee cloned = (Employee) super.clone();
            cloned.address = address.clone(); // clone reference types
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    public String toString() {
        return String.format("Employee{name='%s', age=%d, address=%s}", name, age, address);
    }
    
    public static void main(String[] args) {
        
        Employee original = new Employee("Alice", 30, new Address("123 Main St", "Bangkok"));
        
        // Shallow clone
        Employee shallow = original.shallowClone();
        System.out.println("Original: " + original);
        System.out.println("Shallow: " + shallow);
        
        // Modifying shallow clone's address affects original
        shallow.address.city = "Chiang Mai";
        System.out.println("\nAfter modifying shallow clone's city:");
        System.out.println("Original: " + original);    // Bangkok changed!
        System.out.println("Shallow: " + shallow);
        
        // Reset
        original.address.city = "Bangkok";
        
        // Deep clone
        Employee deep = original.clone();
        deep.address.city = "Phuket";
        
        System.out.println("\nAfter modifying deep clone's city:");
        System.out.println("Original: " + original);  // Bangkok unchanged
        System.out.println("Deep: " + deep);
    }
}
```

---

## 10.6 Full Program: E-Commerce System

```java
import java.util.*;

// Interfaces
interface Priceable {
    double getPrice();
    String getCurrency();
    default String getFormattedPrice() {
        return String.format("%s %.2f", getCurrency(), getPrice());
    }
}

interface Discountable {
    double applyDiscount(double discountPercent);
    double getOriginalPrice();
}

interface Categorizable {
    String getCategory();
    List<String> getTags();
}

// Product class
class Product implements Priceable, Categorizable {
    private final String id;
    private final String name;
    private double price;
    private final String category;
    private List<String> tags;
    private int stockQuantity;
    
    public Product(String id, String name, double price, String category, int stock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.category = category;
        this.tags = new ArrayList<>();
        this.stockQuantity = stock;
    }
    
    public void addTag(String tag) { tags.add(tag); }
    
    @Override public double getPrice() { return price; }
    @Override public String getCurrency() { return "฿"; }
    @Override public String getCategory() { return category; }
    @Override public List<String> getTags() { return Collections.unmodifiableList(tags); }
    
    public String getId() { return id; }
    public String getName() { return name; }
    public int getStockQuantity() { return stockQuantity; }
    
    public boolean isAvailable() { return stockQuantity > 0; }
    
    public void sell(int qty) {
        if (qty > stockQuantity) throw new IllegalStateException("Insufficient stock");
        stockQuantity -= qty;
    }
    
    @Override
    public String toString() {
        return String.format("[%s] %s - %s (Stock: %d)", 
            id, name, getFormattedPrice(), stockQuantity);
    }
}

// Cart Item
record CartItem(Product product, int quantity) {
    public double subtotal() { return product.getPrice() * quantity; }
}

// Shopping Cart
class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();
    private String discountCode;
    private static final Map<String, Double> DISCOUNT_CODES = Map.of(
        "SAVE10", 0.10,
        "SAVE20", 0.20,
        "NEWUSER", 0.15
    );
    
    public void addItem(Product product, int quantity) {
        if (!product.isAvailable()) throw new IllegalStateException("Out of stock: " + product.getName());
        if (quantity > product.getStockQuantity()) throw new IllegalStateException("Not enough stock");
        
        items.add(new CartItem(product, quantity));
        System.out.printf("Added: %s ×%d = ฿%.2f%n", product.getName(), quantity, 
            product.getPrice() * quantity);
    }
    
    public void applyDiscount(String code) {
        if (DISCOUNT_CODES.containsKey(code)) {
            this.discountCode = code;
            System.out.println("Discount code applied: " + code + " (-" + 
                (int)(DISCOUNT_CODES.get(code) * 100) + "%)");
        } else {
            System.out.println("Invalid code: " + code);
        }
    }
    
    public double getSubtotal() {
        return items.stream().mapToDouble(CartItem::subtotal).sum();
    }
    
    public double getDiscount() {
        if (discountCode == null) return 0;
        return getSubtotal() * DISCOUNT_CODES.getOrDefault(discountCode, 0.0);
    }
    
    public double getTotal() {
        return getSubtotal() - getDiscount();
    }
    
    public void printReceipt() {
        System.out.println("\n╔═══════════════════════════════════════════╗");
        System.out.println("║              ใบเสร็จ / Receipt             ║");
        System.out.println("╠═══════════════════════════════════════════╣");
        
        for (CartItem item : items) {
            System.out.printf("║ %-20s ×%2d = ฿%8.2f ║%n",
                item.product().getName(), item.quantity(), item.subtotal());
        }
        
        System.out.println("╠═══════════════════════════════════════════╣");
        System.out.printf("║ %-29s ฿%8.2f ║%n", "ยอดรวม:", getSubtotal());
        
        if (getDiscount() > 0) {
            System.out.printf("║ %-28s -฿%8.2f ║%n", "ส่วนลด (" + discountCode + "):", getDiscount());
        }
        
        System.out.println("╠═══════════════════════════════════════════╣");
        System.out.printf("║ %-29s ฿%8.2f ║%n", "ยอดสุทธิ:", getTotal());
        System.out.println("╚═══════════════════════════════════════════╝");
    }
    
    public List<CartItem> getItems() { return Collections.unmodifiableList(items); }
    
    public void checkout() {
        for (CartItem item : items) {
            item.product().sell(item.quantity());
        }
        System.out.println("✅ ชำระเงินสำเร็จ!");
        items.clear();
    }
}

// Store
class Store {
    private final String name;
    private final List<Product> inventory = new ArrayList<>();
    
    public Store(String name) { this.name = name; }
    
    public void addProduct(Product product) { inventory.add(product); }
    
    public Optional<Product> findById(String id) {
        return inventory.stream().filter(p -> p.getId().equals(id)).findFirst();
    }
    
    public List<Product> findByCategory(String category) {
        return inventory.stream()
                       .filter(p -> p.getCategory().equalsIgnoreCase(category))
                       .filter(Product::isAvailable)
                       .toList();
    }
    
    public List<Product> findByMaxPrice(double maxPrice) {
        return inventory.stream()
                       .filter(p -> p.getPrice() <= maxPrice && p.isAvailable())
                       .sorted(Comparator.comparingDouble(Product::getPrice))
                       .toList();
    }
    
    public void printInventory() {
        System.out.println("\n=== " + name + " - สินค้า ===");
        inventory.forEach(System.out::println);
    }
}

public class ECommerceSystem {
    
    public static void main(String[] args) {
        
        Store store = new Store("Java Shop");
        
        // เพิ่มสินค้า
        Product laptop = new Product("P001", "MacBook Air M2", 45000, "Electronics", 10);
        laptop.addTag("apple"); laptop.addTag("laptop");
        
        Product phone = new Product("P002", "iPhone 15", 35000, "Electronics", 20);
        phone.addTag("apple"); phone.addTag("smartphone");
        
        Product book1 = new Product("P003", "Clean Code", 800, "Books", 50);
        book1.addTag("programming"); book1.addTag("best-practices");
        
        Product book2 = new Product("P004", "Effective Java", 900, "Books", 30);
        book2.addTag("java"); book2.addTag("programming");
        
        Product earphone = new Product("P005", "AirPods Pro", 9500, "Electronics", 15);
        
        store.addProduct(laptop);
        store.addProduct(phone);
        store.addProduct(book1);
        store.addProduct(book2);
        store.addProduct(earphone);
        
        store.printInventory();
        
        // ค้นหา
        System.out.println("\n=== Electronics ===");
        store.findByCategory("Electronics").forEach(System.out::println);
        
        System.out.println("\n=== สินค้าราคา ≤ ฿1000 ===");
        store.findByMaxPrice(1000).forEach(System.out::println);
        
        // ซื้อของ
        System.out.println("\n=== ตะกร้าสินค้า ===");
        ShoppingCart cart = new ShoppingCart();
        cart.addItem(laptop, 1);
        cart.addItem(book1, 2);
        cart.addItem(book2, 1);
        
        cart.applyDiscount("SAVE10");
        cart.printReceipt();
        
        System.out.println();
        cart.checkout();
        
        // ตรวจสอบ stock หลัง checkout
        System.out.println("\nAfter checkout:");
        store.findById("P003").ifPresent(System.out::println);
    }
}
```

---

## สรุป Part 10 และ ภาค 1 (Part 01-10)

```
╔═══════════════════════════════════════════════════════════════╗
║          สรุปภาค 1: Java พื้นฐาน (Part 01-10)               ║
╠═══════════════════════════════════════════════════════════════╣
║  Part 01: Java Basics, JDK, Hello World                      ║
║  Part 02: Variables, Data Types, Operators                   ║
║  Part 03: Control Flow (if/else, switch)                     ║
║  Part 04: Loops (for, while, do-while, recursion)            ║
║  Part 05: Arrays & Multidimensional Arrays                   ║
║  Part 06: Methods & Functions                                ║
║  Part 07: OOP - Classes & Objects                            ║
║  Part 08: OOP - Inheritance                                  ║
║  Part 09: OOP - Polymorphism & Abstraction                   ║
║  Part 10: OOP - Interfaces & Design Patterns                 ║
╚═══════════════════════════════════════════════════════════════╝
```

### สิ่งที่เรียนรู้ใน Part 10:
| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Advanced Interface | Default/Static methods, Sealed Interface |
| Result Type | Error handling ด้วย sealed interface |
| Factory Pattern | Abstract Factory |
| Decorator Pattern | เพิ่มฟีเจอร์แบบ dynamic |
| Comparable/Comparator | การเรียงลำดับ |
| Iterable | custom iteration |
| E-Commerce System | โปรแกรมขนาดใหญ่ |

---

## ขั้นตอนต่อไป

➡️ [Part 11: Exception Handling](./Part-11-Exception-Handling.md)
