# PRD-001: Exfiltrator API Server — Fly.io + Managed Postgres

**Date:** 2026-02-24
**Status:** Draft
**Author:** Claude Opus 4.6 + mason

---

## 1. Goal

Stand up the minimum viable Exfiltrator API server on Fly.io with managed Postgres, authenticated against the existing `auth.axhxrx.com` SSO system, with a database schema that supports user profiles and artifact metadata. This is the foundational infrastructure that everything else (CLI uploads, dashboard UI, etc.) builds on.

**What "done" looks like:** A deployed Bun container on Fly.io that can:
1. Validate a user's session via `auth.axhxrx.com`
2. Look up or create the user's Exfiltrator profile in Postgres
3. Expose a health check endpoint and a `GET /api/me` endpoint that returns the authenticated user's profile
4. Run Tern migrations against managed Postgres

---

## 2. Context & Constraints

### Existing Auth System

The SSO system is a Deno Deploy app at `auth.axhxrx.com` backed by Deno KV. Key facts:

- **User identity:** email (canonical, lowercase). Stored in Deno KV as `User { email, hashedPassword, createdAt, lastLoginAt, emailVerified }`.
- **Session tokens:** Opaque, cryptographically random 64-hex-char strings. Stored in Deno KV with 7-day sliding TTL.
- **Cookie:** Named per environment (`sso-local`, `sso-development`, `sso-production`). Domain `.axhxrx.com` in prod. `SameSite: None`, `Secure: true`.
- **Validation endpoint:** `GET /api/session/validate?token=<token>` returns `{ valid: true, email: "..." }` or `{ valid: false }`. This is how apps that can't access auth's KV directly validate sessions — Panopticon already uses this pattern.
- **Shared library:** `@axhxrx/sso` provides `RUNTIME_CONFIGURATION`, `middlewareAuth`, `middlewareJwt`, `createMiddlewareSession`, and session CRUD functions.

### Integration Pattern (from Panopticon)

Panopticon is the existing reference implementation for cross-app SSO. Its pattern:

1. Read the SSO cookie from the request (cookie name from `RUNTIME_CONFIGURATION`)
2. Call `auth.axhxrx.com/api/session/validate?token=<token>` server-to-server
3. On success, set `authenticatedUser` in the Hono context to the email
4. Use `middlewareAuth` to enforce authentication (redirects to auth or returns 401)

**Key difference for Exfiltrator:** Panopticon runs on Deno Deploy and uses `@axhxrx/sso` directly. The Exfiltrator API runs on Fly.io with Bun. It cannot import `@axhxrx/sso` (Deno-only, uses `Deno.Kv`). Instead, it must implement the session validation call itself — a simple `fetch` to the validate endpoint, exactly as `multron/src/lib/auth-server.ts` already does.

### Runtime & Stack

Per DESIGN.md decisions:

| Layer | Choice |
|-------|--------|
| Runtime | Bun |
| Framework | Hono |
| Database | Fly.io Managed Postgres |
| Query builder | Kysely |
| Migrations | Tern |
| API protocol | tRPC (future — plain Hono routes for V1 MVP) |
| Object store | Tigris (future — not in this PRD) |

### Why Hono

- Runs on Bun natively
- Same framework as the existing auth system (familiar patterns)
- Lightweight, fast, type-safe middleware
- The SSO middleware patterns already exist in Hono

---

## 3. Scope

### In Scope

1. **Fly.io app scaffold** — `fly.toml`, `Dockerfile`, deploy config
2. **Bun + Hono API server** — minimal HTTP server
3. **Auth middleware** — session validation via `auth.axhxrx.com`
4. **Postgres connection** — Kysely setup, connection pooling
5. **Tern migrations** — migration tooling and initial schema
6. **Database schema** — users, nodes tables (minimum viable)
7. **API endpoints:**
   - `GET /health` — health check (unauthenticated)
   - `GET /api/me` — returns current user's Exfiltrator profile (authenticated)
8. **Local dev setup** — Docker Compose with Postgres for local development

### Out of Scope (future PRDs)

