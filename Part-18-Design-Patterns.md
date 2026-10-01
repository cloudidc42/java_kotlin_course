# Part 18: Design Patterns ขั้นสูง
## ขั้นตอนที่ 1141-1210: GoF Patterns ระดับมืออาชีพ

---

## 18.1 Creational Patterns

### Singleton Pattern

```java
// Thread-safe Singleton
public class AppConfig {
    private static volatile AppConfig instance;
    
    private final Map<String, String> properties = new HashMap<>();
    
    private AppConfig() {
        properties.put("app.name", "MyApp");
        properties.put("app.version", "1.0");
        properties.put("db.host", "localhost");
        properties.put("db.port", "5432");
    }
    
    public static AppConfig getInstance() {
        if (instance == null) {
            synchronized (AppConfig.class) {
                if (instance == null) {  // double-checked locking
                    instance = new AppConfig();
                }
            }
        }
        return instance;
    }
    
    public String get(String key) { return properties.getOrDefault(key, ""); }
    public void set(String key, String value) { properties.put(key, value); }
    
    // Enum Singleton (preferred)
    public enum DB {
        INSTANCE;
        
        private final String url = "jdbc:postgresql://localhost/mydb";
        private int connectionCount = 0;
        
        public String getUrl() { return url; }
        public int getConnectionCount() { return connectionCount++; }
    }
}
```

### Builder Pattern

```java
// Builder with validation
class Pizza {
    private final String size;          // required
    private final String crust;         // required
    private final List<String> toppings;
    private final boolean extraCheese;
    private final boolean thinCrust;
    
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.crust = builder.crust;
        this.toppings = List.copyOf(builder.toppings);
        this.extraCheese = builder.extraCheese;
        this.thinCrust = builder.thinCrust;
    }
    
    @Override
    public String toString() {
        return String.format("Pizza{size=%s, crust=%s, toppings=%s, cheese=%b, thin=%b}",
            size, crust, toppings, extraCheese, thinCrust);
    }
    
    public static class Builder {
        private final String size;
        private final String crust;
        private List<String> toppings = new ArrayList<>();
        private boolean extraCheese = false;
        private boolean thinCrust = false;
        
        public Builder(String size, String crust) {
            if (size == null || size.isBlank()) throw new IllegalArgumentException("size required");
            if (crust == null || crust.isBlank()) throw new IllegalArgumentException("crust required");
            this.size = size;
            this.crust = crust;
        }
        
        public Builder addTopping(String topping) {
            toppings.add(topping);
            return this;
        }
        
        public Builder extraCheese() { this.extraCheese = true; return this; }
        public Builder thinCrust() { this.thinCrust = true; return this; }
        
        public Pizza build() { return new Pizza(this); }
    }
    
    public static void main(String[] args) {
        Pizza margherita = new Builder("Large", "Classic")
            .addTopping("Mozzarella")
            .addTopping("Tomato")
            .extraCheese()
            .build();
        
        Pizza bbqChicken = new Builder("Medium", "Thin")
            .addTopping("BBQ Chicken")
            .addTopping("Onion")
            .addTopping("Bacon")
            .thinCrust()
            .build();
        
        System.out.println(margherita);
        System.out.println(bbqChicken);
    }
}
```

### Prototype Pattern

```java
import java.util.*;

interface Prototype<T> {
    T deepClone();
}

class GameCharacter implements Prototype<GameCharacter> {
    private String name;
    private int health;
    private int attack;
    private List<String> abilities;
    private Map<String, Integer> stats;
    
    public GameCharacter(String name, int health, int attack) {
        this.name = name;
        this.health = health;
        this.attack = attack;
        this.abilities = new ArrayList<>();
        this.stats = new HashMap<>();
    }
    
    public void addAbility(String ability) { abilities.add(ability); }
    public void setStat(String stat, int value) { stats.put(stat, value); }
    
    @Override
    public GameCharacter deepClone() {
        GameCharacter clone = new GameCharacter(name, health, attack);
        clone.abilities = new ArrayList<>(this.abilities);
        clone.stats = new HashMap<>(this.stats);
        return clone;
    }
    
    @Override
    public String toString() {
        return String.format("%s(HP:%d, ATK:%d, abilities:%s)", name, health, attack, abilities);
    }
    
    // Setters
    public void setName(String name) { this.name = name; }
    public void setHealth(int health) { this.health = health; }
    
    public static void main(String[] args) {
        // Template character
        GameCharacter warrior = new GameCharacter("Warrior", 100, 50);
        warrior.addAbility("Sword Strike");
        warrior.addAbility("Shield Block");
        warrior.setStat("strength", 80);
        warrior.setStat("defense", 60);
        
        // Clone and customize
        GameCharacter warrior2 = warrior.deepClone();
        warrior2.setName("Elite Warrior");
        warrior2.setHealth(150);
        warrior2.addAbility("Berserker Rage");  // only warrior2 has this
        
        System.out.println("Original: " + warrior);
        System.out.println("Clone:    " + warrior2);
        System.out.println("Abilities shared? " + (warrior.abilities == warrior2.abilities));
    }
}
```

