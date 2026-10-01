# Part 16: Lambda Expressions & Stream API
## ขั้นตอนที่ 1001-1080: Functional Programming ใน Java 8+

---

## 16.1 Lambda Expressions ทบทวนและขั้นสูง

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class LambdaAdvanced {
    
    // ====== Functional Interfaces ======
    
    @FunctionalInterface
    interface TriFunction<A, B, C, R> {
        R apply(A a, B b, C c);
    }
    
    @FunctionalInterface
    interface Memoizable<T, R> {
        R compute(T input);
        
        default Memoizable<T, R> memoize() {
            Map<T, R> cache = new HashMap<>();
            return input -> cache.computeIfAbsent(input, this::compute);
        }
    }
    
    // Closure - lambda captures variable
    static Supplier<Integer> makeCounter(int start) {
        int[] count = {start};  // array trick for effectively final
        return () -> count[0]++;
    }
    
    // Partial application
    static <A, B, C> Function<B, C> partial(BiFunction<A, B, C> f, A a) {
        return b -> f.apply(a, b);
    }
    
    // Currying
    static <A, B, C> Function<A, Function<B, C>> curry(BiFunction<A, B, C> f) {
        return a -> b -> f.apply(a, b);
    }
    
    // Function composition
    static <T> Function<T, T> compose(Function<T, T>... fns) {
        return Arrays.stream(fns).reduce(Function.identity(), Function::andThen);
    }
    
    public static void main(String[] args) {
        
        // TriFunction
        TriFunction<Integer, Integer, Integer, Integer> clamp =
            (value, min, max) -> Math.min(Math.max(value, min), max);
        System.out.println("clamp(15, 0, 10) = " + clamp.apply(15, 0, 10));
        System.out.println("clamp(-5, 0, 10) = " + clamp.apply(-5, 0, 10));
        
        // Memoize
        Memoizable<Integer, Long> fib = n -> {
            if (n <= 1) return (long) n;
            long a = 0, b = 1;
            for (int i = 2; i <= n; i++) { long t = a + b; a = b; b = t; }
            return b;
        };
        var memoFib = fib.memoize();
        System.out.println("\nFib(40) = " + memoFib.compute(40));
        System.out.println("Fib(40) = " + memoFib.compute(40));  // cached
        
        // Counter closure
        Supplier<Integer> counter = makeCounter(1);
        System.out.println("\nCounter: " + counter.get() + ", " + counter.get() + ", " + counter.get());
        
        // Partial application
        BiFunction<Integer, Integer> multiply = (a, b) -> a * b;
        Function<Integer, Integer> triple = partial(multiply, 3);
        System.out.println("\ntriple(5) = " + triple.apply(5));
        System.out.println("triple(7) = " + triple.apply(7));
        
        // Currying
        Function<Integer, Function<Integer, Integer>> curriedAdd = curry(Integer::sum);
        Function<Integer, Integer> add10 = curriedAdd.apply(10);
        System.out.println("\nadd10(5) = " + add10.apply(5));
        System.out.println("add10(20) = " + add10.apply(20));
        
        // Compose
        Function<String, String> pipeline = compose(
            String::trim,
            String::toLowerCase,
            s -> s.replaceAll("\\s+", "-")
        );
        System.out.println("\nPipeline: " + pipeline.apply("  Hello World  "));
        System.out.println("Pipeline: " + pipeline.apply("  Java Programming  "));
        
        // Method references
        List<String> names = List.of("Charlie", "Alice", "Bob", "David");
        names.stream()
             .sorted(String::compareTo)          // instance method ref
             .map(String::toUpperCase)           // instance method ref
             .forEach(System.out::println);      // instance method ref
    }
}
```

---

## 16.2 Stream API พื้นฐาน

```java
import java.util.*;
import java.util.stream.*;

public class StreamBasics {
    
