# Part 67: Zero-Downtime Database Migrations
## ขั้นตอนที่ 4591-4660: Flyway, Safe Schema Changes, Expand-Contract Pattern

---

## 67.1 ปัญหา Database Migration ใน Production

```
ปัญหา: การ migrate DB ขณะมี traffic

สถานการณ์:
  - App v1 รันอยู่ กำลัง serve request
  - Deploy app v2 ที่ต้องการ schema ใหม่
  - ถ้า rename column ทันที → v1 ที่ยังรันอยู่จะ error

Bad Practice:
  1. Stop all traffic (downtime)
  2. Run migration
  3. Deploy new version
  
Good Practice (Zero-Downtime):
  ใช้ Expand-Contract Pattern:
  
  Phase 1 (Expand): เพิ่ม column ใหม่ ไม่ลบเก่า
    → App v1 ยังทำงานได้, app v2 ใช้ column ใหม่
  
  Phase 2 (Migrate): copy data จาก column เก่าไปใหม่
  
  Phase 3 (Contract): ลบ column เก่า
    → ทำหลัง app v2 deploy สำเร็จแล้วเท่านั้น
```

---

## 67.2 Flyway Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false  # strict ordering
```

```
Migration File Naming Convention:
  V{version}__{description}.sql
  R{version}__{description}.sql  (repeatable - runs when checksum changes)
  
  db/migration/
  ├── V1__create_users_table.sql
  ├── V2__create_orders_table.sql
  ├── V3__add_email_index.sql
  └── V4__rename_total_column.sql   ← ระวัง!
```

---

## 67.3 Safe Migration Examples

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- V2__create_orders_table.sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

```sql
-- SAFE: Adding new nullable column (no downtime)
-- V3__add_phone_to_users.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- SAFE: Adding column with default (PostgreSQL 11+: instant, no table rewrite)
-- V4__add_country_to_users.sql  
ALTER TABLE users ADD COLUMN country VARCHAR(2) NOT NULL DEFAULT 'TH';

