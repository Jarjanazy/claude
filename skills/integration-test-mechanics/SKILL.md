---
name: integration-test-mechanics
description: Use when writing or debugging integration tests in a Kotlin/Maven Spring Boot project that spins up the full Docker stack via Testcontainers DockerComposeContainer — covers container lifecycle, singleton pattern, wait strategies, and email testing with MailHog
---

# Integration Test Mechanics

## Overview

True end-to-end tests: spin up the **full stack** — Postgres, the compiled Spring Boot app, and any supporting services (mail trap, optional auth) — inside Docker, then make real HTTP calls against it using OkHttp3.

No mocking. No Spring test context. No embedded database. The app runs as it would in production.

---

## When to Use

- Writing new integration tests against a live Docker stack
- Debugging why the stack starts/fails to start
- Understanding why seed data is missing or tests are colliding
- Adding a new service to the Docker Compose setup

---

## Prerequisites

**You must run `mvn package -pl <module> -am` (or build the whole project) before running tests.**

The Dockerfile copies the already-compiled JAR at image build time. If the JAR is stale or missing, `docker compose build` will either fail or the container will run old code.

---

## The Docker Compose File

`<backend-module>/docker-compose.yml` defines the services. Minimal required set:

| Service | Image | Role in tests |
|---|---|---|
| `postgres` | `postgres:latest` | App database |
| `mailserver` | `mailhog/mailhog` | Captures outgoing emails; tests read them via HTTP API |
| `app` | built from `Dockerfile` | The Spring Boot app under test |

**Optional (auth):**

| Service | Image | Role in tests |
|---|---|---|
| `keycloak` | `quay.io/keycloak/keycloak:26.2` | Auth server; issues JWTs the app validates |
| `keycloak-ready` | `alpine` | One-shot health gate: polls until the Keycloak realm is available, then exits |

`app` should declare `depends_on` on its required services so Docker Compose starts them in order. For Keycloak, depend on `keycloak-ready` (completed) rather than `keycloak` (started) to ensure the realm is configured before the app boots.

Projects without auth can omit the `keycloak` and `keycloak-ready` services entirely.

---

## How the App JAR Enters the Container

```yaml
# docker-compose.yml
app:
  build:
    context: .
    dockerfile: Dockerfile
```

```dockerfile
# Dockerfile
FROM bellsoft/liberica-runtime-container:jre-21-slim-musl
WORKDIR /app
COPY /<module>/target/<artifact>.jar /app/<artifact>.jar
EXPOSE 8080
ENTRYPOINT ["java", "-Dspring.profiles.active=prod", "-jar", "/app/<artifact>.jar"]
```

The Dockerfile copies the JAR from `<module>/target/` at image build time — hence the build prerequisite above.

---

## DockerComposeContainer — What It Does

`ContainerRunner.kt` uses Testcontainers' `DockerComposeContainer` class to manage the compose lifecycle from within the JVM:

```kotlin
// ContainerRunner.kt
private val environment =
    DockerComposeContainer(File("../docker-compose.yml"))
        .withExposedService("postgres", 5432, Wait.forListeningPort())
        .withExposedService("mailserver", 8025, Wait.forListeningPort())
        // Optional: only if your project uses Keycloak
        .withExposedService("keycloak", 8080, Wait.forLogMessage(".*Keycloak.*started.*\\n", 1))
        .withExposedService("app", 8080, Wait.forLogMessage(".*Started <AppName>ApplicationKt.*\\n", 1))

init {
    environment.start()
}
```

`environment.start()` does the equivalent of `docker compose up --build`. Testcontainers:
1. Builds the `app` image from the Dockerfile (using the JAR already on disk).
2. Starts all services.
3. Blocks until each `withExposedService` wait strategy is satisfied.

**Wait strategies:**
- `Wait.forListeningPort()` — TCP port accepts connections.
- `Wait.forLogMessage(regex, 1)` — container stdout matches the regex at least once.

After startup, `DockerComposeContainer` maps each container's internal port to a random free port on the host:

```kotlin
val applicationUri: String
    get() = "http://${environment.getServiceHost("app", 8080)}:${environment.getServicePort("app", 8080)}"
```

**Tests never hardcode `localhost:8080`** — the port is random and only known after containers start.

---

## Why `@Testcontainers` Is NOT Used

`@Testcontainers` + `@Container` automatically calls `start()` / `stop()` around each test class. This project deliberately avoids it for two reasons:

**1. Speed.** The compose stack (especially app startup) takes ~30–60 seconds. With `@Testcontainers`, that cost is paid once per test class — dozens of times for a large suite. The manual singleton approach pays it exactly once per JVM process.

**2. Shared seed data.** The `BaseIntegTest.initialized` block creates seed data once, shared across every test class as fixtures. If containers restarted between test classes, seed data would be gone.

---

## The Singleton Pattern

`ContainerRunner` is instantiated exactly once via Kotlin's `by lazy` delegate:

```kotlin
// BaseIntegTest.kt (companion object)
val containerRunner: ContainerRunner by lazy { ContainerRunner() }
val baseUrl: String by lazy { containerRunner.applicationUri }
val emailMessagesUrl: String by lazy { "http://localhost:${containerRunner.getMailServerPort()}/api/v2/messages" }

val initialized: Boolean by lazy {
    createUsers()
    createResources()
    createRelationships()
    // ... all seed setup via HTTP against the running app
    true
}
```

Every test class calls `init { initialized }` from `BaseIntegTest`:

```kotlin
open class BaseIntegTest {
    init {
        initialized  // lazy; executes only on first access across the entire JVM
    }
}
```

**Sequence on first test run:**

1. JUnit 5 instantiates the first test class (extends `BaseIntegTest`).
2. `init` accesses `initialized`.
3. `initialized` (lazy) → triggers `containerRunner` (lazy) → `ContainerRunner()` → `environment.start()` → blocks until stack is up.
4. Once containers are up, `initialized` runs seed setup functions via HTTP.
5. `initialized` becomes `true` and is cached.
6. Every subsequent test class returns `true` immediately — no re-initialization.

JUnit 5 creates a new instance per test method by default, but the `companion object` holding the lazy properties lives for the entire JVM process, so containers and seed data survive across all test methods and classes.

---

## MailHog — How Email-Based Flows Work

The `mailserver` service (MailHog) is an SMTP trap. The Spring Boot app sends email to `mailserver:1025` (internal Docker network). MailHog stores everything in memory and exposes an HTTP API at port `8025`.

Tests that need to read emails (activation codes, invitation tokens) call helpers that hit `http://localhost:<mapped-8025>/api/v2/messages`, parse the most recent message, and extract the token using a regex.

**Important:** MailHog accumulates all emails during the test run. `items[0]` is the most recent message. This works when flows are sequential — one email is sent, then immediately read, then the next is sent.

---

## Startup Sequence Summary

```
mvn package (build JAR)
        │
        ▼
ContainerRunner() ──► docker compose up --build
                             │
                             ├─ postgres (port 5432) ──► Wait.forListeningPort()
                             ├─ mailserver (port 8025) ──► Wait.forListeningPort()
                             ├─ [keycloak (port 8080)] ──► Wait.forLogMessage("Keycloak started")  [optional]
                             │      └─ [keycloak-ready] polls /realms/<realm> until 200, exits      [optional]
                             └─ app (port 8080) ──► Wait.forLogMessage("Started <AppName>ApplicationKt")
                                    └─ built from Dockerfile (copies <module>/target/<artifact>.jar)
        │
        ▼
BaseIntegTest.initialized runs seed setup (HTTP calls against app)
        │
        ▼
Tests run — all share the same live containers and seed data
```