    public static void main(String[] args) {
        
        // ====== Creating streams ======
        Stream<Integer> fromList = List.of(1, 2, 3, 4, 5).stream();
        Stream<String> ofValues = Stream.of("a", "b", "c");
        IntStream range = IntStream.range(1, 11);      // 1-10
        IntStream rangeClosed = IntStream.rangeClosed(1, 10);  // 1-10
        Stream<Double> random = Stream.generate(Math::random).limit(5);
        Stream<Integer> iterate = Stream.iterate(0, n -> n + 2).limit(10);
        
        System.out.println("=== Basic Streams ===");
        System.out.print("range: ");
        range.forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        System.out.print("iterate (even): ");
        iterate.forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // ====== Intermediate operations ======
        System.out.println("\n=== Intermediate Operations ===");
        
        List<Integer> nums = List.of(1,2,3,4,5,6,7,8,9,10);
        
        // filter
        System.out.print("filter (>5): ");
        nums.stream().filter(n -> n > 5).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // map
        System.out.print("map (×2): ");
        nums.stream().map(n -> n * 2).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // flatMap
        List<List<Integer>> nested = List.of(List.of(1,2,3), List.of(4,5), List.of(6,7,8,9));
        System.out.print("flatMap: ");
        nested.stream().flatMap(Collection::stream).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // sorted
        List<String> words = List.of("banana", "apple", "cherry", "date", "elderberry");
        System.out.print("sorted: ");
        words.stream().sorted().forEach(w -> System.out.print(w + " "));
        System.out.println();
        
        System.out.print("sorted (by length): ");
        words.stream().sorted(Comparator.comparingInt(String::length))
                      .forEach(w -> System.out.print(w + " "));
        System.out.println();
        
        // distinct
        List<Integer> dups = List.of(1,2,2,3,3,3,4,4,5);
        System.out.print("distinct: ");
        dups.stream().distinct().forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // limit & skip
        System.out.print("skip(3).limit(4): ");
        nums.stream().skip(3).limit(4).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // peek (for debugging)
        long count = nums.stream()
            .peek(n -> System.out.print("before:" + n + " "))
            .filter(n -> n % 2 == 0)
            .peek(n -> System.out.print("after:" + n + " "))
            .count();
        System.out.println("\neven count: " + count);
        
        // ====== Terminal operations ======
        System.out.println("\n=== Terminal Operations ===");
        
        System.out.println("count: " + nums.stream().filter(n -> n > 5).count());
        System.out.println("sum: " + nums.stream().mapToInt(Integer::intValue).sum());
        System.out.println("min: " + nums.stream().min(Integer::compareTo).orElse(-1));
        System.out.println("max: " + nums.stream().max(Integer::compareTo).orElse(-1));
        System.out.println("average: " + nums.stream().mapToInt(Integer::intValue).average().orElse(0));
        
        // anyMatch / allMatch / noneMatch
        System.out.println("anyMatch >8: " + nums.stream().anyMatch(n -> n > 8));
        System.out.println("allMatch >0: " + nums.stream().allMatch(n -> n > 0));
        System.out.println("noneMatch >10: " + nums.stream().noneMatch(n -> n > 10));
        
        // findFirst / findAny
        System.out.println("findFirst >5: " + nums.stream().filter(n -> n > 5).findFirst().orElse(-1));
        
        // reduce
        int product = nums.stream().reduce(1, (a, b) -> a * b);
        System.out.println("product: " + product);
        
        // collect to list
        List<Integer> filtered = nums.stream().filter(n -> n % 3 == 0).collect(Collectors.toList());
        System.out.println("divisible by 3: " + filtered);
        
        // joining
        String joined = words.stream().collect(Collectors.joining(", ", "[", "]"));
        System.out.println("joined: " + joined);
    }
}
```

---

## 16.3 Collectors ขั้นสูง

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

record Employee(String name, String dept, double salary, int age) {}

public class CollectorsDemo {
    
    public static void main(String[] args) {
        
        List<Employee> employees = List.of(
            new Employee("Alice", "Engineering", 85000, 28),
            new Employee("Bob", "Engineering", 92000, 35),
            new Employee("Charlie", "Marketing", 67000, 30),
            new Employee("Diana", "Engineering", 78000, 25),
            new Employee("Eve", "Marketing", 72000, 32),
            new Employee("Frank", "HR", 55000, 40),
            new Employee("Grace", "HR", 58000, 27),
            new Employee("Henry", "Engineering", 95000, 45)
        );
        
        // ====== groupingBy ======
        System.out.println("=== Group by Department ===");
        Map<String, List<Employee>> byDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept));
        byDept.forEach((dept, emps) -> {
            System.out.println(dept + ": " + emps.stream().map(Employee::name).toList());
        });
        
        // ====== counting ======
        System.out.println("\n=== Count per Department ===");
        Map<String, Long> countByDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
        countByDept.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(e -> System.out.printf("  %-15s: %d%n", e.getKey(), e.getValue()));
        
        // ====== averagingDouble ======
        System.out.println("\n=== Average Salary per Dept ===");
        Map<String, Double> avgSalary = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept,
                     Collectors.averagingDouble(Employee::salary)));
        avgSalary.forEach((dept, avg) ->
            System.out.printf("  %-15s: ฿%,.2f%n", dept, avg));
        
        // ====== summarizingDouble ======
        System.out.println("\n=== Salary Statistics ===");
        DoubleSummaryStatistics stats = employees.stream()
            .collect(Collectors.summarizingDouble(Employee::salary));
        System.out.printf("  Count: %d, Min: %,.0f, Max: %,.0f, Avg: %,.0f%n",
            stats.getCount(), stats.getMin(), stats.getMax(), stats.getAverage());
        
        // ====== partitioningBy ======
        System.out.println("\n=== Partitioning (salary > 75000) ===");
        Map<Boolean, List<Employee>> partition = employees.stream()
            .collect(Collectors.partitioningBy(e -> e.salary() > 75000));
        partition.forEach((isHigh, emps) ->
            System.out.println((isHigh ? "High" : "Regular") + ": " +
                emps.stream().map(Employee::name).toList()));
        
        // ====== toMap ======
        System.out.println("\n=== Name to Salary Map ===");
        Map<String, Double> nameToSalary = employees.stream()
            .collect(Collectors.toMap(Employee::name, Employee::salary));
        nameToSalary.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(3)
            .forEach(e -> System.out.printf("  %-10s: ฿%,.0f%n", e.getKey(), e.getValue()));
        
        // ====== downstream collectors ======
        System.out.println("\n=== Max Salary per Dept ===");
        Map<String, Optional<Employee>> maxSalaryPerDept = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept,
                     Collectors.maxBy(Comparator.comparingDouble(Employee::salary))));
        maxSalaryPerDept.forEach((dept, emp) ->
            emp.ifPresent(e -> System.out.printf("  %-15s: %s (฿%,.0f)%n",
                dept, e.name(), e.salary())));
        
        // ====== Custom Collector ======
        System.out.println("\n=== Custom: Department Report ===");
        Map<String, String> deptReport = employees.stream()
            .collect(Collectors.groupingBy(Employee::dept,
                     Collectors.mapping(
                         e -> String.format("%s(฿%,.0f)", e.name(), e.salary()),
                         Collectors.joining(", ")
                     )));
        deptReport.forEach((dept, report) ->
            System.out.println("  " + dept + ": " + report));
    }
}
```

