# Part 26: Testing with JUnit 5 & Mockito
## ขั้นตอนที่ 1721-1790: Unit Testing ระดับ Professional

---

## 26.1 JUnit 5 Architecture

```
JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage

JUnit Platform  = foundation สำหรับ run tests (TestEngine API)
JUnit Jupiter   = API ใหม่ (annotations, assertions)  ← เราใช้นี้
JUnit Vintage   = backward compat กับ JUnit 3/4

Dependencies (Maven):
  junit-jupiter-api       = @Test, @BeforeEach, assertions
  junit-jupiter-engine    = run Jupiter tests
  junit-jupiter-params    = parameterized tests
  mockito-core            = mocking framework
  mockito-junit-jupiter   = Mockito + JUnit 5 integration
```

---

## 26.2 Basic Tests

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.condition.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    
    private Calculator calc;
    
    @BeforeAll
    static void globalSetup() {
        System.out.println("=== Tests Starting ===");
    }
    
    @AfterAll
    static void globalTeardown() {
        System.out.println("=== Tests Finished ===");
    }
    
    @BeforeEach
    void setup() {
        calc = new Calculator();
    }
    
    @AfterEach
    void teardown() {
        // cleanup after each test
    }
    
    @Test
    @DisplayName("2 + 3 should equal 5")
    void addition() {
        assertEquals(5, calc.add(2, 3));
    }
    
    @Test
    void subtraction() {
        assertEquals(7, calc.subtract(10, 3));
    }
    
    @Test
    void multiplication() {
        assertEquals(12, calc.multiply(3, 4));
    }
    
    @Test
    void division() {
        assertEquals(2.5, calc.divide(5, 2), 0.001);
    }
    
    @Test
    @DisplayName("Division by zero should throw ArithmeticException")
    void divisionByZero() {
        ArithmeticException ex = assertThrows(
            ArithmeticException.class,
            () -> calc.divide(10, 0)
        );
        assertTrue(ex.getMessage().contains("zero"));
    }
    
    @Test
    void multipleAssertions() {
        // assertAll runs all assertions even if some fail
        assertAll("calculation",
            () -> assertEquals(5, calc.add(2, 3)),
            () -> assertEquals(7, calc.subtract(10, 3)),
            () -> assertEquals(12, calc.multiply(3, 4))
        );
    }
    
    @Test
    @Disabled("Feature not yet implemented")
    void futureFeature() {
        // will be skipped
    }
    
    @Test
    @EnabledOnOs(OS.LINUX)
    void linuxOnly() {
        assertTrue(true, "Runs on Linux");
    }
    
    @Test
    @EnabledIfSystemProperty(named = "env", matches = "ci")
    void ciOnly() {
        assertTrue(true, "Runs in CI");
    }
    
    @Test
    void assertionsDemo() {
        String name = "Alice";
        int age = 30;
        
        assertNotNull(name);
        assertEquals("Alice", name);
        assertTrue(age >= 18, "User must be adult");
        assertFalse(name.isEmpty());
        
        int[] arr = {1, 2, 3};
        assertArrayEquals(new int[]{1, 2, 3}, arr);
        
        // Timeout
        assertTimeout(java.time.Duration.ofMillis(100), () -> {
            // operation must complete within 100ms
            Thread.sleep(50);
        });
    }
}

// Simple Calculator to test
class Calculator {
    public int add(int a, int b) { return a + b; }
    public int subtract(int a, int b) { return a - b; }
    public int multiply(int a, int b) { return a * b; }
    
    public double divide(double a, double b) {
        if (b == 0) throw new ArithmeticException("Cannot divide by zero");
        return a / b;
    }
}
```

---

## 26.3 Parameterized Tests

```java
import org.junit.jupiter.params.*;
import org.junit.jupiter.params.provider.*;
import static org.junit.jupiter.api.Assertions.*;
import java.util.stream.Stream;

class ParameterizedTestsDemo {
    
