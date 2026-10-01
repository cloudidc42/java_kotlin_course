# Part 13: Generics
## ขั้นตอนที่ 821-890: Generic Types และ Wildcards

---

## 13.1 Generic Classes และ Methods

```java
import java.util.*;
import java.util.function.*;

// ====== Generic Class ======
class Pair<A, B> {
    private final A first;
    private final B second;
    
    public Pair(A first, B second) {
        this.first = first;
        this.second = second;
    }
    
    public A getFirst() { return first; }
    public B getSecond() { return second; }
    
    public <C, D> Pair<C, D> map(Function<A, C> f1, Function<B, D> f2) {
        return new Pair<>(f1.apply(first), f2.apply(second));
    }
    
    public Pair<B, A> swap() { return new Pair<>(second, first); }
    
    @Override
    public String toString() { return "(" + first + ", " + second + ")"; }
    
    public static <A, B> Pair<A, B> of(A a, B b) { return new Pair<>(a, b); }
}

// Triple
class Triple<A, B, C> {
    private final A first;
    private final B second;
    private final C third;
    
    public Triple(A first, B second, C third) {
        this.first = first;
        this.second = second;
        this.third = third;
    }
    
    @Override
    public String toString() {
        return "(" + first + ", " + second + ", " + third + ")";
    }
    
    public static <A, B, C> Triple<A, B, C> of(A a, B b, C c) {
        return new Triple<>(a, b, c);
    }
}

// Generic Stack
class Stack<T> {
    private final LinkedList<T> data = new LinkedList<>();
    
    public void push(T item) { data.addFirst(item); }
    
    public T pop() {
        if (isEmpty()) throw new NoSuchElementException("Stack is empty");
        return data.removeFirst();
    }
    
    public T peek() {
        if (isEmpty()) throw new NoSuchElementException("Stack is empty");
        return data.getFirst();
    }
    
    public boolean isEmpty() { return data.isEmpty(); }
    public int size() { return data.size(); }
    
    @Override
    public String toString() { return data.toString(); }
}

// Generic methods
class GenericUtils {
    
    // Generic swap
    public static <T> void swap(T[] arr, int i, int j) {
        T temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    // Generic min
    public static <T extends Comparable<T>> T min(T a, T b) {
        return a.compareTo(b) <= 0 ? a : b;
    }
    
    // Generic max in array
    public static <T extends Comparable<T>> T max(T[] arr) {
        if (arr == null || arr.length == 0) throw new IllegalArgumentException("Empty array");
        T result = arr[0];
        for (T item : arr) {
            if (item.compareTo(result) > 0) result = item;
        }
        return result;
    }
    
    // Generic filter
    public static <T> List<T> filter(List<T> list, Predicate<T> predicate) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (predicate.test(item)) result.add(item);
        }
        return result;
    }
    
    // Generic transform
    public static <T, R> List<R> transform(List<T> list, Function<T, R> mapper) {
        List<R> result = new ArrayList<>(list.size());
        for (T item : list) result.add(mapper.apply(item));
        return result;
    }
}

public class GenericsBasics {
    
    public static void main(String[] args) {
        
        // Pair
        Pair<String, Integer> person = Pair.of("Alice", 30);
        System.out.println("Person: " + person);
        System.out.println("Name: " + person.getFirst());
        System.out.println("Swapped: " + person.swap());
        
        Pair<String, Boolean> status = person.map(String::toUpperCase, age -> age >= 18);
        System.out.println("Status: " + status);
        
        // Triple
        Triple<String, Integer, Double> record = Triple.of("Bob", 25, 75.5);
        System.out.println("Record: " + record);
        
        // Stack
        System.out.println("\n=== Generic Stack ===");
        Stack<Integer> stack = new Stack<>();
        stack.push(1); stack.push(2); stack.push(3);
        System.out.println("Stack: " + stack);
        System.out.println("Pop: " + stack.pop());
        System.out.println("Peek: " + stack.peek());
        
        // Generic methods
        System.out.println("\n=== Generic Methods ===");
        System.out.println("min(3, 7): " + GenericUtils.min(3, 7));
        System.out.println("min('apple','banana'): " + GenericUtils.min("apple", "banana"));
        
        Integer[] nums = {5, 2, 8, 1, 9, 3};
        System.out.println("max: " + GenericUtils.max(nums));
        
        List<Integer> list = List.of(1,2,3,4,5,6,7,8,9,10);
        List<Integer> evens = GenericUtils.filter(list, n -> n % 2 == 0);
        System.out.println("Evens: " + evens);
        
        List<String> strs = GenericUtils.transform(list, n -> "Item" + n);
        System.out.println("Transformed: " + strs.subList(0, 5));
    }
}
```

---

## 13.2 Bounded Type Parameters

