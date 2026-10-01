# Part 11: Exception Handling
## ขั้นตอนที่ 681-750: การจัดการข้อผิดพลาดแบบมืออาชีพ

---

## 11.1 Exception Hierarchy

```
Throwable
├── Error (ไม่ควร catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── ArithmeticException
    │   ├── NumberFormatException
    │   └── IllegalArgumentException
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        ├── FileNotFoundException
        └── ParseException
```

---

## 11.2 try-catch-finally

```java
import java.io.*;
import java.util.*;

public class ExceptionBasics {
    
    // Basic try-catch
    public static void divisionExample() {
        int[] nums = {10, 5, 0, 8};
        int divisor = 2;
        
        for (int num : nums) {
            try {
                int result = num / divisor;
                System.out.printf("%d / %d = %d%n", num, divisor, result);
                divisor = 0; // cause ArithmeticException next
            } catch (ArithmeticException e) {
                System.out.printf("Error: %s (num=%d, div=%d)%n", e.getMessage(), num, divisor);
                divisor = 1; // recover
            }
        }
    }
    
    // Multi-catch
    public static void multiCatchExample(String input, int index) {
        try {
            int[] arr = new int[5];
            int num = Integer.parseInt(input);
            arr[index] = num;
            System.out.printf("arr[%d] = %d%n", index, num);
        } catch (NumberFormatException e) {
            System.out.println("NumberFormatException: '" + input + "' ไม่ใช่ตัวเลข");
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("IndexOutOfBounds: index " + index + " เกินขนาดอาร์เรย์");
        } catch (Exception e) {
            System.out.println("Unexpected: " + e.getMessage());
        } finally {
            System.out.println("finally: ทำงานเสมอ");
        }
    }
    
    // Multi-catch single handler (Java 7+)
    public static void unionCatch(String[] args) {
        try {
            String s = args[0];
            int n = Integer.parseInt(s);
            System.out.println("Parsed: " + n);
        } catch (ArrayIndexOutOfBoundsException | NumberFormatException e) {
            System.out.println("Error: " + e.getMessage());
        }
    }
    
    // Try-with-resources (Java 7+)
    public static void readFileExample(String filename) {
        try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
            String line;
            int lineNum = 0;
            while ((line = reader.readLine()) != null) {
                System.out.printf("%3d: %s%n", ++lineNum, line);
            }
        } catch (FileNotFoundException e) {
            System.out.println("ไม่พบไฟล์: " + filename);
        } catch (IOException e) {
            System.out.println("อ่านไฟล์ผิดพลาด: " + e.getMessage());
        }
        // reader.close() ถูกเรียกอัตโนมัติ
    }
    
    public static void main(String[] args) {
        System.out.println("=== Division ===");
        divisionExample();
        
        System.out.println("\n=== Multi-catch ===");
        multiCatchExample("42", 2);
        multiCatchExample("abc", 2);
        multiCatchExample("42", 10);
        
        System.out.println("\n=== File Read ===");
        readFileExample("nonexistent.txt");
    }
}
```

---

## 11.3 Custom Exceptions

