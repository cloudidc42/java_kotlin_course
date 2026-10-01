# Part 38: Redis Advanced
## ขั้นตอนที่ 2561-2630: Caching, Pub/Sub, Distributed Locks

---

## 38.1 Redis Overview

```
Redis = Remote Dictionary Server
  - In-memory key-value store
  - Persistence options: RDB snapshots, AOF append-only log
  - Data structures: String, Hash, List, Set, Sorted Set, Stream, Bitmap

Use cases:
  ✓ Caching (most common)
  ✓ Session store
  ✓ Pub/Sub messaging
  ✓ Rate limiting
  ✓ Distributed locks
  ✓ Leaderboards (Sorted Sets)
  ✓ Real-time analytics
  ✓ Message queues (Streams)
```

---

## 38.2 Spring Boot + Redis Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD:}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 20
          max-idle: 10
          min-idle: 2
          max-wait: 1000ms
  
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default
      cache-null-values: false
```

```java
import org.springframework.context.annotation.*;
import org.springframework.data.redis.cache.*;
import org.springframework.data.redis.connection.*;
import org.springframework.data.redis.core.*;
import org.springframework.data.redis.serializer.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.time.Duration;

@Configuration
@org.springframework.cache.annotation.EnableCaching
public class RedisConfig {
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Keys: String
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Values: JSON
        var jsonSerializer = new GenericJackson2JsonRedisSerializer();
        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory,
                                           ObjectMapper objectMapper) {
        var jsonSerializer = new GenericJackson2JsonRedisSerializer(objectMapper);
        
        var config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(jsonSerializer))
            .disableCachingNullValues();
        
        // Per-cache TTL configuration
        var cacheConfigs = new java.util.HashMap<String, RedisCacheConfiguration>();
        cacheConfigs.put("users", config.entryTtl(Duration.ofMinutes(30)));
        cacheConfigs.put("products", config.entryTtl(Duration.ofHours(1)));
        cacheConfigs.put("sessions", config.entryTtl(Duration.ofHours(24)));
        cacheConfigs.put("rate-limits", config.entryTtl(Duration.ofMinutes(1)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}
```

---

## 38.3 Caching with @Cacheable

```java
import org.springframework.cache.annotation.*;
import org.springframework.stereotype.*;

@Service
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Cacheable(value = "users", key = "#id")
    public UserDTO findById(Long id) {
        return userRepository.findById(id)
            .map(this::toDTO)
            .orElseThrow(() -> new RuntimeException("User not found: " + id));
    }
    
    @Cacheable(value = "users", key = "#email",
               condition = "#email != null",
               unless = "#result == null")
    public UserDTO findByEmail(String email) {
        return userRepository.findByEmail(email)
            .map(this::toDTO)
            .orElse(null);
    }
    
    @CachePut(value = "users", key = "#result.id")
    public UserDTO updateUser(Long id, UpdateUserRequest req) {
        User user = userRepository.findById(id)
            .orElseThrow();
        user.setName(req.name());
        user.setEmail(req.email());
        return toDTO(userRepository.save(user));
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    
    @CacheEvict(value = "users", allEntries = true)
    public void clearAllUserCache() {
        // Evict entire cache
    }
    
    @Caching(
        cacheable = @Cacheable(value = "users", key = "#username"),
        put = @CachePut(value = "users", key = "#result.id")
    )
    public UserDTO findByUsername(String username) {
        return userRepository.findByUsername(username)
            .map(this::toDTO)
            .orElseThrow();
    }
    
    private UserDTO toDTO(User u) {
        return new UserDTO(u.getId(), u.getName(), u.getEmail());
    }
}
```

---

## 38.4 RedisTemplate Operations

```java
import org.springframework.data.redis.core.*;
import org.springframework.stereotype.*;
import java.time.Duration;
import java.util.*;

@Service
public class RedisService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;
    
    public RedisService(RedisTemplate<String, Object> redisTemplate,
                        StringRedisTemplate stringRedisTemplate) {
        this.redisTemplate = redisTemplate;
        this.stringRedisTemplate = stringRedisTemplate;
    }
    
    // ====== String operations ======
    public void setString(String key, String value, Duration ttl) {
        stringRedisTemplate.opsForValue().set(key, value, ttl);
    }
    
    public Optional<String> getString(String key) {
        return Optional.ofNullable(stringRedisTemplate.opsForValue().get(key));
    }
    
    public long increment(String key) {
        return stringRedisTemplate.opsForValue().increment(key);
    }
    
    // ====== Hash operations (like Map) ======
    public void setHashField(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
    
    public Object getHashField(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }
    
    public Map<Object, Object> getAllHash(String key) {
        return redisTemplate.opsForHash().entries(key);
    }
    
    // ====== List operations ======
    public void pushToList(String key, Object... values) {
        redisTemplate.opsForList().rightPushAll(key, values);
    }
    
    public Object popFromList(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }
    
    public List<Object> getListRange(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }
    
    // ====== Set operations ======
    public void addToSet(String key, Object... values) {
        redisTemplate.opsForSet().add(key, values);
    }
    
    public boolean isMember(String key, Object value) {
        return Boolean.TRUE.equals(redisTemplate.opsForSet().isMember(key, value));
    }
    
    public Set<Object> getSetMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }
    
    // ====== Sorted Set (Leaderboard) ======
    public void addToLeaderboard(String board, String player, double score) {
        redisTemplate.opsForZSet().add(board, player, score);
    }
    
    public void incrementScore(String board, String player, double delta) {
        redisTemplate.opsForZSet().incrementScore(board, player, delta);
    }
    
    public Set<Object> getTopPlayers(String board, int count) {
        return redisTemplate.opsForZSet().reverseRange(board, 0, count - 1);
    }
    
    public Long getPlayerRank(String board, String player) {
        Long rank = redisTemplate.opsForZSet().reverseRank(board, player);
        return rank != null ? rank + 1 : null;  // 1-indexed
    }
    
    // ====== Key management ======
    public boolean exists(String key) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(key));
    }
    
    public void delete(String key) {
        redisTemplate.delete(key);
    }
    
    public void expire(String key, Duration ttl) {
        redisTemplate.expire(key, ttl);
    }
    
    public Duration getTimeToLive(String key) {
        Long ttl = redisTemplate.getExpire(key, java.util.concurrent.TimeUnit.MILLISECONDS);
        return ttl != null && ttl > 0 ? Duration.ofMillis(ttl) : null;
    }
}
```

---

## 38.5 Distributed Lock

```java
import org.springframework.data.redis.core.*;
import org.springframework.stereotype.*;
import java.time.Duration;
import java.util.UUID;
import java.util.function.Supplier;

