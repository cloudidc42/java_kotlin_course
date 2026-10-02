# Part 92: Advanced Spring Boot Configuration & Customization
## ขั้นตอนที่ 6341-6410: Auto-configuration, Conditional Beans, Custom Starters

---

## 92.1 Spring Boot Auto-configuration

```kotlin
// How Spring Boot auto-configuration works:
// 1. spring-boot-autoconfigure jar contains many @Configuration classes
// 2. Each has conditions (@ConditionalOn...)
// 3. When conditions met → beans registered automatically
// 4. Your beans override auto-configured beans

// Debug: see which auto-configs applied
// --debug flag or spring.boot.debug=true
// Shows: CONDITIONS EVALUATION REPORT

// Exclude auto-config
@SpringBootApplication(
    exclude = [
        DataSourceAutoConfiguration::class,  // manual datasource
        SecurityAutoConfiguration::class     // custom security
    ]
)
class Application

// application.yaml
/*
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
*/
```

---

## 92.2 Custom Auto-configuration

```kotlin
// Create a Spring Boot Starter library

// 1. Define configuration properties
@ConfigurationProperties(prefix = "audit")
@ConstructorBinding
data class AuditProperties(
    val enabled: Boolean = true,
    val includeRequestBody: Boolean = false,
    val includeResponseBody: Boolean = false,
    val excludePaths: List<String> = listOf("/actuator/**", "/health"),
    val storage: StorageType = StorageType.DATABASE,
    val asyncMode: Boolean = true
) {
    enum class StorageType { DATABASE, KAFKA, ELASTICSEARCH }
}

// 2. Create auto-configuration class
@AutoConfiguration
@ConditionalOnClass(AuditService::class)
@EnableConfigurationProperties(AuditProperties::class)
@Import(AuditRepositoryConfig::class)
class AuditAutoConfiguration(private val properties: AuditProperties) {
    
    @Bean
    @ConditionalOnMissingBean  // User can override by defining their own AuditService bean
    fun auditService(auditRepository: AuditRepository): AuditService =
        DefaultAuditService(auditRepository, properties)
    
    @Bean
    @ConditionalOnProperty(prefix = "audit", name = ["enabled"], havingValue = "true", matchIfMissing = true)
    @ConditionalOnBean(AuditService::class)
    fun auditFilter(auditService: AuditService): AuditFilter =
        AuditFilter(auditService, properties)
    
    @Bean
    @ConditionalOnClass(name = ["org.springframework.kafka.core.KafkaTemplate"])
    @ConditionalOnProperty(prefix = "audit.storage", name = ["type"], havingValue = "KAFKA")
    fun kafkaAuditRepository(kafkaTemplate: KafkaTemplate<String, String>): AuditRepository =
        KafkaAuditRepository(kafkaTemplate)
    
    @Bean
    @ConditionalOnProperty(prefix = "audit.storage", name = ["type"], havingValue = "DATABASE", matchIfMissing = true)
    fun jdbcAuditRepository(jdbcTemplate: JdbcTemplate): AuditRepository =
        JdbcAuditRepository(jdbcTemplate)
}

// 3. Register auto-configuration
// src/main/resources/META-INF/spring/
//   org.springframework.boot.autoconfigure.AutoConfiguration.imports
/*
com.example.audit.AuditAutoConfiguration
*/

// 4. Include in pom.xml
/*
<dependency>
    <groupId>com.example</groupId>
    <artifactId>audit-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>

Then in application.yaml:
audit:
  enabled: true
  include-request-body: true
  exclude-paths: /actuator/**, /public/**
  storage: DATABASE
*/
```

---

## 92.3 Conditional Bean Registration

