# Part 12: Collections Framework
## ขั้นตอนที่ 751-820: คอลเลกชันใน Java

---

## 12.1 Collections Overview

```
Collection<E>
├── List<E>       - ลำดับ, อนุญาต duplicate
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector (legacy)
├── Set<E>        - ไม่มี duplicate
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet (sorted)
└── Queue<E>      - FIFO
    ├── LinkedList
    ├── PriorityQueue
    └── ArrayDeque

Map<K,V>          - key-value pairs
├── HashMap
├── LinkedHashMap
├── TreeMap (sorted)
└── Hashtable (legacy)
```

---

## 12.2 List Implementations

```java
import java.util.*;
import java.util.stream.*;

public class ListDemo {
    
    public static void main(String[] args) {
        
        // ====== ArrayList - dynamic array ======
        System.out.println("=== ArrayList ===");
        ArrayList<String> list = new ArrayList<>();
        
        // Add
        list.add("Java");
        list.add("Python");
        list.add("Kotlin");
        list.add(1, "C++");    // insert at index
        list.addAll(List.of("Go", "Rust"));
        System.out.println(list);
        
        // Access
        System.out.println("get(2): " + list.get(2));
        System.out.println("size: " + list.size());
        System.out.println("indexOf('Go'): " + list.indexOf("Go"));
        System.out.println("contains('Rust'): " + list.contains("Rust"));
        
        // Update
        list.set(0, "Java 21");
        
        // Remove
        list.remove("Python");
        list.remove(0);  // by index
        System.out.println("After modifications: " + list);
        
        // Iterate
        System.out.print("ForEach: ");
        list.forEach(s -> System.out.print(s + " "));
        System.out.println();
        
        // Sort
        list.sort(String::compareTo);
        System.out.println("Sorted: " + list);
        
        // ====== LinkedList - doubly linked list ======
        System.out.println("\n=== LinkedList ===");
        LinkedList<Integer> linked = new LinkedList<>();
        linked.addFirst(1);
        linked.addLast(3);
        linked.add(1, 2);
        linked.addFirst(0);
        System.out.println("LinkedList: " + linked);
        System.out.println("First: " + linked.peekFirst());
        System.out.println("Last: " + linked.peekLast());
        linked.pollFirst();
        linked.pollLast();
        System.out.println("After poll: " + linked);
        
        // ====== Immutable lists ======
        System.out.println("\n=== Immutable Lists ===");
        List<String> immutable = List.of("a", "b", "c");
        List<String> copy = new ArrayList<>(immutable);  // mutable copy
        copy.add("d");
        System.out.println("Original: " + immutable);
        System.out.println("Copy: " + copy);
        
        // ====== List operations ======
        System.out.println("\n=== Sublist & Copy ===");
        List<Integer> nums = new ArrayList<>(List.of(1,2,3,4,5,6,7,8,9,10));
        List<Integer> sub = nums.subList(3, 7); // [4,5,6,7]
        System.out.println("subList(3,7): " + sub);
        
        Collections.shuffle(nums);
        System.out.println("Shuffled: " + nums);
        Collections.sort(nums);
        System.out.println("Sorted back: " + nums);
        Collections.reverse(nums);
        System.out.println("Reversed: " + nums);
        System.out.println("Min: " + Collections.min(nums));
        System.out.println("Max: " + Collections.max(nums));
        System.out.println("Frequency of 5: " + Collections.frequency(nums, 5));
    }
}
```

---

## 12.3 Set Implementations