@Component
public class DistributedLockService {
    
    private final StringRedisTemplate redisTemplate;
    
    public DistributedLockService(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    // Execute with lock (auto-release)
    public <T> T withLock(String lockKey, Duration timeout, Supplier<T> action) {
        String lockValue = UUID.randomUUID().toString();
        String fullKey = "lock:" + lockKey;
        
        boolean acquired = Boolean.TRUE.equals(
            redisTemplate.opsForValue()
                .setIfAbsent(fullKey, lockValue, timeout)
        );
        
        if (!acquired) {
            throw new RuntimeException("Could not acquire lock: " + lockKey);
        }
        
        try {
            return action.get();
        } finally {
            releaseLock(fullKey, lockValue);
        }
    }
    
    // Release lock only if we own it (atomic via Lua script)
    private void releaseLock(String key, String value) {
        String luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        
        redisTemplate.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            value
        );
    }
    
    // Try lock with retry
    public <T> T withLockRetry(String lockKey, Duration timeout, int maxRetries,
                                Supplier<T> action) {
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return withLock(lockKey, timeout, action);
            } catch (RuntimeException e) {
                if (attempt == maxRetries) throw e;
                try {
                    Thread.sleep(100L * (attempt + 1));
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(ie);
                }
            }
        }
        throw new RuntimeException("Should not reach here");
    }
}

// Usage example
@Service
class InventoryService {
    
    private final DistributedLockService lockService;
    private final ProductRepository productRepository;
    
    InventoryService(DistributedLockService lockService, 
                     ProductRepository productRepository) {
        this.lockService = lockService;
        this.productRepository = productRepository;
    }
    