```java
// ====== Custom Exception Hierarchy ======

// Base exception
class AppException extends RuntimeException {
    private final String errorCode;
    private final Object[] details;
    
    public AppException(String errorCode, String message, Object... details) {
        super(message);
        this.errorCode = errorCode;
        this.details = details;
    }
    
    public AppException(String errorCode, String message, Throwable cause, Object... details) {
        super(message, cause);
        this.errorCode = errorCode;
        this.details = details;
    }
    
    public String getErrorCode() { return errorCode; }
    public Object[] getDetails() { return details; }
    
    @Override
    public String toString() {
        return String.format("[%s] %s", errorCode, getMessage());
    }
}

// Domain-specific exceptions
class ValidationException extends AppException {
    private final String field;
    private final Object value;
    
    public ValidationException(String field, Object value, String message) {
        super("VALIDATION_ERROR", message, field, value);
        this.field = field;
        this.value = value;
    }
    
    public String getField() { return field; }
    public Object getValue() { return value; }
}

class NotFoundException extends AppException {
    public NotFoundException(String resource, Object id) {
        super("NOT_FOUND", resource + " not found: " + id, resource, id);
    }
}

class AuthenticationException extends AppException {
    public AuthenticationException(String reason) {
        super("AUTH_ERROR", "Authentication failed: " + reason);
    }
}

class InsufficientFundsException extends AppException {
    private final double available;
    private final double required;
    
    public InsufficientFundsException(double available, double required) {
        super("INSUFFICIENT_FUNDS",
              String.format("ยอดเงินไม่เพียงพอ: มี ฿%.2f ต้องการ ฿%.2f", available, required));
        this.available = available;
        this.required = required;
    }
    
    public double getShortfall() { return required - available; }
}

// ====== Bank Account with Exceptions ======
class SecureBankAccount {
    private final String accountId;
    private String owner;
    private double balance;
    private boolean locked;
    private int failedAttempts;
    private static final int MAX_ATTEMPTS = 3;
    private static final double DAILY_LIMIT = 50000;
    private double todayWithdrawn = 0;
    
    public SecureBankAccount(String accountId, String owner, double initialBalance) {
        this.accountId = accountId;
        this.owner = owner;
        this.balance = initialBalance;
    }
    
    public void deposit(double amount) {
        if (amount <= 0) throw new ValidationException("amount", amount, "จำนวนต้องมากกว่า 0");
        if (locked) throw new AppException("ACCOUNT_LOCKED", "บัญชีถูกล็อค");
        balance += amount;
        System.out.printf("ฝาก ฿%.2f | ยอดคงเหลือ: ฿%.2f%n", amount, balance);
    }
    
    public void withdraw(double amount) {
        if (amount <= 0) throw new ValidationException("amount", amount, "จำนวนต้องมากกว่า 0");
        if (locked) throw new AppException("ACCOUNT_LOCKED", "บัญชีถูกล็อค");
        if (amount > balance) throw new InsufficientFundsException(balance, amount);
        if (todayWithdrawn + amount > DAILY_LIMIT) {
            throw new AppException("DAILY_LIMIT_EXCEEDED",
                String.format("เกินลิมิตรายวัน (ถอนไปแล้ว ฿%.2f / ฿%.2f)", todayWithdrawn, DAILY_LIMIT));
        }
        balance -= amount;
        todayWithdrawn += amount;
        System.out.printf("ถอน ฿%.2f | ยอดคงเหลือ: ฿%.2f%n", amount, balance);
    }
    
    public void transfer(SecureBankAccount target, double amount) {
        try {
            withdraw(amount);
            target.deposit(amount);
            System.out.printf("โอน ฿%.2f → %s สำเร็จ%n", amount, target.accountId);
        } catch (InsufficientFundsException e) {
            System.out.println("โอนไม่สำเร็จ: " + e.getMessage() + 
                " (ขาดอีก ฿" + e.getShortfall() + ")");
            throw e; // re-throw
        }
    }
    
    public double getBalance() { return balance; }
    public String getAccountId() { return accountId; }
    
    public static void main(String[] args) {
        SecureBankAccount acc1 = new SecureBankAccount("ACC001", "Alice", 10000);
        SecureBankAccount acc2 = new SecureBankAccount("ACC002", "Bob", 5000);
        
        // Normal operations
        try {
            acc1.deposit(5000);
            acc1.withdraw(3000);
            acc1.transfer(acc2, 2000);
        } catch (AppException e) {
            System.out.println("Error [" + e.getErrorCode() + "]: " + e.getMessage());
        }
        
        // Error cases
        System.out.println("\n=== Error cases ===");
        
        // Insufficient funds
        try {
            acc1.withdraw(100000);
        } catch (InsufficientFundsException e) {
            System.out.println("Caught: " + e.getMessage() + " | Shortfall: ฿" + e.getShortfall());
        }
        
        // Validation error
        try {
            acc1.deposit(-100);
        } catch (ValidationException e) {
            System.out.println("Validation error on field '" + e.getField() + "': " + e.getMessage());
        }
    }
}
```