```java
import java.util.*;

public class SetDemo {
    
    public static void main(String[] args) {
        
        // ====== HashSet - no order, O(1) ops ======
        System.out.println("=== HashSet ===");
        HashSet<String> hashSet = new HashSet<>();
        hashSet.add("Banana");
        hashSet.add("Apple");
        hashSet.add("Mango");
        hashSet.add("Apple");  // duplicate, ignored
        hashSet.add("Cherry");
        System.out.println("HashSet: " + hashSet);  // no guaranteed order
        System.out.println("contains Apple: " + hashSet.contains("Apple"));
        hashSet.remove("Mango");
        System.out.println("After remove: " + hashSet);
        
        // ====== LinkedHashSet - insertion order ======
        System.out.println("\n=== LinkedHashSet ===");
        LinkedHashSet<String> linkedSet = new LinkedHashSet<>();
        linkedSet.add("Banana");
        linkedSet.add("Apple");
        linkedSet.add("Mango");
        linkedSet.add("Apple");  // ignored
        linkedSet.add("Cherry");
        System.out.println("LinkedHashSet: " + linkedSet);  // insertion order
        
        // ====== TreeSet - sorted ======
        System.out.println("\n=== TreeSet ===");
        TreeSet<String> treeSet = new TreeSet<>();
        treeSet.add("Banana");
        treeSet.add("Apple");
        treeSet.add("Mango");
        treeSet.add("Cherry");
        System.out.println("TreeSet: " + treeSet);  // sorted alphabetically
        System.out.println("first(): " + treeSet.first());
        System.out.println("last(): " + treeSet.last());
        System.out.println("headSet('M'): " + treeSet.headSet("M"));  // < M
        System.out.println("tailSet('M'): " + treeSet.tailSet("M"));  // >= M
        System.out.println("floor('Coconut'): " + treeSet.floor("Coconut"));
        System.out.println("ceiling('Coconut'): " + treeSet.ceiling("Coconut"));
        
        // ====== Set operations ======
        System.out.println("\n=== Set Operations ===");
        Set<Integer> setA = new HashSet<>(Set.of(1, 2, 3, 4, 5));
        Set<Integer> setB = new HashSet<>(Set.of(3, 4, 5, 6, 7));
        
        // Union
        Set<Integer> union = new HashSet<>(setA);
        union.addAll(setB);
        System.out.println("Union: " + new TreeSet<>(union));
        
        // Intersection
        Set<Integer> intersection = new HashSet<>(setA);
        intersection.retainAll(setB);
        System.out.println("Intersection: " + new TreeSet<>(intersection));
        
        // Difference (A - B)
        Set<Integer> difference = new HashSet<>(setA);
        difference.removeAll(setB);
        System.out.println("Difference (A-B): " + new TreeSet<>(difference));
        
        // Subset check
        Set<Integer> small = Set.of(3, 4);
        System.out.println("Is {3,4} subset of A? " + setA.containsAll(small));
        
        // ====== Remove duplicates from List ======
        System.out.println("\n=== Remove Duplicates ===");
        List<Integer> withDups = Arrays.asList(1, 2, 3, 2, 4, 1, 5, 3);
        Set<Integer> unique = new LinkedHashSet<>(withDups);  // preserve order
        System.out.println("Original: " + withDups);
        System.out.println("Unique: " + unique);
    }
}
```

---

## 12.4 Map Implementations

```java
import java.util.*;
import java.util.stream.*;

public class MapDemo {
    
    public static void main(String[] args) {
        
        // ====== HashMap ======
        System.out.println("=== HashMap ===");
        HashMap<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 95);
        scores.put("Bob", 87);
        scores.put("Charlie", 92);
        scores.put("Alice", 98);  // update
        
        System.out.println("Scores: " + scores);
        System.out.println("Alice: " + scores.get("Alice"));
        System.out.println("Dave (default): " + scores.getOrDefault("Dave", 0));
        System.out.println("containsKey Bob: " + scores.containsKey("Bob"));
        
        // putIfAbsent
        scores.putIfAbsent("Alice", 100);  // won't update
        scores.putIfAbsent("Eve", 88);
        
        // compute
        scores.compute("Bob", (k, v) -> v + 5);  // Bob gets +5
        scores.computeIfAbsent("Frank", k -> 75);
        scores.computeIfPresent("Charlie", (k, v) -> v - 2);
        
        // merge
        Map<String, Integer> newScores = Map.of("Alice", 2, "Bob", 3);
        newScores.forEach((name, bonus) ->
            scores.merge(name, bonus, Integer::sum));
        
        System.out.println("After updates: " + scores);
        
        // Iteration
        System.out.println("\nEntry iteration:");
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.printf("  %-10s: %d%n", entry.getKey(), entry.getValue());
        }
        
        System.out.print("Keys: ");
        scores.keySet().forEach(k -> System.out.print(k + " "));
        System.out.println();
        
        // ====== LinkedHashMap - insertion order ======
        System.out.println("\n=== LinkedHashMap ===");
        LinkedHashMap<String, String> capitals = new LinkedHashMap<>();
        capitals.put("Thailand", "Bangkok");
        capitals.put("Japan", "Tokyo");
        capitals.put("France", "Paris");
        capitals.put("USA", "Washington D.C.");
        capitals.forEach((country, capital) ->
            System.out.println(country + " -> " + capital));
        
        // LRU Cache with LinkedHashMap
        LinkedHashMap<Integer, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
                return size() > 3;  // max 3 entries
            }
        };
        lruCache.put(1, "One");
        lruCache.put(2, "Two");
        lruCache.put(3, "Three");
        lruCache.get(1);  // access 1 (moves to recent)
        lruCache.put(4, "Four");  // evicts 2 (least recently used)
        System.out.println("\nLRU Cache: " + lruCache);
        
        // ====== TreeMap - sorted keys ======
        System.out.println("\n=== TreeMap ===");
        TreeMap<String, Integer> sorted = new TreeMap<>(scores);
        System.out.println("Sorted by name: " + sorted);
        System.out.println("firstKey: " + sorted.firstKey());
        System.out.println("lastKey: " + sorted.lastKey());
        System.out.println("headMap('D'): " + sorted.headMap("D"));
        
        // ====== Frequency count pattern ======
        System.out.println("\n=== Word Frequency ===");
        String text = "the quick brown fox jumps over the lazy dog the fox";
        String[] words = text.split("\\s+");
        
        Map<String, Long> frequency = Arrays.stream(words)
            .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
        
        frequency.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(e -> System.out.printf("%-10s: %d%n", e.getKey(), e.getValue()));
    }
}
```

