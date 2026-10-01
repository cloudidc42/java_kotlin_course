# Part 15: Multithreading & Concurrency
## ขั้นตอนที่ 961-1000+: การเขียนโปรแกรมแบบ Concurrent

---

## 15.1 Thread Basics

```java
import java.util.concurrent.*;

public class ThreadBasics {
    
    // Creating threads: 3 ways
    
    // 1. Extend Thread
    static class MyThread extends Thread {
        private String name;
        public MyThread(String name) { this.name = name; }
        
        @Override
        public void run() {
            for (int i = 1; i <= 5; i++) {
                System.out.printf("[%s] Step %d (Thread: %s)%n",
                    name, i, Thread.currentThread().getName());
                try { Thread.sleep(100); } catch (InterruptedException e) { break; }
            }
        }
    }
    
    // 2. Implement Runnable
    static class MyRunnable implements Runnable {
        private String task;
        public MyRunnable(String task) { this.task = task; }
        
        @Override
        public void run() {
            System.out.println("Running: " + task + " on " + Thread.currentThread().getName());
        }
    }
    
    // 3. Lambda (Runnable)
    static Runnable createTask(String name, int steps) {
        return () -> {
            for (int i = 1; i <= steps; i++) {
                System.out.printf("[%s] %d/%d%n", name, i, steps);
                try { Thread.sleep(50); } catch (InterruptedException e) { break; }
            }
        };
    }
    
    public static void main(String[] args) throws InterruptedException {
        
        // Method 1: Extend Thread
        System.out.println("=== Thread subclass ===");
        MyThread t1 = new MyThread("Worker-A");
        MyThread t2 = new MyThread("Worker-B");
        t1.start();
        t2.start();
        t1.join();  // wait for completion
        t2.join();
        
        // Method 2: Runnable
        System.out.println("\n=== Runnable ===");
        Thread t3 = new Thread(new MyRunnable("Fetch data"));
        Thread t4 = new Thread(new MyRunnable("Process data"));
        t3.start(); t4.start();
        t3.join(); t4.join();
        
        // Method 3: Lambda
        System.out.println("\n=== Lambda threads ===");
        Thread a = new Thread(createTask("Alpha", 3));
        Thread b = new Thread(createTask("Beta", 3));
        a.start(); b.start();
        a.join(); b.join();
        
        // Thread info
        Thread current = Thread.currentThread();
        System.out.printf("\nMain thread: id=%d, name=%s, priority=%d, alive=%b%n",
            current.getId(), current.getName(), current.getPriority(), current.isAlive());
        
        // Thread states
        Thread sleeping = new Thread(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) {}
        });
        sleeping.start();
        System.out.println("Thread state: " + sleeping.getState());
        sleeping.join();
        System.out.println("Thread state after join: " + sleeping.getState());
    }
}
```

---

## 15.2 Synchronization

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class SynchronizationDemo {
    
    // ====== Race Condition (BAD) ======
    static int unsafeCounter = 0;
    
    static void unsafeIncrement() {
        unsafeCounter++; // NOT atomic!
    }
    
    // ====== synchronized method (SAFE) ======
    static int safeCounter = 0;
    
    static synchronized void safeIncrement() {
        safeCounter++;
    }
    
    // ====== synchronized block ======
    static int blockCounter = 0;
    static final Object lock = new Object();
    
    static void blockIncrement() {
        synchronized (lock) {
            blockCounter++;
        }
    }
    
    // ====== AtomicInteger (BEST) ======
    static AtomicInteger atomicCounter = new AtomicInteger(0);
    
    // ====== Bank Account with locks ======
    static class BankAccount {
        private double balance;
        private final Object balanceLock = new Object();
        
        public BankAccount(double initial) { this.balance = initial; }
        
        public void deposit(double amount) {
            synchronized (balanceLock) {
                double old = balance;
                balance += amount;
                System.out.printf("Deposit ฿%.0f: ฿%.0f → ฿%.0f [%s]%n",
                    amount, old, balance, Thread.currentThread().getName());
            }
        }
        
        public boolean withdraw(double amount) {
            synchronized (balanceLock) {
                if (amount > balance) {
                    System.out.println("Withdraw failed: insufficient funds");
                    return false;
                }
                double old = balance;
                balance -= amount;
                System.out.printf("Withdraw ฿%.0f: ฿%.0f → ฿%.0f [%s]%n",
                    amount, old, balance, Thread.currentThread().getName());
                return true;
            }
        }
        
        public double getBalance() {
            synchronized (balanceLock) { return balance; }
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        
        int numThreads = 10;
        int incrementsPerThread = 1000;
        
        // Test counters
        Thread[] threads = new Thread[numThreads];
        
        for (int i = 0; i < numThreads; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    unsafeIncrement();
                    safeIncrement();
                    blockIncrement();
                    atomicCounter.incrementAndGet();
                }
            });
        }
        
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
        
        int expected = numThreads * incrementsPerThread;
        System.out.println("Expected: " + expected);
        System.out.println("Unsafe counter: " + unsafeCounter + (unsafeCounter != expected ? " ⚠️" : " ✅"));
        System.out.println("Safe counter: " + safeCounter + " ✅");
        System.out.println("Block counter: " + blockCounter + " ✅");
        System.out.println("Atomic counter: " + atomicCounter.get() + " ✅");
        
        // Bank account test
        System.out.println("\n=== BankAccount Concurrency ===");
        BankAccount account = new BankAccount(1000);
        
        ExecutorService executor = Executors.newFixedThreadPool(4);
        for (int i = 0; i < 5; i++) {
            executor.submit(() -> account.deposit(100));
            executor.submit(() -> account.withdraw(50));
        }
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        System.out.printf("Final balance: ฿%.0f%n", account.getBalance());
    }
}
```

---

## 15.3 ExecutorService

```java
import java.util.concurrent.*;
import java.util.*;
import java.util.stream.*;

