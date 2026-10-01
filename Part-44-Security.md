# Part 44: Security Best Practices
## ขั้นตอนที่ 2981-3050: Application Security

---

## 44.1 OWASP Top 10

```
OWASP Top 10 (2021) - Most critical web application security risks:

1. Broken Access Control      → authorization failures
2. Cryptographic Failures     → weak encryption, plaintext secrets  
3. Injection                  → SQL, LDAP, OS command injection
4. Insecure Design            → missing threat modeling
5. Security Misconfiguration  → default passwords, stack traces exposed
6. Vulnerable Components      → outdated libraries with known CVEs
7. Authentication Failures    → weak passwords, no MFA, session fixation
8. Data Integrity Failures    → unsigned objects, deserialization
9. Logging Failures           → no audit log, sensitive data logged
10. SSRF                      → server-side request forgery
```

---

## 44.2 Input Validation & Sanitization

```java
import jakarta.validation.constraints.*;
import jakarta.validation.*;
import org.springframework.validation.annotation.*;

// ====== Bean Validation ======
record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    @Pattern(regexp = "^[a-zA-Z\\s'-]+$", message = "Name contains invalid characters")
    String name,
    
    @NotBlank
    @Email(message = "Invalid email format")
    @Size(max = 255)
    String email,
    
    @NotBlank
    @Size(min = 8, max = 100, message = "Password must be at least 8 characters")
    @Pattern(
        regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]+$",
        message = "Password must contain uppercase, lowercase, digit, and special character"
    )
    String password,
    
    @NotNull
    @Min(0) @Max(150)
    Integer age,
    
    @NotNull
    @DecimalMin("0.01")
    @DecimalMax("999999.99")
    java.math.BigDecimal balance,
    
    @Size(max = 10)
    List<@NotBlank @Size(max = 50) String> tags
) {}

// Validate in controller
@RestController
@Validated
class UserController {
    
    @PostMapping("/users")
    public ResponseEntity<UserDTO> create(
            @Valid @RequestBody CreateUserRequest request) {
        // Only reached if validation passes
        return ResponseEntity.ok(userService.create(request));
    }
    
    @GetMapping("/users/{id}")
    public ResponseEntity<UserDTO> getById(
            @PathVariable @Min(1) Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}

// Custom validator
@jakarta.validation.Constraint(validatedBy = PhoneNumberValidator.class)
@interface ValidPhoneNumber {
    String message() default "Invalid phone number";
    Class<?>[] groups() default {};
    Class<? extends jakarta.validation.Payload>[] payload() default {};
}

class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {
    @Override
    public boolean isValid(String phone, ConstraintValidatorContext ctx) {
        if (phone == null) return true;  // @NotNull handles null
        return phone.matches("^\\+?[1-9]\\d{7,14}$");
    }
}
```

---

## 44.3 SQL Injection Prevention

```java
import org.springframework.jdbc.core.*;
import jakarta.persistence.*;

// ====== VULNERABLE (never do this) ======
class VulnerableRepository {
    
    @Autowired JdbcTemplate jdbc;
    
    // DANGEROUS: string concatenation → SQL injection
    public List<User> findByEmail_DANGEROUS(String email) {
        String sql = "SELECT * FROM users WHERE email = '" + email + "'";
        // Input: "' OR '1'='1" → returns ALL users!
        return jdbc.query(sql, userMapper);
    }
    
    // DANGEROUS: native query with concatenation
    @Query(value = "SELECT * FROM users WHERE name LIKE '%" + "#{#name}" + "%'",
           nativeQuery = true)
    List<User> searchByName_DANGEROUS(@Param("name") String name);
}

// ====== SAFE approaches ======
class SafeRepository {
    
    @Autowired JdbcTemplate jdbc;
    
    // SAFE: parameterized query
    public List<User> findByEmail(String email) {
        return jdbc.query(
            "SELECT * FROM users WHERE email = ?",
            new Object[]{email},
            userMapper
        );
    }
    
    // SAFE: named parameters
    @Autowired NamedParameterJdbcTemplate namedJdbc;
    
    public List<User> findByNameAndRole(String name, String role) {
        String sql = "SELECT * FROM users WHERE name LIKE :name AND role = :role";
        return namedJdbc.query(sql,
            Map.of("name", "%" + name + "%", "role", role),
            userMapper
        );
    }
    
    // SAFE: JPA JPQL with parameters
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmail(@Param("email") String email);
    
    // SAFE: criteria API (for dynamic queries)
    public List<User> dynamicSearch(String name, String email, String role) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        if (name != null) predicates.add(cb.like(root.get("name"), "%" + name + "%"));
        if (email != null) predicates.add(cb.equal(root.get("email"), email));
        if (role != null) predicates.add(cb.equal(root.get("role"), role));
        
        query.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(query).getResultList();
    }
    
    @PersistenceContext EntityManager entityManager;
    RowMapper<User> userMapper = (rs, n) -> new User(
        rs.getLong("id"), rs.getString("name"), rs.getString("email")
    );
}
```

