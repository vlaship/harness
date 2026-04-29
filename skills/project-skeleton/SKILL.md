---
name: project-skeleton
description: "Create Maven multi-module project structure with parent POM, api/client/app modules, and all dependencies. Use for initial Spring Boot project scaffolding."
---

## Preconditions
None. This is the first skill.

## Parameters
Before generating any files, confirm these values with the user (show defaults, accept overrides):

| Parameter | Default | Used in |
|-----------|---------|---------|
| `ROOT_DIR` | `sample-spring-boot-web-with-db` | root directory name, artifact ID |
| `GROUP_ID` | `com.example` | all `pom.xml` groupId fields |
| `BASE_PACKAGE` | `com.example.sample` | all Java package declarations |
| `ENTITY_NAME` | `Person` | entity class, service, controller, mapper names |
| `DB_SCHEMA` | `sample` | Liquibase schema, datasource default-schema |
| `APP_NAME` | `sample-app` | `spring.application.name`, Docker image name |
| `SB_VERSION` | `3.5.x` (latest stable) | Spring Boot parent version |
| `JAVA_VERSION` | `21` | compiler source/target, JDK in CI and Dockerfile |

Apply substitutions consistently across **all** skills. All skills in this harness use the defaults above — override once here and every downstream skill picks up the change.

## Goal
Create the Maven multi-module project structure with all `pom.xml` files and dependency management.

## Instructions

1. Create root directory `sample-spring-boot-web-with-db/`
2. Create **parent `pom.xml`** with:
   - `groupId`: `com.example`
   - `artifactId`: `sample-spring-boot-web-with-db`
   - `packaging`: `pom`
   - `version`: `0.1.0`
   - Spring Boot 4 parent (latest stable)
   - Java 25 compiler settings
   - `<modules>`: `api`, `client`, `app`
   - `<dependencyManagement>` with shared versions for: MapStruct, Lombok, Springdoc OpenAPI, Testcontainers, Liquibase, java-uuid-generator (com.fasterxml.uuid), JSpecify (`org.jspecify:jspecify`)
   - Lombok + MapStruct annotation processor configuration in `<pluginManagement>`
3. Create **`api/pom.xml`**:
   - OpenAPI Generator Maven plugin
   - Only dependencies needed for generated code (Jakarta Validation, Jackson annotations, Spring Web annotations)
4. Create **`client/pom.xml`**:
   - OpenAPI Generator Maven plugin (configured in `openapi-spec` skill)
   - Dependencies for generated client code:
     - `spring-web` (RestClient)
     - `jackson-databind`, `jackson-datatype-jsr310`
     - `jakarta.annotation-api`
     - Jackson Nullable (`org.openapitools:jackson-databind-nullable`) if needed, but prefer `openApiNullable: false`
   - Does **NOT** depend on `api` module — client is independently generated from the same YAML
   - Consumers add `client` as a dependency to call our service
5. Create **`app/pom.xml`**:
   - Depends on `api` module (for generated server interfaces and models)
   - Does **NOT** depend on `client` module (client is for external consumers)
   - All runtime dependencies: Spring Boot Starter Web, Spring Boot Starter Security, Spring Boot Starter OAuth2 Resource Server, Spring Boot Starter Actuator, Spring Boot Starter Cache, Caffeine, Spring Boot Starter Data Redis (optional/prod), Micrometer Tracing Bridge OTel, OpenTelemetry Exporter, Micrometer Registry Prometheus, Spring Data JDBC/JPA, PostgreSQL driver, Liquibase, MapStruct, Lombok, Springdoc, Testcontainers (test scope), Spring Security Test (test scope), ArchUnit (`com.tngtech.archunit:archunit-junit5`, test scope)
6. Create placeholder `src/main/java` directories in each module so Maven compiles

## Output
```
sample-spring-boot-web-with-db/
├── pom.xml
├── api/
│   ├── pom.xml
│   └── src/main/java/
├── client/
│   ├── pom.xml
│   └── src/main/java/
└── app/
    ├── pom.xml
    └── src/main/java/
```

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn compile
```
Must complete with `BUILD SUCCESS`.
