---
name: integration-tests
description: "Create Testcontainers-based integration tests for all CRUD endpoints happy path, auth flow, and traceId verification. Primary test layer. Requires Docker; CI fallback is @MockitoBean in unit-tests skill."
---

## Preconditions
`rest-controller` completed. Full project compiles.

## Goal
Create integration tests with Testcontainers as the **primary test layer** covering happy path for all CRUD endpoints.

## Instructions

1. Create **`AbstractIntegrationTest`** base class (`app/src/test/java/com/example/sample/`):
   - `@SpringBootTest(webEnvironment = RANDOM_PORT)`
   - `@Testcontainers`
   - Static `PostgreSQLContainer` (PostgreSQL 17), singleton pattern
   - `@DynamicPropertySource` for datasource properties
   - Inject `TestRestTemplate`
   - Cleanup between tests via `@Sql` scripts — **not** `@Transactional` (HTTP calls run in a different thread, so rollback won't happen)
   - **Auth helper**: method to obtain JWT token via `POST /api/v1/auth/token` with test credentials (`user`/`password`), cache token for reuse across tests in the class
   - Helper method `authenticatedHeaders()` returning `HttpHeaders` with `Authorization: Bearer <token>`
   - **CI note**: if Testcontainers cannot run (no Docker on the agent), the `unit-tests` skill's MockMvc tests use `@MockitoBean` on the repository as a fallback. These ITs require Docker and should be skipped with `-DskipITs` or a Maven profile when TC is unavailable.
2. Create **`PersonControllerIT`** extending `AbstractIntegrationTest`:
   - **Create Person** — POST `/api/v1/persons`
     - Assert `201`, response has `id` (valid UUID format), `createdAt` not null, `createdBy` equals authenticated username, `Location` header present
     - Assert `X-Trace-Id` response header is present and non-empty
   - **Get by ID** — create first, then GET by returned UUID
     - Assert `200`, all fields match
   - **Get All (paginated)** — create 3+ persons, GET `?page=0&size=5&sort=lastName,asc`
     - Assert `200`, `content` size, `totalElements`, `totalPages`, sort order
   - **Update** — create, then PUT with updated data
     - Assert `200`, fields updated, `updatedAt` changed, `modifiedBy` equals authenticated username
   - **Delete** — create, then DELETE
     - Assert `204`, subsequent GET returns `404`
3. Test data setup via `@Sql` scripts or setup methods. Clean after each test.
4. Naming: `should_expectedBehavior_when_condition`
5. **All CRUD requests must include the JWT token** from `authenticatedHeaders()`
6. Create **`AuthControllerIT`** extending `AbstractIntegrationTest`:
   - **Get Token** — POST `/api/v1/auth/token` with valid credentials → `200`, response has `accessToken`, `tokenType`, `expiresIn`
   - **Invalid Credentials** — POST with wrong password → `401`
   - **Unauthenticated Access** — GET `/api/v1/persons` without token → `401`
7. Create **`ActuatorIT`** extending `AbstractIntegrationTest`:
   - **Health** — GET `/actuator/health` without token → `200`, body contains `status: UP`
   - **Prometheus** — GET `/actuator/prometheus` without token → `200`, body contains `http_server_requests_seconds`
   - **Custom metrics** — create a person, then GET `/actuator/prometheus` → body contains `persons_created_total`

## Outbound REST calls
If any service method calls an external API via `RestClient` or the generated client, add a **WireMock** server to `AbstractIntegrationTest`:
```java
@RegisterExtension
static WireMockExtension wireMock = WireMockExtension.newInstance()
    .options(wireMockConfig().dynamicPort())
    .build();

@DynamicPropertySource
static void wireMockProps(DynamicPropertyRegistry registry) {
    registry.add("downstream-service.base-url", wireMock::baseUrl);
}
```
Stub the downstream in each test that triggers an outbound call. Never mock the HTTP client class.

## Do NOT Test Here
- Validation errors (→ `unit-tests`)
- Exception edge cases (→ `unit-tests`)
- MapStruct edge cases (→ `unit-tests`)
- Cache TTL / eviction timing (unit-test or manual)

## Output
- `AbstractIntegrationTest.java`, `PersonControllerIT.java`, `AuthControllerIT.java`, `ActuatorIT.java`
- Test data SQL scripts in `app/src/test/resources/` if needed

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn verify -pl app
```
All integration tests must pass.
