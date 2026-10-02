# Part 71: Advanced Spring Security
## ขั้นตอนที่ 4871-4940: OAuth2, Method Security, Custom Filters, CSRF

---

## 71.1 Spring Security Architecture

```
Security Filter Chain (ทำงานตามลำดับ):

Request
  → CorsFilter
  → CsrfFilter
  → SecurityContextHolderFilter
  → UsernamePasswordAuthenticationFilter (form login)
  → BearerTokenAuthenticationFilter (JWT)
  → ExceptionTranslationFilter
  → AuthorizationFilter
  → Servlet (Controller)

Authentication vs Authorization:
  Authentication = "คุณเป็นใคร?" (login, JWT, API key)
  Authorization  = "คุณทำอะไรได้บ้าง?" (roles, permissions)
```

---

## 71.2 JWT Security Configuration

```java
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.config.annotation.web.configuration.*;
import org.springframework.security.web.*;
import org.springframework.security.oauth2.jwt.*;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
class SecurityConfig {
    
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // stateless API, no CSRF needed
            .sessionManagement(session -> session
                .sessionCreationPolicy(org.springframework.security.config.http.SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(org.springframework.http.HttpMethod.GET, "/api/products/**").permitAll()
                
                // Admin endpoints
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                
                // All other endpoints require authentication
                .anyRequest().authenticated()
            )
            // JWT token validation
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            // Custom JWT exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setStatus(401);
                    response.setContentType("application/json");
                    response.getWriter().write("""
                        {"error": "UNAUTHORIZED", "message": "Authentication required"}
                        """);
                })
                .accessDeniedHandler((request, response, deniedException) -> {
                    response.setStatus(403);
                    response.setContentType("application/json");
                    response.getWriter().write("""
                        {"error": "FORBIDDEN", "message": "Insufficient permissions"}
                        """);
                })
            )
            .build();
    }
    
    @Bean
    JwtDecoder jwtDecoder() {
        // RS256: asymmetric key (public key for verification)
        var publicKey = loadPublicKey();
        return org.springframework.security.oauth2.jwt.NimbusJwtDecoder
            .withPublicKey(publicKey).build();
    }
    
    @Bean
    org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter 
    jwtAuthenticationConverter() {
        
        var grantedAuthoritiesConverter = 
            new org.springframework.security.oauth2.server.resource.authentication
                .JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        var converter = 
            new org.springframework.security.oauth2.server.resource.authentication
                .JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        converter.setPrincipalClaimName("sub");
        
        return converter;
    }
    
    private java.security.interfaces.RSAPublicKey loadPublicKey() {
        // Load from file, Vault, or env var
        try {
            var keyFactory = java.security.KeyFactory.getInstance("RSA");
            var keyBytes = java.util.Base64.getDecoder().decode(
                System.getenv("JWT_PUBLIC_KEY")
            );
            return (java.security.interfaces.RSAPublicKey) keyFactory.generatePublic(
                new java.security.spec.X509EncodedKeySpec(keyBytes)
            );
        } catch (Exception e) {
            throw new RuntimeException("Failed to load public key", e);
        }
    }
}
```

---

## 71.3 Method-Level Security

```java
import org.springframework.security.access.prepost.*;
import org.springframework.security.core.annotation.*;

@Service
class OrderService {
    
    private final OrderRepository orderRepository;
    
    OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    // Only authenticated users
    @PreAuthorize("isAuthenticated()")
    public java.util.List<Order> getMyOrders(
            @AuthenticationPrincipal org.springframework.security.oauth2.jwt.Jwt jwt) {
        String userId = jwt.getSubject();
        return orderRepository.findByUserId(userId);
    }
    
    // Admin only
    @PreAuthorize("hasRole('ADMIN')")
    public java.util.List<Order> getAllOrders() {
        return orderRepository.findAll();
    }
    
    // User can access their own order, OR admin can access any
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(authentication, #orderId)")
    public Order getOrder(java.util.UUID orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found"));
    }
    
    // Permission-based (fine-grained)
    @PreAuthorize("hasPermission(#orderId, 'Order', 'CANCEL')")
    public void cancelOrder(java.util.UUID orderId) {
        var order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("Order not found"));
        order.setStatus("CANCELLED");
        orderRepository.save(order);
    }
    
    // Post-filter: filter returned list
    @PostFilter("filterObject.userId == authentication.name or hasRole('ADMIN')")
    public java.util.List<Order> getOrdersByStatus(String status) {
        return orderRepository.findByStatus(status);
    }
    
    // Pre-filter: filter input list
    @PreFilter("filterObject.ownerId == authentication.name")
    public void bulkUpdateOrders(java.util.List<Order> orders) {
        orderRepository.saveAll(orders);
    }
}

// Custom security expression
@Component("orderSecurity")
class OrderSecurityService {
    
    private final OrderRepository orderRepository;
    
    OrderSecurityService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public boolean isOwner(org.springframework.security.core.Authentication auth,
                            java.util.UUID orderId) {
        var order = orderRepository.findById(orderId).orElse(null);
        return order != null && order.getUserId().equals(auth.getName());
    }
    
    public boolean canCancel(org.springframework.security.core.Authentication auth,
                              java.util.UUID orderId) {
        var order = orderRepository.findById(orderId).orElse(null);
        if (order == null) return false;
        
        boolean isOwner = order.getUserId().equals(auth.getName());
        boolean isCancellable = "PENDING".equals(order.getStatus()) || 
                                "PROCESSING".equals(order.getStatus());
        
        return isOwner && isCancellable;
    }
}
```

---

## 71.4 Custom Authentication Filter