    // ====== @ValueSource ======
    @ParameterizedTest
    @ValueSource(strings = {"racecar", "radar", "level", "madam"})
    void isPalindrome(String word) {
        String reversed = new StringBuilder(word).reverse().toString();
        assertEquals(word, reversed);
    }
    
    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 4, 5})
    void positiveNumbers(int number) {
        assertTrue(number > 0);
    }
    
    // ====== @CsvSource ======
    @ParameterizedTest
    @CsvSource({
        "2, 3, 5",
        "10, -5, 5",
        "0, 0, 0",
        "-3, -4, -7"
    })
    void addition(int a, int b, int expected) {
        assertEquals(expected, a + b);
    }
    
    // ====== @CsvFileSource ======
    @ParameterizedTest
    @CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
    void fromCsvFile(String input, String expected) {
        assertEquals(expected, input.toUpperCase());
    }
    
    // ====== @MethodSource ======
    static Stream<Arguments> provideStrings() {
        return Stream.of(
            Arguments.of("hello", 5),
            Arguments.of("world", 5),
            Arguments.of("java", 4),
            Arguments.of("", 0)
        );
    }
    
    @ParameterizedTest
    @MethodSource("provideStrings")
    void stringLength(String str, int expectedLength) {
        assertEquals(expectedLength, str.length());
    }
    
    // ====== @EnumSource ======
    enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }
    
    @ParameterizedTest
    @EnumSource(value = Day.class, names = {"SAT", "SUN"})
    void weekendsOnly(Day day) {
        assertTrue(day == Day.SAT || day == Day.SUN);
    }
    
    @ParameterizedTest
    @EnumSource(value = Day.class, mode = EnumSource.Mode.EXCLUDE, names = {"SAT", "SUN"})
    void weekdaysOnly(Day day) {
        assertNotEquals(Day.SAT, day);
        assertNotEquals(Day.SUN, day);
    }
    
    // ====== @NullAndEmptySource ======
    @ParameterizedTest
    @NullAndEmptySource
    @ValueSource(strings = {"  ", "\t", "\n"})
    void blankStrings(String input) {
        assertTrue(input == null || input.isBlank());
    }
}
```

---

## 26.4 Test Organization & Nested Tests

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

@DisplayName("BankAccount Tests")
class BankAccountTest {
    
    @Nested
    @DisplayName("Account Creation")
    class CreationTests {
        
        @Test
        @DisplayName("Account with positive balance")
        void validCreation() {
            BankAccount acc = new BankAccount("ACC001", 1000.0);
            assertEquals(1000.0, acc.getBalance());
            assertEquals("ACC001", acc.getId());
        }
        
        @Test
        @DisplayName("Account with zero balance")
        void zeroBalance() {
            BankAccount acc = new BankAccount("ACC002", 0.0);
            assertEquals(0.0, acc.getBalance());
        }
        
        @Test
        @DisplayName("Account with negative balance throws")
        void negativeBalance() {
            assertThrows(IllegalArgumentException.class,
                () -> new BankAccount("ACC003", -100.0));
        }
    }
    
    @Nested
    @DisplayName("Deposit Operations")
    class DepositTests {
        
        private BankAccount account;
        
        @BeforeEach
        void setup() {
            account = new BankAccount("ACC001", 500.0);
        }
        
        @Test
        void depositPositiveAmount() {
            account.deposit(200.0);
            assertEquals(700.0, account.getBalance());
        }
        
        @Test
        void depositZeroThrows() {
            assertThrows(IllegalArgumentException.class,
                () -> account.deposit(0));
        }
        
        @Test
        void depositNegativeThrows() {
            assertThrows(IllegalArgumentException.class,
                () -> account.deposit(-100));
        }
    }
    
    @Nested
    @DisplayName("Withdrawal Operations")
    class WithdrawalTests {
        
        private BankAccount account;
        
        @BeforeEach
        void setup() {
            account = new BankAccount("ACC001", 1000.0);
        }
        
        @Test
        void withdrawValidAmount() {
            account.withdraw(300.0);
            assertEquals(700.0, account.getBalance());
        }
        
        @Test
        void withdrawExactBalance() {
            account.withdraw(1000.0);
            assertEquals(0.0, account.getBalance());
        }
        
        @Test
        void withdrawMoreThanBalance() {
            assertThrows(InsufficientFundsException.class,
                () -> account.withdraw(1500.0));
        }
    }
    
    @TestMethodOrder(MethodOrderer.OrderAnnotation.class)
    @Nested
    @DisplayName("Transaction History")
    class TransactionTests {
        
        private BankAccount account;
        
        @BeforeEach
        void setup() { account = new BankAccount("ACC001", 1000.0); }
        
        @Test
        @Order(1)
        void initiallyEmpty() {
            assertTrue(account.getTransactions().isEmpty());
        }
        
        @Test
        @Order(2)
        void depositCreatesTransaction() {
            account.deposit(200);
            assertEquals(1, account.getTransactions().size());
        }
    }
}

// Classes to test
class BankAccount {
    private final String id;
    private double balance;
    private final java.util.List<String> transactions = new java.util.ArrayList<>();
    
    public BankAccount(String id, double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("Negative balance");
        this.id = id;
        this.balance = initialBalance;
    }
    
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance += amount;
        transactions.add("DEPOSIT: " + amount);
    }
    
    public void withdraw(double amount) {
        if (amount > balance) throw new InsufficientFundsException("Insufficient funds");
        balance -= amount;
        transactions.add("WITHDRAWAL: " + amount);
    }
    
    public String getId() { return id; }
    public double getBalance() { return balance; }
    public java.util.List<String> getTransactions() { return java.util.List.copyOf(transactions); }
}

class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}
```

