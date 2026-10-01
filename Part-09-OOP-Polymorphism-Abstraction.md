# Part 09: OOP - Polymorphism และ Abstraction
## ขั้นตอนที่ 541-610: หลักการขั้นสูงของ OOP

---

## 9.1 Polymorphism (พหุรูป)

Polymorphism หมายถึงความสามารถของ Object ที่จะมีหลายรูปแบบ มีสองประเภทหลัก:
1. **Compile-time (Method Overloading)**
2. **Runtime (Method Overriding)**

```java
// ตัวอย่าง Polymorphism แบบครบถ้วน
abstract class Shape {
    protected String color;
    
    public Shape(String color) { this.color = color; }
    
    // Abstract methods
    public abstract double area();
    public abstract double perimeter();
    public abstract void draw();
    
    // Concrete method
    public void display() {
        System.out.printf("Shape: %s | Color: %s | Area: %.2f | Perimeter: %.2f%n",
            getClass().getSimpleName(), color, area(), perimeter());
    }
}

class Circle extends Shape {
    private double radius;
    
    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }
    
    @Override
    public double area() { return Math.PI * radius * radius; }
    
    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
    
    @Override
    public void draw() {
        System.out.println("Drawing circle with radius " + radius);
    }
}

class Rectangle extends Shape {
    protected double width, height;
    
    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double area() { return width * height; }
    
    @Override
    public double perimeter() { return 2 * (width + height); }
    
    @Override
    public void draw() {
        System.out.println("Drawing rectangle " + width + "×" + height);
    }
}

class Triangle extends Shape {
    private double a, b, c;
    
    public Triangle(String color, double a, double b, double c) {
        super(color);
        this.a = a; this.b = b; this.c = c;
    }
    
    @Override
    public double area() {
        double s = (a + b + c) / 2;
        return Math.sqrt(s * (s-a) * (s-b) * (s-c)); // Heron's formula
    }
    
    @Override
    public double perimeter() { return a + b + c; }
    
    @Override
    public void draw() {
        System.out.println("Drawing triangle " + a + "+" + b + "+" + c);
    }
}

public class PolymorphismDemo {
    
    // ฟังก์ชันที่รับ Shape และทำงานได้กับทุก subclass
    public static void processShape(Shape shape) {
        shape.draw();
        shape.display();
    }
    
    public static double totalArea(Shape[] shapes) {
        double total = 0;
        for (Shape s : shapes) total += s.area();
        return total;
    }
    
    public static Shape findLargest(Shape[] shapes) {
        Shape largest = shapes[0];
        for (Shape s : shapes) {
            if (s.area() > largest.area()) largest = s;
        }
        return largest;
    }
    
    public static void main(String[] args) {
        
        Shape[] shapes = {
            new Circle("Red", 5),
            new Rectangle("Blue", 4, 6),
            new Triangle("Green", 3, 4, 5),
            new Circle("Yellow", 3),
            new Rectangle("Purple", 10, 2)
        };
        
        System.out.println("=== All Shapes ===");
        for (Shape s : shapes) {
            processShape(s);
        }
        
        System.out.printf("%nTotal area: %.2f%n", totalArea(shapes));
        
        Shape largest = findLargest(shapes);
        System.out.println("Largest: " + largest.getClass().getSimpleName() + 
                          " (area=" + String.format("%.2f", largest.area()) + ")");
        
        // ====== Compile-time Polymorphism (Overloading) ======
        System.out.println("\n=== Overloading ===");
        Calculator calc = new Calculator();
        System.out.println("add(5, 3) = " + calc.add(5, 3));
        System.out.println("add(5.5, 3.2) = " + calc.add(5.5, 3.2));
        System.out.println("add(1, 2, 3) = " + calc.add(1, 2, 3));
        System.out.println("add('Hello', ' World') = " + calc.add("Hello", " World"));
    }
}

class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
    public String add(String a, String b) { return a + b; }
}
```

---

## 9.2 Interface

