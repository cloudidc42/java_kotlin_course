# Part 19: JDBC & Database Access
## ขั้นตอนที่ 1211-1280: เชื่อมต่อฐานข้อมูลด้วย Java

---

## 19.1 JDBC Basics

```java
import java.sql.*;
import java.util.*;

public class JDBCBasics {
    
    // Connection URL formats
    // PostgreSQL: jdbc:postgresql://host:5432/database
    // MySQL:      jdbc:mysql://host:3306/database
    // H2 (in-memory): jdbc:h2:mem:testdb
    // SQLite:     jdbc:sqlite:path/to/database.db
    
    static final String URL = "jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1";
    static final String USER = "sa";
    static final String PASSWORD = "";
    
    static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASSWORD);
    }
    
    static void setupDatabase(Connection conn) throws SQLException {
        String createTable = """
            CREATE TABLE IF NOT EXISTS employees (
                id          INTEGER     PRIMARY KEY AUTO_INCREMENT,
                name        VARCHAR(100) NOT NULL,
                department  VARCHAR(50),
                salary      DECIMAL(10,2),
                hire_date   DATE,
                email       VARCHAR(100) UNIQUE
            )
            """;
        
        try (Statement stmt = conn.createStatement()) {
            stmt.execute(createTable);
        }
    }
    
    // Insert with PreparedStatement (prevents SQL injection)
    static int insertEmployee(Connection conn, String name, String dept,
                               double salary, String email) throws SQLException {
        String sql = "INSERT INTO employees (name, department, salary, hire_date, email) " +
                     "VALUES (?, ?, ?, CURRENT_DATE, ?)";
        
        try (PreparedStatement ps = conn.prepareStatement(sql,
                Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, name);
            ps.setString(2, dept);
            ps.setDouble(3, salary);
            ps.setString(4, email);
            
            ps.executeUpdate();
            
            try (ResultSet rs = ps.getGeneratedKeys()) {
                if (rs.next()) return rs.getInt(1);
            }
        }
        return -1;
    }
    
    // Query
    static void queryAll(Connection conn) throws SQLException {
        String sql = "SELECT * FROM employees ORDER BY department, name";
        
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            
            ResultSetMetaData meta = rs.getMetaData();
            int cols = meta.getColumnCount();
            
            // Print header
            for (int i = 1; i <= cols; i++) {
                System.out.printf("%-15s", meta.getColumnName(i));
            }
            System.out.println();
            System.out.println("-".repeat(cols * 15));
            
            // Print rows
            while (rs.next()) {
                for (int i = 1; i <= cols; i++) {
                    System.out.printf("%-15s", rs.getString(i));
                }
                System.out.println();
            }
        }
    }
    
    // Query with parameters
    static void queryByDepartment(Connection conn, String dept) throws SQLException {
        String sql = "SELECT name, salary FROM employees WHERE department = ? ORDER BY salary DESC";
        
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, dept);
            
            try (ResultSet rs = ps.executeQuery()) {
                System.out.println("\nDepartment: " + dept);
                while (rs.next()) {
                    System.out.printf("  %-20s ฿%,10.2f%n",
                        rs.getString("name"), rs.getDouble("salary"));
                }
            }
        }
    }
    
    // Update
    static int giveRaise(Connection conn, String dept, double percent) throws SQLException {
        String sql = "UPDATE employees SET salary = salary * ? WHERE department = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setDouble(1, 1 + percent / 100);
            ps.setString(2, dept);
            return ps.executeUpdate();
        }
    }
    
    // Delete
    static int deleteEmployee(Connection conn, int id) throws SQLException {
        String sql = "DELETE FROM employees WHERE id = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setInt(1, id);
            return ps.executeUpdate();
        }
    }
    
    // Transaction
    static void transferDepartment(Connection conn, int empId, String newDept,
                                    double salaryChange) throws SQLException {
        conn.setAutoCommit(false);  // begin transaction
        try {
            // Update department
            String updDept = "UPDATE employees SET department = ? WHERE id = ?";
            try (PreparedStatement ps = conn.prepareStatement(updDept)) {
                ps.setString(1, newDept);
                ps.setInt(2, empId);
                ps.executeUpdate();
            }
            
            // Update salary
            String updSalary = "UPDATE employees SET salary = salary + ? WHERE id = ?";
            try (PreparedStatement ps = conn.prepareStatement(updSalary)) {
                ps.setDouble(1, salaryChange);
                ps.setInt(2, empId);
                ps.executeUpdate();
            }
            
            conn.commit();  // commit transaction
            System.out.println("Transfer successful!");
        } catch (SQLException e) {
            conn.rollback();  // rollback on error
            System.out.println("Transfer failed, rolled back: " + e.getMessage());
            throw e;
        } finally {
            conn.setAutoCommit(true);
        }
    }
    
    // Aggregate queries
    static void statistics(Connection conn) throws SQLException {
        String sql = """
            SELECT 
                department,
                COUNT(*) as count,
                MIN(salary) as min_salary,
                MAX(salary) as max_salary,
                AVG(salary) as avg_salary,
                SUM(salary) as total_salary
            FROM employees
            GROUP BY department
            ORDER BY avg_salary DESC
            """;
        
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            
            System.out.println("\n=== Department Statistics ===");
            System.out.printf("%-15s %5s %10s %10s %10s %12s%n",
                "DEPT", "CNT", "MIN", "MAX", "AVG", "TOTAL");
            System.out.println("-".repeat(65));
            
            while (rs.next()) {
                System.out.printf("%-15s %5d %10.0f %10.0f %10.0f %12.0f%n",
                    rs.getString("department"),
                    rs.getInt("count"),
                    rs.getDouble("min_salary"),
                    rs.getDouble("max_salary"),
                    rs.getDouble("avg_salary"),
                    rs.getDouble("total_salary"));
            }
        }
    }
    
    public static void main(String[] args) {
        try (Connection conn = getConnection()) {
            
            setupDatabase(conn);
            
            // Insert employees
            System.out.println("=== Inserting Employees ===");
            insertEmployee(conn, "Alice Smith", "Engineering", 85000, "alice@company.com");
            insertEmployee(conn, "Bob Jones", "Engineering", 92000, "bob@company.com");
            insertEmployee(conn, "Charlie Brown", "Marketing", 67000, "charlie@company.com");
            insertEmployee(conn, "Diana Prince", "Engineering", 78000, "diana@company.com");
            insertEmployee(conn, "Eve Wilson", "Marketing", 72000, "eve@company.com");
            insertEmployee(conn, "Frank Miller", "HR", 55000, "frank@company.com");
            System.out.println("Inserted 6 employees");
            
            // Query all
            System.out.println("\n=== All Employees ===");
            queryAll(conn);
            
            // Query by department
            queryByDepartment(conn, "Engineering");
            queryByDepartment(conn, "Marketing");
            
            // Statistics
            statistics(conn);
            
            // Give raise to Engineering
            System.out.println("\n=== Giving 10% raise to Engineering ===");
            int updated = giveRaise(conn, "Engineering", 10);
            System.out.println("Updated " + updated + " employees");
            queryByDepartment(conn, "Engineering");
            
            // Transaction
            System.out.println("\n=== Transfer Employee ===");
            transferDepartment(conn, 3, "Sales", 5000);
            
        } catch (SQLException e) {
            System.err.println("Database error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

---

## 19.2 DAO Pattern

```java
import java.sql.*;
import java.util.*;

