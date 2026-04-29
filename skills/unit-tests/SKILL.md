---
name: unit-tests
description: "Add MockMvc-based edge case tests (real service, mock or TC repo) plus pure unit tests for mapper, token, repository slice, and architecture. No happy path duplication with IT."
---

## Preconditions
`integration-tests` completed. All integration tests pass.

## Goal
Cover edge cases not tested by integration tests. All MockMvc tests call through the **real service** — never mock the service layer.

## Instructions

### Database choice — ask before writing MockMvc tests
> "Should I use Testcontainers (full stack, real DB) or `@MockitoBean` on the repository (no Docker needed, CI-safe)?"

- **Testcontainers** (preferred): extend `AbstractIntegrationTest`, switch `webEnvironment` to `MOCK`, inject `MockMvc`
- **`@MockitoBean` on repository**: `@SpringBootTest(webEnvironment = MOCK)` + `@AutoConfigureMockMvc`, then `@MockitoBean PersonRepository personRepository;` — stub only what the specific test needs

### 1. `PersonControllerMockMvcTest`
(`@SpringBootTest(webEnvironment = MOCK)`, `@AutoConfigureMockMvc`)

**Validation errors** — controller validates before the service is called; no DB interaction needed regardless of DB choice:
- Missing/blank `firstName` → `422` ProblemDetail with field error
- Missing `lastName` → `422`
- Invalid `email` format → `422`
- `birthDate` in the future → `422`
- Null `email` → `422`

**Exception mapping** — stub the repo (or insert no data in TC) to produce the condition:
- Repo returns `Optional.empty()` → service throws `PersonNotFoundException` → `404` ProblemDetail containing `traceId`
- Repo throws `DataIntegrityViolationException` → service wraps as `DuplicateEmailException` → `409` ProblemDetail
- Unhandled `RuntimeException` from service → `500` ProblemDetail, **no stack trace in body**

**Security**:
- Unauthenticated request (no token) → `401`
- If `@PreAuthorize` is used: wrong role → `403`

### 2. `PersonMapperTest` (pure unit — no Spring context):
- Null input handling
- `updateEntity` preserves `id`, `createdAt`
- `toResponse` with null optional fields (`phone`)

### 3. `TokenServiceTest` (pure unit — no Spring context):
- Generated token is non-empty and parseable
- Token contains correct subject and expiration
- Expired token: use `@TestPropertySource(properties = "oauth2.expiration=PT1S")`, wait 2 s with Awaitility, assert decoder rejects it

### 4. `PersonRepositoryTest` (`@DataJpaTest`, Testcontainers PostgreSQL):
- Custom queries return correct results
- Pagination: verify `Page` metadata (`totalElements`, `totalPages`)
- Email uniqueness: duplicate insert throws `DataIntegrityViolationException`
- Use `@AutoConfigureTestDatabase(replace = NONE)` + `@DynamicPropertySource` for TC datasource

### 5. WireMock test (add only if the service makes outbound REST calls):
- Register `WireMockExtension` as a `@RegisterExtension static` field
- Point the downstream `base-url` property to `wireMockServer.baseUrl()` via `@DynamicPropertySource`
- Stub the expected request: `stubFor(get(urlEqualTo("...")).willReturn(aResponse()...))`
- **Never mock `RestClient`, `PersonsApi`, or any HTTP client class** — always stub at the wire
- Test success path and error path (downstream returns 500 → verify service error handling)

### 6. `ArchitectureTest` (ArchUnit, `@AnalyzeClasses(packages = "com.example.sample")`):
- Controllers must not access repositories directly
- Services must not import controller classes
- Classes in `entity` package must implement `Domain`
- No class may use `@Autowired` for field injection

## Rules
- One assertion concern per test method
- Do NOT test any happy path already covered in `integration-tests`

## Output
- `PersonControllerMockMvcTest.java`
- `PersonMapperTest.java`
- `TokenServiceTest.java`
- `PersonRepositoryTest.java`
- `PersonServiceWireMockTest.java` (only if service makes outbound REST calls)
- `ArchitectureTest.java`

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn verify -pl app
```
All unit AND integration tests must pass.