```java
// Interface คือ contract ที่ class ต้องปฏิบัติตาม
interface Drawable {
    // Abstract methods (implicitly public abstract)
    void draw();
    void resize(double factor);
    
    // Default method (Java 8+)
    default void drawWithBorder() {
        System.out.println("╔══════════╗");
        draw();
        System.out.println("╚══════════╝");
    }
    
    // Static method (Java 8+)
    static Drawable create(String type) {
        return switch (type) {
            case "circle" -> new Circle2("black", 5);
            case "rect" -> new Rectangle2("black", 4, 3);
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
    
    // Constant
    double DEFAULT_SIZE = 1.0; // implicitly public static final
}

interface Colorable {
    void setColor(String color);
    String getColor();
    
    default void printColor() {
        System.out.println("Color: " + getColor());
    }
}

interface Saveable {
    void save(String filename);
    void load(String filename);
}

// Multiple Interface Implementation
class Circle2 implements Drawable, Colorable {
    private String color;
    private double radius;
    
    public Circle2(String color, double radius) {
        this.color = color;
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.printf("Drawing %s circle (r=%.1f)%n", color, radius);
    }
    
    @Override
    public void resize(double factor) {
        radius *= factor;
        System.out.println("Resized to r=" + radius);
    }
    
    @Override
    public void setColor(String color) { this.color = color; }
    
    @Override
    public String getColor() { return color; }
}

class Rectangle2 implements Drawable, Colorable, Saveable {
    private String color;
    private double width, height;
    
    public Rectangle2(String color, double width, double height) {
        this.color = color;
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.printf("Drawing %s rectangle (%.1f×%.1f)%n", color, width, height);
    }
    
    @Override
    public void resize(double factor) {
        width *= factor;
        height *= factor;
    }
    
    @Override
    public void setColor(String color) { this.color = color; }
    
    @Override
    public String getColor() { return color; }
    
    @Override
    public void save(String filename) {
        System.out.println("Saving rectangle to " + filename);
    }
    
    @Override
    public void load(String filename) {
        System.out.println("Loading from " + filename);
    }
}

public class InterfaceDemo {
    
    public static void main(String[] args) {
        
        // ====== Interface usage ======
        Drawable circle = new Circle2("red", 5);
        Drawable rect = new Rectangle2("blue", 4, 3);
        
        circle.draw();
        circle.drawWithBorder();
        circle.resize(2);
        circle.draw();
        
        // ====== Interface as type ======
        Drawable[] drawables = {circle, rect};
        System.out.println("\n=== Drawing all ===");
        for (Drawable d : drawables) {
            d.draw();
        }
        
        // ====== instanceof with interface ======
        System.out.println("\n=== instanceof check ===");
        for (Drawable d : drawables) {
            System.out.print(d.getClass().getSimpleName() + " is: ");
            if (d instanceof Colorable c) {
                System.out.print("Colorable (color=" + c.getColor() + ") ");
            }
            if (d instanceof Saveable) {
                System.out.print("Saveable ");
            }
            System.out.println();
        }
        
        // ====== Static factory method ======
        Drawable fromFactory = Drawable.create("circle");
        fromFactory.draw();
        
        // ====== Functional Interface + Lambda ======
        Drawable lambda = new Drawable() {
            @Override
            public void draw() { System.out.println("Lambda draw!"); }
            @Override
            public void resize(double factor) {}
        };
        lambda.draw();
    }
}
```

---

## 9.3 Interface ขั้นสูง