---

## 11.4 Exception Chaining

```java
import java.sql.SQLException;

public class ExceptionChaining {
    
    // Low-level database operation
    static void executeQuery(String sql) throws SQLException {
        if (sql.contains("DROP")) {
            throw new SQLException("Dangerous SQL detected", "42000", 1001);
        }
        System.out.println("Executed: " + sql);
    }
    
    // Mid-level data access layer
    static Object[] fetchUserById(int id) {
        try {
            executeQuery("SELECT * FROM users WHERE id = " + id);
            return new Object[]{"Alice", "alice@example.com"};
        } catch (SQLException e) {
            // Wrap with context
            throw new AppException("DB_ERROR", 
                "Failed to fetch user with id: " + id, e);
        }
    }
    
    // High-level service
    static String getUserEmail(int userId) {
        try {
            Object[] user = fetchUserById(userId);
            return (String) user[1];
        } catch (AppException e) {
            // Add more context
            throw new AppException("SERVICE_ERROR",
                "UserService: cannot get email for userId=" + userId, e.getCause());
        }
    }
    
    public static void main(String[] args) {
        try {
            // Simulate bad input that causes chained exceptions
            fetchUserById(-1);
        } catch (AppException e) {
            System.out.println("Error: " + e.getMessage());
            System.out.println("Caused by: " + e.getCause());
        }
        
        // Print full stack trace
        try {
            getUserEmail(999);
        } catch (AppException e) {
            System.out.println("\nException chain:");
            Throwable t = e;
            while (t != null) {
                System.out.println("  " + t.getClass().getSimpleName() + ": " + t.getMessage());
                t = t.getCause();
            }
        }
    }
}
```

---

## 11.5 Best Practices

```java
import java.util.*;
import java.util.logging.*;

public class ExceptionBestPractices {
    
    private static final Logger logger = Logger.getLogger(ExceptionBestPractices.class.getName());
    
    // ❌ Bad: catch and ignore
    static void bad_catchAndIgnore() {
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            // empty! don't do this
        }
    }
    
    // ❌ Bad: too broad catch
    static void bad_tooBroad() {
        try {
            // many operations
        } catch (Exception e) {
            System.out.println("Something went wrong"); // lost info
        }
    }
    
    // ✅ Good: specific catch, log, and handle
    static Optional<Integer> divide(int a, int b) {
        try {
            return Optional.of(a / b);
        } catch (ArithmeticException e) {
            logger.warning("Division by zero: " + a + " / " + b);
            return Optional.empty();
        }
    }
    
    // ✅ Good: validate then operate
    static int safeDivide(int a, int b) {
        if (b == 0) throw new ArithmeticException("Division by zero");
        return a / b;
    }
    
    // ✅ Good: use Optional instead of exception for "not found"
    static Optional<String> findUser(List<String> users, String name) {
        return users.stream()
                    .filter(u -> u.equalsIgnoreCase(name))
                    .findFirst();
    }
    
    // ✅ Good: resource cleanup with try-with-resources
    static String readFirstLine(String filename) throws IOException {
        try (var reader = new java.io.BufferedReader(new java.io.FileReader(filename))) {
            return reader.readLine();
        }
        // reader auto-closed even on exception
    }
    
    // ✅ Global error handler pattern
    interface ExceptionHandler<T> {
        T handle(Exception e);
    }
    
    static <T> T trySafe(java.util.function.Supplier<T> operation, ExceptionHandler<T> handler) {
        try {
            return operation.get();
        } catch (Exception e) {
            return handler.handle(e);
        }
    }
    
    public static void main(String[] args) {
        
        // Using Optional
        divide(10, 2).ifPresent(r -> System.out.println("10/2 = " + r));
        divide(10, 0).ifPresentOrElse(
            r -> System.out.println("Result: " + r),
            () -> System.out.println("Cannot divide by zero")
        );
        
        // trySafe pattern
        String result = trySafe(
            () -> "Value: " + Integer.parseInt("42"),
            e -> "Error: " + e.getMessage()
        );
        System.out.println(result);
        
        String result2 = trySafe(
            () -> "Value: " + Integer.parseInt("not a number"),
            e -> "Error: " + e.getMessage()
        );
        System.out.println(result2);
        
        // findUser
        List<String> users = List.of("Alice", "Bob", "Charlie");
        findUser(users, "bob")
            .ifPresent(u -> System.out.println("Found: " + u));
        findUser(users, "dave")
            .ifPresentOrElse(
                u -> System.out.println("Found: " + u),
                () -> System.out.println("User 'dave' not found")
            );
    }
}
```