```java
import org.springframework.security.web.authentication.*;
import org.springframework.security.core.*;
import org.springframework.web.filter.*;

// Custom API Key authentication
@Component
class ApiKeyAuthenticationFilter extends OncePerRequestFilter {
    
    private final ApiKeyRepository apiKeyRepository;
    private final org.springframework.security.web.util.matcher.RequestMatcher apiKeyMatcher;
    
    ApiKeyAuthenticationFilter(ApiKeyRepository apiKeyRepository) {
        this.apiKeyRepository = apiKeyRepository;
        this.apiKeyMatcher = new org.springframework.security.web.util.matcher
            .AntPathRequestMatcher("/api/webhooks/**");
    }
    
    @Override
    protected void doFilterInternal(
            jakarta.servlet.http.HttpServletRequest request,
            jakarta.servlet.http.HttpServletResponse response,
            jakarta.servlet.FilterChain filterChain) throws java.io.IOException, jakarta.servlet.ServletException {
        
        if (!apiKeyMatcher.matches(request)) {
            filterChain.doFilter(request, response);
            return;
        }
        
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey == null) {
            response.setStatus(401);
            response.getWriter().write("{\"error\": \"API key required\"}");
            return;
        }
        
        var keyRecord = apiKeyRepository.findByKeyHash(hashApiKey(apiKey));
        if (keyRecord == null || !keyRecord.isActive()) {
            response.setStatus(401);
            response.getWriter().write("{\"error\": \"Invalid API key\"}");
            return;
        }
        
        // Set authentication in context
        var auth = new org.springframework.security.authentication.UsernamePasswordAuthenticationToken(
            keyRecord.getOwnerId(),
            null,
            java.util.List.of(new org.springframework.security.core.authority.SimpleGrantedAuthority("ROLE_API"))
        );
        
        org.springframework.security.core.context.SecurityContextHolder.getContext()
            .setAuthentication(auth);
        
        filterChain.doFilter(request, response);
    }
    
    private String hashApiKey(String key) {
        try {
            var digest = java.security.MessageDigest.getInstance("SHA-256");
            return java.util.Base64.getEncoder().encodeToString(
                digest.digest(key.getBytes(java.nio.charset.StandardCharsets.UTF_8))
            );
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

// Register custom filter in SecurityConfig
// http.addFilterBefore(apiKeyFilter, BearerTokenAuthenticationFilter.class)
```

---

## 71.5 OAuth2 Login (Google, GitHub)

```java
// application.yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: email, profile
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email

@Configuration
@EnableWebSecurity
class OAuth2SecurityConfig {
    
    private final CustomOAuth2UserService customOAuth2UserService;
    
    OAuth2SecurityConfig(CustomOAuth2UserService customOAuth2UserService) {
        this.customOAuth2UserService = customOAuth2UserService;
    }
    
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(info -> info
                    .userService(customOAuth2UserService)
                )
                .successHandler((request, response, authentication) -> {
                    // Generate JWT after successful OAuth2 login
                    var user = (CustomOAuth2User) authentication.getPrincipal();
                    var jwt = generateJwt(user.getUserId(), user.getEmail());
                    response.sendRedirect("/dashboard?token=" + jwt);
                })
                .failureUrl("/login?error")
            )
            .build();
    }
}

@Service
class CustomOAuth2UserService implements
        org.springframework.security.oauth2.client.userinfo.OAuth2UserService<
            org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest,
            org.springframework.security.oauth2.core.user.OAuth2User> {
    
    private final UserRepository userRepository;
    
    CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public org.springframework.security.oauth2.core.user.OAuth2User loadUser(
            org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest userRequest) {
        
        var delegate = new org.springframework.security.oauth2.client.userinfo
            .DefaultOAuth2UserService();
        var oauthUser = delegate.loadUser(userRequest);
        
        String email = oauthUser.getAttribute("email");
        String name = oauthUser.getAttribute("name");
        String provider = userRequest.getClientRegistration().getRegistrationId();
        
        // Create or update user in our DB
        var user = userRepository.findByEmail(email)
            .orElseGet(() -> userRepository.save(new User(email, name, provider)));
        
        return new CustomOAuth2User(user, oauthUser.getAttributes());
    }
}
```

---

## สรุป Part 71

```
Spring Security Key Patterns:

SecurityFilterChain:
  .authorizeHttpRequests() → matchers + rules
  .oauth2ResourceServer(oauth2 -> oauth2.jwt())
  .sessionManagement(STATELESS) for API

JWT Setup:
  JwtDecoder  = NimbusJwtDecoder.withPublicKey(rsaPublicKey)
  Converter   = extract "roles" claim → ROLE_ADMIN, ROLE_USER

Method Security (@EnableMethodSecurity):
  @PreAuthorize("hasRole('ADMIN')")
  @PreAuthorize("@myService.check(authentication, #id)")
  @PostFilter("filterObject.userId == authentication.name")

Custom Expressions:
  @Component("myBean")
  → @PreAuthorize("@myBean.canAccess(authentication, #resource)")

Custom Filter:
  extends OncePerRequestFilter
  addFilterBefore(myFilter, BearerTokenAuthenticationFilter.class)

OAuth2:
  .oauth2Login() for Google/GitHub social login
  CustomOAuth2UserService = link OAuth account to local user
  Generate JWT after OAuth2 login for stateless API

Security Rules:
  ✓ Fail-safe: default DENY (not ALLOW)
  ✓ Test every @PreAuthorize with both allowed and denied cases
  ✓ Rotate JWT keys regularly (RS256 makes rotation easy)
  ✓ Always use HTTPS (HSTS header)
  ✗ Never log sensitive data (password, token, PII)
```

➡️ [Part 72: Data Privacy & GDPR Compliance](./Part-72-DataPrivacy.md)