    public boolean reserveItem(Long productId, int quantity) {
        return lockService.withLock("inventory:" + productId, Duration.ofSeconds(5), () -> {
            Product product = productRepository.findById(productId).orElseThrow();
            if (product.getStock() >= quantity) {
                product.setStock(product.getStock() - quantity);
                productRepository.save(product);
                return true;
            }
            return false;
        });
    }
}
```

---

## 38.6 Rate Limiting with Redis

```java
import org.springframework.data.redis.core.*;
import org.springframework.stereotype.*;
import java.time.Duration;

@Component
public class RateLimiter {
    
    private final StringRedisTemplate redisTemplate;
    
    public RateLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    // Sliding window rate limiter
    // Limit: maxRequests per windowSize
    public boolean isAllowed(String identifier, int maxRequests, Duration window) {
        String key = "rate:" + identifier;
        long now = System.currentTimeMillis();
        long windowMs = window.toMillis();
        
        // Atomic Lua script: add current timestamp, remove old, count
        String luaScript = """
            local key = KEYS[1]
            local now = tonumber(ARGV[1])
            local window = tonumber(ARGV[2])
            local max = tonumber(ARGV[3])
            
            -- Remove timestamps outside window
            redis.call('zremrangebyscore', key, 0, now - window)
            
            -- Count current requests
            local count = redis.call('zcard', key)
            
            if count < max then
                -- Add this request
                redis.call('zadd', key, now, now)
                redis.call('expire', key, math.ceil(window / 1000))
                return 1
            else
                return 0
            end
            """;
        
        Long result = redisTemplate.execute(
            new org.springframework.data.redis.core.script.DefaultRedisScript<>(luaScript, Long.class),
            List.of(key),
            String.valueOf(now),
            String.valueOf(windowMs),
            String.valueOf(maxRequests)
        );
        
        return Long.valueOf(1).equals(result);
    }
    
    // Fixed window rate limiter (simpler)
    public boolean isAllowedFixed(String identifier, int maxRequests, Duration window) {
        String key = "rate:fixed:" + identifier + ":" + 
            (System.currentTimeMillis() / window.toMillis());
        
        Long count = redisTemplate.opsForValue().increment(key);
        
        if (count == 1) {
            redisTemplate.expire(key, window);
        }
        
        return count <= maxRequests;
    }
}

// Rate limiting filter
@org.springframework.stereotype.Component
class RateLimitFilter implements jakarta.servlet.Filter {
    
    private final RateLimiter rateLimiter;
    
    RateLimitFilter(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }
    
    @Override
    public void doFilter(jakarta.servlet.ServletRequest request,
                         jakarta.servlet.ServletResponse response,
                         jakarta.servlet.FilterChain chain) 
            throws java.io.IOException, jakarta.servlet.ServletException {
        
        var httpReq = (jakarta.servlet.http.HttpServletRequest) request;
        var httpRes = (jakarta.servlet.http.HttpServletResponse) response;
        
        String ip = httpReq.getRemoteAddr();
        String identifier = ip;
        
        // 100 requests per minute per IP
        if (!rateLimiter.isAllowed(identifier, 100, Duration.ofMinutes(1))) {
            httpRes.setStatus(429);
            httpRes.setHeader("Retry-After", "60");
            httpRes.getWriter().write("{\"error\": \"Too Many Requests\"}");
            return;
        }
        
        chain.doFilter(request, response);
    }
}
```

---

## 38.7 Pub/Sub Messaging

```java
import org.springframework.data.redis.connection.*;
import org.springframework.data.redis.core.*;
import org.springframework.data.redis.listener.*;
import org.springframework.stereotype.*;

// ====== Publisher ======
@Service
public class RedisPublisher {
    
    private final RedisTemplate<String, Object> redisTemplate;
    
    public RedisPublisher(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }
    
    public void publish(String channel, Object message) {
        redisTemplate.convertAndSend(channel, message);
        System.out.println("Published to [" + channel + "]: " + message);
    }
    
    public void publishOrderUpdate(String orderId, String status) {
        publish("orders.updates", new java.util.Map.Entry<String, String>() {
            public String getKey() { return orderId; }
            public String getValue() { return status; }
            public String setValue(String v) { return null; }
        });
    }
}