---

## 44.4 Password Security

```java
import org.springframework.security.crypto.bcrypt.*;
import org.springframework.security.crypto.password.*;
import org.springframework.stereotype.*;

@Service
public class PasswordService {
    
    private final PasswordEncoder passwordEncoder = new BCryptPasswordEncoder(12);
    // Strength 12 = ~250ms per hash (balance security vs performance)
    
    public String hashPassword(String plaintext) {
        return passwordEncoder.encode(plaintext);
    }
    
    public boolean verifyPassword(String plaintext, String hashed) {
        return passwordEncoder.matches(plaintext, hashed);
    }
    
    // Password strength check
    public PasswordStrength checkStrength(String password) {
        int score = 0;
        
        if (password.length() >= 8) score++;
        if (password.length() >= 12) score++;
        if (password.matches(".*[A-Z].*")) score++;
        if (password.matches(".*[a-z].*")) score++;
        if (password.matches(".*\\d.*")) score++;
        if (password.matches(".*[!@#$%^&*].*")) score++;
        
        // Check against common passwords
        if (COMMON_PASSWORDS.contains(password.toLowerCase())) {
            return PasswordStrength.VERY_WEAK;
        }
        
        return switch (score) {
            case 0, 1, 2 -> PasswordStrength.WEAK;
            case 3, 4 -> PasswordStrength.FAIR;
            case 5 -> PasswordStrength.STRONG;
            default -> PasswordStrength.VERY_STRONG;
        };
    }
    
    enum PasswordStrength { VERY_WEAK, WEAK, FAIR, STRONG, VERY_STRONG }
    
    private static final Set<String> COMMON_PASSWORDS = Set.of(
        "password", "123456", "qwerty", "password123", "admin",
        "letmein", "welcome", "monkey", "dragon", "master"
    );
}

// Account lockout (prevent brute force)
@Service
class LoginAttemptService {
    
    private final org.springframework.data.redis.core.StringRedisTemplate redis;
    
    LoginAttemptService(org.springframework.data.redis.core.StringRedisTemplate redis) {
        this.redis = redis;
    }
    
    private static final int MAX_ATTEMPTS = 5;
    private static final java.time.Duration LOCKOUT_DURATION = java.time.Duration.ofMinutes(15);
    
    public void recordFailedAttempt(String username) {
        String key = "login:attempts:" + username;
        Long attempts = redis.opsForValue().increment(key);
        if (attempts == 1) {
            redis.expire(key, LOCKOUT_DURATION);
        }
    }
    
    public void recordSuccess(String username) {
        redis.delete("login:attempts:" + username);
    }
    
    public boolean isLocked(String username) {
        String key = "login:attempts:" + username;
        String attempts = redis.opsForValue().get(key);
        return attempts != null && Long.parseLong(attempts) >= MAX_ATTEMPTS;
    }
    
    public long getRemainingAttempts(String username) {
        String key = "login:attempts:" + username;
        String attempts = redis.opsForValue().get(key);
        long current = attempts == null ? 0 : Long.parseLong(attempts);
        return Math.max(0, MAX_ATTEMPTS - current);
    }
}
```

---

## 44.5 Secrets Management