---

## 26.5 Mockito

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.*;
import org.mockito.*;
import org.mockito.junit.jupiter.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;
import java.util.*;

// Dependencies to mock
interface UserRepository {
    Optional<User> findById(Long id);
    List<User> findAll();
    User save(User user);
    void deleteById(Long id);
    boolean existsById(Long id);
}

interface EmailService {
    void sendWelcomeEmail(String email, String name);
    void sendPasswordResetEmail(String email, String token);
}

record User(Long id, String name, String email, boolean active) {}

class UserService {
    private final UserRepository userRepo;
    private final EmailService emailService;
    
    UserService(UserRepository userRepo, EmailService emailService) {
        this.userRepo = userRepo;
        this.emailService = emailService;
    }
    
    User getUserById(Long id) {
        return userRepo.findById(id)
            .orElseThrow(() -> new RuntimeException("User not found: " + id));
    }
    
    User createUser(String name, String email) {
        User user = new User(null, name, email, true);
        User saved = userRepo.save(user);
        emailService.sendWelcomeEmail(email, name);
        return saved;
    }
    
    List<User> getActiveUsers() {
        return userRepo.findAll().stream()
            .filter(User::active)
            .toList();
    }
    
    void deleteUser(Long id) {
        if (!userRepo.existsById(id)) {
            throw new RuntimeException("User not found: " + id);
        }
        userRepo.deleteById(id);
    }
}

