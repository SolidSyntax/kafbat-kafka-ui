# Copilot Instructions for Kafbat UI

Kafbat UI is a web UI for managing Apache Kafka clusters. It is a multi-module Gradle project with a **Java 25 Spring Boot reactive backend** and a **React 18 + TypeScript frontend**.

## Architecture

### Module structure

| Module | Purpose |
|---|---|
| `api/` | Spring Boot 3 WebFlux backend — the main application |
| `frontend/` | React 18 SPA built with Vite and pnpm |
| `contract-typespec/` | TypeSpec `.tsp` files that define the REST API |
| `contract/` | OpenAPI Generator config — produces Java server interfaces, DTOs, and TypeScript fetch clients from the TypeSpec-generated OpenAPI spec |
| `serde-api/` | Published library (`io.kafbat.ui:serde-api`) — pluggable Serde interface for custom message (de)serialization |
| `e2e-playwright/` | Playwright + Cucumber BDD end-to-end tests |

### API contract flow

```
contract-typespec/*.tsp  →  tsp compile  →  openapi.yaml
    →  OpenAPI Generator  →  Java server interfaces + DTO classes (backend)
    →  OpenAPI Generator  →  TypeScript fetch client (frontend/src/generated-sources/)
```

When changing an API endpoint: edit the `.tsp` files in `contract-typespec/api/`, then regenerate. Do not edit generated code directly.

### Backend (api/)

- **Reactive stack**: Spring WebFlux — controllers return `Mono<ResponseEntity<...>>` and `Flux<...>`, not traditional MVC.
- **Package layout**: `io.kafbat.ui.{controller,service,model,mapper,config,exception,serdes,emitter,util}`
- **Mappers**: MapStruct interfaces with `@Mapper(componentModel = "spring")` — compile-time generated, mapping internal models ↔ DTOs.
- **DTOs**: All generated from the OpenAPI spec with a `DTO` suffix (e.g. `ClusterDTO`, `TopicDTO`). Annotated with Lombok `@AllArgsConstructor @NoArgsConstructor`.
- **Exceptions**: Custom hierarchy rooted at `CustomBaseException` with an `ErrorCode` enum mapping to HTTP status codes. Global handler: `GlobalErrorWebExceptionHandler`.
- **External clients**: Generated OpenAPI clients for Kafka Connect, Schema Registry, and Prometheus APIs (see `contract/build.gradle`).
- **Auth**: Pluggable — OAuth 2.0, LDAP, Basic Auth, or disabled. RBAC via `AccessControlService`.

### Frontend (frontend/)

- **Stack**: React 18, TypeScript (strict), Vite, styled-components 6
- **State management**: React Query 5 for server state, Zustand for client state, React Context for global UI state (theme, RBAC, confirmations).
- **API layer**: Generated TypeScript fetch clients in `src/generated-sources/`, wrapped in React Query hooks at `src/lib/hooks/api/`.
- **Component structure**: Feature-based folders under `src/components/` (Topics, Brokers, Schemas, etc.), shared components in `src/components/common/`.
- **Routing**: React Router v6 with centralized path definitions in `src/lib/paths.ts`.
- **Test helpers**: Custom `render()` in `src/lib/testHelpers.tsx` wraps components with all required providers (Router, QueryClient, Theme, RBAC context).

## Build & Test Commands

### Backend

```bash
./gradlew api:build                  # Build backend
./gradlew api:test                   # Run all backend tests
./gradlew api:test --tests '*.TopicsServiceTest'  # Single test class
./gradlew api:test --tests '*.TopicsServiceTest.methodName'  # Single test method
./gradlew api:checkstyleMain         # Run checkstyle
```

Integration tests use Testcontainers (Kafka, Schema Registry, Kafka Connect, KSQL, Prometheus). Test base class: `AbstractIntegrationTest`.

### Frontend

```bash
cd frontend
pnpm install                         # Install dependencies
pnpm dev                             # Dev server on port 3000
pnpm build                           # Full build (gen:sources → tsc → vite build)
pnpm gen:sources                     # Regenerate API client from OpenAPI spec
pnpm test                            # Jest in watch mode
pnpm test -- path/to/file.spec.tsx   # Single test file
pnpm test -- --testNamePattern="pattern"  # Single test by name
pnpm lint                            # ESLint
pnpm lint:fix                        # ESLint with auto-fix
```

Node 22+ and pnpm 10+ required (versions specified in `gradle.properties` and `frontend/.nvmrc`).

### E2E tests

```bash
cd e2e-playwright
pnpm test                            # Run all E2E tests
pnpm debug                           # Run with Playwright debug mode
```

### Full project

```bash
./gradlew build                      # Backend only (default)
./gradlew build -Pinclude-frontend   # Backend + frontend
```

## Code Conventions

### Java

- **Style**: Google Java Style enforced via Checkstyle (`etc/checkstyle/checkstyle.xml`). Line limit: 120 chars. Indent: 2 spaces.
- **Checkstyle is strict**: `maxWarnings = 0`, `maxErrors = 0`. Excluded packages: `ksql`, `promql`.
- **Lombok** is used throughout — `@Data`, `@Builder`, `@AllArgsConstructor`, etc.
- **Testing**: JUnit 5 + Mockito + AssertJ + Reactor Test. Integration tests extend `AbstractIntegrationTest` and use Testcontainers with `WebTestClient`.

### TypeScript / React

- **ESLint**: Airbnb base + TypeScript strict. `@typescript-eslint/no-explicit-any: error`. Circular imports disallowed (`import/no-cycle`).
- **Prettier**: Single quotes, trailing commas (es5), semicolons, LF line endings.
- **Import order**: builtin → external → parent → sibling → index. No relative parent imports (use baseUrl `src/`).
- **Testing**: Jest 29 + React Testing Library + fetch-mock. Tests in `__tests__/` directories as `*.spec.{ts,tsx}`.
- **Styling**: styled-components with theming (light/dark). Global theme in `src/theme/theme.ts`.

### REST API conventions (from CONTRIBUTING.md)

- Paths: **lowercase**, **plural** nouns, words separated by hyphens.
- Query parameters: `camelCase`.
- Model names: plural nouns, `camelCase`.

### Branch naming

- `issues/123`, `feature/feature_name`, `bugfix/fix_thing`
