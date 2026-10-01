# Part 27: Spring Security & JWT Authentication
## ขั้นตอนที่ 1791-1860: Security ระดับ Production

---

## 27.1 Spring Security Overview

```
Spring Security Architecture:

Request → FilterChain → DispatcherServlet → Controller

FilterChain:
  SecurityContextPersistenceFilter
  → UsernamePasswordAuthenticationFilter (สำหรับ form login)
  → JwtAuthenticationFilter (custom ของเรา)
  → ExceptionTranslationFilter
  → FilterSecurityInterceptor
  
Authentication Flow:
  1. Client ส่ง credentials (username/password)
  2. AuthenticationManager.authenticate()
  3. UserDetailsService.loadUserByUsername()
  4. ตรวจสอบ password ด้วย PasswordEncoder
  5. สร้าง JWT token
  6. Client ส่ง JWT ใน Authorization header
  7. JwtFilter ตรวจสอบทุก request
```

---

## 27.2 Dependencies & Configuration

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <!-- JWT library -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
app:
  jwt:
    secret: "mySecretKey1234567890abcdefghijklmnopqrstuvwxyz"  # min 256 bits
    expiration: 86400000  # 24 hours in ms
    refresh-expiration: 604800000  # 7 days

spring:
  datasource:
    url: jdbc:h2:mem:securitydb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
  h2:
    console:
      enabled: true
```

---

## 27.3 User Entity & Repository

```java
import jakarta.persistence.*;
import org.springframework.security.core.*;
import org.springframework.security.core.authority.*;
import org.springframework.security.core.userdetails.*;
import java.util.*;

@Entity
@Table(name = "users")
public class AppUser implements UserDetails {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Enumerated(EnumType.STRING)
    private Role role = Role.USER;
    
    private boolean enabled = true;
    private boolean accountNonLocked = true;
    
    public enum Role {
        USER, ADMIN, MODERATOR
    }
    
    // UserDetails implementation
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + role.name()));
    }
    
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return accountNonLocked; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled() { return enabled; }
    
    // Getters & Setters
    public Long getId() { return id; }
    public String getEmail() { return email; }
    public Role getRole() { return role; }
    
    public void setId(Long id) { this.id = id; }
    public void setUsername(String username) { this.username = username; }
    public void setPassword(String password) { this.password = password; }
    public void setEmail(String email) { this.email = email; }
    public void setRole(Role role) { this.role = role; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
}

// Repository
import org.springframework.data.jpa.repository.*;
import java.util.Optional;

@org.springframework.stereotype.Repository
interface UserRepository extends JpaRepository<AppUser, Long> {
    Optional<AppUser> findByUsername(String username);
    Optional<AppUser> findByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

---

## 27.4 JWT Service

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.*;
import java.util.*;
import java.util.function.*;
import javax.crypto.*;

@Service
public class JwtService {
    
    @Value("${app.jwt.secret}")
    private String secretKey;
    
    @Value("${app.jwt.expiration}")
    private long jwtExpiration;
    
    @Value("${app.jwt.refresh-expiration}")
    private long refreshExpiration;
    
    private SecretKey getSigningKey() {
        byte[] keyBytes = io.jsonwebtoken.io.Decoders.BASE64.decode(
            Base64.getEncoder().encodeToString(secretKey.getBytes())
        );
        return Keys.hmacShaKeyFor(keyBytes);
    }
    
    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }
    
    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return buildToken(extraClaims, userDetails, jwtExpiration);
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        return buildToken(new HashMap<>(), userDetails, refreshExpiration);
    }
    
    private String buildToken(Map<String, Object> extraClaims, UserDetails userDetails,
                               long expiration) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
    }
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
    
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }
    
    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