---

## 12.5 Queue & Stack

```java
import java.util.*;

public class QueueStackDemo {
    
    public static void main(String[] args) {
        
        // ====== Queue (FIFO) ======
        System.out.println("=== Queue (FIFO) ===");
        Queue<String> queue = new LinkedList<>();
        queue.offer("First");
        queue.offer("Second");
        queue.offer("Third");
        System.out.println("Queue: " + queue);
        System.out.println("peek: " + queue.peek());    // look without remove
        System.out.println("poll: " + queue.poll());    // remove and return
        System.out.println("After poll: " + queue);
        
        // ====== Deque (double-ended queue) ======
        System.out.println("\n=== Deque ===");
        Deque<String> deque = new ArrayDeque<>();
        deque.addFirst("Middle");
        deque.addFirst("First");
        deque.addLast("Last");
        System.out.println("Deque: " + deque);
        System.out.println("peekFirst: " + deque.peekFirst());
        System.out.println("peekLast: " + deque.peekLast());
        
        // Use as Stack (LIFO)
        System.out.println("\n=== Stack (using Deque) ===");
        Deque<String> stack = new ArrayDeque<>();
        stack.push("1st");
        stack.push("2nd");
        stack.push("3rd");
        System.out.println("Stack: " + stack);
        System.out.println("pop: " + stack.pop());
        System.out.println("peek: " + stack.peek());
        System.out.println("After pop: " + stack);
        
        // ====== PriorityQueue ======
        System.out.println("\n=== PriorityQueue (min-heap) ===");
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        minHeap.addAll(List.of(5, 1, 3, 2, 4));
        while (!minHeap.isEmpty()) {
            System.out.print(minHeap.poll() + " ");
        }
        System.out.println();
        
        // Max-heap
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        maxHeap.addAll(List.of(5, 1, 3, 2, 4));
        System.out.print("MaxHeap: ");
        while (!maxHeap.isEmpty()) {
            System.out.print(maxHeap.poll() + " ");
        }
        System.out.println();
        
        // Custom priority
        record Task(String name, int priority) {}
        PriorityQueue<Task> taskQueue = new PriorityQueue<>(
            Comparator.comparingInt(Task::priority).reversed()
        );
        taskQueue.add(new Task("Low priority", 1));
        taskQueue.add(new Task("High priority", 10));
        taskQueue.add(new Task("Medium priority", 5));
        taskQueue.add(new Task("Critical", 100));
        
        System.out.println("\nTask execution order:");
        while (!taskQueue.isEmpty()) {
            Task t = taskQueue.poll();
            System.out.printf("  [P%3d] %s%n", t.priority(), t.name());
        }
        
        // ====== Application: Balanced Parentheses ======
        System.out.println("\n=== Balanced Parentheses ===");
        String[] expressions = { "(())", "({[]})", "([)]", "(((" };
        for (String expr : expressions) {
            System.out.printf("%-10s : %s%n", expr, isBalanced(expr) ? "✅" : "❌");
        }
    }
    
    static boolean isBalanced(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if ("([{".indexOf(c) >= 0) {
                stack.push(c);
            } else if (")]}".indexOf(c) >= 0) {
                if (stack.isEmpty()) return false;
                char top = stack.pop();
                if (c == ')' && top != '(') return false;
                if (c == ']' && top != '[') return false;
                if (c == '}' && top != '{') return false;
            }
        }
        return stack.isEmpty();
    }
}
```

---

## 12.6 Full Program: Student Management System