@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepo;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void getUserById_exists() {
        // Arrange
        User expected = new User(1L, "Alice", "alice@example.com", true);
        when(userRepo.findById(1L)).thenReturn(Optional.of(expected));
        
        // Act
        User result = userService.getUserById(1L);
        
        // Assert
        assertEquals(expected, result);
        verify(userRepo).findById(1L);
    }
    
    @Test
    void getUserById_notFound() {
        when(userRepo.findById(99L)).thenReturn(Optional.empty());
        
        RuntimeException ex = assertThrows(RuntimeException.class,
            () -> userService.getUserById(99L));
        assertTrue(ex.getMessage().contains("99"));
    }
    
    @Test
    void createUser_sendsEmail() {
        User saved = new User(1L, "Bob", "bob@example.com", true);
        when(userRepo.save(any(User.class))).thenReturn(saved);
        
        User result = userService.createUser("Bob", "bob@example.com");
        
        assertEquals(saved, result);
        
        // Verify email was sent
        verify(emailService).sendWelcomeEmail("bob@example.com", "Bob");
        
        // Verify no other email interactions
        verifyNoMoreInteractions(emailService);
    }
    
    @Test
    void createUser_savesWithCorrectData() {
        when(userRepo.save(any(User.class))).thenAnswer(inv -> {
            User u = inv.getArgument(0);
            return new User(1L, u.name(), u.email(), u.active());
        });
        
        User result = userService.createUser("Charlie", "charlie@example.com");
        
        // Capture what was passed to save
        ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
        verify(userRepo).save(captor.capture());
        
        User captured = captor.getValue();
        assertEquals("Charlie", captured.name());
        assertEquals("charlie@example.com", captured.email());
        assertTrue(captured.active());
        assertNull(captured.id());
    }
    
    @Test
    void getActiveUsers_filtersCorrectly() {
        List<User> all = List.of(
            new User(1L, "Alice", "alice@example.com", true),
            new User(2L, "Bob", "bob@example.com", false),
            new User(3L, "Charlie", "charlie@example.com", true)
        );
        when(userRepo.findAll()).thenReturn(all);
        
        List<User> result = userService.getActiveUsers();
        
        assertEquals(2, result.size());
        assertTrue(result.stream().allMatch(User::active));
    }
    
    @Test
    void deleteUser_exists() {
        when(userRepo.existsById(1L)).thenReturn(true);
        
        assertDoesNotThrow(() -> userService.deleteUser(1L));
        
        verify(userRepo).deleteById(1L);
    }
    
    @Test
    void deleteUser_notFound() {
        when(userRepo.existsById(99L)).thenReturn(false);
        
        assertThrows(RuntimeException.class, () -> userService.deleteUser(99L));
        verify(userRepo, never()).deleteById(any());
    }
    
    @Test
    void verifyInteractionOrder() {
        InOrder inOrder = inOrder(userRepo, emailService);
        
        when(userRepo.save(any())).thenReturn(new User(1L, "Dave", "dave@example.com", true));
        
        userService.createUser("Dave", "dave@example.com");
        
        inOrder.verify(userRepo).save(any());
        inOrder.verify(emailService).sendWelcomeEmail(anyString(), anyString());
    }
    
    @Test
    void mockException() {
        when(userRepo.findById(1L))
            .thenThrow(new RuntimeException("Database connection failed"));
        
        assertThrows(RuntimeException.class, () -> userService.getUserById(1L));
    }
    
    @Test
    void mockMultipleCalls() {
        when(userRepo.findById(1L))
            .thenReturn(Optional.of(new User(1L, "First call", "", true)))
            .thenReturn(Optional.of(new User(1L, "Second call", "", true)))
            .thenThrow(new RuntimeException("Third call fails"));
        
        assertEquals("First call", userService.getUserById(1L).name());
        assertEquals("Second call", userService.getUserById(1L).name());
        assertThrows(RuntimeException.class, () -> userService.getUserById(1L));
    }
}
```

---

## 26.6 TDD (Test-Driven Development)

```java
// TDD Cycle: Red → Green → Refactor

// ====== Step 1: Write failing test (Red) ======
class StringCalculatorTest {
    
    private StringCalculator calc;
    
    @BeforeEach
    void setup() { calc = new StringCalculator(); }
    
    @Test void emptyStringReturns0() { assertEquals(0, calc.add("")); }
    @Test void singleNumber() { assertEquals(5, calc.add("5")); }
    @Test void twoNumbers() { assertEquals(3, calc.add("1,2")); }
    @Test void multipleNumbers() { assertEquals(15, calc.add("1,2,3,4,5")); }
    @Test void newlineDelimiter() { assertEquals(6, calc.add("1\n2,3")); }
    
