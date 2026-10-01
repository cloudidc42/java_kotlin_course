# Part 20: Spring Boot - Web Applications
## ขั้นตอนที่ 1281-1360: สร้าง REST API ระดับ Production

---

## 20.1 Spring Boot Project Structure

```
my-spring-app/
├── src/
│   ├── main/
│   │   ├── java/com/example/app/
│   │   │   ├── Application.java          (Main class)
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   └── SwaggerConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── ProductController.java
│   │   │   │   └── UserController.java
│   │   │   ├── service/
│   │   │   │   ├── ProductService.java
│   │   │   │   └── UserService.java
│   │   │   ├── repository/
│   │   │   │   ├── ProductRepository.java
│   │   │   │   └── UserRepository.java
│   │   │   ├── entity/
│   │   │   │   ├── Product.java
│   │   │   │   └── User.java
│   │   │   ├── dto/
│   │   │   │   ├── ProductDTO.java
│   │   │   │   └── ApiResponse.java
│   │   │   └── exception/
│   │   │       └── GlobalExceptionHandler.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── application-prod.yml
│   └── test/
│       └── java/com/example/app/
│           └── controller/
│               └── ProductControllerTest.java
└── pom.xml
```

---

## 20.2 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>my-spring-app</artifactId>
    <version>1.0.0</version>
    <name>My Spring App</name>
    
    <properties>
        <java.version>21</java.version>
    </properties>
    
    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- JPA / Database -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        
        <!-- Lombok (optional) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <!-- H2 for tests -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 20.3 Application Entry Point & Config

```java
// Application.java
package com.example.app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication  // = @Configuration + @ComponentScan + @EnableAutoConfiguration
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  application:
    name: my-spring-app
  
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true

logging:
  level:
    com.example: DEBUG
    org.springframework.security: INFO
```

---

## 20.4 Entity, Repository, Service

```java
// Entity
package com.example.app.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "products")
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank(message = "Name is required")
    @Size(max = 100)
    @Column(nullable = false)
    private String name;
    
    @NotBlank
    @Column(nullable = false)
    private String category;
    
    @Positive(message = "Price must be positive")
    @Column(nullable = false)
    private Double price;
    
    @Min(0)
    private Integer stock = 0;
    
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
    
    // Constructors, getters, setters
    public Product() {}
    
    public Product(String name, String category, double price, int stock) {
        this.name = name; this.category = category;
        this.price = price; this.stock = stock;
    }
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getCategory() { return category; }
    public Double getPrice() { return price; }
    public Integer getStock() { return stock; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    
    // Setters
    public void setName(String name) { this.name = name; }
    public void setCategory(String category) { this.category = category; }
    public void setPrice(Double price) { this.price = price; }
    public void setStock(Integer stock) { this.stock = stock; }
}

// Repository
package com.example.app.repository;

import com.example.app.entity.Product;
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.Param;
import java.util.*;

public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // Method name convention
    List<Product> findByCategory(String category);
    List<Product> findByPriceLessThanEqual(Double price);
    List<Product> findByCategoryAndPriceLessThanEqual(String category, Double price);
    
    // JPQL
    @Query("SELECT p FROM Product p WHERE p.stock > 0 ORDER BY p.price")
    List<Product> findInStock();
    
    @Query("SELECT p FROM Product p WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    List<Product> searchByName(@Param("keyword") String keyword);
    
    // Native SQL
    @Query(value = "SELECT category, COUNT(*) FROM products GROUP BY category", nativeQuery = true)
    List<Object[]> countByCategory();
    
    // Aggregates
    @Query("SELECT AVG(p.price) FROM Product p WHERE p.category = :category")
    Optional<Double> averagePriceByCategory(@Param("category") String category);
    
    // Update
    @Modifying
    @Transactional
    @Query("UPDATE Product p SET p.price = p.price * :factor WHERE p.category = :category")
    int updatePriceByCategory(@Param("category") String category, @Param("factor") double factor);
}

// Service
package com.example.app.service;

import com.example.app.entity.Product;
import com.example.app.exception.*;
import com.example.app.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepo;
    
    public List<Product> getAllProducts() {
        return productRepo.findAll();
    }
    
    public Product getById(Long id) {
        return productRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }
    
    public List<Product> getByCategory(String category) {
        return productRepo.findByCategory(category);
    }
    
    public List<Product> searchProducts(String keyword) {
        return productRepo.searchByName(keyword);
    }
    
    @Transactional
    public Product createProduct(Product product) {
        if (product.getId() != null) {
            throw new IllegalArgumentException("New product cannot have ID");
        }
        return productRepo.save(product);
    }
    
    @Transactional
    public Product updateProduct(Long id, Product updated) {
        Product existing = getById(id);
        existing.setName(updated.getName());
        existing.setCategory(updated.getCategory());
        existing.setPrice(updated.getPrice());
        existing.setStock(updated.getStock());
        return productRepo.save(existing);
    }
    
    @Transactional
    public void deleteProduct(Long id) {
        if (!productRepo.existsById(id)) {
            throw new ResourceNotFoundException("Product", id);
        }
        productRepo.deleteById(id);
    }
    
    @Transactional
    public boolean purchaseProduct(Long id, int quantity) {
        Product product = getById(id);
        if (product.getStock() < quantity) {
            throw new InsufficientStockException(id, product.getStock(), quantity);
        }
        product.setStock(product.getStock() - quantity);
        productRepo.save(product);
        return true;
    }
}
```

---

## 20.5 REST Controller

