# Part 52: Database Performance & Query Optimization
## ขั้นตอนที่ 3541-3610: Advanced SQL, Indexing, Query Plans

---

## 52.1 Understanding Query Plans

```sql
-- EXPLAIN ANALYZE: understand what the database actually does
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    o.id,
    o.created_at,
    u.name AS customer_name,
    SUM(oi.price * oi.quantity) AS total
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'PENDING'
  AND o.created_at > NOW() - INTERVAL '7 days'
GROUP BY o.id, o.created_at, u.name
ORDER BY o.created_at DESC
LIMIT 20;

-- Reading the output:
-- Seq Scan = full table scan (bad for large tables)
-- Index Scan = uses index (good)
-- Index Only Scan = reads only from index (best)
-- Hash Join = good for large sets
-- Nested Loop = good for small sets with index
-- Merge Join = good for pre-sorted data

-- Key metrics to watch:
-- actual rows vs estimated rows (large gap = stale stats)
-- actual time = execution time
-- Buffers: shared hit = from cache, shared read = from disk
```

---

## 52.2 Index Design

```sql
-- =========================================
-- 1. Composite index: order matters
-- =========================================
-- Query: WHERE status = 'PENDING' AND created_at > '2024-01-01'
-- Good: (status, created_at) → filter status first (low cardinality), then range
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Bad: (created_at, status) → range scan on created_at, then filter status
-- Correct rule: equality columns first, range columns last

-- =========================================
-- 2. Covering index: avoid heap fetch
-- =========================================
-- Query: SELECT id, total FROM orders WHERE user_id = ? AND status = 'CONFIRMED'
CREATE INDEX idx_orders_user_status_covering
ON orders(user_id, status)
INCLUDE (id, total, created_at);  -- INCLUDE = store in index leaf, not key
-- → Index Only Scan (no heap access)

-- =========================================
-- 3. Partial index: for filtered queries
-- =========================================
-- Only index rows we actually query
CREATE INDEX idx_orders_pending
ON orders(created_at, user_id)
WHERE status = 'PENDING';  -- 99% of queries filter on PENDING

-- Much smaller index, faster writes for non-PENDING orders

-- =========================================
-- 4. Expression index
-- =========================================
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Supports: WHERE LOWER(email) = LOWER(?)

-- Function-based
CREATE INDEX idx_orders_year_month ON orders(
    DATE_TRUNC('month', created_at)
);

-- =========================================
-- 5. GIN index for full-text search
-- =========================================
ALTER TABLE products ADD COLUMN search_vector tsvector;
CREATE INDEX idx_products_fts ON products USING GIN(search_vector);

-- Full-text search query
SELECT id, name, ts_rank(search_vector, query) AS rank
FROM products,
     plainto_tsquery('english', 'laptop gaming') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- =========================================
-- 6. BRIN index for time-series
-- =========================================
-- BRIN = Block Range INdex: tiny, great for time-sorted data
CREATE INDEX idx_events_time_brin
ON audit_events USING BRIN(occurred_at)
WITH (pages_per_range = 128);
-- Useful when rows are physically ordered by time (append-only tables)
```

---

## 52.3 N+1 Query Problem

```java
import org.springframework.data.jpa.repository.*;
import org.springframework.data.domain.*;

// Problem: N+1 queries
// 1 query to load orders, N queries to load items for each
@GetMapping("/orders")
List<OrderDTO> getOrders() {
    var orders = orderRepo.findAll();           // 1 query
    return orders.stream()
        .map(o -> new OrderDTO(
            o.getId(),
            o.getItems().stream()...            // N queries (lazy load)
        ))
        .toList();
}

// Solution 1: @EntityGraph
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @EntityGraph(attributePaths = {"items", "items.product", "user"})
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    
    // JOIN FETCH in JPQL
    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.items i
        LEFT JOIN FETCH i.product
        WHERE o.userId = :userId
        ORDER BY o.createdAt DESC
    """)
    List<Order> findByUserIdWithItems(@Param("userId") Long userId);
}

// Solution 2: Projection with custom DTO (avoids loading full entity)
public interface OrderSummaryProjection {
    String getId();
    String getStatus();
    @Value("#{target.items.size()}")
    int getItemCount();
}

// Solution 3: Batch fetching (property in persistence.xml or annotation)
@Entity
@org.hibernate.annotations.BatchSize(size = 100)
class OrderItem { ... }

// Solution 4: Native query with JOIN
@Query(value = """
    SELECT 
        o.id, o.status, o.created_at,
        json_agg(json_build_object(
            'productId', oi.product_id,
            'quantity', oi.quantity,
            'price', oi.price
        )) AS items
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.user_id = :userId
    GROUP BY o.id
    ORDER BY o.created_at DESC
    """, nativeQuery = true)
List<Object[]> findOrdersWithItemsNative(@Param("userId") Long userId);
```

---

## 52.4 Bulk Operations

