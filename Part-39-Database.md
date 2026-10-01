# Part 39: Database Design & Optimization
## ขั้นตอนที่ 2631-2700: SQL ระดับ Professional

---

## 39.1 Database Design Principles

```
Normalization:
  1NF: Atomic values, no repeating groups
  2NF: All non-key attributes depend on FULL primary key
  3NF: No transitive dependencies
  BCNF: Every determinant is a candidate key

When to denormalize:
  - Read-heavy workloads (reports, analytics)
  - Join performance is too slow
  - Acceptable data redundancy

Index types:
  B-Tree    = default, range queries, equality
  Hash      = equality only, faster than B-Tree for =
  GiST      = geometric, full-text search
  GIN       = multi-value (arrays, jsonb, full-text)
  BRIN      = block range, large tables with sequential data
  Partial   = WHERE condition, smaller index
  Expression = index on function result
```

---

## 39.2 Advanced SQL Queries

```sql
-- ====== Window Functions ======

-- Running total, rank, and moving average
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue)   OVER (ORDER BY order_date) AS running_total,
    AVG(daily_revenue)   OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7d,
    RANK()               OVER (ORDER BY daily_revenue DESC) AS revenue_rank,
    ROW_NUMBER()         OVER (PARTITION BY DATE_TRUNC('month', order_date) ORDER BY daily_revenue DESC) AS rank_in_month,
    LAG(daily_revenue)   OVER (ORDER BY order_date) AS prev_day,
    LEAD(daily_revenue)  OVER (ORDER BY order_date) AS next_day,
    daily_revenue - LAG(daily_revenue) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_sales
ORDER BY order_date;

-- ====== CTE (Common Table Expressions) ======

WITH
  -- Base: orders with totals
  order_totals AS (
    SELECT
      o.id,
      o.user_id,
      o.created_at,
      SUM(oi.price * oi.quantity) AS total
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.status = 'COMPLETED'
    GROUP BY o.id, o.user_id, o.created_at
  ),
  
  -- User stats
  user_stats AS (
    SELECT
      user_id,
      COUNT(*)       AS order_count,
      SUM(total)     AS lifetime_value,
      AVG(total)     AS avg_order_value,
      MAX(created_at) AS last_order_date,
      MIN(created_at) AS first_order_date
    FROM order_totals
    GROUP BY user_id
  ),
  
  -- Percentile buckets
  ltv_percentiles AS (
    SELECT
      user_id,
      lifetime_value,
      NTILE(10) OVER (ORDER BY lifetime_value) AS decile,
      PERCENT_RANK() OVER (ORDER BY lifetime_value) AS percentile
    FROM user_stats
  )
  
SELECT
  u.name,
  u.email,
  us.order_count,
  us.lifetime_value,
  us.avg_order_value,
  lp.decile,
  CASE
    WHEN lp.percentile >= 0.9 THEN 'VIP'
    WHEN lp.percentile >= 0.7 THEN 'High Value'
    WHEN lp.percentile >= 0.3 THEN 'Regular'
    ELSE 'Low Value'
  END AS customer_segment
FROM users u
JOIN user_stats us ON us.user_id = u.id
JOIN ltv_percentiles lp ON lp.user_id = u.id
ORDER BY us.lifetime_value DESC;

-- ====== Recursive CTE (hierarchical data) ======

WITH RECURSIVE category_tree AS (
  -- Anchor: root categories
  SELECT
    id, name, parent_id, 0 AS depth,
    name::TEXT AS full_path
  FROM categories
  WHERE parent_id IS NULL
  
  UNION ALL
  
  -- Recursive: children
  SELECT
    c.id, c.name, c.parent_id,
    ct.depth + 1,
    ct.full_path || ' > ' || c.name
  FROM categories c
  JOIN category_tree ct ON ct.id = c.parent_id
  WHERE ct.depth < 10  -- prevent infinite loop
)
SELECT * FROM category_tree ORDER BY full_path;
```

---

## 39.3 Advanced Indexing

```sql
-- ====== Index examples (PostgreSQL) ======

-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
-- Good for: WHERE user_id = ? ORDER BY created_at DESC
-- Bad for:  WHERE created_at > ? (without user_id)

-- Partial index (only index active users)
CREATE INDEX idx_users_active_email ON users(email)
WHERE status = 'ACTIVE';

-- Expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Enables: WHERE LOWER(email) = LOWER(?)

-- GIN index for JSONB
CREATE INDEX idx_products_attrs ON products USING GIN(attributes);
-- Enables: WHERE attributes @> '{"color": "red"}'

-- Full-text search index
CREATE INDEX idx_products_fts ON products 
USING GIN(to_tsvector('english', name || ' ' || description));
-- Query: WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('laptop & gaming')

-- BRIN for time-series (very small index)
CREATE INDEX idx_logs_created_brin ON logs USING BRIN(created_at);
-- Good for append-only tables with sequential data

-- ====== EXPLAIN ANALYZE ======
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 0
ORDER BY order_count DESC
LIMIT 20;

-- Look for:
--   Seq Scan     = missing index (bad for large tables)
--   Index Scan   = using index (good)
--   Nested Loop  = can be slow with large sets
--   Hash Join    = good for larger joins
--   Merge Join   = good for sorted inputs
--   Buffers: hit = cache hit (fast), read = disk (slow)
```

---

## 39.4 JPA/Hibernate Optimization

