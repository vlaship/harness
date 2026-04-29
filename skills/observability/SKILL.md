---
name: observability
description: "Add Micrometer Tracing (OTel bridge) and Prometheus metrics. Covers traceId/spanId in logs and response headers, custom business metrics, MeterBinder, and health indicators."
---

## Preconditions
`security-layer` completed. Full project compiles.

## Goal
Phase 1 — distributed tracing with `traceId`/`spanId` propagation through logs, HTTP response headers, and MDC context.
Phase 2 — Actuator endpoints, Prometheus export, custom business metrics, and health checks.

## Instructions

Create `package-info.java` with `@NullMarked` in every new package created below.

---

## Phase 1: Distributed Tracing

### 1. Dependencies

Add to `app/pom.xml`:
- `io.micrometer:micrometer-tracing-bridge-otel` — OpenTelemetry bridge for Micrometer Tracing
- `io.opentelemetry:opentelemetry-exporter-otlp` — OTLP exporter (prod, optional)
- `io.opentelemetry:opentelemetry-exporter-logging` — logging exporter (local/test fallback)

Spring Boot auto-configures Micrometer Tracing — no manual bean wiring needed.

### 2. Application Configuration

Add to `application.yml`:
```yaml
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0   # 100% in dev, lower in prod
  endpoints:
    web:
      exposure:
        include: health,info,caches,metrics,httpexchanges

logging:
  pattern:
    correlation: "[${spring.application.name:},%mdc{traceId:-},%mdc{spanId:-}] "
```

`application-prod.yml`:
```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # 10% sampling in prod
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318/v1/traces}
```

`application-local.yml`:
```yaml
management:
  tracing:
    sampling:
      probability: 1.0
```

### 3. Logback Configuration

Create `app/src/main/resources/logback-spring.xml`:
```xml
<configuration>
  <springProperty scope="context" name="appName"
                  source="spring.application.name" defaultValue="sample-app"/>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{traceId:-},%X{spanId:-}] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <!-- JSON format for prod -->
  <springProfile name="prod">
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="JSON"/>
    </root>
  </springProfile>

  <springProfile name="!prod">
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>
</configuration>
```

If using Logstash encoder for prod JSON logging, add dependency:
- `net.logstash.logback:logstash-logback-encoder` (optional, prod only)

If you prefer to keep it simple, skip Logstash encoder and use the same `CONSOLE` pattern for all profiles — traceId/spanId will still appear via `%X{traceId}`.

### 4. TraceId Response Header Filter

Create **`TraceIdResponseFilter`** (`@Component`, `@Order(Ordered.HIGHEST_PRECEDENCE)`):
```java
@Component
@RequiredArgsConstructor
public class TraceIdResponseFilter extends OncePerRequestFilter {

    private final Tracer tracer;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {

        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            String traceId = currentSpan.context().traceId();
            String spanId = currentSpan.context().spanId();
            response.setHeader("X-Trace-Id", traceId);
            response.setHeader("X-Span-Id", spanId);
        }

        filterChain.doFilter(request, response);
    }
}
```

This lets clients (and IntelliJ HTTP Client) see the `traceId` in response headers for debugging.

### 5. ProblemDetail Trace Enrichment

Update **`GlobalExceptionHandler`** — include `traceId` in every ProblemDetail response:
```java
@Autowired
private Tracer tracer;

private void enrichWithTraceId(ProblemDetail problem) {
    Span span = tracer.currentSpan();
    if (span != null) {
        problem.setProperty("traceId", span.context().traceId());
    }
}
```

Call `enrichWithTraceId(problem)` in each `@ExceptionHandler` before returning. This allows clients to reference `traceId` in error reports.

### 6. Security Config Update

In `SecurityConfig`, ensure the `TraceIdResponseFilter` runs in the filter chain. No special config needed if it's a `@Component` — Spring picks it up. Just verify it's not blocked by security filters.

### 7. Correlation in RestClient / WebClient (if using)

If `client` module uses `RestClient` or `WebClient` to call external services, ensure tracing propagation:
- `RestClient`: auto-instrumented by Micrometer if built via `RestClient.builder()`
- `WebClient`: auto-instrumented if built via `WebClient.builder()`
- W3C `traceparent` header is propagated automatically by OpenTelemetry

---

## Phase 2: Metrics & Prometheus

### 1. Dependencies

Add to `app/pom.xml`:
- `spring-boot-starter-actuator` (if not already present)
- `io.micrometer:micrometer-registry-prometheus` — Prometheus exporter

### 2. Actuator Configuration

Update `application.yml` (extend the `management:` block added in Phase 1):
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,caches,httpexchanges
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true   # Kubernetes liveness/readiness
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name:sample-app}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      sla:
        http.server.requests: 50ms,100ms,200ms,500ms,1s
