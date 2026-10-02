# Part 65: Multi-tenancy Patterns
## ขั้นตอนที่ 4451-4520: SaaS Architecture, Data Isolation, Tenant Routing

---

## 65.1 Multi-tenancy คืออะไร

```
Multi-tenancy = หนึ่ง application instance → รองรับหลาย customers (tenants)

SaaS Examples:
  Slack: แต่ละบริษัทคือ tenant (แยก workspace, data)
  GitHub: แต่ละ organization คือ tenant
  Shopify: แต่ละ store คือ tenant

3 Isolation Strategies:
  
  1. Separate Database (สูงสุด):
     Tenant A → DB_A
     Tenant B → DB_B
     + ปลอดภัยสูง, migrate/backup per tenant
     - ค่าใช้จ่ายสูง, จัดการยาก
  
  2. Shared Database, Separate Schema:
     Tenant A → schema_a.orders
     Tenant B → schema_b.orders
     + Balance ระหว่างต้นทุนและ isolation
     - ซับซ้อนกว่า single schema
  
  3. Shared Database, Shared Schema (ถูกที่สุด):
     orders table มี tenant_id column
     + ง่ายสุด, ประหยัดทรัพยากร
     - Bug ง่ายหากลืม filter tenant_id
```

---

## 65.2 Tenant Context & Identification

```java
import org.springframework.stereotype.*;
import org.springframework.web.filter.*;

// Tenant context holder (ThreadLocal)
public class TenantContext {
    
    private static final ThreadLocal<String> TENANT = new ThreadLocal<>();
    
    public static void setTenantId(String tenantId) {
        TENANT.set(tenantId);
    }
    
    public static String getTenantId() {
        var tenantId = TENANT.get();
        if (tenantId == null) throw new IllegalStateException("No tenant set in context");
        return tenantId;
    }
    
    public static void clear() {
        TENANT.remove();  // IMPORTANT: clear after request to prevent leak
    }
}

// Identify tenant from request (subdomain, header, JWT claim, or path)
@Component
@org.springframework.core.annotation.Order(1)
class TenantResolutionFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
            jakarta.servlet.http.HttpServletRequest request,
            jakarta.servlet.http.HttpServletResponse response,
            jakarta.servlet.FilterChain filterChain) throws java.io.IOException, jakarta.servlet.ServletException {
        
        try {
            String tenantId = resolveTenantId(request);
            TenantContext.setTenantId(tenantId);
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();  // always clean up
        }
    }
    
    private String resolveTenantId(jakarta.servlet.http.HttpServletRequest request) {
        // Strategy 1: From subdomain (tenant1.myapp.com)
        String host = request.getServerName();
        if (host.contains(".")) {
            String subdomain = host.split("\\.")[0];
            if (!subdomain.equals("www") && !subdomain.equals("api")) {
                return subdomain;
            }
        }
        
        // Strategy 2: From header
        String headerTenant = request.getHeader("X-Tenant-Id");
        if (headerTenant != null) return headerTenant;
        
        // Strategy 3: From JWT claim
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            // Parse JWT and extract tenant claim
            return extractTenantFromJwt(authHeader.substring(7));
        }
        
        // Strategy 4: From path prefix (/api/tenants/{tenantId}/...)
        String path = request.getRequestURI();
        if (path.startsWith("/api/tenants/")) {
            String[] parts = path.split("/");
            if (parts.length > 3) return parts[3];
        }
        
        throw new RuntimeException("Cannot resolve tenant from request");
    }
    
    private String extractTenantFromJwt(String token) {
        // Decode JWT and extract "tenant" claim
        return "tenant-id-from-jwt";
    }
}
```

---

## 65.3 Shared Schema with Row-Level Security

