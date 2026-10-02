# Part 66: Feature Flags & A/B Testing
## ขั้นตอนที่ 4521-4590: Gradual Rollout, Experimentation, Dark Launch

---

## 66.1 Feature Flags คืออะไร

```
Feature Flag (Feature Toggle) คือ:
  - สวิตช์ใน code ที่เปิด/ปิด feature ได้โดยไม่ต้อง deploy ใหม่
  - ควบคุมได้แบบ runtime จาก dashboard
  - Target specific users/groups/percentage

Use Cases:
  1. Gradual Rollout: เปิด feature ให้ 10% → 50% → 100% ของ users
  2. A/B Testing: แสดง version A ให้ 50%, version B ให้ 50%
  3. Kill Switch: ปิด feature ฉุกเฉินถ้ามี bug
  4. Dark Launch: deploy code แล้วซ่อนไว้ก่อน
  5. Beta Features: เปิดให้เฉพาะ beta users

Example Timeline:
  Sprint 1: Deploy code (flag OFF)
  Week 1:   Enable 5% users (observe metrics)
  Week 2:   Enable 25% users
  Week 3:   Enable 100% (flag always ON)
  Month 2:  Remove flag from code
```

---

## 66.2 Simple Feature Flag Implementation

```java
import org.springframework.stereotype.*;
import org.springframework.cache.annotation.*;

// Feature flag config stored in DB or properties
@Entity
@Table(name = "feature_flags")
class FeatureFlag {
    
    @Id
    private String name;
    private boolean enabled;
    private String description;
    private double rolloutPercentage;  // 0.0 - 1.0 (0% to 100%)
    private String targetUserIds;  // comma-separated user IDs (or null = all)
    private java.time.Instant updatedAt;
    
    // getters/setters
}

@Service
class FeatureFlagService {
    
    private final FeatureFlagRepository flagRepository;
    
    FeatureFlagService(FeatureFlagRepository flagRepository) {
        this.flagRepository = flagRepository;
    }
    
    // Check if feature is enabled (global)
    @Cacheable(value = "feature-flags", key = "#featureName")
    public boolean isEnabled(String featureName) {
        return flagRepository.findById(featureName)
            .map(FeatureFlag::isEnabled)
            .orElse(false);
    }
    
    // Check if feature is enabled for specific user
    public boolean isEnabledForUser(String featureName, String userId) {
        var flag = flagRepository.findById(featureName).orElse(null);
        if (flag == null || !flag.isEnabled()) return false;
        
        // Check whitelist
        if (flag.getTargetUserIds() != null) {
            return java.util.Arrays.asList(flag.getTargetUserIds().split(","))
                .contains(userId);
        }
        
        // Percentage rollout: hash userId to consistent bucket
        if (flag.getRolloutPercentage() < 1.0) {
            int bucket = Math.abs(userId.hashCode()) % 100;
            return bucket < (flag.getRolloutPercentage() * 100);
        }
        
        return true;
    }
    
    @CacheEvict(value = "feature-flags", key = "#featureName")
    public void updateFlag(String featureName, boolean enabled, double percentage) {
        var flag = flagRepository.findById(featureName)
            .orElse(new FeatureFlag());
        flag.setName(featureName);
        flag.setEnabled(enabled);
        flag.setRolloutPercentage(percentage);
        flag.setUpdatedAt(java.time.Instant.now());
        flagRepository.save(flag);
    }
}

// Usage in service
@Service
class ProductService {
    
    private final FeatureFlagService featureFlags;
    
    ProductService(FeatureFlagService featureFlags) {
        this.featureFlags = featureFlags;
    }
    
    public java.util.List<Product> getProducts(String userId) {
        var products = loadProductsFromDb();
        
        // New recommendation algorithm (gradual rollout)
        if (featureFlags.isEnabledForUser("new-recommendation-algo", userId)) {
            return applyNewRecommendationAlgorithm(products, userId);
        }
        
        return products;
    }
    
    public ProductSearchResult search(String query, String userId) {
        // AI-powered search (beta feature)
        if (featureFlags.isEnabledForUser("ai-search", userId)) {
            return searchWithAI(query);
        }
        return searchWithElasticsearch(query);
    }
    
    private java.util.List<Product> loadProductsFromDb() { return java.util.List.of(); }
    private java.util.List<Product> applyNewRecommendationAlgorithm(java.util.List<Product> p, String uid) { return p; }
    private ProductSearchResult searchWithAI(String q) { return new ProductSearchResult(java.util.List.of()); }
    private ProductSearchResult searchWithElasticsearch(String q) { return new ProductSearchResult(java.util.List.of()); }
    
    record ProductSearchResult(java.util.List<Product> items) {}
}
```

