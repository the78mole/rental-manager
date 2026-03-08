# rental-manager

Web application for rental management (landlords, tenants, contracts, utility billing) with:

- FastAPI backend
- React frontend
- Keycloak (OIDC)
- PostgreSQL + Redis
- Caddy reverse proxy

## Stack Initial Setup (Docker Compose)

### Automated setup (recommended)

From repository root:

```bash
./scripts/setup-stack.sh --with-demo-accounts
```

What it does:

- builds and starts the full compose stack
- ensures the `keycloak` Postgres DB exists (also for older volumes)
- waits for critical services to be ready
- creates demo accounts for all roles

### 1. Start the full stack

Run from repository root:

```bash
docker compose up -d --build
```

Check status:

```bash
docker compose ps
```

Expected: `db`, `redis`, `rustfs`, `step-ca`, `keycloak` become `healthy`; `api`, `web`, `caddy`, `worker`, `beat` are `Up`.

### 2. Recovery steps if initial startup is incomplete

If `api` or `caddy` are stuck in `Created`:

```bash
docker compose up -d api caddy
```

If `keycloak` is unhealthy and logs show `database "keycloak" does not exist`:

```bash
./scripts/setup-stack.sh
```

If you changed `keycloak/rental-realm.json`, rebuild Keycloak image:

```bash
docker compose up -d --build keycloak
```

### 3. Access URLs

Local:

- App: `http://localhost`
- Keycloak: `http://localhost/auth`

GitHub Codespaces:

- Open forwarded port `80`
- Use URL like: `https://<codespace-name>-80.app.github.dev`

## Example Accounts For All Roles

Create sample users for all app roles:

- `SUPER_ADMIN`
- `ADMIN`
- `OPERATOR`
- `CARETAKER`
- `LANDLORD`
- `TENANT`

It creates users in Keycloak and synchronizes admin sub-roles in PostgreSQL.

```bash
./scripts/create-demo-accounts.sh
```

Default demo credentials:

- `SUPER_ADMIN`: `super-admin@example.com` / `SuperAdminTest2026`
- `ADMIN`: `test-admin@example.com` / `AdminTest2026`
- `OPERATOR`: `test-operator@example.com` / `OperatorTest2026`
- `CARETAKER`: `hausverwalter@example.com` / `HausverwalterTest2026`
- `LANDLORD`: `demo-landlord@example.com` / `LandlordTest2026`
- `TENANT`: `demo-tenant@example.com` / `TenantTest2026`

### Role behavior in UI

- `SUPER_ADMIN`: may manage `SUPER_ADMIN`, `ADMIN`, `OPERATOR`
- `ADMIN`: may manage `ADMIN` and `OPERATOR`
- `OPERATOR`: may manage landlord tenants only (no admin user management)
- `CARETAKER`: has landlord-area access (`/landlord/*`) to assigned buildings/apartments
- `LANDLORD`: landlord area
- `TENANT`: tenant area

## Domain Structure (Building and Apartment)

The platform now models real estate in two explicit layers:

- `Building` (`Property` in DB/API compatibility layer): physical building object
- `Apartment` (`Unit` in DB/API compatibility layer): rentable unit inside a building

Compatibility:

- Existing `/landlord/properties` and `/landlord/properties/{id}/units` endpoints remain available.
- New semantic aliases are available as well:
  - `/landlord/buildings`
  - `/landlord/buildings/{building_id}/apartments`

### Multiple tenants per apartment (WG / shared flats)

An apartment can have multiple tenants via multiple contracts, e.g. one contract per person.

- `Contract` references `unit_id` (apartment) and `tenant_id`.
- There is no one-tenant-only constraint on apartments.
- This allows parallel and/or overlapping individual contracts for shared living scenarios.

### Caretaker assignment model

Caretakers can be assigned either:

- to a whole building (`caretaker_building_assignments`), or
- to a specific apartment (`caretaker_apartment_assignments`).

Landlords/Admins can manage assignments via landlord API endpoints:

- `POST/DELETE /landlord/buildings/{building_id}/caretakers/{caretaker_id}`
- `POST/DELETE /landlord/apartments/{apartment_id}/caretakers/{caretaker_id}`

## Backend Local Development (without Compose)

The backend uses [uv](https://docs.astral.sh/uv/) for dependency management.

```bash
cd backend
uv sync
uv run uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Add a dependency:

```bash
cd backend
uv add <package>
```