```java
import java.util.*;
import java.util.stream.*;

record Student(String id, String name, int age, String major, double gpa) {
    public boolean isHonors() { return gpa >= 3.5; }
}

class StudentDB {
    private final Map<String, Student> students = new LinkedHashMap<>();
    
    public void add(Student s) {
        students.put(s.id(), s);
        System.out.println("Added: " + s.name());
    }
    
    public Optional<Student> findById(String id) {
        return Optional.ofNullable(students.get(id));
    }
    
    public List<Student> findByMajor(String major) {
        return students.values().stream()
            .filter(s -> s.major().equalsIgnoreCase(major))
            .sorted(Comparator.comparing(Student::name))
            .toList();
    }
    
    public List<Student> getTopStudents(int n) {
        return students.values().stream()
            .sorted(Comparator.comparingDouble(Student::gpa).reversed())
            .limit(n)
            .toList();
    }
    
    public Map<String, Double> averageGpaByMajor() {
        return students.values().stream()
            .collect(Collectors.groupingBy(
                Student::major,
                Collectors.averagingDouble(Student::gpa)
            ));
    }
    
    public Map<String, List<Student>> groupByMajor() {
        return students.values().stream()
            .collect(Collectors.groupingBy(Student::major));
    }
    
    public DoubleSummaryStatistics gpaStatistics() {
        return students.values().stream()
            .mapToDouble(Student::gpa)
            .summaryStatistics();
    }
    
    public Set<String> getHonorsIds() {
        return students.values().stream()
            .filter(Student::isHonors)
            .map(Student::id)
            .collect(Collectors.toSet());
    }
    
    public void printReport() {
        System.out.println("\n╔═══════════════════════════════════════════════════════════════╗");
        System.out.println("║                    รายงานนักศึกษา                            ║");
        System.out.println("╠═══════════════════════════════════════════════════════════════╣");
        System.out.printf("║ %-10s %-15s %-5s %-12s %5s ║%n",
            "รหัส", "ชื่อ", "อายุ", "สาขา", "GPA");
        System.out.println("╠═══════════════════════════════════════════════════════════════╣");
        
        students.values().stream()
            .sorted(Comparator.comparing(Student::major).thenComparing(Student::name))
            .forEach(s -> System.out.printf("║ %-10s %-15s %5d %-12s %5.2f ║%n",
                s.id(), s.name(), s.age(), s.major(), s.gpa()));
        
        System.out.println("╚═══════════════════════════════════════════════════════════════╝");
        
        DoubleSummaryStatistics stats = gpaStatistics();
        System.out.printf("  Total: %d | Min GPA: %.2f | Max GPA: %.2f | Avg GPA: %.2f%n",
            (int) stats.getCount(), stats.getMin(), stats.getMax(), stats.getAverage());
    }
    
    public static void main(String[] args) {
        StudentDB db = new StudentDB();
        
        db.add(new Student("S001", "Alice Smith", 20, "CS", 3.8));
        db.add(new Student("S002", "Bob Jones", 21, "Math", 3.2));
        db.add(new Student("S003", "Charlie Brown", 22, "CS", 3.9));
        db.add(new Student("S004", "Diana Prince", 20, "Physics", 3.7));
        db.add(new Student("S005", "Eve Wilson", 21, "CS", 3.4));
        db.add(new Student("S006", "Frank Miller", 22, "Math", 3.6));
        db.add(new Student("S007", "Grace Lee", 20, "Physics", 3.1));
        db.add(new Student("S008", "Henry Ford", 21, "CS", 3.5));
        
        db.printReport();
        
        System.out.println("\nTop 3 Students:");
        db.getTopStudents(3)
            .forEach(s -> System.out.printf("  %s (%s) GPA: %.2f%n", s.name(), s.major(), s.gpa()));
        
        System.out.println("\nCS Students:");
        db.findByMajor("CS")
            .forEach(s -> System.out.printf("  %s - GPA: %.2f%n", s.name(), s.gpa()));
        
        System.out.println("\nAverage GPA by Major:");
        db.averageGpaByMajor().entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .forEach(e -> System.out.printf("  %-10s: %.2f%n", e.getKey(), e.getValue()));
        
        System.out.println("\nHonors Students (GPA >= 3.5): " + db.getHonorsIds());
    }
}
```

---

## สรุป Part 12

| Collection | Use Case | Complexity |
|-----------|----------|------------|
| ArrayList | Random access, dynamic | O(1) get, O(n) insert |
| LinkedList | Queue/Deque, frequent insert | O(1) insert ends, O(n) access |
| HashSet | Fast lookup, no duplicates | O(1) add/contains |
| TreeSet | Sorted set | O(log n) |
| HashMap | Key-value lookup | O(1) avg |
| TreeMap | Sorted key-value | O(log n) |
| PriorityQueue | Priority ordering | O(log n) |

➡️ [Part 13: Generics](./Part-13-Generics.md)