// Entity
record Product(Integer id, String name, String category, double price, int stock) {
    public Product withId(int id) {
        return new Product(id, name, category, price, stock);
    }
}

// DAO Interface
interface ProductDAO {
    Optional<Product> findById(int id);
    List<Product> findAll();
    List<Product> findByCategory(String category);
    List<Product> findByMaxPrice(double maxPrice);
    Product save(Product product);
    boolean update(Product product);
    boolean delete(int id);
    long count();
    Map<String, Double> averagePriceByCategory();
}

// JDBC Implementation
class JdbcProductDAO implements ProductDAO {
    private final Connection conn;
    
    public JdbcProductDAO(Connection conn) { this.conn = conn; }
    
    static void createTable(Connection conn) throws SQLException {
        conn.createStatement().execute("""
            CREATE TABLE IF NOT EXISTS products (
                id       INTEGER PRIMARY KEY AUTO_INCREMENT,
                name     VARCHAR(100) NOT NULL,
                category VARCHAR(50),
                price    DECIMAL(10,2),
                stock    INTEGER DEFAULT 0
            )
            """);
    }
    
    private Product mapRow(ResultSet rs) throws SQLException {
        return new Product(
            rs.getInt("id"),
            rs.getString("name"),
            rs.getString("category"),
            rs.getDouble("price"),
            rs.getInt("stock")
        );
    }
    