```java
import org.springframework.jdbc.core.*;
import jakarta.persistence.*;

// Problem: inserting 10,000 records one-by-one = 10,000 round trips
// Solution 1: Spring JDBC batch update
@Service
class BulkInsertService {
    
    private final JdbcTemplate jdbc;
    
    BulkInsertService(JdbcTemplate jdbc) { this.jdbc = jdbc; }
    
    public void bulkInsertProducts(List<Product> products) {
        jdbc.batchUpdate(
            "INSERT INTO products(id, name, price, category_id) VALUES (?, ?, ?, ?)",
            products,
            500,  // batch size
            (ps, product) -> {
                ps.setObject(1, product.getId());
                ps.setString(2, product.getName());
                ps.setBigDecimal(3, product.getPrice());
                ps.setLong(4, product.getCategoryId());
            }
        );
    }
    
    // COPY command: even faster (PostgreSQL specific)
    public void copyInsertProducts(List<Product> products) throws Exception {
        var conn = jdbc.getDataSource().getConnection();
        var copyManager = ((org.postgresql.core.BaseConnection) conn)
            .createCopyManager();
        
        var data = new StringBuilder("id,name,price\n");
        products.forEach(p -> 
            data.append(p.getId()).append(',')
                .append(p.getName()).append(',')
                .append(p.getPrice()).append('\n')
        );
        
        copyManager.copyIn(
            "COPY products(id, name, price) FROM STDIN CSV HEADER",
            new java.io.StringReader(data.toString())
        );
    }
}

// Solution 2: JPA with flush/clear
@Service
class JpaBulkService {
    
    @PersistenceContext EntityManager em;
    
    @Transactional
    public void saveInBatches(List<Order> orders, int batchSize) {
        for (int i = 0; i < orders.size(); i++) {
            em.persist(orders.get(i));
            
            if ((i + 1) % batchSize == 0) {
                em.flush();   // write SQL to DB
                em.clear();   // detach entities from context (prevent memory bloat)
            }
        }
        em.flush();  // flush remaining
    }
}
```

---

## 52.5 Pagination Best Practices

```sql
-- Problem: OFFSET pagination is slow on large tables
-- Scanning + skipping 1,000,000 rows to get page 100

-- Bad: OFFSET-based
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 20000;
-- Must scan 20,020 rows, discard first 20,000

-- Good: Keyset (cursor) pagination
-- First page
SELECT id, created_at, status FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page: use last row's (created_at, id) as cursor
SELECT id, created_at, status FROM orders
WHERE (created_at, id) < ('2024-01-15 10:30:00', 'abc123')
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Jumps directly to the cursor position using index
```

```java
// Spring Data keyset pagination
public interface OrderRepository extends JpaRepository<Order, String> {
    
    @Query("""
        SELECT o FROM Order o
        WHERE o.userId = :userId
          AND (o.createdAt < :lastCreatedAt
               OR (o.createdAt = :lastCreatedAt AND o.id < :lastId))
        ORDER BY o.createdAt DESC, o.id DESC
    """)
    List<Order> findNextPage(
        @Param("userId") String userId,
        @Param("lastCreatedAt") java.time.Instant lastCreatedAt,
        @Param("lastId") String lastId,
        Pageable pageable
    );
}

// Response with cursor
record PageResponse<T>(
    List<T> items,
    String nextCursor,  // base64-encoded (createdAt, id)
    boolean hasMore
) {
    static <T extends HasCursor> PageResponse<T> of(List<T> items, int requestedSize) {
        boolean hasMore = items.size() > requestedSize;
        var page = hasMore ? items.subList(0, requestedSize) : items;
        String cursor = hasMore ? encodeCursor(page.get(page.size() - 1)) : null;
        return new PageResponse<>(page, cursor, hasMore);
    }
    
    private static String encodeCursor(HasCursor item) {
        return java.util.Base64.getEncoder().encodeToString(
            (item.getCreatedAt() + "," + item.getId()).getBytes()
        );
    }
}

interface HasCursor {
    java.time.Instant getCreatedAt();
    String getId();
}
```

---

## 52.6 Database Partitioning

```sql
-- Range partitioning for time-series data
CREATE TABLE orders (
    id          VARCHAR PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    total       NUMERIC(15,2),
    created_at  TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Auto-create partitions with pg_partman extension
-- Or create future partitions in advance

-- Indexes on parent table apply to all partitions
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- Partition pruning: WHERE created_at > '2024-01-01' only scans relevant partitions
EXPLAIN SELECT * FROM orders WHERE created_at > '2024-01-01' AND status = 'PENDING';
-- → Scans only 2024-01, 2024-02, ... partitions (pruned older ones)

-- Drop old partition (instant, no DELETE)
DROP TABLE orders_2022_01;
```

---

## สรุป Part 52

```
Database Performance Rules:

Indexes:
  1. Equality columns before range columns in composite index
  2. INCLUDE columns to avoid heap fetch (covering index)
  3. Partial index: WHERE clause limits index size
  4. GIN for full-text search, BRIN for time-series

N+1 Prevention:
  @EntityGraph = JOIN FETCH in single query
  BatchSize = group lazy loads
  Native query with json_agg = single query with nested data

Bulk Operations:
  JdbcTemplate.batchUpdate() = 500 records per batch
  COPY command = fastest (PostgreSQL native, ~100k rows/sec)
  JPA: flush() + clear() every 500 records

Pagination:
  Avoid OFFSET for large pages → use keyset (cursor) pagination
  Cursor = (created_at, id) of last item in previous page

Partitioning:
  Range partition by date = fast pruning, instant DROP
  Good for: time-series, audit logs, historical data
```

➡️ [Part 53: API Design & REST Best Practices](./Part-53-APIDesign.md)