// ====== Subscriber ======
@Component
public class RedisSubscriber {
    
    @org.springframework.data.redis.listener.adapter.RedisMessageListenerContainer
    public void onMessage(Message message, byte[] pattern) {
        // Used via MessageListenerAdapter
    }
    
    // Actual handler method (via adapter)
    public void handleMessage(String message, String channel) {
        System.out.printf("[Redis PubSub] channel=%s msg=%s%n", channel, message);
        processMessage(channel, message);
    }
    
    private void processMessage(String channel, String message) {
        switch (channel) {
            case "orders.updates" -> System.out.println("Order update: " + message);
            case "notifications" -> System.out.println("Notification: " + message);
            default -> System.out.println("Unknown channel: " + channel);
        }
    }
}

// ====== Redis Listener Config ======
@org.springframework.context.annotation.Configuration
class RedisListenerConfig {
    
    @org.springframework.context.annotation.Bean
    public RedisMessageListenerContainer redisContainer(
            RedisConnectionFactory connectionFactory,
            RedisSubscriber subscriber) {
        
        var container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        
        var adapter = new org.springframework.data.redis.listener.adapter.MessageListenerAdapter(
            subscriber, "handleMessage"
        );
        
        container.addMessageListener(adapter, new ChannelTopic("orders.updates"));
        container.addMessageListener(adapter, new ChannelTopic("notifications"));
        container.addMessageListener(adapter, new PatternTopic("events.*")); // wildcard
        
        return container;
    }
}
```

---

## 38.8 Redis Streams (Advanced Queue)

```java
import org.springframework.data.redis.connection.stream.*;
import org.springframework.data.redis.core.*;
import org.springframework.stereotype.*;
import java.time.Duration;
import java.util.*;

@Service
public class RedisStreamService {
    
    private final StreamOperations<String, String, String> streamOps;
    
    public RedisStreamService(RedisTemplate<String, String> redisTemplate) {
        this.streamOps = redisTemplate.opsForStream();
    }
    
    // Produce message to stream
    public String produce(String streamKey, Map<String, String> payload) {
        RecordId id = streamOps.add(
            MapRecord.create(streamKey, payload)
        );
        System.out.println("Produced to stream: " + id);
        return id.getValue();
    }
    
    // Read from consumer group
    public List<MapRecord<String, Object, Object>> consume(
            String streamKey, String groupName, String consumerName, int count) {
        
        return streamOps.read(
            Consumer.from(groupName, consumerName),
            StreamReadOptions.empty().count(count).block(Duration.ofSeconds(2)),
            StreamOffset.create(streamKey, ReadOffset.lastConsumed())
        );
    }
    
    // Acknowledge processed
    public void acknowledge(String streamKey, String groupName, RecordId... ids) {
        streamOps.acknowledge(streamKey, groupName, ids);
    }
    
    // Create consumer group (once)
    public void createConsumerGroup(String streamKey, String groupName) {
        try {
            streamOps.createGroup(streamKey, ReadOffset.from("0"), groupName);
        } catch (Exception e) {
            // Group already exists
        }
    }
    
    // Trim old messages (keep last 1000)
    public void trimStream(String streamKey, long maxLen) {
        streamOps.trim(streamKey, maxLen);
    }
}
```

---

## สรุป Part 38

| Feature | Use case |
|---------|---------|
| `@Cacheable` | Cache method results |
| `@CacheEvict` | Invalidate cache |
| `RedisTemplate` | Direct Redis operations |
| Sorted Set | Leaderboards, rankings |
| Distributed Lock | Prevent race conditions |
| Pub/Sub | Real-time notifications |
| Rate Limiter | API throttling |
| Streams | Durable message queue |

**Redis Data Structure Cheatsheet:**
```
String    → counters, sessions, simple values
Hash      → objects (user profile)
List      → queues, timelines (ordered)
Set       → tags, unique visitors
Sorted Set → leaderboards, rankings
Stream    → event log, message queue
```

➡️ [Part 39: Database Design & Optimization](./Part-39-Database.md)