```java
import java.util.*;
import java.util.function.*;

// ====== Functional Interfaces ======
@FunctionalInterface
interface MathOperation {
    double calculate(double a, double b);
    
    // Static factory methods
    static MathOperation add() { return (a, b) -> a + b; }
    static MathOperation subtract() { return (a, b) -> a - b; }
    static MathOperation multiply() { return (a, b) -> a * b; }
    static MathOperation divide() { return (a, b) -> {
        if (b == 0) throw new ArithmeticException("Division by zero");
        return a / b;
    }};
    
    // Composition
    default MathOperation andThen(MathOperation after, double secondArg) {
        return (a, b) -> after.calculate(this.calculate(a, b), secondArg);
    }
}

@FunctionalInterface
interface Transformer<T> {
    T transform(T input);
    
    default Transformer<T> andThen(Transformer<T> after) {
        return input -> after.transform(this.transform(input));
    }
}

// ====== Interface Segregation ======
// ❌ Fat Interface
interface Worker {
    void work();
    void eat();
    void sleep();
    void manage();  // ไม่ใช่ทุก worker ต้อง manage
}

// ✅ Segregated Interfaces
interface Workable { void work(); }
interface Eatable { void eat(); }
interface Sleepable { void sleep(); }
interface Manageable { void manage(); }

class Developer implements Workable, Eatable, Sleepable {
    private String name;
    public Developer(String name) { this.name = name; }
    
    @Override public void work() { System.out.println(name + " กำลัง coding"); }
    @Override public void eat() { System.out.println(name + " กินข้าว"); }
    @Override public void sleep() { System.out.println(name + " นอนหลับ"); }
}

class Manager implements Workable, Eatable, Sleepable, Manageable {
    private String name;
    public Manager(String name) { this.name = name; }
    
    @Override public void work() { System.out.println(name + " ทำงาน"); }
    @Override public void eat() { System.out.println(name + " กินข้าว"); }
    @Override public void sleep() { System.out.println(name + " นอน"); }
    @Override public void manage() { System.out.println(name + " ประชุม"); }
}

public class AdvancedInterface {
    
    public static void main(String[] args) {
        
        // Functional Interface
        MathOperation add = MathOperation.add();
        MathOperation multiply = MathOperation.multiply();
        
        System.out.printf("5 + 3 = %.1f%n", add.calculate(5, 3));
        System.out.printf("5 * 3 = %.1f%n", multiply.calculate(5, 3));
        
        // Lambda
        MathOperation power = (a, b) -> Math.pow(a, b);
        System.out.printf("2^10 = %.1f%n", power.calculate(2, 10));
        
        // Transformer
        Transformer<String> trim = String::trim;
        Transformer<String> upper = String::toUpperCase;
        Transformer<String> trimAndUpper = trim.andThen(upper);
        
        System.out.println(trimAndUpper.transform("  hello world  "));
        
        // Interface segregation
        Developer dev = new Developer("Alice");
        Manager mgr = new Manager("Bob");
        
        dev.work();
        mgr.manage();
        
        // Using interface type
        List<Workable> workers = Arrays.asList(dev, mgr);
        workers.forEach(Workable::work);
    }
}
```

---

## 9.4 Strategy Pattern (Design Pattern with Interface)

```java
import java.util.*;

// Strategy Interface
interface SortStrategy {
    void sort(int[] arr);
    String getName();
}

// Concrete Strategies
class BubbleSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] arr) {
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j]; arr[j] = arr[j + 1]; arr[j + 1] = temp;
                }
            }
        }
    }
    @Override public String getName() { return "Bubble Sort"; }
}

class QuickSortStrategy implements SortStrategy {
    @Override
    public void sort(int[] arr) {
        quickSort(arr, 0, arr.length - 1);
    }
    
    private void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pi = partition(arr, low, high);
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
    }
    
    private int partition(int[] arr, int low, int high) {
        int pivot = arr[high], i = low - 1;
        for (int j = low; j < high; j++) {
            if (arr[j] <= pivot) {
                i++;
                int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
            }
        }
        int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
        return i + 1;
    }
    
    @Override public String getName() { return "Quick Sort"; }
}

// Context class ที่ใช้ Strategy
class Sorter {
    private SortStrategy strategy;
    
    public Sorter(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    public int[] sortAndMeasure(int[] arr) {
        int[] copy = Arrays.copyOf(arr, arr.length);
        long start = System.nanoTime();
        strategy.sort(copy);
        long elapsed = System.nanoTime() - start;
        System.out.printf("%s: %.3f ms%n", strategy.getName(), elapsed / 1e6);
        return copy;
    }
}

public class StrategyPatternDemo {
    
    public static void main(String[] args) {
        Random random = new Random(42);
        int[] arr = new int[1000];
        for (int i = 0; i < arr.length; i++) arr[i] = random.nextInt(10000);
        
        Sorter sorter = new Sorter(new BubbleSortStrategy());
        int[] result1 = sorter.sortAndMeasure(arr);
        
        sorter.setStrategy(new QuickSortStrategy());
        int[] result2 = sorter.sortAndMeasure(arr);
        
        // Using Lambda as Strategy
        sorter.setStrategy(new SortStrategy() {
            @Override
            public void sort(int[] a) { Arrays.sort(a); }
            @Override
            public String getName() { return "Java Arrays.sort (TimSort)"; }
        });
        int[] result3 = sorter.sortAndMeasure(arr);
        
        // Verify all results are same
        System.out.println("All results equal: " + 
            Arrays.equals(result1, result2) && Arrays.equals(result2, result3));
    }
}
```