```kotlin
// @ConditionalOn... annotations

@Configuration
class ConditionalExamples {
    
    // Only register if specific class is on classpath
    @Bean
    @ConditionalOnClass(name = ["com.google.cloud.storage.Storage"])
    fun gcsStorageService(): StorageService = GcsStorageService()
    
    // Register only if property has value
    @Bean
    @ConditionalOnProperty(prefix = "feature", name = ["new-checkout"], havingValue = "true")
    fun newCheckoutService(): CheckoutService = NewCheckoutService()
    
    // Register only if another bean exists
    @Bean
    @ConditionalOnBean(CacheManager::class)
    fun cacheableProductService(cacheManager: CacheManager): ProductService =
        CacheableProductService(cacheManager)
    
    // Register only if no other bean of type exists
    @Bean
    @ConditionalOnMissingBean(StorageService::class)
    fun localStorageService(): StorageService = LocalFileStorageService()
    
    // Custom condition
    @Bean
    @Conditional(ProductionEnvironmentCondition::class)
    fun productionOnlyBean(): ProductionConfig = ProductionConfig()
    
    // Expression-based (Spring Expression Language)
    @Bean
    @ConditionalOnExpression("\${app.feature.enabled} && '\${app.environment}' == 'production'")
    fun advancedFeature(): AdvancedFeatureService = AdvancedFeatureService()
}

// Custom condition
class ProductionEnvironmentCondition : Condition {
    override fun matches(context: ConditionContext, metadata: AnnotatedTypeMetadata): Boolean {
        val profiles = context.environment.activeProfiles
        return "production" in profiles
    }
}
```

---

## 92.4 Application Events & Lifecycle

```kotlin
// Spring Boot lifecycle events

@Component
class ApplicationLifecycleHandler {
    
    // Early startup: before context refreshed
    @EventListener
    fun onStarting(event: ApplicationStartingEvent) {
        println("Application starting...")
    }
    
    // Context ready but before beans activated
    @EventListener
    fun onEnvironmentPrepared(event: ApplicationEnvironmentPreparedEvent) {
        val profiles = event.environment.activeProfiles
        println("Active profiles: ${profiles.joinToString()}")
    }
    
    // Context fully initialized (all beans ready)
    @EventListener
    fun onContextRefreshed(event: ContextRefreshedEvent) {
        println("All beans initialized")
    }
    
    // Application ready to serve requests
    @EventListener
    fun onReady(event: ApplicationReadyEvent) {
        println("Application is ready!")
        // Good place to: warm up caches, connect to external systems, start consumers
    }
    
    // Bean initialization order control
    @Component
    @DependsOn("databaseConfig")  // ensure databaseConfig initialized first
    class OrderServiceInit(private val orderRepository: OrderRepository) : InitializingBean {
        override fun afterPropertiesSet() {
            // Called after bean fully initialized
            orderRepository.ensureIndexes()
        }
    }
    
    // Shutdown hook
    @PreDestroy
    fun onShutdown() {
        println("Application shutting down...")
        // Graceful: wait for in-flight requests, close connections
    }
}

// ApplicationRunner vs CommandLineRunner
@Component
@Order(1)  // run first
class DatabaseMigrationRunner(private val flyway: Flyway) : ApplicationRunner {
    override fun run(args: ApplicationArguments) {
        flyway.migrate()
        println("Database migrations complete")
    }
}

@Component
@Order(2)  // run second
class CacheWarmupRunner(
    private val productService: ProductService,
    private val cacheManager: CacheManager
) : CommandLineRunner {
    override fun run(vararg args: String) {
        println("Warming up product cache...")
        productService.findAll()  // populate cache
    }
}
```

---

## 92.5 Dynamic Configuration with @RefreshScope

```kotlin
// Spring Cloud Config: reload config without restart

// application.yaml
/*
spring:
  config:
    import: configserver:http://config-server:8888
  cloud:
    config:
      label: main
      fail-fast: true
*/

// Mark beans that should refresh when config changes
@Component
@RefreshScope
class FeatureFlagService(
    @Value("\${feature.new-checkout.enabled:false}") val newCheckoutEnabled: Boolean,
    @Value("\${feature.recommendation.model:v1}") val recommendationModel: String
) {
    fun isNewCheckoutEnabled() = newCheckoutEnabled
    fun getRecommendationModel() = recommendationModel
}

// Trigger refresh via Actuator
// POST /actuator/refresh
// → Refreshes all @RefreshScope beans with new config values

// Or use Spring Cloud Bus for broadcast refresh to all instances
// POST /actuator/busrefresh
// → Notifies all app instances via message broker

// Custom configuration refresh listener
@Component
class ConfigRefreshListener {
    
    @EventListener(RefreshScopeRefreshedEvent::class)
    fun onRefresh() {
        log.info("Configuration refreshed at ${java.time.Instant.now()}")
        // Clear caches that depend on configuration
    }
}

// Externalized configuration priority (highest to lowest):
// 1. Command line args: --server.port=8080
// 2. SPRING_APPLICATION_JSON env var
// 3. Java System properties: -Dserver.port=8080
// 4. OS environment variables: SERVER_PORT=8080
// 5. application-{profile}.yaml
// 6. application.yaml
// 7. @PropertySource annotated configs
// 8. Default properties
```