    @Test void customDelimiter() {
        assertEquals(3, calc.add("//;\n1;2"));
        assertEquals(6, calc.add("//|\n1|2|3"));
    }
    
    @Test void negativeNumbersThrow() {
        Exception ex = assertThrows(IllegalArgumentException.class,
            () -> calc.add("1,-2,3,-4"));
        assertTrue(ex.getMessage().contains("-2"));
        assertTrue(ex.getMessage().contains("-4"));
    }
    
    @Test void ignoreOver1000() { assertEquals(2, calc.add("2,1001")); }
    
    @Test void multiCharDelimiter() {
        assertEquals(6, calc.add("//[***]\n1***2***3"));
    }
    
    @Test void multipleDelimiters() {
        assertEquals(6, calc.add("//[*][%]\n1*2%3"));
    }
}

// ====== Step 2: Make tests pass (Green) ======
class StringCalculator {
    
    public int add(String numbers) {
        if (numbers.isEmpty()) return 0;
        
        String delimiter = "[,\n]";
        String input = numbers;
        
        if (numbers.startsWith("//")) {
            int newlinePos = numbers.indexOf('\n');
            String delimPart = numbers.substring(2, newlinePos);
            
            if (delimPart.startsWith("[")) {
                // Multiple or multi-char: //[***][%]\n
                delimiter = java.util.Arrays.stream(delimPart.split("\\]\\["))
                    .map(d -> d.replace("[", "").replace("]", ""))
                    .map(java.util.regex.Pattern::quote)
                    .reduce((a, b) -> a + "|" + b)
                    .orElse(delimPart);
            } else {
                delimiter = java.util.regex.Pattern.quote(delimPart);
            }
            input = numbers.substring(newlinePos + 1);
        }
        
        String[] parts = input.split(delimiter);
        List<Integer> nums = new java.util.ArrayList<>();
        List<Integer> negatives = new java.util.ArrayList<>();
        
        for (String part : parts) {
            int n = Integer.parseInt(part.trim());
            if (n < 0) negatives.add(n);
            else if (n <= 1000) nums.add(n);
        }
        
        if (!negatives.isEmpty()) {
            String msg = "Negatives not allowed: " + negatives;
            throw new IllegalArgumentException(msg);
        }
        
        return nums.stream().mapToInt(Integer::intValue).sum();
    }
}
```

---

## 26.7 Spring Boot Test

```java
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.mock.mockito.*;
import org.springframework.test.web.servlet.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.mockito.Mockito.*;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.*;
import com.fasterxml.jackson.databind.*;

@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    void getUser_success() throws Exception {
        User user = new User(1L, "Alice", "alice@example.com", true);
        when(userService.getUserById(1L)).thenReturn(user);
        
        mockMvc.perform(get("/api/users/1")
                   .contentType("application/json"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.id").value(1))
               .andExpect(jsonPath("$.name").value("Alice"))
               .andExpect(jsonPath("$.email").value("alice@example.com"));
    }
    
    @Test
    void getUser_notFound() throws Exception {
        when(userService.getUserById(99L))
            .thenThrow(new RuntimeException("User not found: 99"));
        
        mockMvc.perform(get("/api/users/99"))
               .andExpect(status().isNotFound());
    }
    
    @Test
    void createUser_success() throws Exception {
        var request = Map.of("name", "Bob", "email", "bob@example.com");
        User created = new User(2L, "Bob", "bob@example.com", true);
        
        when(userService.createUser("Bob", "bob@example.com")).thenReturn(created);
        
        mockMvc.perform(post("/api/users")
                   .contentType("application/json")
                   .content(objectMapper.writeValueAsString(request)))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.id").value(2))
               .andExpect(jsonPath("$.name").value("Bob"));
    }
    
    @Test
    void createUser_validationFails() throws Exception {
        var request = Map.of("name", "", "email", "invalid-email");
        
        mockMvc.perform(post("/api/users")
                   .contentType("application/json")
                   .content(objectMapper.writeValueAsString(request)))
               .andExpect(status().isBadRequest());
    }
}
```

---

## 26.8 Integration Test with @SpringBootTest

```java
import org.springframework.boot.test.context.*;
import org.springframework.test.context.*;
import org.springframework.transaction.annotation.*;

@SpringBootTest
@ActiveProfiles("test")
@Transactional  // rollback after each test
class UserIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void createAndRetrieve() {
        User created = userService.createUser("Alice", "alice@test.com");
        
        assertNotNull(created.id());
        
        User found = userService.getUserById(created.id());
        assertEquals("Alice", found.name());
        assertEquals("alice@test.com", found.email());
    }
    
    @Test
    void listActiveUsers() {
        userService.createUser("Active1", "a1@test.com");
        userService.createUser("Active2", "a2@test.com");
        
        List<User> active = userService.getActiveUsers();
        assertTrue(active.size() >= 2);
        assertTrue(active.stream().allMatch(User::active));
    }
}