---

## 11.6 Full Program: ATM Machine with Exception Handling

```java
import java.util.*;

class ATMException extends RuntimeException {
    private final String code;
    public ATMException(String code, String message) {
        super(message);
        this.code = code;
    }
    public String getCode() { return code; }
}

class ATMMachine {
    private double balance;
    private final String cardNumber;
    private final String pin;
    private boolean cardInserted = false;
    private boolean authenticated = false;
    private int pinAttempts = 0;
    private static final int MAX_PIN_ATTEMPTS = 3;
    private static final double TRANSACTION_LIMIT = 20000;
    private final List<String> history = new ArrayList<>();
    
    public ATMMachine(String cardNumber, String pin, double balance) {
        this.cardNumber = cardNumber;
        this.pin = pin;
        this.balance = balance;
    }
    
    public void insertCard(String card) {
        if (cardInserted) throw new ATMException("CARD_ALREADY_INSERTED", "การ์ดถูกใส่แล้ว");
        if (!card.equals(cardNumber)) throw new ATMException("INVALID_CARD", "การ์ดไม่ถูกต้อง");
        cardInserted = true;
        System.out.println("✅ ใส่การ์ดสำเร็จ");
    }
    
    public void enterPin(String inputPin) {
        if (!cardInserted) throw new ATMException("NO_CARD", "กรุณาใส่การ์ดก่อน");
        if (authenticated) throw new ATMException("ALREADY_AUTH", "ยืนยันตัวตนแล้ว");
        if (pinAttempts >= MAX_PIN_ATTEMPTS) throw new ATMException("CARD_BLOCKED", "การ์ดถูกบล็อก");
        
        pinAttempts++;
        if (!inputPin.equals(pin)) {
            int remaining = MAX_PIN_ATTEMPTS - pinAttempts;
            if (remaining == 0) throw new ATMException("CARD_BLOCKED", "PIN ผิด การ์ดถูกบล็อก");
            throw new ATMException("WRONG_PIN", 
                "PIN ผิด เหลืออีก " + remaining + " ครั้ง");
        }
        
        authenticated = true;
        pinAttempts = 0;
        System.out.println("✅ ยืนยันตัวตนสำเร็จ");
    }
    
    public double checkBalance() {
        ensureAuthenticated();
        System.out.printf("💰 ยอดเงิน: ฿%.2f%n", balance);
        return balance;
    }
    
    public void withdraw(double amount) {
        ensureAuthenticated();
        if (amount <= 0) throw new ATMException("INVALID_AMOUNT", "จำนวนต้องมากกว่า 0");
        if (amount % 100 != 0) throw new ATMException("INVALID_DENOMINATION", "ต้องเป็นทวีคูณของ 100");
        if (amount > TRANSACTION_LIMIT) throw new ATMException("LIMIT_EXCEEDED",
            "เกินลิมิต ฿" + TRANSACTION_LIMIT + " ต่อครั้ง");
        if (amount > balance) throw new ATMException("INSUFFICIENT_FUNDS",
            String.format("ยอดเงินไม่พอ (มี ฿%.2f)", balance));
        
        balance -= amount;
        String txn = String.format("ถอน ฿%.0f | คงเหลือ ฿%.2f", amount, balance);
        history.add(txn);
        System.out.println("💵 " + txn);
    }
    
    public void deposit(double amount) {
        ensureAuthenticated();
        if (amount <= 0) throw new ATMException("INVALID_AMOUNT", "จำนวนต้องมากกว่า 0");
        balance += amount;
        String txn = String.format("ฝาก ฿%.0f | คงเหลือ ฿%.2f", amount, balance);
        history.add(txn);
        System.out.println("✅ " + txn);
    }
    
    public void printHistory() {
        ensureAuthenticated();
        System.out.println("\n=== ประวัติธุรกรรม ===");
        if (history.isEmpty()) {
            System.out.println("ไม่มีรายการ");
        } else {
            history.forEach(h -> System.out.println("  " + h));
        }
    }
    
    public void ejectCard() {
        if (!cardInserted) System.out.println("ไม่มีการ์ดในเครื่อง");
        else {
            cardInserted = false;
            authenticated = false;
            System.out.println("✅ นำการ์ดออก");
        }
    }
    
    private void ensureAuthenticated() {
        if (!cardInserted) throw new ATMException("NO_CARD", "กรุณาใส่การ์ดก่อน");
        if (!authenticated) throw new ATMException("NOT_AUTH", "กรุณายืนยันตัวตนก่อน");
    }
    
    static void runOperation(Runnable op) {
        try {
            op.run();
        } catch (ATMException e) {
            System.out.println("❌ Error [" + e.getCode() + "]: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        
        ATMMachine atm = new ATMMachine("1234-5678", "1234", 25000);
        
        System.out.println("=== ATM Simulation ===\n");
        
        // Scenario 1: Normal flow
        System.out.println("--- สถานการณ์ที่ 1: ใช้งานปกติ ---");
        runOperation(() -> atm.insertCard("1234-5678"));
        runOperation(() -> atm.enterPin("1234"));
        runOperation(atm::checkBalance);
        runOperation(() -> atm.deposit(5000));
        runOperation(() -> atm.withdraw(3000));
        runOperation(() -> atm.withdraw(500));
        atm.printHistory();
        atm.ejectCard();
        
        // Scenario 2: Wrong PIN
        System.out.println("\n--- สถานการณ์ที่ 2: PIN ผิด ---");
        ATMMachine atm2 = new ATMMachine("9999-0000", "5678", 10000);
        runOperation(() -> atm2.insertCard("9999-0000"));
        runOperation(() -> atm2.enterPin("0000")); // wrong
        runOperation(() -> atm2.enterPin("1111")); // wrong
        runOperation(() -> atm2.enterPin("2222")); // blocked!
        
        // Scenario 3: Error operations
        System.out.println("\n--- สถานการณ์ที่ 3: ข้อผิดพลาดต่างๆ ---");
        ATMMachine atm3 = new ATMMachine("5555-1111", "4321", 1000);
        runOperation(() -> atm3.insertCard("5555-1111"));
        runOperation(() -> atm3.enterPin("4321"));
        runOperation(() -> atm3.withdraw(150));    // not multiple of 100
        runOperation(() -> atm3.withdraw(50000));  // over limit
        runOperation(() -> atm3.withdraw(5000));   // insufficient funds
        runOperation(() -> atm3.withdraw(1000));   // OK
    }
}
```

---

## สรุป Part 11

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Exception Hierarchy | Error, Exception, Checked, Unchecked |
| try-catch-finally | การจับ exception หลายชนิด |
| try-with-resources | Auto-close resources |
| Custom Exceptions | สร้าง exception เอง |
| Exception Chaining | เก็บ cause chain |
| Best Practices | Optional, validate first, specific catch |

➡️ [Part 12: Collections Framework](./Part-12-Collections-Framework.md)
