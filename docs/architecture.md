# Architecture

Rental Manager is a fully containerised, self-hosted application. All components run as
Docker services orchestrated by Docker Compose, fronted by a Caddy reverse proxy.

## System Overview

```mermaid
graph LR
    subgraph Client
        Browser
    end

    subgraph Edge ["Edge (port 80)"]
        Caddy["Caddy\nReverse Proxy"]
    end

    subgraph App ["Application"]
        Web["React\nFrontend"]
        API["FastAPI\nBackend"]
    end

    subgraph Auth ["Authentication"]
        KC["Keycloak\nOIDC Provider"]
    end

    subgraph Data ["Data Layer"]
        DB[("PostgreSQL\n• rental DB\n• keycloak DB")]
        Redis[("Redis\nTask Queue")]
        S3["RustFS\nS3 Storage"]
    end

    subgraph Workers ["Background Workers"]
        Worker["Celery Worker\n(billing, tasks)"]
        Beat["Celery Beat\n(scheduler)"]
    end

    Browser -->|"HTTP :80"| Caddy
    Caddy -->|"/"| Web
    Caddy -->|"/api/v1"| API
    Caddy -->|"/auth"| KC

    Web -->|"OIDC login"| KC
    Web -->|"API calls + JWT"| API

    API -->|"validates JWT"| KC
    API --- DB
    API --- Redis
    API --- S3

    Worker --- DB
    Worker --- Redis
    Worker --- S3
    Beat --- Redis
```

## Authentication Flow

```mermaid
sequenceDiagram
    participant Browser
    participant Caddy
    participant Keycloak
    participant API

    Browser->>Caddy: GET /
    Caddy-->>Browser: React SPA

    Browser->>Keycloak: OIDC Authorization Request
    Keycloak-->>Browser: Login form
    Browser->>Keycloak: Credentials
    Keycloak-->>Browser: Authorization code

    Browser->>Keycloak: Token exchange (code → access_token + id_token)
    Keycloak-->>Browser: JWT access_token (with roles)

    Browser->>Caddy: GET /api/v1/... Bearer <token>
    Caddy->>API: Proxy request
    API->>Keycloak: JWKS fetch (cached)
    API-->>API: Validate JWT signature + claims
    API-->>Browser: 200 OK / 403 Forbidden
```

## Request Routing

```mermaid
flowchart TD
    Request["Incoming Request\n:80"] --> Caddy

    Caddy -->|"path: /auth/*"| KC["Keycloak :8080"]
    Caddy -->|"path: /api/v1/*"| API["FastAPI :8000"]
    Caddy -->|"all other paths"| Web["Caddy :80 → /srv\n(React build)"]

    API -->|"async ORM"| PG["PostgreSQL\nrental DB"]
    API -->|"task dispatch"| Redis
    API -->|"S3 PutObject/GetObject"| RustFS

    Redis --> Worker["Celery Worker"]
    Worker --> PG
    Worker --> RustFS
```

## Tech Stack

| Layer           | Technology            | Notes                                       |
| --------------- | --------------------- | ------------------------------------------- |
| Frontend        | React 18 + Vite       | OIDC via `oidc-client-ts`                   |
| Backend         | FastAPI + Python 3.12 | Async, `uvicorn`                            |
| Auth            | Keycloak 26           | Realm `rental`, OIDC/OAuth2                 |
| Database        | PostgreSQL 16         | Two databases: `rental_manager`, `keycloak` |
| Cache / Queue   | Redis 7               | Celery broker + result backend              |
| Object Storage  | RustFS                | S3-compatible, for documents                |
| Reverse Proxy   | Caddy 2               | Automatic HTTPS, routing                    |
| TLS CA          | step-ca               | Internal CA for local HTTPS                 |
| Dependency Mgmt | uv                    | Python lockfile-based installs              |
| Container       | Docker Compose v2     | Requires Docker API ≥ 1.44                  |

## Docker Services

```mermaid
graph TD
    subgraph "Always starts first"
        db["db\n(postgres:16)"]
        redis["redis\n(redis:7)"]
        rustfs["rustfs\n(rustfs/rustfs)"]
        stepca["step-ca\n(smallstep/step-ca)"]
    end

    subgraph "Starts after DB is healthy"
        keycloak["keycloak\n(custom build)"]
    end

    subgraph "Starts after Keycloak is healthy"
        api["api\n(Python/FastAPI)"]
        worker["worker\n(Celery)"]
        beat["beat\n(Celery Beat)"]
    end

    subgraph "Frontend + Edge"
        web["web\n(React/Caddy)"]
        caddy["caddy\n(Caddy)"]
    end

    db -->|"healthcheck"| keycloak
    db -->|"healthcheck"| api
    redis -->|"healthcheck"| api
    keycloak -->|"healthcheck"| api
    rustfs -->|"healthcheck"| api
    api -->|"ready"| caddy
    web -->|"ready"| caddy
```