    @Override
    public Optional<Product> findById(int id) {
        try (PreparedStatement ps = conn.prepareStatement("SELECT * FROM products WHERE id = ?")) {
            ps.setInt(1, id);
            ResultSet rs = ps.executeQuery();
            return rs.next() ? Optional.of(mapRow(rs)) : Optional.empty();
        } catch (SQLException e) { throw new RuntimeException(e); }
    }
    
    @Override
    public List<Product> findAll() {
        List<Product> list = new ArrayList<>();
        try (ResultSet rs = conn.createStatement().executeQuery("SELECT * FROM products ORDER BY id")) {
            while (rs.next()) list.add(mapRow(rs));
        } catch (SQLException e) { throw new RuntimeException(e); }
        return list;
    }
    
    @Override
    public List<Product> findByCategory(String category) {
        List<Product> list = new ArrayList<>();
        try (PreparedStatement ps = conn.prepareStatement(
                "SELECT * FROM products WHERE category = ?")) {
            ps.setString(1, category);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) list.add(mapRow(rs));
        } catch (SQLException e) { throw new RuntimeException(e); }
        return list;
    }
    
    @Override
    public List<Product> findByMaxPrice(double maxPrice) {
        List<Product> list = new ArrayList<>();
        try (PreparedStatement ps = conn.prepareStatement(
                "SELECT * FROM products WHERE price <= ? ORDER BY price")) {
            ps.setDouble(1, maxPrice);
            ResultSet rs = ps.executeQuery();
            while (rs.next()) list.add(mapRow(rs));
        } catch (SQLException e) { throw new RuntimeException(e); }
        return list;
    }
    
    @Override
    public Product save(Product product) {
        try (PreparedStatement ps = conn.prepareStatement(
                "INSERT INTO products (name, category, price, stock) VALUES (?, ?, ?, ?)",
                Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, product.name());
            ps.setString(2, product.category());
            ps.setDouble(3, product.price());
            ps.setInt(4, product.stock());
            ps.executeUpdate();
            ResultSet keys = ps.getGeneratedKeys();
            if (keys.next()) return product.withId(keys.getInt(1));
        } catch (SQLException e) { throw new RuntimeException(e); }
        return product;
    }
    
    @Override
    public boolean update(Product product) {
        try (PreparedStatement ps = conn.prepareStatement(
                "UPDATE products SET name=?, category=?, price=?, stock=? WHERE id=?")) {
            ps.setString(1, product.name());
            ps.setString(2, product.category());
            ps.setDouble(3, product.price());
            ps.setInt(4, product.stock());
            ps.setInt(5, product.id());
            return ps.executeUpdate() > 0;
        } catch (SQLException e) { throw new RuntimeException(e); }
    }
    
    @Override
    public boolean delete(int id) {
        try (PreparedStatement ps = conn.prepareStatement("DELETE FROM products WHERE id = ?")) {
            ps.setInt(1, id);
            return ps.executeUpdate() > 0;
        } catch (SQLException e) { throw new RuntimeException(e); }
    }
    
    @Override
    public long count() {
        try (ResultSet rs = conn.createStatement().executeQuery("SELECT COUNT(*) FROM products")) {
            return rs.next() ? rs.getLong(1) : 0;
        } catch (SQLException e) { throw new RuntimeException(e); }
    }
    
    @Override
    public Map<String, Double> averagePriceByCategory() {
        Map<String, Double> result = new LinkedHashMap<>();
        try (ResultSet rs = conn.createStatement().executeQuery(
                "SELECT category, AVG(price) as avg_price FROM products GROUP BY category ORDER BY avg_price DESC")) {
            while (rs.next()) result.put(rs.getString("category"), rs.getDouble("avg_price"));
        } catch (SQLException e) { throw new RuntimeException(e); }
        return result;
    }
}