---

## 18.2 Structural Patterns

### Adapter Pattern

```java
// Legacy API
interface OldPaymentGateway {
    boolean processPayment(String cardNumber, String expiry, int amountCents);
}

// New interface
interface PaymentProcessor {
    boolean charge(double amount, String currency, Map<String, String> cardDetails);
}

// Old implementation
class LegacyStripe implements OldPaymentGateway {
    @Override
    public boolean processPayment(String cardNumber, String expiry, int amountCents) {
        System.out.printf("Legacy Stripe: charging %s/%s for %d cents%n",
            cardNumber, expiry, amountCents);
        return true;
    }
}

// Adapter: wraps old API to new interface
class StripeAdapter implements PaymentProcessor {
    private final OldPaymentGateway legacy;
    
    public StripeAdapter(OldPaymentGateway legacy) {
        this.legacy = legacy;
    }
    
    @Override
    public boolean charge(double amount, String currency, Map<String, String> details) {
        int cents = (int)(amount * 100);
        String cardNumber = details.get("number");
        String expiry = details.get("expiry");
        System.out.printf("Adapting: ฿%.2f %s → %d cents%n", amount, currency, cents);
        return legacy.processPayment(cardNumber, expiry, cents);
    }
}

// Object Adapter example
public class AdapterDemo {
    public static void main(String[] args) {
        OldPaymentGateway legacy = new LegacyStripe();
        PaymentProcessor processor = new StripeAdapter(legacy);
        
        Map<String, String> card = Map.of(
            "number", "4111111111111111",
            "expiry", "12/25",
            "cvv", "123"
        );
        
        boolean result = processor.charge(1500.00, "THB", card);
        System.out.println("Payment " + (result ? "succeeded" : "failed"));
    }
}
```

### Composite Pattern

```java
// File system tree
interface FileSystemNode {
    String getName();
    long getSize();
    void print(String indent);
}

class FileNode implements FileSystemNode {
    private final String name;
    private final long size;
    
    public FileNode(String name, long size) {
        this.name = name;
        this.size = size;
    }
    
    @Override public String getName() { return name; }
    @Override public long getSize() { return size; }
    
    @Override
    public void print(String indent) {
        System.out.printf("%s📄 %-20s %,8d bytes%n", indent, name, size);
    }
}

class DirectoryNode implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();
    
    public DirectoryNode(String name) { this.name = name; }
    
    public void add(FileSystemNode node) { children.add(node); }
    
    @Override public String getName() { return name; }
    
    @Override
    public long getSize() {
        return children.stream().mapToLong(FileSystemNode::getSize).sum();
    }
    
    @Override
    public void print(String indent) {
        System.out.printf("%s📁 %s/ (%,d bytes)%n", indent, name, getSize());
        children.forEach(c -> c.print(indent + "  "));
    }
    
    public static void main(String[] args) {
        DirectoryNode root = new DirectoryNode("project");
        
        DirectoryNode src = new DirectoryNode("src");
        src.add(new FileNode("Main.java", 2048));
        src.add(new FileNode("App.java", 3072));
        
        DirectoryNode test = new DirectoryNode("test");
        test.add(new FileNode("MainTest.java", 1024));
        
        DirectoryNode resources = new DirectoryNode("resources");
        resources.add(new FileNode("config.properties", 512));
        resources.add(new FileNode("messages.txt", 256));
        
        root.add(src);
        root.add(test);
        root.add(resources);
        root.add(new FileNode("pom.xml", 4096));
        root.add(new FileNode("README.md", 2048));
        
        root.print("");
        System.out.printf("Total size: %,d bytes%n", root.getSize());
    }
}
```

