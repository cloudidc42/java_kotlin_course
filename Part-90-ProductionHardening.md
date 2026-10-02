# Part 90: Production Hardening & Zero-Trust Architecture
## ขั้นตอนที่ 6201-6270: Security Hardening, mTLS, Secret Management, Vault

---

## 90.1 Zero-Trust Architecture

```
Zero Trust Principles:
  "Never trust, always verify"
  "Assume breach"
  "Verify explicitly"
  
Traditional (Perimeter) Security:
  Trust everything inside the network
  → VPN access = full network access
  → One compromised machine = entire network at risk

Zero Trust:
  Every request authenticated + authorized
  Even internal service-to-service calls
  Least privilege access
  Network position doesn't grant trust
  
Pillars:
  1. Identity    = strong authentication (mTLS, OIDC)
  2. Device      = device health verification
  3. Network     = microsegmentation, no implicit trust
  4. Application = per-request authorization
  5. Data        = classify and protect data
  6. Visibility  = comprehensive monitoring
```

---

## 90.2 Mutual TLS (mTLS)

```kotlin
// mTLS: both client AND server present certificates
// Server verifies client cert (not just client verifying server)

// Use case: service-to-service authentication
// order-service → payment-service: both present certs

// ====== Spring Boot mTLS Server Config ======
/*
server:
  ssl:
    enabled: true
    key-store: classpath:server.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    client-auth: need  # require client cert
    trust-store: classpath:ca.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
*/

// ====== Spring Boot mTLS Client Config ======
@Configuration
class RestTemplateConfig {
    
    @Bean
    fun restTemplate(
        @Value("\${client.keystore.path}") keystorePath: String,
        @Value("\${client.keystore.password}") keystorePassword: String,
        @Value("\${client.truststore.path}") truststorePath: String,
        @Value("\${client.truststore.password}") truststorePassword: String
    ): RestTemplate {
        val keyStore = KeyStore.getInstance("PKCS12")
        keystorePath.let { path ->
            FileInputStream(path).use { fis ->
                keyStore.load(fis, keystorePassword.toCharArray())
            }
        }
        
        val trustStore = KeyStore.getInstance("PKCS12")
        truststorePath.let { path ->
            FileInputStream(path).use { fis ->
                trustStore.load(fis, truststorePassword.toCharArray())
            }
        }
        
        val sslContext = SSLContextBuilder()
            .loadKeyMaterial(keyStore, keystorePassword.toCharArray())
            .loadTrustMaterial(trustStore, null)
            .build()
        
        val httpClient = HttpClients.custom()
            .setSSLContext(sslContext)
            .setSSLHostnameVerifier(SSLConnectionSocketFactory.getDefaultHostnameVerifier())
            .build()
        
        val factory = HttpComponentsClientHttpRequestFactory(httpClient)
        return RestTemplate(factory)
    }
}

// ====== Extract client identity from cert ======
@RestController
class ServiceController {
    
    @GetMapping("/internal/health")
    fun internalHealth(request: HttpServletRequest): ResponseEntity<Map<String, String>> {
        // Extract service identity from client certificate
        val certs = request.getAttribute("jakarta.servlet.request.X509Certificate")
                as? Array<java.security.cert.X509Certificate>
        
        val clientSubject = certs?.firstOrNull()?.subjectX500Principal?.name
        log.info("Request from service: $clientSubject")
        
        return ResponseEntity.ok(mapOf("status" to "OK", "calledBy" to (clientSubject ?: "unknown")))
    }
}

// ====== Certificate generation ======
/*
# Root CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt \
    -subj "/CN=Internal CA/O=Example/C=TH"

# Service certificate
openssl genrsa -out service.key 2048
openssl req -new -key service.key -out service.csr \
    -subj "/CN=order-service/O=Example/C=TH"
openssl x509 -req -days 365 -in service.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out service.crt

# Convert to PKCS12
openssl pkcs12 -export -in service.crt -inkey service.key \
    -out service.p12 -passout pass:changeme
*/
```

---

## 90.3 HashiCorp Vault Integration

```kotlin
// Vault: secret management, dynamic credentials, PKI

// build.gradle.kts
// implementation("org.springframework.cloud:spring-cloud-starter-vault-config")

// ====== application.yaml ======
/*
spring:
  config:
    import: "vault://"
  cloud:
    vault:
      authentication: kubernetes  # use K8s service account
      kubernetes:
        role: shop-api
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      uri: https://vault.example.com
      kv:
        enabled: true
        backend: secret
        application-name: shop-api

# Vault path: secret/data/shop-api
# Contains: db.password, redis.password, jwt.secret, stripe.api-key
*/

// ====== Vault Dynamic Database Credentials ======
@Configuration
class VaultDatabaseConfig {
    
    @Bean
    @VaultPropertySource("database/creds/shop-api")
    fun dataSource(
        @Value("\${username}") username: String,
        @Value("\${password}") password: String,
        @Value("\${spring.datasource.url}") url: String
    ): DataSource {
        return HikariDataSource().apply {
            jdbcUrl = url
            this.username = username
            this.password = password
            // Short max-lifetime because Vault creds expire
            maxLifetime = 600000  // 10 min
        }
    }
}

// ====== Vault PKI (dynamic certificates) ======
@Component
class VaultPkiManager(
    private val vaultTemplate: VaultTemplate
) {
    fun issueCertificate(commonName: String, ttl: String = "24h"): CertificateInfo {
        val response = vaultTemplate.write(
            "pki/issue/shop-internal",
            mapOf(
                "common_name" to commonName,
                "ttl" to ttl,
                "alt_names" to "localhost,127.0.0.1"
            )
        )
        
        val data = response.data!!
        return CertificateInfo(
            certificate = data["certificate"] as String,
            privateKey = data["private_key"] as String,
            issuingCa = data["issuing_ca"] as String,
            expiresAt = data["expiration"] as Long
        )
    }
}
```