-- SAFE: Adding a new index CONCURRENTLY (doesn't lock table)
-- V5__add_index_users_name.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_name ON users(name);

-- NOT SAFE: Rename column (breaks existing queries immediately)
-- V6__rename_total_to_amount.sql  ← DANGEROUS if app v1 still running
-- ALTER TABLE orders RENAME COLUMN total TO amount;  -- DON'T DO THIS

-- SAFE: Expand-Contract rename (3-phase)
-- Phase 1: Add new column (app v2 reads both, writes both)
-- V6__expand_add_amount_column.sql
ALTER TABLE orders ADD COLUMN amount DECIMAL(15,2);
UPDATE orders SET amount = total WHERE amount IS NULL;

-- Phase 2: After app v2 fully deployed (all pods use new column)
-- V7__contract_remove_old_total.sql  (run after v2 is stable)
-- ALTER TABLE orders DROP COLUMN total;  ← only after app v2 in 100%
```

---

## 67.4 Expand-Contract Pattern in Practice

```java
// Phase 1: App V2 - read from new column, fallback to old, write to both

@Entity
@Table(name = "orders")
class Order {
    
    @Id
    private java.util.UUID id;
    
    // Old column (still exists in DB during migration)
    @Column(name = "total")
    @Deprecated
    private Double totalOld;
    
    // New column (added in V6 migration)
    @Column(name = "amount")
    private Double amount;
    
    // Business logic uses the new name
    public Double getAmount() {
        // Fallback to old column if new is null (data migration in progress)
        return amount != null ? amount : totalOld;
    }
    
    public void setAmount(Double amount) {
        this.amount = amount;
        this.totalOld = amount;  // write to both during transition
    }
}

// Phase 3 (after V7 migration removes old column):
// Remove totalOld field and the fallback logic
```

---

## 67.5 Large Table Migrations

```sql
-- ปัญหา: migration บน table ขนาดใหญ่ (100M rows) ทำให้ lock นาน

-- SAFE: เพิ่ม NOT NULL column บน large table
-- ขั้น 1: เพิ่ม nullable ก่อน (fast, no table rewrite)
ALTER TABLE orders ADD COLUMN currency VARCHAR(3);

-- ขั้น 2: ใส่ data ทีละ batch (ไม่ lock table นาน)
-- V8__backfill_currency.sql
DO $$
DECLARE
    batch_size INT := 10000;
    offset_val INT := 0;
    updated_count INT;
BEGIN
    LOOP
        UPDATE orders
        SET currency = 'THB'
        WHERE id IN (
            SELECT id FROM orders
            WHERE currency IS NULL
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS updated_count = ROW_COUNT;
        EXIT WHEN updated_count = 0;
        
        COMMIT;  -- commit each batch
        PERFORM pg_sleep(0.1);  -- brief pause to let other queries run
    END LOOP;
END $$;

-- ขั้น 3: Set NOT NULL constraint (fast, validate=false first)
ALTER TABLE orders ADD CONSTRAINT orders_currency_not_null
    CHECK (currency IS NOT NULL) NOT VALID;  -- fast! validates new rows only

-- ขั้น 4: Validate existing rows (runs without exclusive lock on PostgreSQL 12+)
ALTER TABLE orders VALIDATE CONSTRAINT orders_currency_not_null;

-- ขั้น 5: สุดท้าย เปลี่ยนเป็น NOT NULL จริงๆ
ALTER TABLE orders ALTER COLUMN currency SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT orders_currency_not_null;
```

---

## 67.6 Flyway Callbacks & Repair

```java
import org.flywaydb.core.api.callback.*;

// Custom callback: log migration events
@Component
class MigrationAuditCallback implements Callback {
    
    private static final org.slf4j.Logger log = 
        org.slf4j.LoggerFactory.getLogger(MigrationAuditCallback.class);
    
    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_EACH_MIGRATE;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }
    
    @Override
    public void handle(Event event, Context context) {
        log.info("Migration completed: {} - {}",
            context.getMigrationInfo().getVersion(),
            context.getMigrationInfo().getDescription());
    }
    
    @Override
    public String getCallbackName() {
        return "MigrationAuditCallback";
    }
}

// Flyway repair (fix checksum mismatch after hotfix)
@RestController
@RequestMapping("/api/admin/db")
@org.springframework.security.access.prepost.PreAuthorize("hasRole('SUPER_ADMIN')")
class DatabaseAdminController {
    
    private final org.flywaydb.core.Flyway flyway;
    
    DatabaseAdminController(org.flywaydb.core.Flyway flyway) {
        this.flyway = flyway;
    }
    
    @PostMapping("/migrate")
    String migrate() {
        var result = flyway.migrate();
        return "Migrated " + result.migrationsExecuted + " migrations";
    }
    
    @PostMapping("/repair")
    String repair() {
        flyway.repair();
        return "Flyway checksums repaired";
    }
    
    @GetMapping("/info")
    String info() {
        var info = flyway.info();
        var sb = new StringBuilder();
        for (var migration : info.all()) {
            sb.append(String.format("%s - %s - %s%n",
                migration.getVersion(),
                migration.getDescription(),
                migration.getState()));
        }
        return sb.toString();
    }
}
```

---

## สรุป Part 67

```
Zero-Downtime Migration Rules:

ALWAYS SAFE:
  ✓ ADD nullable column
  ✓ ADD column with DEFAULT (PostgreSQL 11+)
  ✓ ADD new table
  ✓ ADD index CONCURRENTLY
  ✓ ADD constraint NOT VALID, then VALIDATE separately
  ✓ DROP constraint (except NOT NULL)

DANGEROUS (requires Expand-Contract):
  ✗ RENAME column → Add new, copy data, drop old
  ✗ RENAME table → Add view with old name, migrate apps, drop view
  ✗ DROP column → ensure no code references it first
  ✗ CHANGE column type → Add new column, copy/transform, drop old

Expand-Contract Steps:
  Phase 1 (Expand): Add new structure, keep old
  Phase 2 (Migrate): Deploy app that uses both
  Phase 3 (Contract): Remove old structure

Large Table Tips:
  - Batch UPDATE (10,000 rows), COMMIT each batch
  - Use pg_sleep(0.1) between batches (avoid overload)
  - ADD CONSTRAINT ... NOT VALID = fast (no full scan)
  - VALIDATE CONSTRAINT = separate step, non-blocking

Flyway Tips:
  - Never modify committed migration files (use repair if hotfix needed)
  - out-of-order: false in production
  - Use Java-based migrations for complex data transforms
```

➡️ [Part 68: Advanced Kotlin Patterns](./Part-68-AdvancedKotlin.md)