---

## 18.3 Behavioral Patterns

### Command Pattern

```java
import java.util.*;

interface Command {
    void execute();
    void undo();
    String getDescription();
}

class TextEditor {
    private StringBuilder content = new StringBuilder();
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();
    
    public void executeCommand(Command cmd) {
        cmd.execute();
        history.push(cmd);
        redoStack.clear();  // clear redo on new action
    }
    
    public void undo() {
        if (history.isEmpty()) { System.out.println("Nothing to undo"); return; }
        Command cmd = history.pop();
        cmd.undo();
        redoStack.push(cmd);
        System.out.println("Undone: " + cmd.getDescription());
    }
    
    public void redo() {
        if (redoStack.isEmpty()) { System.out.println("Nothing to redo"); return; }
        Command cmd = redoStack.pop();
        cmd.execute();
        history.push(cmd);
        System.out.println("Redone: " + cmd.getDescription());
    }
    
    public String getContent() { return content.toString(); }
    
    // Commands
    class InsertCommand implements Command {
        private final String text;
        private final int pos;
        
        InsertCommand(String text, int pos) { this.text = text; this.pos = pos; }
        
        @Override public void execute() { content.insert(pos, text); }
        @Override public void undo() { content.delete(pos, pos + text.length()); }
        @Override public String getDescription() { return "Insert '" + text + "' at " + pos; }
    }
    
    class DeleteCommand implements Command {
        private final int from, to;
        private String deleted;
        
        DeleteCommand(int from, int to) { this.from = from; this.to = to; }
        
        @Override public void execute() {
            deleted = content.substring(from, to);
            content.delete(from, to);
        }
        @Override public void undo() { content.insert(from, deleted); }
        @Override public String getDescription() { return "Delete [" + from + "," + to + "]"; }
    }
    
    Command insertAt(String text, int pos) { return new InsertCommand(text, pos); }
    Command deleteRange(int from, int to) { return new DeleteCommand(from, to); }
    
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        
        editor.executeCommand(editor.insertAt("Hello", 0));
        System.out.println("After insert: " + editor.getContent());
        
        editor.executeCommand(editor.insertAt(", World!", 5));
        System.out.println("After insert: " + editor.getContent());
        
        editor.executeCommand(editor.deleteRange(5, 7));  // delete ", "
        System.out.println("After delete: " + editor.getContent());
        
        editor.undo();
        System.out.println("After undo: " + editor.getContent());
        
        editor.undo();
        System.out.println("After undo: " + editor.getContent());
        
        editor.redo();
        System.out.println("After redo: " + editor.getContent());
    }
}
```

### Chain of Responsibility

