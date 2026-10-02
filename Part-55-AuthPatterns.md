# Part 55: Authentication & Authorization Patterns
## ขั้นตอนที่ 3751-3820: JWT, OAuth2, RBAC, Multi-tenancy

---

## 55.1 JWT Authentication Flow

```
JWT Authentication Flow:

1. Client: POST /auth/login {username, password}
2. Server: validate credentials → create JWT
3. Server: return { accessToken, refreshToken }
4. Client: store tokens (memory/cookie)

5. Client: GET /api/orders (Authorization: Bearer <accessToken>)
6. Server: validate JWT signature + expiry
7. Server: extract userId + roles from claims
8. Server: check authorization
9. Server: return response

Token types:
  Access Token:
    - Short-lived (15min - 1hr)
    - Contains: userId, roles, expiry
    - Stateless: server doesn't store it
    
  Refresh Token:
    - Long-lived (7-30 days)
    - Stored in DB (can be revoked)
    - Used only to get new access token

JWT Structure: header.payload.signature
  header:    {"alg":"RS256","typ":"JWT"}
  payload:   {"sub":"user123","role":"USER","iat":...,"exp":...}
  signature: RSA256(base64(header) + "." + base64(payload), privateKey)
```

---

## 55.2 JWT Implementation

```java
import io.jsonwebtoken.*;
import java.security.interfaces.*;

@org.springframework.stereotype.Service
class JwtService {
    
    // Use RSA256 (asymmetric): sign with private key, verify with public key
    // Can share public key with other services for verification without sharing secret
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;
    
    private static final long ACCESS_TOKEN_EXPIRY = 15 * 60 * 1000L;   // 15 minutes
    private static final long REFRESH_TOKEN_EXPIRY = 7 * 24 * 60 * 60 * 1000L; // 7 days
    
    JwtService(RSAPrivateKey privateKey, RSAPublicKey publicKey) {
        this.privateKey = privateKey;
        this.publicKey = publicKey;
    }
    
    public String generateAccessToken(String userId, String role, List<String> permissions) {
        return Jwts.builder()
            .subject(userId)
            .claim("role", role)
            .claim("permissions", permissions)
            .issuedAt(new java.util.Date())
            .expiration(new java.util.Date(System.currentTimeMillis() + ACCESS_TOKEN_EXPIRY))
            .signWith(privateKey)
            .compact();
    }
    
    public String generateRefreshToken(String userId) {
        return Jwts.builder()
            .subject(userId)
            .id(java.util.UUID.randomUUID().toString())  // jti = JWT ID (for revocation)
            .issuedAt(new java.util.Date())
            .expiration(new java.util.Date(System.currentTimeMillis() + REFRESH_TOKEN_EXPIRY))
            .signWith(privateKey)
            .compact();
    }
    
    public Claims validateToken(String token) {
        return Jwts.parser()
            .verifyWith(publicKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
    
    public boolean isTokenExpired(String token) {
        try {
            validateToken(token);
            return false;
        } catch (ExpiredJwtException e) {
            return true;
        }
    }
    
    public String extractUserId(String token) {
        return validateToken(token).getSubject();
    }
}

// Auth Controller
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/auth")
class AuthController {
    
    private final AuthService authService;
    
    AuthController(AuthService authService) { this.authService = authService; }
    
    @org.springframework.web.bind.annotation.PostMapping("/login")
    public org.springframework.http.ResponseEntity<TokenResponse> login(
            @jakarta.validation.Valid 
            @org.springframework.web.bind.annotation.RequestBody LoginRequest request,
            jakarta.servlet.http.HttpServletResponse response) {
        
        var tokens = authService.login(request.username(), request.password());
        
        // Store refresh token in HttpOnly cookie (not accessible by JS)
        var cookie = new jakarta.servlet.http.Cookie("refreshToken", tokens.refreshToken());
        cookie.setHttpOnly(true);
        cookie.setSecure(true);    // HTTPS only
        cookie.setPath("/auth/refresh");
        cookie.setMaxAge(7 * 24 * 3600);
        response.addCookie(cookie);
        
        // Return only access token in body
        return org.springframework.http.ResponseEntity.ok(
            new TokenResponse(tokens.accessToken(), "Bearer", 900)
        );
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/refresh")
    public org.springframework.http.ResponseEntity<TokenResponse> refresh(
            @org.springframework.web.bind.annotation.CookieValue("refreshToken") String refreshToken) {
        
        var tokens = authService.refresh(refreshToken);
        return org.springframework.http.ResponseEntity.ok(
            new TokenResponse(tokens.accessToken(), "Bearer", 900)
        );
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/logout")
    @org.springframework.http.ResponseStatus(org.springframework.http.HttpStatus.NO_CONTENT)
    public void logout(
            @org.springframework.web.bind.annotation.CookieValue("refreshToken") String refreshToken,
            jakarta.servlet.http.HttpServletResponse response) {
        authService.revokeRefreshToken(refreshToken);
        
        // Clear cookie
        var cookie = new jakarta.servlet.http.Cookie("refreshToken", "");
        cookie.setMaxAge(0);
        cookie.setPath("/auth/refresh");
        response.addCookie(cookie);
    }
}

record LoginRequest(
    @jakarta.validation.constraints.NotBlank String username,
    @jakarta.validation.constraints.NotBlank String password
) {}
record TokenResponse(String accessToken, String tokenType, int expiresIn) {}
```

