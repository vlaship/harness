---
name: openapi-spec
description: "Create OpenAPI 3.1 YAML spec, configure server code generation in the api module, and generate a typed Java RestClient in the client module. Use when setting up the API contract and both server/client libraries."
---

## Preconditions
`project-skeleton` completed. Project compiles with `mvn compile`.

## Goal
Phase 1 — create the OpenAPI YAML and configure OpenAPI Generator to produce server-side interfaces + DTO models in the `api` module.
Phase 2 — configure OpenAPI Generator in the `client` module to produce a typed Java RestClient from the same YAML.

```
persons-api.yaml (single source of truth)
  ├── api     → server interfaces (spring generator)
  └── client  → Java client (java generator + restclient)
```

## Instructions

---

## Phase 1: OpenAPI Spec & Server Interfaces

1. Create `api/src/main/resources/openapi/persons-api.yaml` with:
   - OpenAPI 3.1
   - Info section with title, version, description
   - Base path: `/api/v1`
   - Paths:
     - `POST /persons` — create person (201)
     - `GET /persons/{id}` — get by id (200, 404)
     - `GET /persons` — list with pagination: `page`, `size`, `sort` query params (200)
     - `PUT /persons/{id}` — full update (200, 404)
     - `DELETE /persons/{id}` — delete (204, 404)
   - Components/Schemas:
     - `PersonRequest` — `firstName` (required, minLength 1), `lastName` (required), `email` (required, format email), `birthDate` (required, format date, must be in the past), `phone` (optional)
     - `PersonResponse` — all above + `id` (type: string, format: uuid), `createdAt`, `updatedAt` (format date-time), `createdBy`, `modifiedBy` (string, nullable)
     - `PersonPageResponse` — `content` (array of PersonResponse), `page`, `size`, `totalElements`, `totalPages`
     - `ProblemDetail` — per RFC 9457: `type`, `title`, `status`, `detail`, `instance`
   - Error responses reference `ProblemDetail`

2. Configure **OpenAPI Generator** in `api/pom.xml` — **server interfaces**:
   - Generator: `spring`
   - Generate into `target/generated-sources/openapi`
   - API package: `com.example.sample.api`
   - Model package: `com.example.sample.api.model`
   - Configuration properties:
     - `interfaceOnly: true` — generate interfaces only, controller implements them
     - `useSpringBoot3: true` — Jakarta namespace
     - `useTags: true` — group endpoints by tags
     - `useBeanValidation: true` — generate `@NotNull`, `@Size`, `@Email` etc. on models
     - `performBeanValidation: true` — enable runtime validation
     - `serializableModel: true` — models implement `Serializable` (needed for cache serialization)
     - `documentationProvider: springdoc` — generate Springdoc OpenAPI annotations instead of Swagger 2
     - `useResponseEntity: false` — controllers return plain objects, HTTP status via `@ResponseStatus`
     - `containerDefaultToNull: true` — lists default to `null`, not empty — explicit null handling
     - `generateSupportFiles: false` — skip ApiUtil.java and other boilerplate
   - Generate models as plain POJOs (not records). Do **NOT** configure Lombok in generator.
   - Date mapping: `java.time.LocalDate` for date, `java.time.OffsetDateTime` for date-time

---

## Phase 2: Client Generation

Create `package-info.java` with `@NullMarked` in every new package created below.

1. Configure **OpenAPI Generator Maven plugin** in `client/pom.xml`:
   - Input spec: `${project.basedir}/../api/src/main/resources/openapi/persons-api.yaml` (same YAML as `api`)
   - Generator: `java`
   - Library: `restclient` (Spring 6+ `RestClient` — no WebFlux, no Feign, no external HTTP deps)
   - Generate into `target/generated-sources/openapi`
   - API package: `com.example.sample.client.api`
   - Model package: `com.example.sample.client.model`
   - Invoker package: `com.example.sample.client`
   - Configuration properties:
     - `useSpringBoot3: true` — Jakarta namespace
     - `serializableModel: true` — for cache compatibility
     - `useBeanValidation: true`
     - `generateSupportFiles: true` — client needs `ApiClient`, auth helpers, etc.
     - `hideGenerationTimestamp: true` — cleaner diffs in VCS
     - `openApiNullable: false` — plain types, no `JsonNullable` wrappers
   - Date mapping: `java.time.LocalDate` for date, `java.time.OffsetDateTime` for date-time (same as `api`)

