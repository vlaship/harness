---
name: service-mapping
description: "Create PersonService, MapStruct mappers, and custom exceptions. Use when building the service layer between repository and controller."
---

## Preconditions
`database-layer` completed. Entity and repository compile.

## Goal
Create the service layer and MapStruct mappers between entities and OpenAPI-generated DTOs.

## Instructions

Create `package-info.java` with `@NullMarked` in every new package created below.

1. Create **`PersonMapper`** (`app/src/main/java/com/example/sample/mapper/`):
   - `@Mapper(componentModel = "spring")`
   - Methods:
     - `Person toEntity(PersonRequest dto)` — ignore `id`, `createdAt`, `updatedAt`
     - `PersonResponse toResponse(Person entity)`
     - `List<PersonResponse> toResponseList(List<Person> entities)`
     - `void updateEntity(PersonRequest dto, @MappingTarget Person entity)` — preserve `id`, `createdAt`
   - Use `@Mapping(target = "...", ignore = true)` where needed
2. Create **`PersonService`** (`app/src/main/java/com/example/sample/service/`):
   - `@Service`, `@Slf4j`, `@RequiredArgsConstructor`, `@Transactional(readOnly = true)` at class level — all methods default to read-only
   - Constructor-injected `PersonRepository` and `PersonMapper`
   - Methods (override class-level `readOnly` with `@Transactional` on writes):
     - `@Transactional PersonResponse create(PersonRequest request)` — map to entity, assign UUID v7 id, `save()`; catch `DataIntegrityViolationException` → throw `DuplicateEmailException`
     - `PersonResponse getById(UUID id)` — `findById` or throw `PersonNotFoundException`
     - `PersonPageResponse getAll(int page, int size, String sort)` — paginated via `PageRequest`, assemble `PersonPageResponse` from `Page<Person>`
     - `@Transactional PersonResponse update(UUID id, PersonRequest request)` — find, `updateEntity()` via mapper, `save()`; catch `DataIntegrityViolationException` → `DuplicateEmailException`
     - `@Transactional void delete(UUID id)` — find or throw `PersonNotFoundException`, then `deleteById()`
   - Log create/update/delete at INFO
   - Do NOT add cache annotations here — they will be added by `cache-layer` skill
   - Do NOT add metrics calls here — they will be added by `metrics` skill
3. Create **`PersonNotFoundException`** extends `RuntimeException` — include `UUID id`
4. Create **`DuplicateEmailException`** extends `RuntimeException` — include the duplicate email value; wrap `DataIntegrityViolationException` in the service as shown:
   ```java
   try {
       return mapper.toResponse(repository.save(entity));
   } catch (DataIntegrityViolationException ex) {
       throw new DuplicateEmailException(request.getEmail());
   }
   ```

## Output
- `PersonMapper.java`, `PersonService.java`
- `PersonNotFoundException.java`, `DuplicateEmailException.java`

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile -pl app
```
MapStruct must generate implementation. Service must compile.