---

## 55.3 Spring Security Configuration

```java
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.config.annotation.web.configurers.*;
import org.springframework.security.web.*;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@org.springframework.context.annotation.Configuration
@org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
@org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
class SecurityConfig {
    
    private final JwtAuthFilter jwtAuthFilter;
    
    SecurityConfig(JwtAuthFilter jwtAuthFilter) { this.jwtAuthFilter = jwtAuthFilter; }
    
    @org.springframework.context.annotation.Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)  // using JWT, no session
            .sessionManagement(session ->
                session.sessionCreationPolicy(
                    org.springframework.security.config.http.SessionCreationPolicy.STATELESS
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/actuator/health", "/swagger-ui/**").permitAll()
                .requestMatchers(org.springframework.http.HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(401);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"code\":\"UNAUTHORIZED\",\"message\":\"Authentication required\"}");
                })
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(403);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"code\":\"FORBIDDEN\",\"message\":\"Access denied\"}");
                })
            )
            .build();
    }
    
    @org.springframework.context.annotation.Bean
    org.springframework.security.crypto.password.PasswordEncoder passwordEncoder() {
        return new org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder(12);
    }
}

// JWT Filter
@org.springframework.stereotype.Component
class JwtAuthFilter extends org.springframework.web.filter.OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final org.springframework.security.core.userdetails.UserDetailsService userDetailsService;
    
    JwtAuthFilter(JwtService jwtService,
                  org.springframework.security.core.userdetails.UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }
    
    @Override
    protected void doFilterInternal(
            jakarta.servlet.http.HttpServletRequest request,
            jakarta.servlet.http.HttpServletResponse response,
            jakarta.servlet.FilterChain filterChain) throws Exception {
        
        var auth = request.getHeader("Authorization");
        if (auth == null || !auth.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        var token = auth.substring(7);
        
        try {
            var claims = jwtService.validateToken(token);
            var userId = claims.getSubject();
            
            if (userId != null && org.springframework.security.core.context.SecurityContextHolder
                    .getContext().getAuthentication() == null) {
                
                var userDetails = userDetailsService.loadUserByUsername(userId);
                var authToken = new org.springframework.security.authentication
                    .UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                authToken.setDetails(
                    new org.springframework.security.web.authentication.WebAuthenticationDetailsSource()
                        .buildDetails(request)
                );
                org.springframework.security.core.context.SecurityContextHolder
                    .getContext().setAuthentication(authToken);
            }
        } catch (JwtException e) {
            // Invalid token → don't set auth, let 401 handler deal with it
        }
        
        filterChain.doFilter(request, response);
    }
}
```