---

## 9.5 Observer Pattern

```java
import java.util.*;

// Observer Interface
interface Observer {
    void update(String event, Object data);
}

// Observable Interface
interface Observable {
    void addObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers(String event, Object data);
}

// Event Store (Observable)
class EventBus implements Observable {
    private Map<String, List<Observer>> listeners = new HashMap<>();
    
    public void subscribe(String event, Observer observer) {
        listeners.computeIfAbsent(event, k -> new ArrayList<>()).add(observer);
    }
    
    public void publish(String event, Object data) {
        List<Observer> eventListeners = listeners.getOrDefault(event, Collections.emptyList());
        for (Observer o : eventListeners) {
            o.update(event, data);
        }
    }
    
    @Override
    public void addObserver(Observer observer) {
        subscribe("all", observer);
    }
    
    @Override
    public void removeObserver(Observer observer) {
        listeners.values().forEach(list -> list.remove(observer));
    }
    
    @Override
    public void notifyObservers(String event, Object data) {
        publish(event, data);
    }
}

// Stock Market Example
class StockMarket implements Observable {
    private Map<String, Double> stocks = new HashMap<>();
    private List<Observer> observers = new ArrayList<>();
    
    @Override
    public void addObserver(Observer o) { observers.add(o); }
    
    @Override
    public void removeObserver(Observer o) { observers.remove(o); }
    
    @Override
    public void notifyObservers(String event, Object data) {
        observers.forEach(o -> o.update(event, data));
    }
    
    public void updateStock(String symbol, double price) {
        double oldPrice = stocks.getOrDefault(symbol, 0.0);
        stocks.put(symbol, price);
        
        Map<String, Object> data = new HashMap<>();
        data.put("symbol", symbol);
        data.put("price", price);
        data.put("oldPrice", oldPrice);
        data.put("change", price - oldPrice);
        data.put("changePercent", oldPrice > 0 ? (price - oldPrice) / oldPrice * 100 : 0);
        
        notifyObservers("PRICE_UPDATE", data);
    }
}

class Investor implements Observer {
    private String name;
    private Map<String, Integer> portfolio = new HashMap<>();
    private double alertThreshold;
    
    public Investor(String name, double alertThreshold) {
        this.name = name;
        this.alertThreshold = alertThreshold;
    }
    
    public void addPosition(String symbol, int shares) {
        portfolio.put(symbol, shares);
    }
    
    @Override
    @SuppressWarnings("unchecked")
    public void update(String event, Object data) {
        if (!"PRICE_UPDATE".equals(event)) return;
        
        Map<String, Object> update = (Map<String, Object>) data;
        String symbol = (String) update.get("symbol");
        
        if (!portfolio.containsKey(symbol)) return;
        
        double price = (Double) update.get("price");
        double changePercent = (Double) update.get("changePercent");
        int shares = portfolio.get(symbol);
        double value = price * shares;
        
        if (Math.abs(changePercent) >= alertThreshold) {
            System.out.printf("[%s] Alert! %s: %.2f%% change | Portfolio value: ฿%,.0f%n",
                name, symbol, changePercent, value);
        }
    }
}

public class ObserverPatternDemo {
    
    public static void main(String[] args) {
        StockMarket market = new StockMarket();
        
        Investor investor1 = new Investor("Alice", 2.0); // alert if 2%+ change
        Investor investor2 = new Investor("Bob", 5.0);   // alert if 5%+ change
        
        investor1.addPosition("AAPL", 100);
        investor1.addPosition("GOOGL", 50);
        investor2.addPosition("AAPL", 200);
        investor2.addPosition("MSFT", 150);
        
        market.addObserver(investor1);
        market.addObserver(investor2);
        
        System.out.println("=== Stock Updates ===");
        market.updateStock("AAPL", 150.00);
        market.updateStock("AAPL", 153.50);  // +2.33%
        market.updateStock("AAPL", 162.00);  // +5.55%
        market.updateStock("GOOGL", 140.00);
        market.updateStock("MSFT", 380.00);
        market.updateStock("MSFT", 361.00);  // -5%
    }
}
```