public class ExecutorServiceDemo {
    
    public static void main(String[] args) throws Exception {
        
        // ====== Fixed Thread Pool ======
        System.out.println("=== Fixed Thread Pool ===");
        ExecutorService fixed = Executors.newFixedThreadPool(3);
        
        List<Future<String>> futures = new ArrayList<>();
        for (int i = 1; i <= 6; i++) {
            final int taskId = i;
            Future<String> f = fixed.submit(() -> {
                Thread.sleep(200);
                return "Task " + taskId + " done by " + Thread.currentThread().getName();
            });
            futures.add(f);
        }
        
        for (Future<String> f : futures) {
            System.out.println(f.get());  // blocking
        }
        fixed.shutdown();
        
        // ====== CompletableFuture ======
        System.out.println("\n=== CompletableFuture ===");
        
        // Async task chain
        CompletableFuture<String> result = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("Fetching data...");
                try { Thread.sleep(100); } catch (Exception e) {}
                return "Raw Data";
            })
            .thenApply(data -> {
                System.out.println("Processing: " + data);
                return data.toUpperCase() + " (processed)";
            })
            .thenApply(data -> {
                System.out.println("Formatting: " + data);
                return "Result: " + data;
            });
        
        System.out.println(result.get());
        
        // Combine futures
        CompletableFuture<Integer> priceTask = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(100); } catch (Exception e) {}
            return 1000;
        });
        
        CompletableFuture<Integer> discountTask = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(150); } catch (Exception e) {}
            return 100;
        });
        
        CompletableFuture<Integer> finalPrice = priceTask.thenCombine(discountTask,
            (price, discount) -> price - discount);
        
        System.out.println("Final price: " + finalPrice.get());
        
        // Run multiple and wait for all
        List<CompletableFuture<String>> tasks = IntStream.rangeClosed(1, 5)
            .mapToObj(i -> CompletableFuture.supplyAsync(() -> {
                try { Thread.sleep((long)(Math.random() * 200)); } catch (Exception e) {}
                return "Task" + i + " completed";
            }))
            .collect(Collectors.toList());
        
        CompletableFuture<Void> all = CompletableFuture.allOf(tasks.toArray(new CompletableFuture[0]));
        all.get(); // wait for all
        
        System.out.println("All tasks done:");
        tasks.forEach(t -> {
            try { System.out.println("  " + t.get()); } catch (Exception e) {}
        });
        
        // Error handling
        System.out.println("\n=== Error Handling ===");
        CompletableFuture<Integer> withError = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.3) throw new RuntimeException("Random failure");
                return 42;
            })
            .exceptionally(ex -> {
                System.out.println("Caught: " + ex.getMessage());
                return -1;  // fallback
            });
        
        System.out.println("Result (or fallback): " + withError.get());
    }
}
```

---

## 15.4 Concurrent Data Structures

```java
import java.util.concurrent.*;
import java.util.*;

public class ConcurrentCollectionsDemo {
    
    public static void main(String[] args) throws InterruptedException {
        
        // ====== ConcurrentHashMap ======
        System.out.println("=== ConcurrentHashMap ===");
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        ExecutorService exec = Executors.newFixedThreadPool(5);
        
        // Concurrent word count
        String[] words = {"java", "kotlin", "java", "python", "java", "kotlin", "go"};
        CountDownLatch latch = new CountDownLatch(words.length);
        
        for (String word : words) {
            exec.submit(() -> {
                map.merge(word, 1, Integer::sum);
                latch.countDown();
            });
        }
        
        latch.await();
        System.out.println("Word counts: " + new TreeMap<>(map));
        
        // ====== BlockingQueue (Producer-Consumer) ======
        System.out.println("\n=== Producer-Consumer with BlockingQueue ===");
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);
        
