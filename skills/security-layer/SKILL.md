---
name: security-layer
description: "Add Spring Security OAuth2 Resource Server with self-signed RSA JWT. Token issuer + resource server in one app. Use when adding authentication."
---

## Preconditions
`rest-controller` completed. Full project compiles. All endpoints work without security.

## Goal
Add Spring Security OAuth2 Resource Server with self-signed RSA JWT tokens. The application acts as both **token issuer** and **resource server**.

## Instructions

Create `package-info.java` with `@NullMarked` in every new package created below.

### 1. Dependencies

Add to `app/pom.xml`:
- `spring-boot-starter-security`
- `spring-boot-starter-oauth2-resource-server`
- `spring-security-test` (test scope)

### 2. RSA Key Pair

Generate RSA key pair for JWT signing (development keys, committed to repo for convenience):

```bash
openssl genrsa -out app/src/main/resources/certs/private-key.pem 2048
openssl rsa -in app/src/main/resources/certs/private-key.pem -pubout -out app/src/main/resources/certs/public-key.pem
```

Copy the same keys to test resources: `app/src/test/resources/certs/`

### 3. Configuration Properties

Create **`RsaKeyProperties`** (`@ConfigurationProperties(prefix = "oauth2.rsa")`):
- `RSAPublicKey publicKey`
- `RSAPrivateKey privateKey`

Create **`JwtProperties`** (`@ConfigurationProperties(prefix = "oauth2")`):
- `Duration expiration` (default `PT1H`)

Add to `application.yml`:
```yaml
oauth2:
  rsa:
    private-key: classpath:certs/private-key.pem
    public-key: classpath:certs/public-key.pem
  expiration: ${JWT_TTL_SECONDS:PT1H}
```

Enable `@EnableConfigurationProperties({RsaKeyProperties.class, JwtProperties.class})`.

### 4. Security Configuration

Create **`SecurityConfig`** (`@Configuration`, `@EnableWebSecurity`, `@EnableMethodSecurity`):
- Disable CSRF (stateless API)
- Stateless session management (`SessionCreationPolicy.STATELESS`)
- Enable CORS: `.cors(cors -> cors.configurationSource(corsConfigurationSource()))`
- Configure OAuth2 Resource Server with JWT decoder using RSA public key
- Endpoint rules:
  - `POST /api/v1/auth/token` — `permitAll()`
  - `GET /swagger-ui/**`, `/api-docs/**`, `/v3/api-docs/**` — `permitAll()`
  - `GET /actuator/health` — `permitAll()`
  - All other requests — `authenticated()`
- Bean `JwtDecoder` using `NimbusJwtDecoder.withPublicKey(rsaPublicKey)`
- Bean `JwtEncoder` using `NimbusJwtEncoder` with RSA key pair (`RSAKey`)
- Bean `CorsConfigurationSource corsConfigurationSource()`:
  ```java
  @Bean
  CorsConfigurationSource corsConfigurationSource(
          @Value("${cors.allowed-origins:http://localhost:3000}") List<String> allowedOrigins) {
      CorsConfiguration config = new CorsConfiguration();
      config.setAllowedOriginPatterns(allowedOrigins);  // use patterns, not setAllowedOrigins
      config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
      config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Trace-Id"));
      config.setExposedHeaders(List.of("X-Trace-Id", "X-Span-Id", "Location"));
      config.setAllowCredentials(true);
      config.setMaxAge(3600L);
      UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
      source.registerCorsConfiguration("/**", config);
      return source;
  }
  ```
  Add to `application.yml`:
  ```yaml
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS:http://localhost:3000}
  ```
  Add to `application-prod.yml`: set `CORS_ALLOWED_ORIGINS` to the real frontend origin — never `*` in production.

### 5. Token Service

Create **`TokenService`** (`@Service`, `@Slf4j`, `@RequiredArgsConstructor`):
- Inject `JwtEncoder`, `JwtProperties`
- Method `String generateToken(Authentication authentication)`:
  - Build `JwtClaimsSet` with:
    - `issuer`: `sample-app`
    - `issuedAt`: now
    - `expiresAt`: now + expiration
    - `subject`: authentication name
    - `scope`: authorities as space-separated string
  - Encode and return token value

### 6. Auth Controller

Create **`AuthController`** (`@RestController`, `@RequestMapping("/api/v1/auth")`):
- `POST /token` — accepts `username`/`password` (form or JSON)
  - Authenticate via `AuthenticationManager`
  - Return `TokenResponse` with `accessToken`, `tokenType: "Bearer"`, `expiresIn`

Add `AuthenticationManager` bean in SecurityConfig (using `DaoAuthenticationProvider` or in-memory users for demo):
```java
@Bean
public InMemoryUserDetailsManager userDetailsService() {
    UserDetails user = User.withUsername("user")
        .password("{noop}password")
        .roles("USER")
        .build();
    UserDetails admin = User.withUsername("admin")
        .password("{noop}admin")
        .roles("USER", "ADMIN")
        .build();
    return new InMemoryUserDetailsManager(user, admin);
}
```

### 7. Method-Level Security with `@PreAuthorize`

`@EnableMethodSecurity` is already set on `SecurityConfig`. Use it for role-based access at the service or controller level when URL rules alone are too coarse:

```java
// Admin-only write operations
@PreAuthorize("hasRole('ADMIN')")
public PersonResponse create(PersonRequest request) { ... }

// Any authenticated user can read
@PreAuthorize("isAuthenticated()")
public PersonResponse getById(UUID id) { ... }

// User can only modify their own data, admin can modify any
@PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
public PersonResponse update(UUID id, PersonRequest request) { ... }
```

**Guideline:** prefer URL rules in `SecurityConfig` for broad access control; use `@PreAuthorize` when the rule depends on the method arguments or the caller's identity relative to the resource.

### 8. OpenAPI Security Scheme

Update Springdoc config (or add `@SecurityScheme` annotation on main class):
```java
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
)
```

Add `@SecurityRequirement(name = "bearerAuth")` on `PersonController`.

### 9. Update OpenAPI YAML

Add security scheme to `persons-api.yaml`:
```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

Add Auth endpoint paths (or create a separate `auth-api.yaml`).

## Output
- `RsaKeyProperties.java`, `JwtProperties.java`
- `SecurityConfig.java`
- `TokenService.java`
- `AuthController.java`, `TokenResponse.java`
- RSA keys in `app/src/main/resources/certs/` and `app/src/test/resources/certs/`
- Updated `application.yml`
- Updated OpenAPI YAML with security scheme

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile
```
Project must compile with security config wired.