```java
// Entity with tenant isolation
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue
    private java.util.UUID id;
    
    @Column(name = "tenant_id", nullable = false)
    private String tenantId;
    
    private String userId;
    private double total;
    private String status;
    
    // getters/setters
}

// Repository with automatic tenant filtering
@Repository
public interface OrderRepository extends JpaRepository<Order, java.util.UUID> {
    
    // Always filter by tenant - explicit approach
    java.util.List<Order> findByTenantIdAndUserId(String tenantId, String userId);
    
    java.util.Optional<Order> findByIdAndTenantId(java.util.UUID id, String tenantId);
    
    long countByTenantId(String tenantId);
}

// Service using TenantContext
@Service
@org.springframework.transaction.annotation.Transactional
class OrderService {
    
    private final OrderRepository orderRepository;
    
    OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public Order getOrder(java.util.UUID orderId) {
        String tenantId = TenantContext.getTenantId();
        return orderRepository.findByIdAndTenantId(orderId, tenantId)
            .orElseThrow(() -> new RuntimeException("Order not found"));
    }
    
    public Order createOrder(CreateOrderRequest request) {
        var order = new Order();
        order.setTenantId(TenantContext.getTenantId());  // always set tenant
        order.setUserId(request.userId());
        order.setTotal(request.total());
        order.setStatus("PENDING");
        return orderRepository.save(order);
    }
}

// Hibernate Filter for automatic tenant filtering (safer approach)
@Entity
@Table(name = "orders")
@org.hibernate.annotations.Filter(
    name = "tenantFilter",
    condition = "tenant_id = :tenantId"
)
class OrderWithFilter {
    
    @Id @GeneratedValue
    private java.util.UUID id;
    
    @Column(nullable = false)
    private String tenantId;
    
    // other fields...
}

// Enable filter on every request
@Component
class TenantHibernateFilterInterceptor implements
        org.springframework.web.servlet.HandlerInterceptor {
    
    private final jakarta.persistence.EntityManager entityManager;
    
    TenantHibernateFilterInterceptor(jakarta.persistence.EntityManager entityManager) {
        this.entityManager = entityManager;
    }
    
    @Override
    public boolean preHandle(jakarta.servlet.http.HttpServletRequest request,
                              jakarta.servlet.http.HttpServletResponse response,
                              Object handler) {
        var session = entityManager.unwrap(org.hibernate.Session.class);
        session.enableFilter("tenantFilter")
            .setParameter("tenantId", TenantContext.getTenantId());
        return true;
    }
}
```

---

## 65.4 Separate Schema per Tenant (PostgreSQL)

```java
import org.springframework.jdbc.datasource.lookup.*;

// Dynamic datasource that routes to correct schema based on tenant
@Configuration
class MultiTenantDataSourceConfig {
    
    @Bean
    @Primary
    javax.sql.DataSource tenantRoutingDataSource(
            @Qualifier("tenantADataSource") javax.sql.DataSource tenantA,
            @Qualifier("tenantBDataSource") javax.sql.DataSource tenantB) {
        
        var routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TenantContext.getTenantId();
            }
        };
        
        routing.setTargetDataSources(java.util.Map.of(
            "tenant-a", tenantA,
            "tenant-b", tenantB
        ));
        routing.setDefaultTargetDataSource(tenantA);
        routing.afterPropertiesSet();
        
        return routing;
    }
    
    // Per-tenant datasources
    @Bean("tenantADataSource")
    javax.sql.DataSource tenantADataSource() {
        var config = new com.zaxxer.hikari.HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost/myapp");
        config.setSchema("tenant_a");
        config.setUsername("app_user");
        config.setPassword("password");
        return new com.zaxxer.hikari.HikariDataSource(config);
    }
    
    @Bean("tenantBDataSource")
    javax.sql.DataSource tenantBDataSource() {
        var config = new com.zaxxer.hikari.HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost/myapp");
        config.setSchema("tenant_b");
        config.setUsername("app_user");
        config.setPassword("password");
        return new com.zaxxer.hikari.HikariDataSource(config);
    }
}

// Flyway migration per tenant schema
@Component
class TenantSchemaMigrator implements
        org.springframework.boot.context.event.ApplicationStartedApplicationEvent {
    
    private final javax.sql.DataSource dataSource;
    
    TenantSchemaMigrator(javax.sql.DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public void migrateSchema(String tenantId) {
        var flyway = org.flywaydb.core.Flyway.configure()
            .dataSource(dataSource)
            .schemas(tenantId)
            .locations("classpath:db/migration/tenants")
            .load();
        flyway.migrate();
    }
}
```