---

## 92.6 Custom Actuator Endpoint

```kotlin
// Custom Actuator endpoint
@Component
@Endpoint(id = "feature-flags")
class FeatureFlagEndpoint(private val featureFlagService: FeatureFlagService) {
    
    @ReadOperation
    fun getAllFlags(): Map<String, Any> =
        mapOf(
            "flags" to featureFlagService.getAllFlags(),
            "lastUpdated" to java.time.Instant.now()
        )
    
    @ReadOperation
    fun getFlag(@Selector flagName: String): Map<String, Any>? {
        val flag = featureFlagService.getFlag(flagName) ?: return null
        return mapOf(
            "name" to flagName,
            "enabled" to flag.enabled,
            "rolloutPercentage" to flag.rolloutPercentage
        )
    }
    
    @WriteOperation
    fun toggleFlag(@Selector flagName: String, @Body request: ToggleRequest) {
        featureFlagService.toggle(flagName, request.enabled)
    }
    
    data class ToggleRequest(val enabled: Boolean)
}

// Access: GET /actuator/feature-flags
//         GET /actuator/feature-flags/new-checkout
//         POST /actuator/feature-flags/new-checkout (body: {"enabled": true})

// Custom Health Indicator
@Component
class ExternalServiceHealthIndicator(
    private val externalService: ExternalPaymentService
) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            val ping = externalService.ping()
            if (ping.latencyMs < 1000) {
                Health.up()
                    .withDetail("latency", "${ping.latencyMs}ms")
                    .withDetail("version", ping.version)
                    .build()
            } else {
                Health.degraded()
                    .withDetail("latency", "${ping.latencyMs}ms (high)")
                    .build()
            }
        } catch (e: Exception) {
            Health.down()
                .withException(e)
                .build()
        }
    }
}
```

---

## สรุป Part 92

```
Spring Boot Configuration:

Auto-configuration:
  Conditional beans based on classpath, properties, existing beans
  @ConditionalOnClass, @ConditionalOnMissingBean
  @ConditionalOnProperty, @ConditionalOnExpression
  
  Creating Starter:
    1. @ConfigurationProperties for config
    2. @AutoConfiguration with conditions
    3. Register in AutoConfiguration.imports
    4. Package as spring-boot-starter-{name}

Lifecycle:
  ApplicationRunner / CommandLineRunner: run on startup
  @Order: control execution order
  @PreDestroy: cleanup on shutdown
  ContextRefreshedEvent: after all beans ready
  ApplicationReadyEvent: after app accepts requests

Dynamic Config:
  Spring Cloud Config: centralized config server
  @RefreshScope: beans reload on config change
  POST /actuator/refresh: trigger reload
  Spring Cloud Bus: broadcast refresh to all instances

Custom Actuator:
  @Endpoint(id = "name") = new endpoint at /actuator/name
  @ReadOperation = GET
  @WriteOperation = POST
  @DeleteOperation = DELETE
  @Selector = path variable

Config Priority:
  CLI args > System props > Env vars > application-{profile}.yaml > application.yaml

Best Practices:
  Use @ConfigurationProperties instead of @Value for groups
  Document config with spring-configuration-metadata.json
  Validate with @Validated + @Valid on @ConfigurationProperties
```

➡️ [Part 93: Java Records, Sealed Classes & Pattern Matching](./Part-93-ModernJava.md)