```java
import java.util.*;

public class BoundedGenerics {
    
    // Upper bounded: T must be Number or subclass
    static <T extends Number> double sum(List<T> list) {
        return list.stream().mapToDouble(Number::doubleValue).sum();
    }
    
    // Multiple bounds: T must implement both Comparable and Serializable
    static <T extends Comparable<T> & java.io.Serializable> T clamp(T value, T min, T max) {
        if (value.compareTo(min) < 0) return min;
        if (value.compareTo(max) > 0) return max;
        return value;
    }
    
    // Generic binary search
    static <T extends Comparable<T>> int binarySearch(List<T> sorted, T target) {
        int lo = 0, hi = sorted.size() - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            int cmp = sorted.get(mid).compareTo(target);
            if (cmp == 0) return mid;
            else if (cmp < 0) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }
    
    // Wildcard: ? extends (producer - read only)
    static double sumNumbers(List<? extends Number> numbers) {
        double total = 0;
        for (Number n : numbers) total += n.doubleValue();
        return total;
    }
    
    // Wildcard: ? super (consumer - write only)
    static void addNumbers(List<? super Integer> list, int count) {
        for (int i = 0; i < count; i++) list.add(i);
    }
    
    // PECS: Producer Extends, Consumer Super
    static <T> void copy(List<? extends T> src, List<? super T> dst) {
        for (T item : src) dst.add(item);
    }
    
    public static void main(String[] args) {
        
        List<Integer> ints = List.of(1, 2, 3, 4, 5);
        List<Double> doubles = List.of(1.1, 2.2, 3.3);
        
        System.out.println("sum(ints): " + sum(ints));
        System.out.println("sum(doubles): " + sum(doubles));
        
        System.out.println("clamp(15, 0, 10): " + clamp(15, 0, 10));
        System.out.println("clamp(-5, 0, 10): " + clamp(-5, 0, 10));
        System.out.println("clamp(5, 0, 10): " + clamp(5, 0, 10));
        
        List<Integer> sorted = List.of(1, 3, 5, 7, 9, 11, 13);
        System.out.println("binarySearch(7): " + binarySearch(sorted, 7));
        System.out.println("binarySearch(6): " + binarySearch(sorted, 6));
        
        // PECS
        List<Integer> source = new ArrayList<>(List.of(10, 20, 30));
        List<Number> dest = new ArrayList<>();
        copy(source, dest);
        System.out.println("Copied: " + dest);
        
        // Wildcard consumer
        List<Number> numList = new ArrayList<>();
        addNumbers(numList, 5);
        System.out.println("Added: " + numList);
    }
}
```

---

## 13.3 Generic Interfaces & Type Erasure

```java
// Generic Repository Interface
interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
    long count();
}

// Generic with constraints
interface Validator<T> {
    boolean validate(T value);
    String getErrorMessage();
    
    default Validator<T> and(Validator<T> other) {
        return new Validator<T>() {
            @Override public boolean validate(T value) {
                return Validator.this.validate(value) && other.validate(value);
            }
            @Override public String getErrorMessage() {
                return Validator.this.getErrorMessage() + " AND " + other.getErrorMessage();
            }
        };
    }
    
    default Validator<T> or(Validator<T> other) {
        return new Validator<T>() {
            @Override public boolean validate(T value) {
                return Validator.this.validate(value) || other.validate(value);
            }
            @Override public String getErrorMessage() {
                return Validator.this.getErrorMessage() + " OR " + other.getErrorMessage();
            }
        };
    }
    
    static <T> Validator<T> of(java.util.function.Predicate<T> predicate, String message) {
        return new Validator<T>() {
            @Override public boolean validate(T value) { return predicate.test(value); }
            @Override public String getErrorMessage() { return message; }
        };
    }
}

// Example: User validator
class UserValidator {
    
    static final Validator<String> notEmpty = 
        Validator.of(s -> s != null && !s.isBlank(), "Must not be empty");
    
    static final Validator<String> emailFormat =
        Validator.of(s -> s != null && s.matches("[\\w.-]+@[\\w.-]+\\.[a-z]{2,}"),
                     "Must be valid email format");
    
    static final Validator<String> minLength8 =
        Validator.of(s -> s != null && s.length() >= 8, "Must be at least 8 characters");
    
    static final Validator<String> hasUpperCase =
        Validator.of(s -> s != null && s.chars().anyMatch(Character::isUpperCase),
                     "Must contain uppercase letter");
    
    static final Validator<String> passwordValidator =
        minLength8.and(hasUpperCase);
    
    static final Validator<String> emailValidator =
        notEmpty.and(emailFormat);
    
    static void validate(String field, String value, Validator<String> validator) {
        if (validator.validate(value)) {
            System.out.printf("✅ %s: '%s'%n", field, value);
        } else {
            System.out.printf("❌ %s: '%s' - %s%n", field, value, validator.getErrorMessage());
        }
    }
    
    public static void main(String[] args) {
        System.out.println("=== Email Validation ===");
        validate("email", "alice@example.com", emailValidator);
        validate("email", "not-an-email", emailValidator);
        validate("email", "", emailValidator);
        
        System.out.println("\n=== Password Validation ===");
        validate("password", "Secret123", passwordValidator);
        validate("password", "short", passwordValidator);
        validate("password", "nouppercase1", passwordValidator);
    }
}
```