---

## 55.4 Role-Based Access Control (RBAC)

```java
import org.springframework.security.access.prepost.*;

// Method-level security with @PreAuthorize
@org.springframework.stereotype.Service
class OrderService {
    
    // Only authenticated users
    @PreAuthorize("isAuthenticated()")
    public List<Order> getMyOrders(String userId) { return List.of(); }
    
    // Admin only
    @PreAuthorize("hasRole('ADMIN')")
    public List<Order> getAllOrders() { return List.of(); }
    
    // Admin or the order owner
    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.name")
    public Order getOrder(String orderId, String userId) { return null; }
    
    // Custom SpEL with service call
    @PreAuthorize("@orderSecurity.canAccess(authentication, #orderId)")
    public void cancelOrder(String orderId) {}
    
    // Permission-based (more flexible than roles)
    @PreAuthorize("hasAuthority('ORDER:WRITE')")
    public Order createOrder(CreateOrderRequest req) { return null; }
    
    // Post-filter: filter results based on ownership
    @PostFilter("filterObject.userId == authentication.name or hasRole('ADMIN')")
    public List<Order> searchOrders(String query) { return List.of(); }
}

// Custom security expression component
@org.springframework.stereotype.Component("orderSecurity")
class OrderSecurityService {
    
    private final OrderRepository orderRepository;
    
    OrderSecurityService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public boolean canAccess(
            org.springframework.security.core.Authentication auth,
            String orderId) {
        // Admin can access anything
        if (auth.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            return true;
        }
        // User can only access their own orders
        return orderRepository.findById(orderId)
            .map(order -> order.getUserId().equals(auth.getName()))
            .orElse(false);
    }
}
```

---

## 55.5 OAuth2 / OpenID Connect

```yaml
# application.yaml: OAuth2 login via Google
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,profile,email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
```

```java
// OAuth2 user service: create local user on first login
@org.springframework.stereotype.Service
class CustomOAuth2UserService extends
        org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService {
    
    private final UserRepository userRepository;
    
    CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public org.springframework.security.oauth2.core.user.OAuth2User loadUser(
            org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest request) {
        
        var oAuth2User = super.loadUser(request);
        
        String email = oAuth2User.getAttribute("email");
        String name = oAuth2User.getAttribute("name");
        String provider = request.getClientRegistration().getRegistrationId();
        
        // Create or update local user
        var user = userRepository.findByEmail(email)
            .orElseGet(() -> userRepository.save(
                new User(name, email, provider, "USER")
            ));
        
        return oAuth2User;
    }
}
```

---

## สรุป Part 55

```
Authentication & Authorization:

JWT Best Practices:
  ✓ RS256 (asymmetric) > HS256 (symmetric) for microservices
  ✓ Short access token (15min) + long refresh token (7 days)
  ✓ Refresh token in HttpOnly cookie (XSS protection)
  ✓ Access token in memory or Authorization header only
  ✓ Store refresh tokens in DB (enables revocation)
  ✓ jti claim for refresh token revocation

Spring Security:
  ✓ Stateless session (STATELESS)
  ✓ @PreAuthorize for method-level control
  ✓ Custom AuthenticationEntryPoint for 401 JSON response
  ✓ JwtAuthFilter before UsernamePasswordAuthenticationFilter

RBAC:
  Roles: USER, ADMIN, MANAGER
  Permissions: ORDER:READ, ORDER:WRITE, USER:ADMIN
  Custom SpEL: @orderSecurity.canAccess(authentication, #id)

OAuth2:
  openid scope = identity (who you are)
  profile + email = basic info
  Create local user on first OAuth login
```

➡️ [Part 56: Distributed Systems & Consistency](./Part-56-DistributedSystems.md)
