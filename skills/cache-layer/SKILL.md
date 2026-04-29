---
name: cache-layer
description: "Add Spring Cache with per-cache TTL via YAML config. Caffeine for local/test, Redis for prod. Use when adding caching to service methods."
---

## Preconditions
`service-mapping` completed. Service layer compiles.

## Goal
Add caching with Spring Cache abstraction. Each cache has its own TTL configured via YAML. Use Caffeine for local/test, Redis for prod.

## Instructions

Create `package-info.java` with `@NullMarked` in every new package created below.

### 1. Dependencies

Add to `app/pom.xml`:
- `spring-boot-starter-cache`
- `com.github.ben-manes.caffeine:caffeine` (default / local / test)
- `spring-boot-starter-data-redis` (optional, prod profile)

### 2. Cache Configuration Properties

Create **`CacheProperties`** (`@ConfigurationProperties(prefix = "app.cache")`):
```java
@Getter
@Setter
public class CacheProperties {
    private Map<String, CacheSpec> caches = new LinkedHashMap<>();

    @Getter
    @Setter
    public static class CacheSpec {
        private Duration ttl = Duration.ofMinutes(10);
        private int maxSize = 500;
    }
}
```

### 3. YAML Configuration

Add to `application.yml`:
```yaml
app:
  cache:
    caches:
      person-by-id:
        ttl: PT30M
        max-size: 1000
      persons-page:
        ttl: PT5M
        max-size: 200
```

### 4. Caffeine Cache Manager (default)

Create **`CaffeineCacheConfig`** (`@Configuration`, `@EnableCaching`, `@Profile("!redis")`):
- `@RequiredArgsConstructor`, inject `CacheProperties`
- Bean `CacheManager caffeineCacheManager()`:
  - Build a `SimpleCacheManager` with individual `CaffeineCache` per entry in `caches` map
  - Each cache uses its own `Caffeine.newBuilder().expireAfterWrite(spec.ttl).maximumSize(spec.maxSize)`
  - This gives **per-cache TTL** — not a global TTL

```java
@Bean
public CacheManager cacheManager(CacheProperties props) {
    List<CaffeineCache> caches = props.getCaches().entrySet().stream()
        .map(entry -> new CaffeineCache(
            entry.getKey(),
            Caffeine.newBuilder()
                .expireAfterWrite(entry.getValue().getTtl())
                .maximumSize(entry.getValue().getMaxSize())
                .recordStats()
                .build()))
        .toList();

    SimpleCacheManager manager = new SimpleCacheManager();
    manager.setCaches(caches);
    return manager;
}
```

### 5. Redis Cache Manager (prod)

Create **`RedisCacheConfig`** (`@Configuration`, `@EnableCaching`, `@Profile("redis")`):
- Bean `RedisCacheManager` with per-cache `RedisCacheConfiguration`:
  - Iterate `CacheProperties.caches`, build `RedisCacheConfiguration.defaultCacheConfig().entryTtl(spec.ttl)` per entry
  - Use `RedisCacheManager.builder(connectionFactory).withInitialCacheConfigurations(configMap)`
  - JSON serialization: `GenericJackson2JsonRedisSerializer`

### 6. Cache Constants

Create **`CacheNames`** (final class with private constructor):
```java
public final class CacheNames {
    private CacheNames() {}

    public static final String PERSON_BY_ID = "person-by-id";
    public static final String PERSONS_PAGE = "persons-page";
}
```

### 7. Annotate Service Methods

Update **`PersonService`**:
- `getById(UUID id)` — `@Cacheable(value = PERSON_BY_ID, key = "#id")`
- `getAll(int page, int size, String sort)` — `@Cacheable(value = PERSONS_PAGE, key = "#page + '-' + #size + '-' + #sort")`
- `create(...)` — `@CacheEvict(value = PERSONS_PAGE, allEntries = true)`
- `update(UUID id, ...)` — `@CacheEvict(value = {PERSON_BY_ID, PERSONS_PAGE}, allEntries = true)` + or `@Caching` to evict specific key from PERSON_BY_ID and allEntries from PERSONS_PAGE
- `delete(UUID id)` — `@Caching(evict = {@CacheEvict(value = PERSON_BY_ID, key = "#id"), @CacheEvict(value = PERSONS_PAGE, allEntries = true)})`

### 8. Profile configs

- `application-local.yml` — Caffeine (default, no extra config)
- `application-prod.yml`:
  ```yaml
  spring:
    profiles:
      include: redis
    data:
      redis:
        host: ${REDIS_HOST:localhost}
        port: ${REDIS_PORT:6379}
  ```
- `application-test.yml` — Caffeine (fast, no external deps)

### 9. Actuator Cache Metrics (optional)

In `application.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,caches,metrics
```

## Output
- `CacheProperties.java`
- `CaffeineCacheConfig.java`
- `RedisCacheConfig.java`
- `CacheNames.java`
- Updated `PersonService.java` with cache annotations
- Updated `application.yml`, `application-prod.yml`

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile -pl app
```