- Tigris / object storage / presigned URLs
- Artifact upload flow
- CLI tool (`exfiltrator sync`)
- Dashboard UI (React + TanStack Router)
- tRPC (overkill for 2 endpoints; adopt when the API grows)
- Node enrollment
- Rate limiting (follow auth's pattern when needed)

---

## 4. Database Schema

### Design Principles

- **Stripe-style prefixed UUIDs** for all entity IDs (e.g., `usr_`, `node_`, `art_`)
- **Email is the join key to the auth system** — no foreign keys to Deno KV, just the email string
- **Timestamps are `timestamptz`** — always UTC, Postgres-native
- **Soft delete where useful** — `deleted_at` column pattern

### Tables

#### `users`

The Exfiltrator-specific user profile. Created on first authenticated access (lazy registration). The auth system owns identity (email/password); this table owns app-specific state.

```sql
CREATE TABLE users (
  id            TEXT PRIMARY KEY,          -- 'usr_' + uuid v7
  email         TEXT NOT NULL UNIQUE,      -- canonical email, lowercase, matches auth system
  display_name  TEXT,                      -- optional friendly name
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ                -- soft delete
);

CREATE INDEX idx_users_email ON users (email);
```

**Why a separate users table?** The auth system's `User` record lives in Deno KV and contains only auth concerns (password hash, login timestamps). Exfiltrator needs app-specific fields (display name, preferences, quotas, etc.) that don't belong in the auth system. The `email` column is the stable link between the two.

**Lazy registration:** When a user hits `GET /api/me` and their email isn't in the `users` table, the API creates a row automatically. No separate signup flow needed — if you can authenticate via SSO, you're a valid Exfiltrator user.

#### `nodes`

Enrolled machines in the user's fleet. Not in this PRD's endpoint scope, but included in the schema because it's the next thing after profiles and the schema should be designed holistically.

```sql
CREATE TABLE nodes (
  id            TEXT PRIMARY KEY,          -- 'node_' + uuid v7
  user_id       TEXT NOT NULL REFERENCES users(id),
  hostname      TEXT,                      -- machine hostname at enrollment time
  label         TEXT,                      -- user-assigned friendly name
  os            TEXT,                      -- 'darwin', 'linux', 'windows'
  arch          TEXT,                      -- 'x64', 'arm64'
  enrolled_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at  TIMESTAMPTZ,              -- updated on each sync
  deleted_at    TIMESTAMPTZ               -- soft delete (unenroll)
);

CREATE INDEX idx_nodes_user_id ON nodes (user_id);
```

#### `artifacts` (schema only — no endpoints yet)

Metadata for uploaded artifacts. Included here so the schema is forward-looking.

```sql
CREATE TABLE artifacts (
  id            TEXT PRIMARY KEY,          -- 'art_' + uuid v7
  node_id       TEXT NOT NULL REFERENCES nodes(id),
  user_id       TEXT NOT NULL REFERENCES users(id),  -- denormalized for fast queries
  artifact_type TEXT NOT NULL,             -- 'claude_transcript', 'gemini_transcript', etc.
  file_path     TEXT NOT NULL,             -- original path on the node
  file_name     TEXT NOT NULL,             -- basename
  file_size     BIGINT NOT NULL,           -- bytes
  checksum      TEXT NOT NULL,             -- SHA-256 hex
  storage_key   TEXT,                      -- Tigris object key (null until uploaded)
  uploaded_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_artifacts_node_id ON artifacts (node_id);
CREATE INDEX idx_artifacts_user_id ON artifacts (user_id);
CREATE INDEX idx_artifacts_type ON artifacts (artifact_type);
CREATE UNIQUE INDEX idx_artifacts_dedup ON artifacts (node_id, file_path, checksum);
```

The dedup index enforces the server-side deduplication key from DESIGN.md: `(nodeId, artifactPath, checksum)`.

### ID Generation

Use UUID v7 (time-sortable) with a type prefix:

```typescript
import { v7 as uuidv7 } from 'uuid'

function generateId(prefix: string): string {
  return `${prefix}${uuidv7()}`
}

// generateId('usr_')  => 'usr_0192d4f0-7b3a-7...'
// generateId('node_') => 'node_0192d4f0-7b3a-7...'
// generateId('art_')  => 'art_0192d4f0-7b3a-7...'
```

---

## 5. API Design

### `GET /health`

Unauthenticated. Returns `200 { status: "ok" }`. Used by Fly.io health checks.

### `GET /api/me`

Authenticated. Returns the current user's Exfiltrator profile.

**Auth flow:**
1. Read SSO cookie from request
2. Call `auth.axhxrx.com/api/session/validate?token=<token>` server-to-server
3. If invalid → 401
4. Look up user by email in Postgres
5. If not found → create user row (lazy registration), then return it
6. Return user profile

**Response (200):**
```json
{
  "id": "usr_0192d4f0-7b3a-7...",
  "email": "mason@example.com",
  "displayName": null,
  "createdAt": "2026-02-24T10:00:00.000Z",
  "updatedAt": "2026-02-24T10:00:00.000Z"
}
```

**Response (401):**
```json
{ "error": "Unauthorized" }
```

---

## 6. Auth Middleware Implementation

The middleware needs to work on Bun/Hono without depending on `@axhxrx/sso` (which is Deno-specific due to `Deno.Kv`). Follow the same pattern as `multron/src/lib/auth-server.ts`:

```typescript
// Pseudocode — actual implementation will follow Hono middleware patterns

async function validateSession(token: string): Promise<string | null> {
  const authUrl = getAuthServerUrl() // https://auth.axhxrx.com or http://localhost:1974
  const res = await fetch(`${authUrl}/api/session/validate?token=${encodeURIComponent(token)}`)
  if (!res.ok) return null
  const data = await res.json()
  return data.valid ? data.email : null
}
```

**Cookie name:** Must match the environment. Use the same naming convention:
- Local: `sso-local`
- Development: `sso-development`
- Production: `sso-production`

**Environment detection:** Use an env var (`EXFILTRATOR_ENV` or `RUNTIME_CONFIGURATION_NAME` for consistency with the SSO system).

---

## 7. Infrastructure

### Fly.io Configuration

```toml
app = 'exfiltrator-api'
primary_region = 'nrt'  # Tokyo, same as multron

[build]
  # Dockerfile builds the Bun app

[env]
  HOST = '0.0.0.0'
  PORT = '3000'
  NODE_ENV = 'production'

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = 'stop'
  auto_start_machines = true
  min_machines_running = 0
  processes = ['app']

[[vm]]
  size = 'shared-cpu-1x'
  memory = '512mb'
```

### Dockerfile (Bun)

```dockerfile
FROM oven/bun:1 AS base
WORKDIR /app

# Install dependencies
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

# Copy source
COPY . .

# Tern binary for migrations (installed separately)
# ...

EXPOSE 3000
CMD ["bun", "run", "src/server.ts"]
```

### Managed Postgres

- **Plan:** Starter ($72/mo, 1 shared CPU, 1GB RAM, 10GB storage)
- **Region:** `nrt` (Tokyo)
- **Create:** `fly postgres create --name exfiltrator-db --region nrt`
- **Attach:** `fly postgres attach exfiltrator-db --app exfiltrator-api`
- This sets `DATABASE_URL` as a secret on the app automatically

### Tern Migrations

Tern connects via standard Postgres wire protocol. Migrations live in a `migrations/` directory:

```
migrations/
  001_create_users.sql
  002_create_nodes.sql
  003_create_artifacts.sql
```

Run migrations:
- **Locally:** `tern migrate --conn-string $DATABASE_URL --migrations migrations/`
- **On deploy:** Run as a release command in `fly.toml` or as part of the Docker entrypoint

### Local Development

Docker Compose for local Postgres:

```yaml
services:
  postgres:
    image: postgres:17
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: exfiltrator
      POSTGRES_USER: exfiltrator
      POSTGRES_PASSWORD: localdev
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Local auth: Point `AUTH_SERVER_URL` at `http://localhost:1974` (the local auth server). Use cookie name `sso-local`.

---

## 8. Kysely Setup

### Type Definitions

```typescript
import type { Generated, Insertable, Selectable, Updateable } from 'kysely'

interface Database {
  users: UsersTable
  nodes: NodesTable
  artifacts: ArtifactsTable
}

interface UsersTable {
  id: string
  email: string
  display_name: string | null
  created_at: Generated<Date>
  updated_at: Generated<Date>
  deleted_at: Date | null
}

interface NodesTable {
  id: string
  user_id: string
  hostname: string | null
  label: string | null
  os: string | null
  arch: string | null
  enrolled_at: Generated<Date>
  last_seen_at: Date | null
  deleted_at: Date | null
}

interface ArtifactsTable {
  id: string
  node_id: string
  user_id: string
  artifact_type: string
  file_path: string
  file_name: string
  file_size: number
  checksum: string
  storage_key: string | null
  uploaded_at: Generated<Date>
  created_at: Generated<Date>
}
```

### Connection

```typescript
import { Kysely, PostgresDialect } from 'kysely'
import pg from 'pg'

const db = new Kysely<Database>({
  dialect: new PostgresDialect({
    pool: new pg.Pool({
      connectionString: process.env.DATABASE_URL,
      max: 10,
    }),
  }),
})
```

---

## 9. Project Structure

```
exfiltrator/
  src/
    server.ts           # Hono app entry point
    db/
      connection.ts     # Kysely connection setup
      types.ts          # Database type definitions
      id.ts             # Prefixed UUID generation
    middleware/
      auth.ts           # Session validation middleware
    routes/
      health.ts         # GET /health
      me.ts             # GET /api/me
  migrations/
    001_create_users.sql
    002_create_nodes.sql
    003_create_artifacts.sql
  Dockerfile
  fly.toml
  docker-compose.yml    # Local dev Postgres
  package.json
  tsconfig.json
```

---

## 10. Environment Variables

| Variable | Local | Production | Notes |
|----------|-------|------------|-------|
| `DATABASE_URL` | `postgres://exfiltrator:localdev@localhost:5432/exfiltrator` | Set by `fly postgres attach` | Standard Postgres connection string |
| `AUTH_SERVER_URL` | `http://localhost:1974` | `https://auth.axhxrx.com` | Where to validate session tokens |
| `RUNTIME_CONFIGURATION_NAME` | `local` | `production` | Determines cookie name (`sso-{env}`) |
| `PORT` | `3000` | `3000` | HTTP listen port |

---

## 11. Security Considerations

- **Session tokens are validated server-to-server** — the Exfiltrator API never sees passwords or handles login. It delegates entirely to `auth.axhxrx.com`.
- **Cookie name must match** — if the API reads the wrong cookie name, auth silently fails. The `sso-{environment}` convention must be respected.
- **DATABASE_URL is a secret** — set via `fly secrets set`, never in `fly.toml`.
- **Lazy user creation is safe** — if someone can pass session validation, they're a legitimate user in the auth system. Creating an Exfiltrator profile for them is a business decision, not a security concern.
- **No direct Deno KV access** — Fly.io can't reach Deno Deploy's KV. The `/api/session/validate` endpoint is the only integration point. If auth is down, Exfiltrator returns 401s (fail-closed).

---

## 12. Rollout Plan

1. **Schema + Kysely types** — Write the Tern migrations and Kysely type definitions
2. **Local dev stack** — Docker Compose + Bun server running locally, validating against local auth
3. **Endpoints** — Implement `/health` and `/api/me` with auth middleware
4. **Fly.io deploy** — Create the app, attach Postgres, deploy, run migrations
5. **Smoke test** — Log in at `auth.axhxrx.com`, hit `exfiltrator-api.fly.dev/api/me`, confirm profile creation

---

## 13. Open Questions (for discussion)

### Should `exfiltrator-api` live at `exfiltrator.axhxrx.com` or `exfiltrator-api.fly.dev`?

- If it's at `*.axhxrx.com`, the SSO cookie is automatically sent by the browser (domain `.axhxrx.com`), enabling future dashboard integration without CORS issues.
- If it's at `*.fly.dev`, the cookie won't be sent automatically — the CLI would need to extract and forward it, and the dashboard would need CORS + `credentials: 'include'`.
- **Recommendation:** Set up a custom domain `exfiltrator.axhxrx.com` pointing at the Fly.io app. This is trivial (CNAME record + `fly certs add`) and makes everything simpler.

### Should the API also serve the dashboard SPA?

- DESIGN.md says the dashboard lives in the existing Astro app. But the Astro app runs on Deno Deploy, and the API runs on Fly.io.
- Options: (a) Dashboard in the Astro app, calls exfiltrator API cross-origin; (b) Dashboard served by the exfiltrator API itself; (c) Dashboard is a separate Fly.io app.
- **Recommendation:** Start with (a) — dashboard in Astro, API on Fly.io, CORS between them. Revisit if the split causes friction.

### CLI auth: cookie forwarding vs. API keys?

- The CLI runs headlessly. It can't open a browser to get an SSO cookie.
- DESIGN.md suggests "generate an API key in the web UI and paste it into the CLI" for V1.
- This PRD doesn't implement API keys, but the `users` table is designed to support them later (add an `api_keys` table with `user_id` FK).
- **For now:** The API validates SSO cookies only. API key support is a follow-up PRD.