        // Producer
        Thread producer = new Thread(() -> {
            String[] items = {"A", "B", "C", "D", "E", "F", "G"};
            for (String item : items) {
                try {
                    queue.put(item);  // blocks if full
                    System.out.println("Produced: " + item + " (queue size: " + queue.size() + ")");
                    Thread.sleep(50);
                } catch (InterruptedException e) { break; }
            }
        });
        
        // Consumer
        Thread consumer = new Thread(() -> {
            int count = 0;
            while (count < 7) {
                try {
                    String item = queue.poll(1, TimeUnit.SECONDS);  // wait up to 1s
                    if (item != null) {
                        System.out.println("  Consumed: " + item);
                        Thread.sleep(120);
                        count++;
                    }
                } catch (InterruptedException e) { break; }
            }
        });
        
        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
        
        // ====== Semaphore ======
        System.out.println("\n=== Semaphore (Resource Pool) ===");
        Semaphore semaphore = new Semaphore(3);  // max 3 concurrent
        
        for (int i = 1; i <= 8; i++) {
            final int id = i;
            exec.submit(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("Worker " + id + " acquired resource");
                    Thread.sleep(200);
                    System.out.println("Worker " + id + " released resource");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    semaphore.release();
                }
            });
        }
        
        exec.shutdown();
        exec.awaitTermination(5, TimeUnit.SECONDS);
        
        // ====== CopyOnWriteArrayList ======
        System.out.println("\n=== CopyOnWriteArrayList ===");
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>();
        cowList.addAll(List.of("A", "B", "C"));
        
        // Safe to iterate while modifying
        for (String item : cowList) {
            System.out.print(item + " ");
            cowList.add("X");  // won't affect current iteration
        }
        System.out.println();
        System.out.println("Final: " + cowList.subList(0, 3));
    }
}
```

---

## 15.5 Virtual Threads (Java 21)

```java
import java.util.concurrent.*;
import java.util.*;

public class VirtualThreadsDemo {
    
    // Simulate I/O bound task
    static String fetchData(int id) {
        try {
            Thread.sleep(100);  // simulate network I/O
            return "Data from source " + id;
        } catch (InterruptedException e) {
            return "Interrupted";
        }
    }
    
