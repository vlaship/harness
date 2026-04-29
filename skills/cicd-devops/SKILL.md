---
name: cicd-devops
description: "Set up GitHub Actions CI pipeline, multi-stage Dockerfile, .dockerignore, docker-compose app service, README, and IntelliJ HTTP Client files. Use for project finalization."
---

## Preconditions
`unit-tests` completed. All tests pass with `mvn verify`.

## Goal
Phase 1 — GitHub Actions pipeline and production Docker image.
Phase 2 — project documentation (README) and IntelliJ HTTP Client files for manual API testing.

## Instructions

---

## Phase 1: CI/CD & Docker

1. Create **`.github/workflows/ci.yml`**:
   - Trigger: push to `main`, PRs to `main`
   - Job `build-and-test` on `ubuntu-latest`:
     - Checkout
     - Setup JDK 25 (`actions/setup-java@v4`, `distribution: temurin`)
     - Cache Maven with key `${{ hashFiles('**/pom.xml') }}` — invalidates when any POM changes
     - `mvn verify --batch-mode --no-transfer-progress`
   - Job `docker-build` (runs after `build-and-test`, on push to `main` only):
     - Build image tagged with both git SHA and `latest`:
       ```
       docker build -t $IMAGE_NAME:${{ github.sha }} -t $IMAGE_NAME:latest .
       ```
     - Security scan with Trivy (fail on CRITICAL/HIGH):
       ```yaml
       - uses: aquasecurity/trivy-action@master
         with:
           image-ref: $IMAGE_NAME:${{ github.sha }}
           format: table
           exit-code: '1'
           severity: CRITICAL,HIGH
           ignore-unfixed: true
       ```
     - Push both tags to registry only if scan passes
2. Create **`Dockerfile`** (multi-stage):
   - Stage 1 (build): `eclipse-temurin:25-jdk`, `mvn package -DskipTests`
   - Stage 2 (runtime): `eclipse-temurin:25-jre`
     - Non-root user `appuser`
     - Expose 8080
     - Health check: `curl -f http://localhost:8080/actuator/health || exit 1`
     - `-XX:+UseContainerSupport` JVM flags
3. Create **`.dockerignore`**: `.git`, `target/`, `*.md`, `.github/`, `.idea/`, `http/`, `.claude/`
4. Update **`docker-compose.yml`** — add `app` service:
   - Builds from Dockerfile
   - Depends on `postgres`
   - Environment for DB connection
   - Port `8080:8080`, profile `local`

---

## Phase 2: Documentation & HTTP Client

### README.md

Create `README.md` in project root:

1. **Project title** and one-line description
2. **Architecture overview** — module diagram, each module's role, tech stack:
   ```
   persons-api.yaml (single source of truth)
     ├── api     → server interfaces + DTOs (OpenAPI Generator: spring)
     │     └── app depends on api → implements interfaces, business logic
     └── client  → typed Java client (OpenAPI Generator: java/restclient)
                   for external consumers
   ```
3. **Prerequisites**: Java 25, Maven 3.9+, Docker
4. **Quick start**:
   ```bash
   docker compose up -d postgres
   mvn spring-boot:run -pl app -Dspring-boot.run.profiles=local
   ```
5. **Build & test**: `mvn verify`
6. **Docker**: `docker compose up --build`
7. **API docs**: link to `http://localhost:8080/swagger-ui.html`
8. **Using the client module**: how consumers add the dependency and configure:
   ```xml
   <dependency>
     <groupId>com.example</groupId>
     <artifactId>sample-spring-boot-web-with-db-client</artifactId>
     <version>0.1.0</version>
   </dependency>
   ```
   Brief example of wiring `PersonsApi` bean with `RestClient` and calling `personsApi.getPersonById(uuid)`.
9. **Project structure**: brief tree of key directories

### IntelliJ HTTP Client

Create `http/` directory in project root:

1. **`http/http-client.env.json`**:
   ```json
   {
     "local": {
       "baseUrl": "http://localhost:8080/api/v1",
       "actuatorUrl": "http://localhost:8080/actuator"
     },
     "docker": {
       "baseUrl": "http://localhost:8080/api/v1",
       "actuatorUrl": "http://localhost:8080/actuator"
     }
   }
   ```

2. **`http/persons-api.http`** — all CRUD operations:
   - `### Get Auth Token` — POST `{{baseUrl}}/auth/token` with `user`/`password`, capture `{{authToken}}` via `> {% %}`
   - `### Create Person` — POST with `Authorization: Bearer {{authToken}}`, capture `{{personId}}`
   - `### Create Second Person` — for pagination testing
   - `### Get Person by ID` — GET `{{personId}}`
   - `### Get All Persons` — GET `?page=0&size=10&sort=lastName,asc`
   - `### Update Person` — PUT with modified body
   - `### Get Updated Person` — verify update
   - `### Delete Person` — DELETE
   - `### Verify Deletion` — GET same id, expect 404
   - `### Unauthenticated Request` — GET without token, expect 401
   - All authenticated requests include `Authorization: Bearer {{authToken}}` header
   - Each request: `###` separator, descriptive comment, `{{baseUrl}}` variable, realistic example data

3. **`http/actuator.http`** — monitoring endpoints (no auth required):
   - `### Health Check` — GET `{{actuatorUrl}}/health`
   - `### Health Liveness` — GET `{{actuatorUrl}}/health/liveness`
   - `### Health Readiness` — GET `{{actuatorUrl}}/health/readiness`
   - `### Prometheus Metrics` — GET `{{actuatorUrl}}/prometheus`
   - `### Application Info` — GET `{{actuatorUrl}}/info`
   - `### All Metrics Names` — GET `{{actuatorUrl}}/metrics` (requires auth)
   - `### Specific Metric` — GET `{{actuatorUrl}}/metrics/http.server.requests` (requires auth)
   - `### Cache Stats` — GET `{{actuatorUrl}}/caches` (requires auth)

---

## Output
- `.github/workflows/ci.yml`
- `Dockerfile`, `.dockerignore`
- Updated `docker-compose.yml`
- `README.md`
- `http/http-client.env.json`
- `http/persons-api.http`
- `http/actuator.http`

## Verification
```bash
cd sample-spring-boot-web-with-db && mvn verify
```
No build verification for docs/HTTP files — review for completeness.