```java
// ====== Environment variables (basic) ======
String dbPassword = System.getenv("DATABASE_PASSWORD");
String jwtSecret = System.getenv("JWT_SECRET");
// NEVER hardcode secrets

// ====== Spring Config with Vault ======
// application.yml:
// spring:
//   config:
//     import: vault://
//   cloud:
//     vault:
//       host: vault.example.com
//       scheme: https
//       authentication: TOKEN
//       token: ${VAULT_TOKEN}
//       kv:
//         enabled: true
//         backend: secret

// ====== AWS Secrets Manager ======
// spring:
//   config:
//     import: aws-secretsmanager:myapp/production/db-credentials

// ====== Kubernetes Secrets (in docker-compose / K8s manifest) ======
// env:
//   - name: DATABASE_PASSWORD
//     valueFrom:
//       secretKeyRef:
//         name: db-credentials
//         key: password

// ====== Encrypting sensitive config ======
// jasypt-spring-boot:
// ENC(encryptedValue) in application.yml
// Encrypted with: java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI
//   input=mypassword password=masterKey algorithm=PBEWITHHMACSHA512ANDAES_256

@org.springframework.context.annotation.Configuration
class SecretsConfig {
    
    @org.springframework.beans.factory.annotation.Value("${database.password}")
    private String dbPassword;
    
    // Never log secrets
    // Never return secrets in API responses
    // Never commit secrets to git
    // Rotate secrets regularly
    // Use principle of least privilege
}
```

---

## 44.6 XSS & CSRF Prevention

```java
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.web.csrf.*;

@org.springframework.context.annotation.Configuration
public class SecurityConfig {
    
    @org.springframework.context.annotation.Bean
    public org.springframework.security.web.SecurityFilterChain securityFilterChain(
            HttpSecurity http) throws Exception {
        
        http
            // CSRF: enabled by default for browser clients
            // For REST APIs with stateless JWT: disable
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // Or disable for pure API:
                // .disable()
            )
            
            // Security headers
            .headers(headers -> headers
                .frameOptions(frame -> frame.sameOrigin())  // X-Frame-Options: SAMEORIGIN
                .xssProtection(xss -> xss.headerValue(
                    org.springframework.security.web.header.writers.XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK
                ))
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; script-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'")
                )
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                )
                .referrerPolicy(ref -> ref
                    .policy(org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                )
            );
        
        return http.build();
    }
}

// Content Security Policy for Thymeleaf
// In HTML template:
// <meta http-equiv="Content-Security-Policy" content="default-src 'self'">

// Output encoding to prevent XSS in Thymeleaf:
// th:text="${userInput}" → auto-escaped
// th:utext="${userInput}" → NOT escaped (dangerous!)
```

---

## 44.7 Dependency Security Scanning

```xml
<!-- pom.xml - OWASP Dependency Check -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.10</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>  <!-- fail on HIGH/CRITICAL -->
        <format>HTML,JSON</format>
        <outputDirectory>${project.build.directory}/security</outputDirectory>
    </configuration>
</plugin>
```

```yaml
# GitHub Actions security scanning
- name: Dependency Check
  uses: jeremylong/DependencyCheck@v4
  with:
    project: MyApp
    scan: '.'
    format: JSON
    failBuildOnCVSS: 7

- name: Code scanning (CodeQL)
  uses: github/codeql-action/analyze@v3
  with:
    languages: java, kotlin

- name: Secret scanning
  uses: trufflesecurity/trufflehog@main
  with:
    scanArguments: "--only-verified"
```

---

## สรุป Part 44

```
Security Checklist:

Authentication:
  □ Strong password hashing (bcrypt strength >= 10)
  □ Account lockout after failed attempts
  □ MFA support
  □ Secure session management
  □ JWT with expiry + refresh tokens

Authorization:
  □ Role-based access control
  □ Principle of least privilege
  □ Method-level security (@PreAuthorize)
  □ Test for horizontal/vertical privilege escalation

Input Validation:
  □ Validate ALL inputs (server-side, never rely on client)
  □ Parameterized queries (no SQL concatenation)
  □ Output encoding (prevent XSS)
  □ Content-Type validation

Secrets:
  □ No hardcoded secrets in code
  □ Environment variables or secret manager
  □ Secrets not in logs
  □ Regular rotation

Dependencies:
  □ OWASP dependency check in CI/CD
  □ Regular updates
  □ Pin versions

Infrastructure:
  □ HTTPS everywhere
  □ Security headers (CSP, HSTS, X-Frame-Options)
  □ CORS properly configured
  □ Least privilege on service accounts
```

➡️ [Part 45: Clean Architecture & DDD](./Part-45-CleanArchitecture.md)