// application-test.yml
/*
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
*/
```

---

## 26.9 Test Coverage & Best Practices

```java
// ====== Test Coverage Tips ======

// 1. AAA Pattern: Arrange - Act - Assert
@Test
void transferFunds_success() {
    // Arrange
    BankAccount source = new BankAccount("SRC", 1000);
    BankAccount dest = new BankAccount("DST", 200);
    
    // Act
    TransferService.transfer(source, dest, 300);
    
    // Assert
    assertEquals(700, source.getBalance());
    assertEquals(500, dest.getBalance());
}

// 2. Test edge cases
@Test
void edgeCases() {
    // Zero
    assertEquals(0, calc.add(0, 0));
    
    // MAX/MIN values
    assertThrows(ArithmeticException.class,
        () -> calc.add(Integer.MAX_VALUE, 1));
    
    // null handling
    assertThrows(NullPointerException.class,
        () -> service.process(null));
}

// 3. One assertion per test (mostly)
@Test
void user_hasCorrectName() {
    User user = createUser("Alice");
    assertEquals("Alice", user.getName());
}

@Test
void user_hasCorrectEmail() {
    User user = createUser("Alice", "alice@example.com");
    assertEquals("alice@example.com", user.getEmail());
}

// 4. Use descriptive test names
// Bad: @Test void test1()
// Good: @Test @DisplayName("User with duplicate email should throw DuplicateEmailException")

// 5. Test behaviors, not implementations
// Don't test: private methods, internal state
// Do test: public API, return values, exceptions, side effects

// 6. Use @ParameterizedTest to eliminate duplication
@ParameterizedTest
@CsvSource({"alice, ALICE", "Bob, BOB", "charlie, CHARLIE"})
void uppercaseName(String input, String expected) {
    assertEquals(expected, input.toUpperCase());
}
```

---

## สรุป Part 26

| Annotation | ความหมาย |
|-----------|---------|
| `@Test` | method เป็น test |
| `@BeforeEach` | ทำก่อนแต่ละ test |
| `@AfterEach` | ทำหลังแต่ละ test |
| `@BeforeAll` | ทำครั้งเดียวก่อนทุก test |
| `@Nested` | จัดกลุ่ม tests |
| `@ParameterizedTest` | รัน test หลายครั้งด้วยข้อมูลต่างกัน |
| `@Mock` | สร้าง mock object |
| `@InjectMocks` | inject mocks เข้า class |
| `@WebMvcTest` | test controller layer |
| `@SpringBootTest` | full application context |

**หลักการ TDD:**
1. **Red**: เขียน test ที่ fail
2. **Green**: เขียนโค้ดให้ test pass
3. **Refactor**: ปรับปรุงโค้ด

➡️ [Part 27: Spring Security & JWT](./Part-27-Spring-Security-JWT.md)