public class DAODemo {
    
    static final String URL = "jdbc:h2:mem:shopdb;DB_CLOSE_DELAY=-1";
    
    public static void main(String[] args) throws Exception {
        try (Connection conn = DriverManager.getConnection(URL, "sa", "")) {
            JdbcProductDAO.createTable(conn);
            ProductDAO dao = new JdbcProductDAO(conn);
            
            // Save products
            Product p1 = dao.save(new Product(null, "MacBook Pro", "Laptop", 75000, 10));
            Product p2 = dao.save(new Product(null, "iPhone 15", "Phone", 35000, 25));
            Product p3 = dao.save(new Product(null, "iPad Air", "Tablet", 28000, 15));
            Product p4 = dao.save(new Product(null, "AirPods Pro", "Audio", 9500, 40));
            Product p5 = dao.save(new Product(null, "MacBook Air", "Laptop", 45000, 20));
            Product p6 = dao.save(new Product(null, "Apple Watch", "Wearable", 15000, 30));
            
            System.out.println("Total products: " + dao.count());
            
            // Find by category
            System.out.println("\nLaptops:");
            dao.findByCategory("Laptop")
               .forEach(p -> System.out.printf("  [%d] %-15s ฿%,8.0f (stock: %d)%n",
                   p.id(), p.name(), p.price(), p.stock()));
            
            // Find by max price
            System.out.println("\nProducts ≤ ฿20,000:");
            dao.findByMaxPrice(20000)
               .forEach(p -> System.out.printf("  %-15s ฿%,8.0f%n", p.name(), p.price()));
            
            // Average by category
            System.out.println("\nAverage price by category:");
            dao.averagePriceByCategory()
               .forEach((cat, avg) -> System.out.printf("  %-12s ฿%,10.2f%n", cat, avg));
            
            // Update
            Product updated = new Product(p1.id(), p1.name(), p1.category(), 70000, 12);
            dao.update(updated);
            System.out.println("\nUpdated " + p1.name() + " price to ฿70,000");
            dao.findById(p1.id()).ifPresent(p ->
                System.out.printf("  Verified: ฿%,.0f (stock: %d)%n", p.price(), p.stock()));
            
            // Delete
            dao.delete(p4.id());
            System.out.println("Deleted: " + p4.name());
            System.out.println("Total after delete: " + dao.count());
        }
    }
}
```

---

## สรุป Part 19

| หัวข้อ | สิ่งสำคัญ |
|--------|---------|
| JDBC Connection | `DriverManager.getConnection()` |
| Statement | `createStatement()` - เสี่ยง SQL injection |
| PreparedStatement | `prepareStatement(sql)` - ปลอดภัย |
| ResultSet | อ่านผลลัพธ์ query |
| Transaction | `setAutoCommit(false)` → commit/rollback |
| DAO Pattern | แยก data access logic ออกจาก business logic |

---

> **โน้ต**: ในโปรเจกต์จริงใช้ Connection Pool (HikariCP) และ ORM (Hibernate/JPA)  
> แทนการเขียน JDBC ตรงๆ เพื่อ performance และ maintainability

➡️ [Part 20: Spring Boot Basics](./Part-20-Spring-Boot.md)
