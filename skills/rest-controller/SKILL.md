---
name: rest-controller
description: "Implement REST controller from generated API interface, GlobalExceptionHandler with ProblemDetail (RFC 9457), and Springdoc config. Use for controller layer."
---

## Preconditions
`service-mapping` completed. Service layer and mapper compile.

## Goal
Implement the REST controller from the generated API interface and add global exception handling with Problem Details (RFC 9457).

## Instructions

Create `package-info.java` with `@NullMarked` in every new package created below.

1. Create **`PersonController`** (`app/src/main/java/com/example/sample/controller/`):
   - `@RestController`, `@Slf4j`, `@RequiredArgsConstructor`
   - **Implements** the generated API interface from `api` module
   - Delegates to `PersonService`
   - Since `useResponseEntity: false` in OpenAPI Generator, methods return plain objects (not `ResponseEntity`)
   - HTTP status codes via `@ResponseStatus` annotations:
     - `@ResponseStatus(HttpStatus.CREATED)` for POST
     - `@ResponseStatus(HttpStatus.OK)` for GET and PUT (default, can omit)
     - `@ResponseStatus(HttpStatus.NO_CONTENT)` for DELETE
   - POST: add `Location` header via `HttpServletResponse.setHeader()` or inject `UriComponentsBuilder`
2. Create **`GlobalExceptionHandler`** (`app/src/main/java/com/example/sample/exception/`):
   - `@RestControllerAdvice`, `@Slf4j`
   - Handlers:
     - `PersonNotFoundException` → `404` ProblemDetail
     - `DuplicateEmailException` → `409` ProblemDetail
     - `MethodArgumentNotValidException` → `422` ProblemDetail + field errors in `properties`
     - `ConstraintViolationException` → `422` ProblemDetail
     - `ObjectOptimisticLockingFailureException` → `409` ProblemDetail ("Resource was modified by another request — please retry")
     - `Exception` (catch-all) → `500` ProblemDetail (log stacktrace, return generic message)
   - Use `ProblemDetail.forStatusAndDetail()` from Spring Framework
   - Set `type` URI and `instance` to request URI
   - TraceId enrichment will be added by `observability` skill later
3. Create **`SampleApplication`** main class:
   - `@SpringBootApplication`, `@EnableJpaAuditing`
4. Configure **Springdoc OpenAPI** in `application.yml`:
   - Swagger UI: `/swagger-ui.html`
   - API docs: `/api-docs`

## Output
- `PersonController.java`, `GlobalExceptionHandler.java`, `SampleApplication.java`
- Updated `application.yml`

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile
```
Full project must compile. Controller must implement generated API interface.