```java
// ProductController.java
package com.example.app.controller;

import com.example.app.dto.*;
import com.example.app.entity.Product;
import com.example.app.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/v1/products")
@CrossOrigin(origins = "*")
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @GetMapping
    public ResponseEntity<ApiResponse<List<Product>>> getAllProducts(
            @RequestParam(required = false) String category,
            @RequestParam(required = false) String search) {
        
        List<Product> products;
        if (search != null) {
            products = productService.searchProducts(search);
        } else if (category != null) {
            products = productService.getByCategory(category);
        } else {
            products = productService.getAllProducts();
        }
        
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<Product>> getById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(productService.getById(id)));
    }
    
    @PostMapping
    public ResponseEntity<ApiResponse<Product>> createProduct(
            @Valid @RequestBody Product product) {
        Product created = productService.createProduct(product);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.success(created, "Product created successfully"));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<Product>> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody Product product) {
        Product updated = productService.updateProduct(id, product);
        return ResponseEntity.ok(ApiResponse.success(updated, "Product updated"));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.ok(ApiResponse.success(null, "Product deleted"));
    }
    
    @PostMapping("/{id}/purchase")
    public ResponseEntity<ApiResponse<String>> purchase(
            @PathVariable Long id,
            @RequestParam int quantity) {
        productService.purchaseProduct(id, quantity);
        return ResponseEntity.ok(ApiResponse.success("Purchase successful"));
    }
}

// DTO: ApiResponse
package com.example.app.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.LocalDateTime;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp = LocalDateTime.now();
    
    public ApiResponse(boolean success, T data, String message) {
        this.success = success; this.data = data; this.message = message;
    }
    
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, data, null);
    }
    
    public static <T> ApiResponse<T> success(T data, String message) {
        return new ApiResponse<>(true, data, message);
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message);
    }
    
    // Getters
    public boolean isSuccess() { return success; }
    public String getMessage() { return message; }
    public T getData() { return data; }
    public LocalDateTime getTimestamp() { return timestamp; }
}
```

---

## 20.6 Exception Handling

```java
// ResourceNotFoundException.java
package com.example.app.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " not found with id: " + id);
    }
}

// GlobalExceptionHandler.java
package com.example.app.exception;

import com.example.app.dto.ApiResponse;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.*;
import org.springframework.web.bind.annotation.*;
import java.util.*;

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException e) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ApiResponse.error(e.getMessage()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Map<String, String>>> handleValidation(
            MethodArgumentNotValidException e) {
        Map<String, String> errors = new LinkedHashMap<>();
        e.getBindingResult().getAllErrors().forEach(err -> {
            String field = ((FieldError) err).getField();
            errors.put(field, err.getDefaultMessage());
        });
        
        ApiResponse<Map<String, String>> response = new ApiResponse<>(false, errors, "Validation failed");
        return ResponseEntity.badRequest().body(response);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleGeneral(Exception e) {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiResponse.error("Internal server error: " + e.getMessage()));
    }
}
```

---

## 20.7 Testing

```java
// ProductControllerTest.java
package com.example.app.controller;

import com.example.app.entity.Product;
import com.example.app.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import java.util.*;

@WebMvcTest(ProductController.class)
class ProductControllerTest {
    
    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;
    @MockBean ProductService productService;
    
    @Test
    void getAllProducts_shouldReturn200() throws Exception {
        Product p = new Product("MacBook", "Laptop", 75000, 10);
        when(productService.getAllProducts()).thenReturn(List.of(p));
        
        mockMvc.perform(get("/v1/products"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.success").value(true))
               .andExpect(jsonPath("$.data[0].name").value("MacBook"));
    }
    
    @Test
    void createProduct_withValidData_shouldReturn201() throws Exception {
        Product input = new Product("iPhone", "Phone", 35000, 20);
        Product saved = new Product("iPhone", "Phone", 35000, 20);
        
        when(productService.createProduct(any())).thenReturn(saved);
        
        mockMvc.perform(post("/v1/products")
                   .contentType(MediaType.APPLICATION_JSON)
                   .content(objectMapper.writeValueAsString(input)))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.data.name").value("iPhone"));
    }
    
    @Test
    void createProduct_withMissingName_shouldReturn400() throws Exception {
        Product invalid = new Product("", "Phone", 35000, 20);
        
        mockMvc.perform(post("/v1/products")
                   .contentType(MediaType.APPLICATION_JSON)
                   .content(objectMapper.writeValueAsString(invalid)))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.success").value(false));
    }
}
```

---

## 20.8 REST API Endpoints Summary

```
BASE URL: http://localhost:8080/api/v1

Products:
  GET    /products              - List all (filter by ?category=&search=)
  GET    /products/{id}         - Get by ID
  POST   /products              - Create new product
  PUT    /products/{id}         - Update product
  DELETE /products/{id}         - Delete product
  POST   /products/{id}/purchase?quantity=N  - Purchase

Response format:
  {
    "success": true/false,
    "message": "...",
    "data": {...} | [...] | null,
    "timestamp": "2024-01-15T10:30:00"
  }

HTTP Status codes:
  200 OK         - Success
  201 Created    - Resource created
  400 Bad Request - Validation error
  401 Unauthorized - Not authenticated
  403 Forbidden  - Not authorized
  404 Not Found  - Resource not found
  500 Internal Server Error - Unexpected error
```

---

## สรุป Part 20

| Component | Annotation | หน้าที่ |
|-----------|-----------|---------|
| Controller | `@RestController` | รับ HTTP request, ส่ง response |
| Service | `@Service` | Business logic |
| Repository | `@Repository` | Data access layer |
| Entity | `@Entity` | JPA entity (database table) |
| Config | `@Configuration` | App configuration |
| Exception Handler | `@RestControllerAdvice` | Global error handling |

➡️ [Part 21: Spring Security & JWT](./Part-21-Spring-Security.md)