---

## 9.6 Abstraction ในทางปฏิบัติ

```java
// ตัวอย่างการใช้ Abstraction ในระบบจริง

// ====== Database Abstraction ======
interface DataRepository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void deleteById(ID id);
    boolean existsById(ID id);
}

// Entity
record User(int id, String name, String email) {}

// In-Memory Implementation
class InMemoryUserRepository implements DataRepository<User, Integer> {
    private Map<Integer, User> storage = new HashMap<>();
    private int nextId = 1;
    
    @Override
    public User findById(Integer id) {
        return storage.get(id);
    }
    
    @Override
    public List<User> findAll() {
        return new ArrayList<>(storage.values());
    }
    
    @Override
    public User save(User user) {
        if (user.id() == 0) {
            User newUser = new User(nextId++, user.name(), user.email());
            storage.put(newUser.id(), newUser);
            return newUser;
        }
        storage.put(user.id(), user);
        return user;
    }
    
    @Override
    public void deleteById(Integer id) {
        storage.remove(id);
    }
    
    @Override
    public boolean existsById(Integer id) {
        return storage.containsKey(id);
    }
}

// Service Layer ที่ใช้ Repository
class UserService {
    private final DataRepository<User, Integer> repository;
    
    public UserService(DataRepository<User, Integer> repository) {
        this.repository = repository;
    }
    
    public User createUser(String name, String email) {
        if (name == null || name.isEmpty()) throw new IllegalArgumentException("Name required");
        if (email == null || !email.contains("@")) throw new IllegalArgumentException("Invalid email");
        
        return repository.save(new User(0, name, email));
    }
    
    public User getUserById(int id) {
        User user = repository.findById(id);
        if (user == null) throw new NoSuchElementException("User not found: " + id);
        return user;
    }
    
    public List<User> getAllUsers() {
        return repository.findAll();
    }
    
    public void deleteUser(int id) {
        if (!repository.existsById(id)) throw new NoSuchElementException("User not found: " + id);
        repository.deleteById(id);
    }
}

public class AbstractionDemo {
    
    public static void main(String[] args) {
        // ใช้ In-Memory implementation
        DataRepository<User, Integer> repo = new InMemoryUserRepository();
        UserService service = new UserService(repo);
        
        // Create users
        User u1 = service.createUser("Alice", "alice@example.com");
        User u2 = service.createUser("Bob", "bob@example.com");
        User u3 = service.createUser("Charlie", "charlie@example.com");
        
        System.out.println("Created: " + u1);
        System.out.println("Created: " + u2);
        System.out.println("Created: " + u3);
        
        // Get all
        System.out.println("\nAll users: " + service.getAllUsers());
        
        // Get by id
        System.out.println("User 2: " + service.getUserById(2));
        
        // Delete
        service.deleteUser(2);
        System.out.println("After delete: " + service.getAllUsers());
        
        // Error handling
        try {
            service.getUserById(2);
        } catch (NoSuchElementException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
}
```

---

## 9.7 แบบฝึกหัด Part 09

### แบบฝึกหัด: Payment System