    public static void main(String[] args) throws Exception {
        
        int taskCount = 100;
        
        // ====== Platform threads (traditional) ======
        long startPlatform = System.currentTimeMillis();
        ExecutorService platform = Executors.newFixedThreadPool(10);
        List<Future<String>> platformFutures = new ArrayList<>();
        
        for (int i = 0; i < taskCount; i++) {
            final int id = i;
            platformFutures.add(platform.submit(() -> fetchData(id)));
        }
        
        for (Future<String> f : platformFutures) f.get();
        platform.shutdown();
        long platformTime = System.currentTimeMillis() - startPlatform;
        
        // ====== Virtual threads (Java 21) ======
        long startVirtual = System.currentTimeMillis();
        ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
        List<Future<String>> virtualFutures = new ArrayList<>();
        
        for (int i = 0; i < taskCount; i++) {
            final int id = i;
            virtualFutures.add(virtual.submit(() -> fetchData(id)));
        }
        
        for (Future<String> f : virtualFutures) f.get();
        virtual.shutdown();
        long virtualTime = System.currentTimeMillis() - startVirtual;
        
        System.out.printf("Platform threads (pool=10): %dms%n", platformTime);
        System.out.printf("Virtual threads: %dms%n", virtualTime);
        System.out.printf("Speedup: %.1fx%n", (double) platformTime / virtualTime);
        
        // Virtual thread demo
        System.out.println("\n=== Virtual Thread Info ===");
        Thread vt = Thread.ofVirtual().name("my-virtual").start(() -> {
            System.out.println("Virtual: " + Thread.currentThread().isVirtual());
            System.out.println("Name: " + Thread.currentThread().getName());
        });
        vt.join();
        
        // Structured Concurrency (Java 21)
        System.out.println("\n=== Structured Tasks ===");
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            var priceTask = scope.fork(() -> {
                Thread.sleep(100);
                return 1500.0;
            });
            
            var discountTask = scope.fork(() -> {
                Thread.sleep(80);
                return 0.1;
            });
            
            scope.join();
            scope.throwIfFailed();
            
            double price = priceTask.get();
            double discount = discountTask.get();
            double final_price = price * (1 - discount);
            
            System.out.printf("Price: ฿%.2f, Discount: %.0f%%, Final: ฿%.2f%n",
                price, discount * 100, final_price);
        }
    }
}
```

---

## 15.6 Full Program: Concurrent Web Crawler (Simulation)

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.stream.*;

public class WebCrawlerSimulation {
    
    record PageResult(String url, int links, int wordCount, long durationMs) {}
    
    // Simulate fetching a web page
    static PageResult fetchPage(String url) {
        long start = System.currentTimeMillis();
        try {
            Thread.sleep(50 + (int)(Math.random() * 200));  // simulate network
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        int links = (int)(Math.random() * 20);
        int words = 100 + (int)(Math.random() * 1000);
        long duration = System.currentTimeMillis() - start;
        return new PageResult(url, links, words, duration);
    }
    
    static List<String> generateUrls(String base, int count) {
        List<String> urls = new ArrayList<>();
        for (int i = 1; i <= count; i++) {
            urls.add(base + "/page/" + i);
        }
        return urls;
    }
    
    public static void main(String[] args) throws Exception {
        
        List<String> urls = generateUrls("https://example.com", 20);
        
        System.out.println("Crawling " + urls.size() + " pages...\n");
        
        // Sequential
        long seqStart = System.currentTimeMillis();
        List<PageResult> seqResults = urls.stream()
            .map(WebCrawlerSimulation::fetchPage)
            .collect(Collectors.toList());
        long seqTime = System.currentTimeMillis() - seqStart;
        
        // Concurrent with CompletableFuture
        long concStart = System.currentTimeMillis();
        List<CompletableFuture<PageResult>> futures = urls.stream()
            .map(url -> CompletableFuture.supplyAsync(() -> fetchPage(url)))
            .collect(Collectors.toList());
        
        List<PageResult> concResults = CompletableFuture
            .allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList()))
            .get();
        long concTime = System.currentTimeMillis() - concStart;
        
        // Statistics
        LongSummaryStatistics durationStats = concResults.stream()
            .mapToLong(PageResult::durationMs).summaryStatistics();
        
        IntSummaryStatistics wordStats = concResults.stream()
            .mapToInt(PageResult::wordCount).summaryStatistics();
        
        int totalLinks = concResults.stream().mapToInt(PageResult::links).sum();
        
        System.out.println("=== Crawl Results ===");
        System.out.println("Pages crawled: " + concResults.size());
        System.out.println("Total links found: " + totalLinks);
        System.out.println("Total words: " + wordStats.getSum());
        System.out.printf("Avg words/page: %.0f%n", wordStats.getAverage());
        System.out.printf("Avg fetch time: %.0fms%n", durationStats.getAverage());
        
        System.out.println("\n=== Performance ===");
        System.out.printf("Sequential: %dms%n", seqTime);
        System.out.printf("Concurrent: %dms%n", concTime);
        System.out.printf("Speedup:    %.1fx%n", (double) seqTime / concTime);
        
        // Top 5 slowest pages
        System.out.println("\n=== Slowest Pages ===");
        concResults.stream()
            .sorted(Comparator.comparingLong(PageResult::durationMs).reversed())
            .limit(5)
            .forEach(r -> System.out.printf("  %-40s %4dms (%4d words)%n",
                r.url(), r.durationMs(), r.wordCount()));
    }
}
```

---

## สรุป Part 15 และ ภาพรวมหลักสูตร

```
╔══════════════════════════════════════════════════════════════════╗
║          Java Fundamentals (Part 01-15) - สรุป                 ║
╠══════════════════════════════════════════════════════════════════╣
║  01  Java Basics & JDK         09  Polymorphism               ║
║  02  Variables & Types         10  Interfaces                  ║
║  03  Control Flow              11  Exception Handling          ║
║  04  Loops & Recursion         12  Collections                 ║
║  05  Arrays                    13  Generics                    ║
║  06  Methods & Lambda          14  File I/O (NIO.2)           ║
║  07  OOP Classes               15  Multithreading             ║
║  08  Inheritance               ...                             ║
╚══════════════════════════════════════════════════════════════════╝

ต่อไป: Stream API, Lambda, Design Patterns, JDBC, Spring Boot,
       Kotlin, Android, APK Decompile & Smali
```

| Concurrency Feature | Use Case |
|--------------------|----------|
| Thread | low-level control |
| synchronized | mutual exclusion |
| AtomicInteger | counter without lock |
| ExecutorService | thread pool management |
| CompletableFuture | async chain / combine |
| BlockingQueue | producer-consumer |
| Semaphore | limit concurrency |
| Virtual Threads (Java 21) | high-throughput I/O |

➡️ [Part 16: Lambda & Stream API](./Part-16-Lambda-Stream-API.md)