---

## 65.5 Tenant Management API

```java
@RestController
@RequestMapping("/api/admin/tenants")
@org.springframework.security.access.prepost.PreAuthorize("hasRole('SUPER_ADMIN')")
class TenantManagementController {
    
    private final TenantService tenantService;
    
    TenantManagementController(TenantService tenantService) {
        this.tenantService = tenantService;
    }
    
    @PostMapping
    @org.springframework.http.ResponseStatus(org.springframework.http.HttpStatus.CREATED)
    TenantDTO createTenant(@RequestBody CreateTenantRequest request) {
        return tenantService.createTenant(request);
    }
    
    @GetMapping("/{tenantId}")
    TenantDTO getTenant(@PathVariable String tenantId) {
        return tenantService.getTenant(tenantId);
    }
    
    @PutMapping("/{tenantId}/suspend")
    void suspendTenant(@PathVariable String tenantId) {
        tenantService.suspendTenant(tenantId);
    }
    
    @DeleteMapping("/{tenantId}")
    void deleteTenant(@PathVariable String tenantId) {
        tenantService.deleteTenant(tenantId);
    }
    
    record CreateTenantRequest(
        String name,
        String subdomain,
        String plan,           // "free", "pro", "enterprise"
        String adminEmail,
        String adminPassword
    ) {}
    
    record TenantDTO(
        String id,
        String name,
        String subdomain,
        String plan,
        String status,
        java.time.Instant createdAt
    ) {}
}

@Service
class TenantService {
    
    private final TenantRepository tenantRepository;
    private final TenantSchemaMigrator migrator;
    
    TenantService(TenantRepository tenantRepository, TenantSchemaMigrator migrator) {
        this.tenantRepository = tenantRepository;
        this.migrator = migrator;
    }
    
    @org.springframework.transaction.annotation.Transactional
    public TenantManagementController.TenantDTO createTenant(
            TenantManagementController.CreateTenantRequest request) {
        
        // Validate subdomain availability
        if (tenantRepository.existsBySubdomain(request.subdomain())) {
            throw new RuntimeException("Subdomain already taken: " + request.subdomain());
        }
        
        var tenant = new Tenant(
            request.subdomain(),  // use subdomain as ID
            request.name(),
            request.subdomain(),
            request.plan(),
            "ACTIVE"
        );
        
        tenantRepository.save(tenant);
        
        // Create schema and run migrations
        migrator.migrateSchema(tenant.getId());
        
        // Create admin user for this tenant
        // ... 
        
        return new TenantManagementController.TenantDTO(
            tenant.getId(), tenant.getName(), tenant.getSubdomain(),
            tenant.getPlan(), tenant.getStatus(), java.time.Instant.now()
        );
    }
}
```

---

## สรุป Part 65

```
Multi-tenancy Strategies:

1. Shared Schema (tenant_id column):
   + ง่ายสุด, ประหยัดทรัพยากร
   - ต้องระวัง tenant leakage (ลืม filter)
   Use: Hibernate @Filter หรือ explicit findByTenantId

2. Separate Schema (per-tenant PostgreSQL schema):
   + Better isolation, per-tenant migration
   - Connection pool ต้องใหญ่ขึ้น
   Use: AbstractRoutingDataSource + Flyway per schema

3. Separate Database:
   + Highest isolation (GDPR-friendly)
   - Expensive, complex management
   Use: Enterprise/regulated industries

Tenant Resolution:
  Subdomain: tenant1.myapp.com
  Header: X-Tenant-Id: tenant1
  JWT claim: {"tenant": "tenant1"}
  Path prefix: /api/tenants/tenant1/...

Best Practices:
  ✓ TenantContext (ThreadLocal) + Filter ดัก resolve
  ✓ Always clear ThreadLocal ใน finally block
  ✓ Row-Level Security ใน PostgreSQL (RLS) เพื่อ DB-level enforcement
  ✓ Test: verify tenant A cannot access tenant B data
```

➡️ [Part 66: Feature Flags & A/B Testing](./Part-66-FeatureFlags.md)