---

## 66.3 A/B Testing Framework

```java
// A/B Test: measure which variant performs better
@Service
class ABTestingService {
    
    private final FeatureFlagService featureFlags;
    private final ABTestMetricsRepository metricsRepository;
    
    ABTestingService(FeatureFlagService featureFlags,
                     ABTestMetricsRepository metricsRepository) {
        this.featureFlags = featureFlags;
        this.metricsRepository = metricsRepository;
    }
    
    // Assign user to a variant (consistent per user)
    public String getVariant(String testName, String userId, String... variants) {
        if (!featureFlags.isEnabled(testName)) {
            return variants[0];  // control variant
        }
        
        int bucket = Math.abs((testName + userId).hashCode()) % variants.length;
        return variants[bucket];
    }
    
    // Track experiment event
    public void trackEvent(String testName, String userId, String variant, String event) {
        metricsRepository.save(new ABTestEvent(
            testName, userId, variant, event, java.time.Instant.now()
        ));
    }
    
    // Get conversion rates per variant
    public ABTestResults getResults(String testName) {
        var events = metricsRepository.findByTestName(testName);
        
        var variantStats = events.stream()
            .collect(java.util.stream.Collectors.groupingBy(
                ABTestEvent::variant,
                java.util.stream.Collectors.toList()
            ));
        
        var results = variantStats.entrySet().stream()
            .collect(java.util.stream.Collectors.toMap(
                java.util.Map.Entry::getKey,
                e -> {
                    long exposures = e.getValue().stream()
                        .filter(ev -> "exposure".equals(ev.event())).count();
                    long conversions = e.getValue().stream()
                        .filter(ev -> "conversion".equals(ev.event())).count();
                    double rate = exposures > 0 ? (double) conversions / exposures : 0;
                    return new VariantStats(exposures, conversions, rate);
                }
            ));
        
        return new ABTestResults(testName, results);
    }
    
    record ABTestEvent(String testName, String userId, String variant,
                       String event, java.time.Instant timestamp) {}
    record VariantStats(long exposures, long conversions, double conversionRate) {}
    record ABTestResults(String testName, java.util.Map<String, VariantStats> variants) {}
}

// Use in controller
@RestController
class CheckoutController {
    
    private final ABTestingService abTesting;
    private final FeatureFlagService featureFlags;
    
    CheckoutController(ABTestingService abTesting, FeatureFlagService featureFlags) {
        this.abTesting = abTesting;
        this.featureFlags = featureFlags;
    }
    
    @GetMapping("/api/checkout/button-config")
    CheckoutButtonConfig getButtonConfig(
            @AuthenticationPrincipal org.springframework.security.core.userdetails.UserDetails user) {
        
        String userId = user.getUsername();
        
        // A/B test: checkout button color
        String variant = abTesting.getVariant("checkout-button-color", userId,
            "control", "green", "orange");
        
        // Track exposure
        abTesting.trackEvent("checkout-button-color", userId, variant, "exposure");
        
        return switch (variant) {
            case "green"  -> new CheckoutButtonConfig("green", "สั่งซื้อเลย!", variant);
            case "orange" -> new CheckoutButtonConfig("orange", "ซื้อตอนนี้", variant);
            default       -> new CheckoutButtonConfig("blue", "ชำระเงิน", variant);
        };
    }
    
    @PostMapping("/api/checkout/complete")
    void checkout(@RequestBody CheckoutRequest request,
                  @AuthenticationPrincipal org.springframework.security.core.userdetails.UserDetails user) {
        String userId = user.getUsername();
        
        // Track conversion for all active experiments
        abTesting.trackEvent("checkout-button-color", userId,
            abTesting.getVariant("checkout-button-color", userId, "control", "green", "orange"),
            "conversion");
        
        // ... process checkout
    }
    
    record CheckoutButtonConfig(String color, String label, String variant) {}
    record CheckoutRequest(String cartId, String paymentMethod) {}
}
```

---

## 66.4 GrowthBook Integration (Open Source)