---

## 16.4 Parallel Streams

```java
import java.util.*;
import java.util.stream.*;
import java.util.concurrent.*;

public class ParallelStreamDemo {
    
    static long sumSequential(List<Long> numbers) {
        return numbers.stream().mapToLong(Long::longValue).sum();
    }
    
    static long sumParallel(List<Long> numbers) {
        return numbers.parallelStream().mapToLong(Long::longValue).sum();
    }
    
    static boolean isPrime(long n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        for (long i = 3; i * i <= n; i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }
    
    public static void main(String[] args) {
        
        // Large sum test
        int N = 10_000_000;
        List<Long> bigList = LongStream.rangeClosed(1, N)
            .boxed().collect(Collectors.toList());
        
        long t1 = System.currentTimeMillis();
        long seqSum = sumSequential(bigList);
        long seqTime = System.currentTimeMillis() - t1;
        
        long t2 = System.currentTimeMillis();
        long parSum = sumParallel(bigList);
        long parTime = System.currentTimeMillis() - t2;
        
        System.out.printf("Sequential sum: %,d in %dms%n", seqSum, seqTime);
        System.out.printf("Parallel sum:   %,d in %dms%n", parSum, parTime);
        System.out.printf("Speedup: %.2fx (cores: %d)%n",
            (double) seqTime / parTime, Runtime.getRuntime().availableProcessors());
        
        // Prime counting
        System.out.println("\n=== Prime counting (1 - 1,000,000) ===");
        long limit = 1_000_000L;
        
        long t3 = System.currentTimeMillis();
        long seqPrimes = LongStream.rangeClosed(2, limit)
            .filter(ParallelStreamDemo::isPrime).count();
        long seqPrimeTime = System.currentTimeMillis() - t3;
        
        long t4 = System.currentTimeMillis();
        long parPrimes = LongStream.rangeClosed(2, limit)
            .parallel().filter(ParallelStreamDemo::isPrime).count();
        long parPrimeTime = System.currentTimeMillis() - t4;
        
        System.out.printf("Sequential: %,d primes in %dms%n", seqPrimes, seqPrimeTime);
        System.out.printf("Parallel:   %,d primes in %dms%n", parPrimes, parPrimeTime);
        System.out.printf("Speedup: %.2fx%n", (double) seqPrimeTime / parPrimeTime);
        
        // When NOT to use parallel
        System.out.println("\n=== WARNING: Order matters ===");
        
        // Sequential - maintains order
        System.out.print("Sequential forEach: ");
        IntStream.range(1, 6).forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // Parallel - may NOT maintain order
        System.out.print("Parallel forEach: ");
        IntStream.range(1, 6).parallel().forEach(n -> System.out.print(n + " "));
        System.out.println();
        
        // forEachOrdered - maintains order even with parallel
        System.out.print("Parallel forEachOrdered: ");
        IntStream.range(1, 6).parallel().forEachOrdered(n -> System.out.print(n + " "));
        System.out.println();
    }
}
```