```

---

## 27.5 JWT Filter

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.lang.*;
import org.springframework.security.authentication.*;
import org.springframework.security.core.context.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.*;
import org.springframework.web.filter.*;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }
    
    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                    @NonNull HttpServletResponse response,
                                    @NonNull FilterChain filterChain) throws Exception {
        
        final String authHeader = request.getHeader("Authorization");
        
        // Skip if no Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        final String jwt = authHeader.substring(7);
        
        try {
            final String username = jwtService.extractUsername(jwt);
            
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                
                if (jwtService.isTokenValid(jwt, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities()
                        );
                    authToken.setDetails(new WebAuthenticationDetailsSource()
                        .buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (Exception e) {
            // Invalid token — just don't authenticate
            // The request will fail at the security check later
        }
        
        filterChain.doFilter(request, response);
    }
}
```

---

## 27.6 Security Configuration

```java
import org.springframework.context.annotation.*;
import org.springframework.security.authentication.*;
import org.springframework.security.authentication.dao.*;
import org.springframework.security.config.annotation.authentication.configuration.*;
import org.springframework.security.config.annotation.method.configuration.*;
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.config.annotation.web.configuration.*;
import org.springframework.security.config.http.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.crypto.bcrypt.*;
import org.springframework.security.crypto.password.*;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserRepository userRepository;
    
    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter, UserRepository userRepository) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userRepository = userRepository;
    }
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/h2-console/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                
                // Admin only
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                
                // Authenticated users
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            // Headers for H2 console
            .headers(h -> h.frameOptions(fo -> fo.sameOrigin()))
            // Add JWT filter before username/password filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public UserDetailsService userDetailsService() {
        return username -> userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
    
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService());
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 27.7 Auth Controller

```java
import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.security.authentication.*;
import org.springframework.security.crypto.password.*;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.constraints.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    private final AuthenticationManager authManager;
    private final JwtService jwtService;
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    public AuthController(AuthenticationManager authManager, JwtService jwtService,
                          UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.authManager = authManager;
        this.jwtService = jwtService;
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    
    // Request/Response DTOs
    record RegisterRequest(
        @NotBlank String username,
        @NotBlank @Size(min = 8) String password,
        @NotBlank @Email String email
    ) {}
    
    record LoginRequest(
        @NotBlank String username,
        @NotBlank String password
    ) {}
    
    record AuthResponse(String token, String refreshToken, String username, String role) {}
    
    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest req) {
        
        if (userRepository.existsByUsername(req.username())) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(null);  // real app: return error message
        }
        if (userRepository.existsByEmail(req.email())) {
            return ResponseEntity.status(HttpStatus.CONFLICT).body(null);
        }
        
        AppUser user = new AppUser();
        user.setUsername(req.username());
        user.setPassword(passwordEncoder.encode(req.password()));
        user.setEmail(req.email());
        user.setRole(AppUser.Role.USER);
        
        userRepository.save(user);
        
        String token = jwtService.generateToken(user);
        String refreshToken = jwtService.generateRefreshToken(user);
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new AuthResponse(token, refreshToken, user.getUsername(),
                                   user.getRole().name()));
    }
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest req) {
        
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.username(), req.password())
        );
        
        AppUser user = userRepository.findByUsername(req.username())
            .orElseThrow();
        
        String token = jwtService.generateToken(user);
        String refreshToken = jwtService.generateRefreshToken(user);
        
        return ResponseEntity.ok(
            new AuthResponse(token, refreshToken, user.getUsername(), user.getRole().name())
        );
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refreshToken(@RequestHeader("Refresh-Token") String refreshToken) {
        
        String username = jwtService.extractUsername(refreshToken);
        AppUser user = userRepository.findByUsername(username)
            .orElseThrow(() -> new RuntimeException("User not found"));
        
        if (!jwtService.isTokenValid(refreshToken, user)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        String newToken = jwtService.generateToken(user);
        return ResponseEntity.ok(
            new AuthResponse(newToken, refreshToken, user.getUsername(), user.getRole().name())
        );
    }
}
```

---

## 27.8 Protected Controllers with @PreAuthorize

```java
import org.springframework.security.access.prepost.*;
import org.springframework.security.core.*;
import org.springframework.security.core.annotation.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/me")
    public ResponseEntity<?> getCurrentUser(@AuthenticationPrincipal AppUser user) {
        return ResponseEntity.ok(Map.of(
            "id", user.getId(),
            "username", user.getUsername(),
            "email", user.getEmail(),
            "role", user.getRole()
        ));
    }
    
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public ResponseEntity<?> getUserById(@PathVariable Long id) {
        // ADMIN can see any user, users can only see themselves
        return ResponseEntity.ok(Map.of("id", id, "message", "Access granted"));
    }
    
    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<?> listUsers() {
        return ResponseEntity.ok(Map.of("users", List.of()));
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        return ResponseEntity.noContent().build();
    }
}