---

## 13.4 Generic Data Structures

```java
import java.util.*;
import java.util.function.Consumer;

// Generic Binary Search Tree
class BST<T extends Comparable<T>> {
    
    private static class Node<T> {
        T data;
        Node<T> left, right;
        Node(T data) { this.data = data; }
    }
    
    private Node<T> root;
    
    public void insert(T value) {
        root = insertRec(root, value);
    }
    
    private Node<T> insertRec(Node<T> node, T value) {
        if (node == null) return new Node<>(value);
        int cmp = value.compareTo(node.data);
        if (cmp < 0) node.left = insertRec(node.left, value);
        else if (cmp > 0) node.right = insertRec(node.right, value);
        return node;
    }
    
    public boolean contains(T value) {
        Node<T> cur = root;
        while (cur != null) {
            int cmp = value.compareTo(cur.data);
            if (cmp == 0) return true;
            cur = cmp < 0 ? cur.left : cur.right;
        }
        return false;
    }
    
    // In-order traversal (sorted)
    public void inOrder(Consumer<T> action) { inOrderRec(root, action); }
    
    private void inOrderRec(Node<T> node, Consumer<T> action) {
        if (node == null) return;
        inOrderRec(node.left, action);
        action.accept(node.data);
        inOrderRec(node.right, action);
    }
    
    public List<T> toSortedList() {
        List<T> result = new ArrayList<>();
        inOrder(result::add);
        return result;
    }
    
    public static void main(String[] args) {
        
        BST<Integer> intTree = new BST<>();
        int[] values = {5, 3, 7, 1, 4, 6, 8};
        for (int v : values) intTree.insert(v);
        
        System.out.print("Sorted: ");
        intTree.inOrder(n -> System.out.print(n + " "));
        System.out.println();
        
        System.out.println("contains(4): " + intTree.contains(4));
        System.out.println("contains(9): " + intTree.contains(9));
        
        BST<String> strTree = new BST<>();
        String[] words = {"banana", "apple", "cherry", "date"};
        for (String w : words) strTree.insert(w);
        System.out.println("Sorted strings: " + strTree.toSortedList());
    }
}
```

---

## 13.5 Full Program: Type-Safe Event System

```java
import java.util.*;
import java.util.function.*;

// Generic Event System
interface Event {}
interface EventHandler<E extends Event> {
    void handle(E event);
}

class EventBus {
    @SuppressWarnings("unchecked")
    private final Map<Class<?>, List<EventHandler<?>>> handlers = new HashMap<>();
    
    public <E extends Event> void subscribe(Class<E> eventType, EventHandler<E> handler) {
        handlers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(handler);
    }
    
    @SuppressWarnings("unchecked")
    public <E extends Event> void publish(E event) {
        List<EventHandler<?>> eventHandlers = handlers.getOrDefault(event.getClass(), List.of());
        for (EventHandler<?> handler : eventHandlers) {
            ((EventHandler<E>) handler).handle(event);
        }
    }
}

// Events
record UserRegistered(String userId, String email) implements Event {}
record OrderPlaced(String orderId, String userId, double amount) implements Event {}
record PaymentProcessed(String orderId, boolean success) implements Event {}

public class EventSystemDemo {
    
    public static void main(String[] args) {
        EventBus bus = new EventBus();
        
        // Subscribe handlers
        bus.subscribe(UserRegistered.class, e -> {
            System.out.println("📧 Send welcome email to: " + e.email());
        });
        
        bus.subscribe(UserRegistered.class, e -> {
            System.out.println("📊 Analytics: New user " + e.userId());
        });
        
        bus.subscribe(OrderPlaced.class, e -> {
            System.out.printf("🛒 Order %s placed by %s (฿%.2f)%n",
                e.orderId(), e.userId(), e.amount());
        });
        
        bus.subscribe(PaymentProcessed.class, e -> {
            if (e.success()) {
                System.out.println("✅ Payment for order " + e.orderId() + " succeeded");
            } else {
                System.out.println("❌ Payment for order " + e.orderId() + " failed");
            }
        });
        
        // Publish events
        System.out.println("=== Event Stream ===");
        bus.publish(new UserRegistered("U001", "alice@example.com"));
        bus.publish(new OrderPlaced("O001", "U001", 1500.00));
        bus.publish(new PaymentProcessed("O001", true));
        bus.publish(new OrderPlaced("O002", "U001", 500.00));
        bus.publish(new PaymentProcessed("O002", false));
    }
}
```

---

## สรุป Part 13

| หัวข้อ | สาระสำคัญ |
|--------|----------|
| Generic Classes | `class Box<T>` type-safe container |
| Bounded Types | `<T extends Comparable<T>>` |
| Wildcards | `? extends`, `? super`, PECS |
| Generic Methods | `static <T> T max(T[] arr)` |
| Type Erasure | Generics เป็น compile-time เท่านั้น |
| Generic Data Structures | BST, Stack, Pair |

➡️ [Part 14: File I/O](./Part-14-File-IO.md)