```java
abstract class RequestHandler {
    protected RequestHandler next;
    
    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;
    }
    
    abstract boolean handle(HttpRequest request);
}

record HttpRequest(String method, String path, String token, Map<String, String> headers) {}

class AuthHandler extends RequestHandler {
    private static final Set<String> validTokens = Set.of("token123", "admin456");
    
    @Override
    public boolean handle(HttpRequest req) {
        if (req.path().startsWith("/public")) {
            System.out.println("AuthHandler: public path, skip auth");
            return next != null && next.handle(req);
        }
        
        String token = req.token();
        if (token == null || !validTokens.contains(token)) {
            System.out.println("AuthHandler: ❌ Unauthorized");
            return false;
        }
        System.out.println("AuthHandler: ✅ Authenticated");
        return next == null || next.handle(req);
    }
}

class RateLimitHandler extends RequestHandler {
    private final Map<String, Integer> requestCount = new HashMap<>();
    private final int limit;
    
    public RateLimitHandler(int limit) { this.limit = limit; }
    
    @Override
    public boolean handle(HttpRequest req) {
        String key = req.token() != null ? req.token() : "anonymous";
        int count = requestCount.merge(key, 1, Integer::sum);
        if (count > limit) {
            System.out.println("RateLimitHandler: ❌ Rate limit exceeded (" + count + "/" + limit + ")");
            return false;
        }
        System.out.println("RateLimitHandler: ✅ Request " + count + "/" + limit);
        return next == null || next.handle(req);
    }
}

class LoggingHandler extends RequestHandler {
    @Override
    public boolean handle(HttpRequest req) {
        System.out.printf("LoggingHandler: [%s] %s%n", req.method(), req.path());
        return next == null || next.handle(req);
    }
}

class BusinessHandler extends RequestHandler {
    @Override
    public boolean handle(HttpRequest req) {
        System.out.println("BusinessHandler: Processing request...");
        System.out.println("Response: 200 OK, data: {result: 'success'}");
        return true;
    }
}

public class ChainDemo {
    public static void main(String[] args) {
        
        // Build chain: Logging → Auth → RateLimit → Business
        RequestHandler logging = new LoggingHandler();
        RequestHandler auth = new AuthHandler();
        RequestHandler rateLimit = new RateLimitHandler(3);
        RequestHandler business = new BusinessHandler();
        
        logging.setNext(auth).setNext(rateLimit).setNext(business);
        
        System.out.println("=== Request 1: Valid token ===");
        logging.handle(new HttpRequest("GET", "/api/users", "token123", Map.of()));
        
        System.out.println("\n=== Request 2: Invalid token ===");
        logging.handle(new HttpRequest("GET", "/api/data", "invalid", Map.of()));
        
        System.out.println("\n=== Request 3: Public path ===");
        logging.handle(new HttpRequest("GET", "/public/home", null, Map.of()));
        
        System.out.println("\n=== Requests 4-6: Rate limiting ===");
        for (int i = 4; i <= 6; i++) {
            System.out.println("\n--- Request " + i + " ---");
            logging.handle(new HttpRequest("GET", "/api/test", "token123", Map.of()));
        }
    }
}
```

---

## 18.4 Template Method Pattern

```java
abstract class DataExporter {
    // Template method
    public final void export(List<?> data, String filename) {
        System.out.println("Starting export to: " + filename);
        String validated = validate(data);
        String transformed = transform(data);
        String formatted = format(transformed);
        write(formatted, filename);
        notifyComplete(filename);
        System.out.println("Export complete!");
    }
    
    protected String validate(List<?> data) {
        if (data == null || data.isEmpty()) throw new IllegalArgumentException("No data");
        System.out.println("Validating " + data.size() + " records...");
        return "valid";
    }
    
    protected abstract String transform(List<?> data);
    protected abstract String format(String data);
    
    protected void write(String content, String filename) {
        System.out.println("Writing to " + filename + " (" + content.length() + " chars)");
    }
    
    protected void notifyComplete(String filename) {
        System.out.println("Notification sent for: " + filename);
    }
}

class CsvExporter extends DataExporter {
    @Override
    protected String transform(List<?> data) {
        return data.stream().map(Object::toString).collect(java.util.stream.Collectors.joining(","));
    }
    
    @Override
    protected String format(String data) {
        return "id,value\n" + data.replace(",", "\n").trim();
    }
}

class JsonExporter extends DataExporter {
    @Override
    protected String transform(List<?> data) {
        return data.stream()
            .map(item -> "\"" + item + "\"")
            .collect(java.util.stream.Collectors.joining(","));
    }
    
    @Override
    protected String format(String data) {
        return "[" + data + "]";
    }
    
    @Override
    protected void notifyComplete(String filename) {
        System.out.println("JSON export ready: " + filename);
    }
}

public class TemplateMethodDemo {
    public static void main(String[] args) {
        List<Integer> data = List.of(1, 2, 3, 4, 5);
        
        System.out.println("=== CSV Export ===");
        new CsvExporter().export(data, "data.csv");
        
        System.out.println("\n=== JSON Export ===");
        new JsonExporter().export(data, "data.json");
    }
}
```

---

## สรุป Part 18

| Pattern | หมวด | Use Case |
|---------|------|---------|
| Singleton | Creational | Config, Logger, Cache |
| Builder | Creational | Complex object creation |
| Prototype | Creational | Clone templates |
| Adapter | Structural | Legacy API integration |
| Composite | Structural | Tree structures (file system, UI) |
| Command | Behavioral | Undo/Redo, queued tasks |
| Chain of Responsibility | Behavioral | Middleware, filters |
| Template Method | Behavioral | Algorithm skeleton |

➡️ [Part 19: JDBC & Database](./Part-19-JDBC-Database.md)