2. The generated client provides:
   - **`PersonsApi`** class with typed methods:
     - `createPerson(PersonRequest)` → `PersonResponse`
     - `getPersonById(UUID)` → `PersonResponse`
     - `getAllPersons(Integer page, Integer size, String sort)` → `PersonPageResponse`
     - `updatePerson(UUID, PersonRequest)` → `PersonResponse`
     - `deletePerson(UUID)` → `void`
   - **`ApiClient`** — configurable with base URL, default headers, request interceptors
   - **Model classes** — same structure as server (generated from same YAML)

3. Create **`PersonClientConfig`** example (`client/src/main/java/com/example/sample/client/config/`):
   ```java
   @Configuration
   public class PersonClientConfig {

       @Bean
       public PersonsApi personsApi(
               @Value("${persons-service.base-url:http://localhost:8080}") String baseUrl) {
           ApiClient apiClient = new ApiClient(RestClient.builder()
               .baseUrl(baseUrl)
               .build());
           return new PersonsApi(apiClient);
       }
   }
   ```
   Also create **`PersonClientWithAuthConfig`** — shows how to forward the caller's JWT to the downstream service:
   ```java
   @Configuration
   public class PersonClientWithAuthConfig {

       @Bean
       public PersonsApi personsApiWithAuth(
               @Value("${persons-service.base-url:http://localhost:8080}") String baseUrl) {
           ApiClient apiClient = new ApiClient(RestClient.builder()
               .baseUrl(baseUrl)
               .requestInterceptor(request -> {
                   Authentication auth = SecurityContextHolder.getContext().getAuthentication();
                   if (auth instanceof JwtAuthenticationToken jwtAuth) {
                       request.getHeaders().setBearerAuth(jwtAuth.getToken().getTokenValue());
                   }
               })
               .build());
           return new PersonsApi(apiClient);
       }
   }
   ```
   Use this variant when the consumer service is itself a resource server and needs to propagate the inbound token. Micrometer auto-instruments `RestClient.builder()` — tracing headers (`traceparent`) are propagated automatically.

4. Create **`application-client.yml`** example in `client/src/main/resources/`:
   ```yaml
   persons-service:
     base-url: ${PERSONS_SERVICE_URL:http://localhost:8080}
   ```

### Consumer Usage

Other services use the client by adding a single dependency:
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>sample-spring-boot-web-with-db-client</artifactId>
  <version>${project.version}</version>
</dependency>
```

Then either use the provided `PersonClientConfig` or wire their own:
```java
@Autowired
private PersonsApi personsApi;

PersonResponse person = personsApi.getPersonById(uuid);
```

---

## Output
- `api/src/main/resources/openapi/persons-api.yaml`
- Updated `api/pom.xml` with OpenAPI Generator plugin (spring server)
- Updated `client/pom.xml` with OpenAPI Generator plugin (java/restclient)
- `PersonClientConfig.java` and `PersonClientWithAuthConfig.java` in `client/src/main/java/`
- `application-client.yml` in `client/src/main/resources/`
- `package-info.java` in all new packages

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn generate-sources -pl api
```
`api` module must generate `PersonsApi` interface + model classes without errors.
```bash
cd sample-spring-boot-web-with-db && mvn generate-sources -pl client && mvn compile -pl client
```
Client module must generate and compile including the hand-written `PersonClientConfig`.