@RestController
@RequestMapping("/api/admin")
@PreAuthorize("hasRole('ADMIN')")  // Class-level security
public class AdminController {
    
    @GetMapping("/stats")
    public ResponseEntity<?> getStats() {
        return ResponseEntity.ok(Map.of("totalUsers", 100, "activeUsers", 85));
    }
    
    @PostMapping("/users/{id}/disable")
    public ResponseEntity<Void> disableUser(@PathVariable Long id) {
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/users/{id}/promote")
    public ResponseEntity<Void> promoteToAdmin(@PathVariable Long id) {
        return ResponseEntity.ok().build();
    }
}
```

---

## 27.9 Exception Handler

```java
import org.springframework.http.*;
import org.springframework.security.access.*;
import org.springframework.security.core.*;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class SecurityExceptionHandler {
    
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse(403, "Access denied", ex.getMessage()));
    }
    
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthentication(AuthenticationException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(401, "Authentication failed", ex.getMessage()));
    }
    
    @ExceptionHandler(io.jsonwebtoken.JwtException.class)
    public ResponseEntity<ErrorResponse> handleJwtException(io.jsonwebtoken.JwtException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(401, "Invalid token", ex.getMessage()));
    }
    
    record ErrorResponse(int status, String error, String message) {}
}
```

---

## 27.10 Testing Security

```java
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.security.test.context.support.*;
import org.springframework.security.test.web.servlet.request.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerSecurityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void unauthenticated_getMe_returns401() throws Exception {
        mockMvc.perform(get("/api/users/me"))
               .andExpect(status().isUnauthorized());
    }
    
    @Test
    @WithMockUser(username = "alice", roles = "USER")
    void authenticated_getMe_returns200() throws Exception {
        mockMvc.perform(get("/api/users/me"))
               .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(roles = "USER")
    void user_listUsers_returns403() throws Exception {
        mockMvc.perform(get("/api/users"))
               .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_listUsers_returns200() throws Exception {
        mockMvc.perform(get("/api/users"))
               .andExpect(status().isOk());
    }
    
    @Test
    void login_withValidCredentials_returnsToken() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                   .contentType("application/json")
                   .content("""
                       {"username": "admin", "password": "password123"}
                       """))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.token").exists());
    }
}
```

---

## สรุป Part 27

| Component | หน้าที่ |
|-----------|---------|
| `SecurityFilterChain` | กำหนด security rules |
| `JwtAuthenticationFilter` | ตรวจสอบ JWT ทุก request |
| `JwtService` | สร้างและตรวจสอบ JWT |
| `UserDetailsService` | โหลด user จาก DB |
| `PasswordEncoder` | encrypt/verify passwords |
| `@PreAuthorize` | method-level security |
| `@AuthenticationPrincipal` | ดึง current user |

**JWT Flow:**
```
POST /api/auth/login → token + refreshToken
GET /api/users/me    Authorization: Bearer <token>
POST /api/auth/refresh Refresh-Token: <refreshToken>
```

➡️ [Part 28: Microservices Architecture](./Part-28-Microservices.md)