```java
// GrowthBook: open-source feature flag & A/B testing platform

// SDK setup
// Maven: com.github.growthbook:growthbook-sdk-java:1.1.0

import com.sdk.growthbook.*;

@org.springframework.context.annotation.Configuration
class GrowthBookConfig {
    
    @org.springframework.context.annotation.Bean
    GBContext growthBookContext(@Value("${growthbook.api-host}") String apiHost,
                                @Value("${growthbook.client-key}") String clientKey) {
        // Load features from GrowthBook API
        var featuresJson = fetchFeaturesFromGrowthBook(apiHost, clientKey);
        
        return new GBContext.Builder()
            .setFeaturesJson(featuresJson)
            .build();
    }
    
    private String fetchFeaturesFromGrowthBook(String host, String key) {
        // HTTP call to GrowthBook API: GET /api/features/{key}
        return "{}"; // placeholder
    }
}

@Service
class GrowthBookFeatureService {
    
    private final GBContext gbContext;
    
    GrowthBookFeatureService(GBContext gbContext) {
        this.gbContext = gbContext;
    }
    
    public boolean isFeatureOn(String feature, String userId) {
        var userContext = new GBContext.Builder()
            .setFeaturesJson(gbContext.getFeaturesJson())
            .setAttributes(new com.google.gson.JsonObject() {{
                addProperty("id", userId);
            }})
            .build();
        
        var gb = new GrowthBook(userContext);
        return gb.isOn(feature);
    }
    
    public <T> T getFeatureValue(String feature, String userId, T defaultValue) {
        var userContext = new GBContext.Builder()
            .setFeaturesJson(gbContext.getFeaturesJson())
            .setAttributes(new com.google.gson.JsonObject() {{
                addProperty("id", userId);
            }})
            .build();
        
        var gb = new GrowthBook(userContext);
        return gb.getFeatureValue(feature, defaultValue);
    }
}
```

---

## 66.5 Feature Flag Admin API

```java
@RestController
@RequestMapping("/api/admin/features")
@org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
class FeatureFlagAdminController {
    
    private final FeatureFlagService featureFlagService;
    
    FeatureFlagAdminController(FeatureFlagService featureFlagService) {
        this.featureFlagService = featureFlagService;
    }
    
    @GetMapping
    java.util.List<FeatureFlag> getAllFlags() {
        return featureFlagService.getAllFlags();
    }
    
    // Toggle feature on/off
    @PostMapping("/{name}/toggle")
    FeatureFlag toggle(@PathVariable String name) {
        return featureFlagService.toggleFlag(name);
    }
    
    // Update rollout percentage
    @PutMapping("/{name}/rollout")
    FeatureFlag updateRollout(@PathVariable String name,
                               @RequestBody RolloutRequest request) {
        featureFlagService.updateFlag(name, true, request.percentage());
        return featureFlagService.getFlag(name);
    }
    
    // Add user to whitelist
    @PostMapping("/{name}/users/{userId}")
    void addUserToWhitelist(@PathVariable String name, @PathVariable String userId) {
        featureFlagService.addUserToWhitelist(name, userId);
    }
    
    record RolloutRequest(double percentage) {}
}
```

---

## สรุป Part 66

```
Feature Flags Best Practices:

Types:
  Release flag   = temporary, remove after full rollout
  Experiment     = A/B test, remove after decision
  Ops flag       = kill switch, keep indefinitely
  Permission     = per user/role, keep indefinitely

Rollout Strategy:
  1% → 5% → 25% → 50% → 100% → Remove flag

Consistent Bucketing (same user always gets same variant):
  bucket = abs(hash(testName + userId)) % 100
  
A/B Test Significance:
  Need: 1,000+ exposures per variant for statistical significance
  Track: conversion rate, revenue per user, bounce rate
  Use: Chi-squared test to determine winner

Technical Rules:
  ✓ Max 2-3 flag checks per request path
  ✓ Default to OFF (fail safe)
  ✓ Monitor performance impact of new flags
  ✗ Don't use flag for permanent config (use config files)
  ✗ Never leave old flags in code > 3 months

Tools:
  DIY: DB table + Spring Cache + REST API
  Open Source: GrowthBook, Unleash
  SaaS: LaunchDarkly, Split.io, Flagsmith
```

➡️ [Part 67: Zero-Downtime Database Migrations](./Part-67-DatabaseMigrations.md)
