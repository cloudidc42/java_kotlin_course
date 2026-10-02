# Part 61: Java Virtual Threads (Project Loom)
## ขั้นตอนที่ 4171-4240: Lightweight Concurrency, Structured Concurrency

---

## 61.1 Virtual Threads คืออะไร

```
Platform Thread (เดิม):
  - Thread ใน Java = OS thread = heavyweight
  - 1 thread ≈ 1MB stack memory
  - Context switch: OS-level overhead
  - Max ~thousands threads per JVM

Virtual Thread (Java 21+):
  - Lightweight thread managed by JVM
  - Runs on a small pool of OS threads (carrier threads)
  - 1 virtual thread ≈ few KB stack
  - Can create MILLIONS of virtual threads
  - Blocking I/O: unmounts from carrier, lets others run

เปรียบเทียบ:
  Platform Thread  : 1 thread = 1MB, max ~10,000
  Virtual Thread   : 1 thread = ~1KB, max ~1,000,000+

Analogy:
  Platform Thread = แท็กซี่ (รอลูกค้าคนนึงจนกว่าจะถึงที่หมาย)
  Virtual Thread  = BTS (รับผู้โดยสารหลายคน, พอคนลงก็รับคนอื่นทันที)
```

---

## 61.2 Basic Usage

```java
import java.util.concurrent.*;

public class VirtualThreadDemo {
    
    public static void main(String[] args) throws Exception {
        
        // Create one virtual thread
        Thread vt = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> {
                System.out.println("Running in: " + Thread.currentThread());
                System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
            });
        vt.join();
        
        // Create millions of virtual threads
        long start = System.currentTimeMillis();
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = new java.util.ArrayList<Future<String>>();
            
            for (int i = 0; i < 100_000; i++) {
                int taskId = i;
                futures.add(executor.submit(() -> {
                    Thread.sleep(100);  // simulate I/O wait
                    return "Task " + taskId + " done";
                }));
            }
            
            futures.forEach(f -> {
                try { f.get(); }
                catch (Exception e) { e.printStackTrace(); }
            });
        }
        
        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("100,000 tasks with 100ms I/O each: %dms total%n", elapsed);
        // ~100ms total (parallel) instead of 10,000,000ms (sequential)
    }
}
```

---

## 61.3 Spring Boot + Virtual Threads

```java
import org.springframework.context.annotation.*;
import org.springframework.boot.autoconfigure.*;

// application.yaml: ใช้ virtual threads กับ Tomcat
// spring.threads.virtual.enabled: true   (Spring Boot 3.2+)

// หรือ config manually:
@Configuration
class VirtualThreadConfig {
    
    // Tomcat: ใช้ virtual thread แทน platform thread สำหรับ HTTP requests
    @Bean
    org.apache.coyote.ProtocolHandler protocolHandler() {
        var handler = new org.apache.coyote.http11.Http11NioProtocol();
        handler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        return handler;
    }
    
    // Spring MVC async: virtual thread executor
    @Bean
    @Primary
    java.util.concurrent.Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
    
    // @Async จะใช้ virtual threads
    @Bean
    org.springframework.scheduling.annotation.AsyncConfigurer asyncConfigurer() {
        return new org.springframework.scheduling.annotation.AsyncConfigurer() {
            @Override
            public java.util.concurrent.Executor getAsyncExecutor() {
                return Executors.newVirtualThreadPerTaskExecutor();
            }
        };
    }
}

// Service ใช้ virtual threads โดยอัตโนมัติ (ถ้า Tomcat ใช้ virtual thread executor)
@org.springframework.stereotype.Service
class OrderService {
    
    // การเรียกนี้จะ block virtual thread แทน platform thread
    // JVM จะ unmount virtual thread ระหว่าง I/O wait → ไม่เปลือง OS thread
    public Order createOrder(CreateOrderRequest request) {
        var user = userRepository.findById(request.getUserId());    // I/O: DB
        var product = productService.getProduct(request.getProductId()); // I/O: HTTP call
        var inventory = inventoryService.check(request.getProductId()); // I/O: DB
        
        // เหล่านี้ block แต่ไม่เปลือง OS thread
        return orderRepository.save(new Order(user, product, inventory));
    }
}
```

---

## 61.4 Structured Concurrency (Java 21+)