---

## 90.4 Service Mesh Security (Istio)

```yaml
# Istio mTLS policy: enforce mutual TLS between all services
# k8s/istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # STRICT = only mTLS, reject plain HTTP

---
# Authorization policy: only allow specific services
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              # Only allow calls from these service accounts
              - "cluster.local/ns/production/sa/api-gateway"
              - "cluster.local/ns/production/sa/notification-service"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/metrics-scraper"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/actuator/prometheus"]

---
# Network policy: restrict pod communication at network level
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    - to:  # allow DNS
        - namespaceSelector: {}
      ports:
        - port: 53
          protocol: UDP
```

---

## 90.5 Security Headers & CORS

```kotlin
// Spring Security: configure security headers
@Configuration
@EnableWebSecurity
class SecurityHeadersConfig {
    
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .headers { headers ->
                headers
                    .contentTypeOptions {}  // X-Content-Type-Options: nosniff
                    .frameOptions { it.deny() }  // X-Frame-Options: DENY
                    .xssProtection { }  // X-XSS-Protection: 1; mode=block
                    .contentSecurityPolicy { csp ->
                        csp.policyDirectives(
                            "default-src 'self'; " +
                            "script-src 'self'; " +
                            "style-src 'self' https://fonts.googleapis.com; " +
                            "img-src 'self' data: https:; " +
                            "font-src 'self' https://fonts.gstatic.com; " +
                            "connect-src 'self' https://api.example.com; " +
                            "frame-ancestors 'none'"
                        )
                    }
                    .httpStrictTransportSecurity { hsts ->
                        hsts.includeSubDomains(true).maxAgeInSeconds(31536000)
                    }
                    .permissionsPolicy { permissions ->
                        permissions.policy("camera=(), microphone=(), geolocation=()")
                    }
                    .referrerPolicy { it.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN) }
            }
            .cors { cors -> cors.configurationSource(corsConfigurationSource()) }
        
        return http.build()
    }
    
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val config = CorsConfiguration().apply {
            allowedOrigins = listOf("https://shop.example.com")  // NOT *
            allowedMethods = listOf("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            allowedHeaders = listOf("Authorization", "Content-Type", "X-Requested-With")
            allowCredentials = true
            maxAge = 3600L
        }
        
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", config)
        }
    }
}
```

---

## 90.6 Secrets Management Best Practices

```kotlin
// ====== NEVER do this ======
class BadConfig {
    val apiKey = "sk-live-abc123secret"  // committed to git!
    val dbPassword = "password123"
}

// ====== Better: environment variables ======
class EnvConfig {
    val apiKey = System.getenv("STRIPE_API_KEY") 
        ?: throw IllegalStateException("STRIPE_API_KEY not set")
    val dbPassword = System.getenv("DB_PASSWORD")
        ?: throw IllegalStateException("DB_PASSWORD not set")
}

// ====== Best: Spring Cloud Vault ======
// Vault injects secrets as Spring properties
// @Value("${stripe.api-key}") comes from Vault, not env vars
// Vault rotates secrets automatically
// Audit log: who accessed what secret, when

// ====== AWS Secrets Manager ======
@Configuration
class SecretsManagerConfig {
    
    @Bean
    fun stripeApiKey(secretsManager: AWSSecretsManager): String {
        val request = GetSecretValueRequest()
            .withSecretId("production/shop/stripe-api-key")
        val result = secretsManager.getSecretValue(request)
        
        return result.secretString?.let {
            JSONObject(it).getString("api_key")
        } ?: throw IllegalStateException("Secret not found")
    }
}

// ====== Secret Rotation ======
// 1. Vault: built-in rotation for DB creds (dynamic secrets)
// 2. AWS: automatic rotation with Lambda function
// 3. K8s: External Secrets Operator syncs from Vault/AWS SM

// K8s ExternalSecret
/*
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: shop-api-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: shop-api-secrets  # K8s Secret name
  data:
    - secretKey: stripe-api-key
      remoteRef:
        key: production/shop
        property: stripe_api_key
    - secretKey: db-password
      remoteRef:
        key: production/db
        property: password
*/
```

---

## สรุป Part 90

```
Production Hardening:

Zero Trust:
  Verify every request (no implicit trust)
  Least privilege access
  Microsegmentation (NetworkPolicy)
  Service mesh (Istio) for mTLS

mTLS:
  Both client + server present certificates
  Service identity = certificate CN
  Certificate rotation: short-lived (24h)
  Vault PKI for automated issuance

Vault:
  Dynamic secrets: DB creds expire → less blast radius
  PKI: issue certificates automatically
  Audit log: who accessed what
  K8s auth: use service account (no long-lived tokens)

Security Headers (HTTP):
  HSTS: always HTTPS
  CSP: prevent XSS
  X-Frame-Options: prevent clickjacking
  CORS: allowedOrigins never use *

Secrets Management:
  Never in code or git
  Environment vars: ok for simple setups
  Vault/AWS SM: best for production
  Rotation: automate it

K8s Security:
  NetworkPolicy: L3/L4 isolation
  PeerAuthentication: L4/L7 mTLS
  AuthorizationPolicy: per-service rules
  RBAC: pod service account permissions
```

➡️ [Part 91: Microservices Design Patterns Advanced](./Part-91-MicroservicesAdvanced.md)