---

## 16.5 Full Program: Data Analysis Pipeline

```java
import java.util.*;
import java.util.stream.*;

record Transaction(String id, String type, double amount, String category, String date) {}

public class DataAnalysisPipeline {
    
    static List<Transaction> generateData() {
        String[] types = {"income", "expense"};
        String[] categories = {"Food", "Transport", "Shopping", "Entertainment", "Salary", "Freelance"};
        String[] dates = {"2024-01", "2024-02", "2024-03", "2024-04", "2024-05", "2024-06"};
        Random rng = new Random(42);
        
        List<Transaction> txns = new ArrayList<>();
        for (int i = 1; i <= 100; i++) {
            String type = categories[rng.nextInt(categories.length)].equals("Salary") ||
                          categories[rng.nextInt(categories.length)].equals("Freelance")
                          ? "income" : "expense";
            String category = categories[rng.nextInt(categories.length)];
            type = (category.equals("Salary") || category.equals("Freelance")) ? "income" : "expense";
            double amount = 100 + rng.nextInt(9900);
            txns.add(new Transaction("T" + String.format("%03d", i),
                type, amount, category, dates[rng.nextInt(dates.length)]));
        }
        return txns;
    }
    
    public static void main(String[] args) {
        
        List<Transaction> transactions = generateData();
        
        System.out.println("=== Financial Analysis Report ===\n");
        System.out.println("Total transactions: " + transactions.size());
        
        // Income vs Expense
        Map<String, DoubleSummaryStatistics> typeSummary = transactions.stream()
            .collect(Collectors.groupingBy(Transaction::type,
                     Collectors.summarizingDouble(Transaction::amount)));
        
        typeSummary.forEach((type, stats) ->
            System.out.printf("%-8s: count=%2d, total=฿%8,.2f, avg=฿%7,.2f%n",
                type, (int) stats.getCount(), stats.getSum(), stats.getAverage()));
        
        // Spending by category
        System.out.println("\n--- Expenses by Category ---");
        transactions.stream()
            .filter(t -> t.type().equals("expense"))
            .collect(Collectors.groupingBy(Transaction::category,
                     Collectors.summingDouble(Transaction::amount)))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .forEach(e -> {
                int bars = (int)(e.getValue() / 1000);
                System.out.printf("  %-15s ฿%8,.2f %s%n",
                    e.getKey(), e.getValue(), "█".repeat(Math.min(bars, 20)));
            });
        
        // Monthly trend
        System.out.println("\n--- Monthly Trend ---");
        transactions.stream()
            .collect(Collectors.groupingBy(Transaction::date,
                     TreeMap::new,
                     Collectors.summarizingDouble(Transaction::amount)))
            .forEach((month, stats) ->
                System.out.printf("  %s: count=%2d, total=฿%8,.2f%n",
                    month, (int) stats.getCount(), stats.getSum()));
        
        // Top 5 largest transactions
        System.out.println("\n--- Top 5 Largest Transactions ---");
        transactions.stream()
            .sorted(Comparator.comparingDouble(Transaction::amount).reversed())
            .limit(5)
            .forEach(t -> System.out.printf("  [%s] %-10s %-8s ฿%8,.2f%n",
                t.date(), t.category(), t.type(), t.amount()));
        
        // Net balance by month
        System.out.println("\n--- Net Balance by Month ---");
        transactions.stream()
            .collect(Collectors.groupingBy(Transaction::date, TreeMap::new,
                     Collectors.summingDouble(t ->
                         t.type().equals("income") ? t.amount() : -t.amount())))
            .forEach((month, net) ->
                System.out.printf("  %s: %s฿%,.2f%n",
                    month, net >= 0 ? "+" : "", net));
    }
}
```

---

## สรุป Part 16

| Feature | คำอธิบาย |
|---------|---------|
| Lambda | `(params) -> expression` |
| Method Reference | `Class::method`, `obj::method` |
| Functional Interface | Interface ที่มี 1 abstract method |
| Stream | Lazy sequence, intermediate + terminal ops |
| Collectors | groupingBy, counting, joining, toMap |
| Parallel Stream | `.parallelStream()` สำหรับงาน CPU-intensive |

➡️ [Part 17: Optional & Date/Time API](./Part-17-Optional-DateTime.md)