```java
import java.util.concurrent.*;

// Structured Concurrency: child tasks tied to parent scope
// ถ้า parent ปิด → ทุก child ถูก cancel อัตโนมัติ

public class StructuredConcurrencyDemo {
    
    record UserDashboard(User user, List<Order> orders, List<Notification> notifications) {}
    
    // แบบเก่า (flat/unstructured):
    UserDashboard getDashboard_OLD(String userId) throws Exception {
        var userFuture = CompletableFuture.supplyAsync(() -> fetchUser(userId));
        var ordersFuture = CompletableFuture.supplyAsync(() -> fetchOrders(userId));
        var notifFuture = CompletableFuture.supplyAsync(() -> fetchNotifications(userId));
        
        // ถ้า fetchUser throw → fetchOrders ยังทำงานอยู่ (resource leak)
        return new UserDashboard(
            userFuture.get(),
            ordersFuture.get(),
            notifFuture.get()
        );
    }
    
    // แบบใหม่ (structured concurrency):
    UserDashboard getDashboard(String userId) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            
            // fork: เริ่ม subtask แบบ parallel
            var userTask = scope.fork(() -> fetchUser(userId));
            var ordersTask = scope.fork(() -> fetchOrders(userId));
            var notifTask = scope.fork(() -> fetchNotifications(userId));
            
            // join: รอทุก subtask จบ
            scope.join()
                .throwIfFailed();  // propagate exception ถ้ามี subtask fail
            
            // ถ้า subtask ใด fail → scope.join() throw → scope.close() cancel subtasks ที่เหลือ
            return new UserDashboard(
                userTask.get(),
                ordersTask.get(),
                notifTask.get()
            );
        }
    }
    
    // ShutdownOnSuccess: ใช้สำหรับ race (เอาค่าแรกที่สำเร็จ)
    String fetchFromFastest(String key) throws Exception {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
            
            scope.fork(() -> fetchFromPrimary(key));    // ลองทั้งคู่พร้อมกัน
            scope.fork(() -> fetchFromSecondary(key));
            
            scope.join();
            return scope.result();  // return ค่าแรกที่สำเร็จ
        }
    }
    
    // Helpers
    User fetchUser(String id) throws InterruptedException {
        Thread.sleep(50); return new User(id, "Alice");
    }
    List<Order> fetchOrders(String id) throws InterruptedException {
        Thread.sleep(80); return List.of();
    }
    List<Notification> fetchNotifications(String id) throws InterruptedException {
        Thread.sleep(30); return List.of();
    }
    String fetchFromPrimary(String key) throws InterruptedException {
        Thread.sleep(100); return "primary:" + key;
    }
    String fetchFromSecondary(String key) throws InterruptedException {
        Thread.sleep(50); return "secondary:" + key;
    }
    
    record User(String id, String name) {}
}
```

---

## 61.5 Virtual Threads กับ JDBC

```java
// JDBC pool + virtual threads: ต้องระวัง

// ปัญหา: virtual thread มีล้านตัว แต่ connection pool มีแค่ 20
// → virtual thread block รอ connection จาก pool
// นี่คือการ block เพราะรอ pool ไม่ใช่ I/O จริง

// การแก้ไข: ขยาย pool size เมื่อใช้ virtual threads
// แต่ DB มี connection limit จำกัด → ต้องหาสมดุล

// HikariCP config สำหรับ virtual threads
spring:
  datasource:
    hikari:
      # virtual threads ทำให้ request per second สูงขึ้นมาก
      # pool ต้องใหญ่พอ แต่ไม่เกิน DB limit
      maximum-pool-size: 50   # ปรับจาก 10 เป็น 50
      minimum-idle: 10

// Alternative: ใช้ R2DBC (reactive) แทน JDBC เมื่อใช้ virtual threads
// R2DBC ไม่ block เลย → เหมาะกับ virtual threads มากกว่า

// ตัวอย่างวัดประสิทธิภาพ
public class VirtualThreadBenchmark {
    
    static void runWithPlatformThreads(int concurrency) throws Exception {
        var exec = Executors.newFixedThreadPool(concurrency);
        var start = System.nanoTime();
        
        var futures = new java.util.ArrayList<Future<?>>();
        for (int i = 0; i < 10000; i++) {
            futures.add(exec.submit(() -> {
                Thread.sleep(10);  // simulate DB call
                return null;
            }));
        }
        futures.forEach(f -> { try { f.get(); } catch (Exception e) {} });
        
        System.out.printf("Platform (%d threads): %dms%n",
            concurrency, (System.nanoTime() - start) / 1_000_000);
        exec.shutdown();
    }
    
    static void runWithVirtualThreads() throws Exception {
        var exec = Executors.newVirtualThreadPerTaskExecutor();
        var start = System.nanoTime();
        
        var futures = new java.util.ArrayList<Future<?>>();
        for (int i = 0; i < 10000; i++) {
            futures.add(exec.submit(() -> {
                Thread.sleep(10);
                return null;
            }));
        }
        futures.forEach(f -> { try { f.get(); } catch (Exception e) {} });
        
        System.out.printf("Virtual threads:        %dms%n",
            (System.nanoTime() - start) / 1_000_000);
        exec.shutdown();
    }
    
    public static void main(String[] args) throws Exception {
        runWithPlatformThreads(10);    // Platform (10 threads): ~10000ms
        runWithPlatformThreads(200);   // Platform (200 threads): ~500ms
        runWithVirtualThreads();        // Virtual threads:        ~10ms
    }
}
```

---

## สรุป Part 61

```
Virtual Threads (Java 21+):

ข้อดี:
  ✓ สร้างได้นับล้าน (RAM ถูกกว่า platform threads 100-1000x)
  ✓ Block I/O โดยไม่เปลือง OS thread
  ✓ Code เหมือนเดิม (ไม่ต้อง async/reactive)
  ✓ ใช้กับ Spring Boot: spring.threads.virtual.enabled=true

ข้อควรระวัง:
  ✗ synchronized block บน virtual thread = pin carrier thread (ไม่ดี)
  ✗ ThreadLocal ยังทำงาน แต่ memory อาจรั่วถ้ามีล้าน virtual threads
  ✗ Connection pool ต้องปรับ size ให้รับ throughput ที่สูงขึ้น

Structured Concurrency:
  StructuredTaskScope.ShutdownOnFailure = รอทุก task, fail ถ้ามีอะไร fail
  StructuredTaskScope.ShutdownOnSuccess = เอาค่าแรกที่สำเร็จ (race)
  
  Parent ปิด → child tasks ถูก cancel อัตโนมัติ (ไม่มี resource leak)

เมื่อไหร่ควรใช้:
  ✓ I/O-heavy: HTTP calls, DB queries, file I/O
  ✗ CPU-heavy: algorithm, encryption, image processing (ใช้ platform thread ปกติ)
```

➡️ [Part 62: Spring AI & LLM Integration](./Part-62-SpringAI.md)