```java
// สร้างระบบชำระเงินที่รองรับหลายวิธี

interface PaymentProcessor {
    boolean processPayment(double amount, String currency);
    boolean refund(String transactionId, double amount);
    String getPaymentMethod();
}

interface TransactionLogger {
    void logTransaction(String type, double amount, boolean success);
    List<String> getTransactionHistory();
}

abstract class BasePaymentProcessor implements PaymentProcessor, TransactionLogger {
    protected List<String> transactionHistory = new ArrayList<>();
    protected String merchantId;
    
    public BasePaymentProcessor(String merchantId) {
        this.merchantId = merchantId;
    }
    
    @Override
    public void logTransaction(String type, double amount, boolean success) {
        String log = String.format("[%s] %s: ฿%.2f - %s", 
            java.time.LocalDateTime.now().toString().substring(0, 16),
            type, amount, success ? "SUCCESS" : "FAILED");
        transactionHistory.add(log);
        System.out.println(log);
    }
    
    @Override
    public List<String> getTransactionHistory() {
        return Collections.unmodifiableList(transactionHistory);
    }
}

class CreditCardProcessor extends BasePaymentProcessor {
    private String cardNumber;
    private String expiryDate;
    
    public CreditCardProcessor(String merchantId, String cardNumber, String expiry) {
        super(merchantId);
        this.cardNumber = maskCard(cardNumber);
        this.expiryDate = expiry;
    }
    
    private String maskCard(String card) {
        return "**** **** **** " + card.substring(card.length() - 4);
    }
    
    @Override
    public boolean processPayment(double amount, String currency) {
        System.out.println("Processing credit card: " + cardNumber);
        boolean success = amount <= 100000; // mock: limit 100k
        logTransaction("CHARGE", amount, success);
        return success;
    }
    
    @Override
    public boolean refund(String transactionId, double amount) {
        System.out.println("Refunding to card: " + cardNumber);
        logTransaction("REFUND", amount, true);
        return true;
    }
    
    @Override
    public String getPaymentMethod() { return "Credit Card (" + cardNumber + ")"; }
}

class PromptPayProcessor extends BasePaymentProcessor {
    private String phoneNumber;
    
    public PromptPayProcessor(String merchantId, String phone) {
        super(merchantId);
        this.phoneNumber = phone;
    }
    
    @Override
    public boolean processPayment(double amount, String currency) {
        System.out.println("PromptPay QR Code generated for ฿" + amount);
        System.out.println("Phone: " + phoneNumber);
        logTransaction("PROMPTPAY", amount, true);
        return true;
    }
    
    @Override
    public boolean refund(String transactionId, double amount) {
        logTransaction("REFUND", amount, true);
        return true;
    }
    
    @Override
    public String getPaymentMethod() { return "PromptPay (" + phoneNumber + ")"; }
}

class PaymentGateway {
    private List<PaymentProcessor> processors = new ArrayList<>();
    
    public void addProcessor(PaymentProcessor processor) {
        processors.add(processor);
    }
    
    public boolean pay(double amount, String method) {
        for (PaymentProcessor p : processors) {
            if (p.getPaymentMethod().toLowerCase().contains(method.toLowerCase())) {
                return p.processPayment(amount, "THB");
            }
        }
        System.out.println("Payment method not found: " + method);
        return false;
    }
    
    public void printAllTransactions() {
        System.out.println("\n=== Transaction History ===");
        for (PaymentProcessor p : processors) {
            System.out.println("--- " + p.getPaymentMethod() + " ---");
            if (p instanceof TransactionLogger logger) {
                logger.getTransactionHistory().forEach(System.out::println);
            }
        }
    }
}

public class PaymentSystemDemo {
    
    public static void main(String[] args) {
        PaymentGateway gateway = new PaymentGateway();
        gateway.addProcessor(new CreditCardProcessor("MERCHANT001", "4532123456789012", "12/26"));
        gateway.addProcessor(new PromptPayProcessor("MERCHANT001", "0812345678"));
        
        System.out.println("=== Processing Payments ===");
        gateway.pay(1500.00, "credit");
        gateway.pay(250.00, "promptpay");
        gateway.pay(200000.00, "credit"); // will fail - exceeds limit
        
        gateway.printAllTransactions();
    }
}
```

---

## สรุป Part 09

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Polymorphism | Compile-time, Runtime |
| Interface | basic, default methods, static methods |
| Multiple Interfaces | implement หลาย interface |
| Functional Interfaces | @FunctionalInterface, lambda |
| Strategy Pattern | design pattern ด้วย interface |
| Observer Pattern | event-driven design |
| Abstraction | repository pattern, layer separation |

---

## ขั้นตอนต่อไป

➡️ [Part 10: OOP - Interfaces ขั้นสูง](./Part-10-OOP-Interfaces.md)