```java
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.*;

// ====== Avoid N+1 with JOIN FETCH ======
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    // BAD: N+1 - loads order, then N queries for items
    List<Order> findByUserId(Long userId);
    
    // GOOD: Single JOIN FETCH
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
    List<Order> findByUserIdWithItems(@Param("userId") Long userId);
    
    // With pagination - cannot use JOIN FETCH directly
    // Use @EntityGraph instead
    @EntityGraph(attributePaths = {"items", "items.product"})
    Page<Order> findByUserId(Long userId, Pageable pageable);
    
    // Projections: only fetch needed fields
    @Query("SELECT new com.example.dto.OrderSummary(o.id, o.total, o.status, o.createdAt) " +
           "FROM Order o WHERE o.userId = :userId")
    List<OrderSummary> findSummariesByUserId(@Param("userId") Long userId);
    
    // Native query for complex SQL
    @Query(value = """
        SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as revenue
        FROM users u
        JOIN orders o ON o.user_id = u.id
        WHERE o.created_at >= :from AND o.created_at < :to
        GROUP BY u.id, u.name
        ORDER BY revenue DESC
        LIMIT :limit
        """, nativeQuery = true)
    List<Object[]> getTopCustomers(
        @Param("from") java.time.Instant from,
        @Param("to") java.time.Instant to,
        @Param("limit") int limit
    );
}

// ====== Batch operations ======
@Service
class BatchInsertService {
    
    @PersistenceContext
    private EntityManager em;
    
    @org.springframework.transaction.annotation.Transactional
    public void batchInsert(List<Product> products) {
        int batchSize = 100;
        
        for (int i = 0; i < products.size(); i++) {
            em.persist(products.get(i));
            
            if ((i + 1) % batchSize == 0) {
                em.flush();   // write to DB
                em.clear();   // free memory
            }
        }
        
        em.flush();  // flush remaining
    }
    
    // Spring Data batch save (configure spring.jpa.properties.hibernate.jdbc.batch_size=100)
    @org.springframework.transaction.annotation.Transactional
    public void batchSave(List<Product> products, ProductRepository repo) {
        repo.saveAll(products);
        // Works automatically if:
        // - @GeneratedValue(strategy = SEQUENCE) (not IDENTITY)
        // - hibernate.jdbc.batch_size is set
        // - hibernate.order_inserts=true
    }
}
```

---

## 39.5 Database Migrations with Flyway

```sql
-- src/main/resources/db/migration/V1__initial_schema.sql
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,
    username   VARCHAR(50)  NOT NULL UNIQUE,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,
    role       VARCHAR(20)  NOT NULL DEFAULT 'USER',
    status     VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE TABLE categories (
    id        BIGSERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    slug      VARCHAR(100) NOT NULL UNIQUE,
    parent_id BIGINT REFERENCES categories(id)
);

CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock       INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id BIGINT NOT NULL REFERENCES categories(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price     ON products(price);

CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users(id),
    status     VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    total      NUMERIC(12,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id    ON orders(user_id);
CREATE INDEX idx_orders_status     ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

CREATE TABLE order_items (
    id         BIGSERIAL PRIMARY KEY,
    order_id   BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    price      NUMERIC(10,2) NOT NULL,
    UNIQUE(order_id, product_id)
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at    BEFORE UPDATE ON users    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER products_updated_at BEFORE UPDATE ON products FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER orders_updated_at   BEFORE UPDATE ON orders   FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

```sql
-- V2__add_product_search.sql
ALTER TABLE products
    ADD COLUMN fts_vector TSVECTOR;

UPDATE products
SET fts_vector = to_tsvector('english', name || ' ' || COALESCE(description, ''));

CREATE INDEX idx_products_fts ON products USING GIN(fts_vector);

-- Auto-maintain FTS vector
CREATE OR REPLACE FUNCTION update_product_fts()
RETURNS TRIGGER AS $$
BEGIN
    NEW.fts_vector = to_tsvector('english', NEW.name || ' ' || COALESCE(NEW.description, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_fts_update
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION update_product_fts();
```

---

## 39.6 Connection Pooling & Performance Config

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    
    hikari:
      pool-name: MainPool
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000    # 30s to get connection from pool
      idle-timeout: 600000         # 10min before idle connection closed
      max-lifetime: 1800000        # 30min max connection lifetime
      connection-test-query: SELECT 1
      
      # PostgreSQL specific
      data-source-properties:
        prepareThreshold: 5         # prepare after 5 executions
        preparedStatementCacheQueries: 256
        defaultRowFetchSize: 1000  # fetch rows in batches
  
  jpa:
    database: postgresql
    show-sql: false
    open-in-view: false              # prevent lazy-loading after HTTP response
    
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          batch_size: 100
          batch_versioned_data: true
          fetch_size: 1000
        order_inserts: true
        order_updates: true
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
          use_query_cache: true
```

---

## สรุป Part 39

```
Database Performance Checklist:

1. Index Strategy
   □ Primary key indexed automatically
   □ Foreign keys indexed
   □ Frequently queried columns indexed
   □ Composite index: most selective column first
   □ Partial indexes for filtered queries

2. Query Optimization
   □ EXPLAIN ANALYZE on slow queries
   □ Avoid SELECT *
   □ Use projections (SELECT only needed columns)
   □ JOIN FETCH or @EntityGraph to prevent N+1
   □ Paginate large result sets

3. Bulk Operations
   □ Batch insert/update (batchSize=100)
   □ Use native SQL for bulk operations
   □ Partition large tables

4. Connection Pool
   □ Set pool size appropriately
   □ open-in-view = false
   □ Connection timeout configured

5. Schema Design
   □ Normalize first, denormalize for performance
   □ Appropriate data types (don't use VARCHAR(255) for everything)
   □ Constraints: NOT NULL, CHECK, UNIQUE
   □ Auto-update timestamps via triggers
```

➡️ [Part 40: Ktor (Kotlin Server-Side)](./Part-40-Ktor.md)