```

`application-local.yml`:
```yaml
management:
  endpoint:
    health:
      show-details: always
```

`application-prod.yml`:
```yaml
management:
  endpoint:
    health:
      show-details: when-authorized
```

### 3. Security Config Update

In `SecurityConfig`, permit metrics endpoints:
```java
// Add to existing permitAll rules:
.requestMatchers("/actuator/health/**", "/actuator/prometheus", "/actuator/info").permitAll()
```

Keep `/actuator/metrics`, `/actuator/caches` behind authentication — they expose internals.

### 4. Custom Business Metrics

Create **`PersonMetrics`** (`app/src/main/java/com/example/sample/metrics/`):
- `@Component`, `@RequiredArgsConstructor`
- Implements `MeterBinder` for self-registration
- Metrics:
  ```java
  @Component
  @RequiredArgsConstructor
  public class PersonMetrics implements MeterBinder {

      private Counter personsCreated;
      private Counter personsUpdated;
      private Counter personsDeleted;
      private final AtomicLong totalPersons = new AtomicLong(0);

      @Override
      public void bindTo(MeterRegistry registry) {
          personsCreated = Counter.builder("persons.created")
              .description("Total persons created")
              .register(registry);

          personsUpdated = Counter.builder("persons.updated")
              .description("Total persons updated")
              .register(registry);

          personsDeleted = Counter.builder("persons.deleted")
              .description("Total persons deleted")
              .register(registry);

          Gauge.builder("persons.total", totalPersons, AtomicLong::get)
              .description("Current total number of persons")
              .register(registry);
      }

      public void recordCreated() { personsCreated.increment(); }
      public void recordUpdated() { personsUpdated.increment(); }
      public void recordDeleted() { personsDeleted.increment(); }
      public void setTotal(long count) { totalPersons.set(count); }
  }
  ```

### 5. Wire Metrics into Service

Update **`PersonService`** — inject `PersonMetrics`:
- `create()` → call `personMetrics.recordCreated()` after successful save
- `update()` → call `personMetrics.recordUpdated()` after successful save
- `delete()` → call `personMetrics.recordDeleted()` after successful delete
- On application startup, initialize `personMetrics.setTotal(personRepository.count())`
  via `@EventListener(ApplicationReadyEvent.class)` or `@PostConstruct`

### 6. Custom Health Indicator

Create **`DatabaseHealthIndicator`** (`app/src/main/java/com/example/sample/health/`):
- `@Component`
- Extends `AbstractHealthIndicator`
- Checks database connectivity with a simple query (`SELECT 1`)
- Reports `UP` with details: `database: PostgreSQL`, `schema: sample`
- Reports `DOWN` on exception with error message

```java
@Component
@RequiredArgsConstructor
public class DatabaseHealthIndicator extends AbstractHealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        jdbcTemplate.queryForObject("SELECT 1", Integer.class);
        builder.up()
            .withDetail("database", "PostgreSQL")
            .withDetail("schema", "sample");
    }
}
```

### 7. Request Timing (auto-configured)

Spring Boot auto-configures `http.server.requests` timer via Micrometer. No manual code needed.
The `percentiles-histogram` and `sla` config in step 2 adds histogram buckets for:
- p50, p95, p99 latency
- SLA buckets: 50ms, 100ms, 200ms, 500ms, 1s

Verify these appear at `/actuator/prometheus`:
```
http_server_requests_seconds_bucket{...,le="0.05"} 
http_server_requests_seconds_bucket{...,le="0.1"}
```

### 8. Cache Metrics

Caffeine cache already has `.recordStats()` from `cache-layer` skill.
Spring Boot auto-exposes cache metrics when Actuator is present:
- `cache.gets{cache=person-by-id,result=hit}`
- `cache.gets{cache=person-by-id,result=miss}`
- `cache.size{cache=person-by-id}`
- `cache.evictions{cache=person-by-id}`

No extra code needed — just verify they appear in `/actuator/prometheus`.

---

## Output
- Updated `app/pom.xml` with tracing and Prometheus dependencies
- Updated `application.yml`, `application-local.yml`, `application-prod.yml`
- `logback-spring.xml`
- `TraceIdResponseFilter.java`
- Updated `GlobalExceptionHandler.java` with traceId in ProblemDetail
- Updated `SecurityConfig` with actuator endpoint rules
- `PersonMetrics.java` in `metrics/`
- Updated `PersonService.java` with metrics calls
- `DatabaseHealthIndicator.java` in `health/`
- `package-info.java` in all new packages

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile
```
Start the app and verify:
- `GET /actuator/health` → `{"status":"UP"}`
- `GET /actuator/prometheus` → Prometheus text format with `persons_created_total`, `http_server_requests_seconds`, `cache_gets_total`
