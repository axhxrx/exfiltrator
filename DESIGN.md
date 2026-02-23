# Exfiltrator — Design Notes

## Core Concept

- Users operate **fleets of machines/VMs** (potentially 100+) running various agents and tasks
- Each machine generates local artifacts: logs, transcripts, git activity, scan results, etc.
- **Exfiltrator** is an agent that runs on each node, detects new/changed artifacts, and uploads them to a centralized service under the user's account
- The centralized service provides a unified view: what happened, on which machine, when, by which agent

## Data Types (non-exhaustive)

| Category | Example artifacts | Size profile |
| -------- | ---------------- | ------------ |
| Claude Code transcripts | JSONL conversation files | Potentially very large (MBs each) |
| Gemini transcripts | Conversation logs | Similar to Claude |
| OpenAI/Codex transcripts | Chat transcription files | Similar to Claude |
| Custom agent logs | Bespoke logs from user-built agentic coding loops | Varies |
| Git activity | Repos with recent commits, branches, diffs | Small metadata, large diffs |
| Agent deployments | Which agents ran, what they did | Small-medium |
| Security scans | Scan results, vulnerability reports | Medium |
| System events | Updates, reboots, config changes | Small |

## First Target: Claude Code Transcripts

The initial implementation focuses on Claude Code JSONL transcript files:

- Located in the local Claude Code history directory (exact path TBD)
- Each file is a JSONL conversation transcript
- Files can be large
- Need to detect new/changed files and upload only what hasn't been seen

Future artifact sources (Gemini, OpenAI Codex, custom agent logs) will follow, each as its own artifact type with its own scan logic. For V1, artifact types are hardcoded. Over time, each artifact type will likely become its own operation with a standard interface.

## Decisions

### Storage Strategy

- **V1**: Store raw JSONL blobs in object storage, index only metadata (file name, machine, timestamp, size, checksum) in the database
- **Later**: Parse and index JSONL contents into the database for search/filtering

### Local State: SQLite

- Each node keeps a local SQLite database tracking what has been uploaded
- Tracks: `{ path, checksum, size, lastModified, lastUploaded, remoteRef }`
- Supports both immutable (write-once) and mutable (append-growing) files — checksumming detects changes either way

### File Mutability

- Must handle both cases: files written once and never changed, and files that grow over time (e.g., logs, JSONL that gets appended to)
- For V1, re-upload the whole file when checksum changes (simple, correct)
- Incremental/append-only upload optimization can come later

### Node Identity & Enrollment

- To join the fleet, a user **runs our CLI tool on the machine** to enroll it
- Enrollment generates a **prefixed UUID** (e.g., `node_a1b2c3d4e5f6...`), persisted locally
- All entity IDs use **Stripe-style prefixed UUIDs** for debuggability — when you're staring at a weird row in prod two years from now, the prefix tells you immediately what kind of thing it is (e.g., `node_`, `art_`, `usr_`)
- Friendly node names (e.g., "dev-laptop", "ci-runner-3") will be added later as a mapping layer on top of the stable UUID
- The node authenticates against the **existing auth system** (cookie-based SSO on `*.axhxrx.com`, Deno KV-backed, opaque session tokens)
- After enrollment, the agent runs autonomously using stored credentials + node ID

### Node-Side Runtime: Bun CLI

- V1 is a **Bun TypeScript CLI tool** that runs on all platforms (macOS, Linux, Windows)
- macOS will eventually get a control plane GUI app; other platforms stay CLI
- Scope will expand later, but CLI is the V1 form factor

### Viewing UI: React Dashboard

- A React SPA page inside the existing Astro web app, at a route like `/dashboard`
- Shows a table of all collected artifacts
- Filterable by: machine, artifact type, time range, agent
- Allows the user to see fleet-wide activity at a glance

### Upload Trigger

- V1: **manual CLI invocation** (`exfiltrator sync`)
- The manual command is the primitive that any future automation (cron, watch, daemon) will call

### Auth Integration

- Existing auth system: Deno Deploy + Deno KV, cookie-based SSO on `.axhxrx.com`
- User model: email (canonical ID) + hashed password, opaque session tokens (32 bytes, 7-day sliding TTL)
- Apps integrate via `@axhxrx/sso` middleware or the `/api/session/validate` endpoint
- For the CLI: will need a login flow that obtains a session token and persists it locally (or a long-lived API key mechanism for headless nodes)
- User email is the stable ID that maps to application-level data; the auth system stays simple

## Cloud Infrastructure: Fly.io

Decision: Fly.io (Machines + Managed Postgres + Tigris)

| Component | Service | Notes |
| --------- | ------- | ----- |
| API layer | Fly Machines | Containerized Bun/Deno, per-second billing |
| Database | Managed Postgres | Full Postgres, ~$72/mo (Starter) |
| Object store | Tigris | S3-compatible, $0.02/GB/mo, zero egress |
| Migrations | Tern | Standard Postgres migration tool |
| Query builder | Kysely | Type-safe SQL query builder |
| API protocol | tRPC | Type-safe API layer |

**Why Fly.io + Postgres over Cloudflare D1:**

- **Postgres confidence** — strong preference for Postgres over SQLite for any database purpose. Postgres is a known quantity with no size ceilings, real transactions, and mature tooling.
- **Stack alignment with day job** — using Kysely + Tern + Postgres mirrors the work stack. Every hour spent here compounds into day-job proficiency and vice versa. This is a pragmatic, high-leverage reason.
- **Tern just works** — Tern connects via standard Postgres wire protocol. With Fly, run `fly proxy 5432` locally or run migrations as a deploy hook. Same workflow as any other managed Postgres.
- **No per-user DB migration headaches** — single shared DB with row-level tenant isolation. Schema changes are one migration, not a script iterating over hundreds of databases.
- **No artificial limits** — no 10 GB ceiling, no write serialization, no 30s CPU time limits for large uploads.
- **Traditional server model** — simpler mental model for a Bun backend. Can run background jobs, crons, long-running uploads natively.
- **No vendor lock-in** — everything is bog-standard. Postgres is Postgres, Tigris is S3-compatible, the Bun API server is just a container. Could migrate off Fly with minimal pain if needed.

**Cost:** ~$95-105/mo minimum ($72 Postgres + $5.70 Machine + storage). Not cheap like D1's ~$5/mo, but well within reason for a real product.

### Cloudflare was also considered

Cloudflare (Workers + D1 + R2) was evaluated as an alternative. D1 is absurdly cheap (~$5/mo even at 1,000 users) and the per-user database isolation is elegant. However, the D1 limitations (10 GB hard cap, write serialization, per-user migration tooling) combined with the practical value of matching the day-job Postgres stack tipped the decision toward Fly.io.

If Fly's costs become a problem or Postgres feels like overkill, Cloudflare remains a viable fallback. The CLI and blob storage layers are decoupled from the DB choice.

### API & Web Stack

| Layer | Choice | Notes |
| ----- | ------ | ----- |
| Query builder | Kysely | Type-safe, works great with Postgres |
| Migrations | Tern | Standard Postgres migration tool |
| API protocol | tRPC | Type-safe RPC — server and client share types |
| Client-side router | TanStack Router | React-first, type-safe routing |
| Dashboard frontend | React | Inside existing Astro app (see below) |
| Other web UIs | SolidJS / Astro | Existing pages stay as they are |

**Why this stack:** Every layer (Kysely, Tern, Postgres, tRPC, TanStack Router, React) mirrors the day-job stack. This is a deliberate choice for knowledge compounding — hours spent here build proficiency that transfers directly to work, and vice versa. The ~$90/mo extra over Cloudflare pays for itself in accelerated learning.

**The Astro + React + TanStack Router integration:**

The existing web app architecture is Astro, which already hosts pages in multiple frameworks (SolidJS, React, static). This stays unchanged. The exfiltrator dashboard will be a React SPA living at a dedicated Astro route. The proven pattern:

1. Catch-all Astro route: `src/pages/dashboard/[...page].astro`
2. Root React component mounted with `client:only="react"` (avoids hydration conflicts)
3. TanStack Router configured with `basepath: "/dashboard"` handles all sub-routing within the SPA
4. TanStack Router Vite plugin added to Astro's Vite config

Astro's React integration is fully router-agnostic — it just mounts components into islands. Multiple real-world implementations confirm this works (see: declanlscott.com, bjoernf.com, github.com/sookmax/astro-tanstack-router). This means existing SolidJS pages, static pages, and mixed-framework pages are completely unaffected.

## Architecture Overview (V1)

```text
┌─────────────────────┐         ┌──────────────────────────────┐
│  Node (machine/VM)  │         │  Fly.io                      │
│                     │         │                              │
│  ┌───────────────┐  │         │  ┌────────────────────────┐  │
│  │ exfiltrator   │──┼─────┐   │  │  tRPC API (Bun)        │  │
│  │ (Bun CLI)     │  │     │   │  │  - presign endpoint    │  │
│  └───────┬───────┘  │     │   │  │  - metadata endpoint   │  │
│          │          │     │   │  └──────────┬─────────────┘  │
│  ┌───────▼───────┐  │     │   │             │                │
│  │ local SQLite  │  │     │   │  ┌──────────▼─────────────┐  │
│  │ (upload state)│  │     └──▶│  │  Tigris (JSONL blobs)  │  │
│  └───────────────┘  │  direct │  │  (presigned URL upload) │  │
│                     │  upload │  └────────────────────────┘  │
│  ┌───────────────┐  │         │             │                │
│  │ ~/.claude/    │  │         │  ┌──────────▼─────────────┐  │
│  │ transcripts   │  │         │  │  Postgres (Kysely)     │  │
│  └───────────────┘  │         │  │  metadata + manifests  │  │
└─────────────────────┘         │  └────────────────────────┘  │
                                └──────────────────────────────┘
                                           │
                                ┌──────────▼──────────────┐
                                │  Astro Web App          │
                                │  /dashboard/* = React   │
                                │    TanStack Router      │
                                │    tRPC client          │
                                │  (other pages: Solid,   │
                                │   static, mixed)        │
                                └─────────────────────────┘
```

## Upload Flow (V1)

1. CLI scans known artifact directories (e.g., `~/.claude/` for transcripts)
2. For each file found, compute SHA-256 checksum
3. Check local SQLite: has this `(path, checksum)` been uploaded?
4. If not, call a tRPC procedure to get a **presigned Tigris upload URL**
5. CLI uploads the blob **directly to Tigris** via the presigned URL (avoids streaming large files through the API server)
6. On successful upload, CLI calls a tRPC procedure to **record the metadata** in Postgres (via Kysely)
7. On success, CLI records the upload in local SQLite
8. Deduplication key on the server: `(nodeId, artifactPath, checksum)`

## TODO

- [ ] **Postgres schema design** — sketch out the actual tables (nodes, artifacts, users, etc.)
- [ ] **Local dev story** — figure out how to run the full stack locally. Options: (a) Docker Compose with Postgres + mock Tigris (lowest common denominator, always works), (b) something better if Fly.io offers dev tooling worth using. Key principle: the Fly stack is bog-standard, no magic — we should be able to simulate it locally without Fly-specific tools.

## Open Questions

### Tigris bucket topology

- Single bucket with per-user key prefixes (`/{userId}/{nodeId}/...`) is simplest
- One bucket per user is also possible and provides stronger isolation
- All metadata about blobs lives in Postgres; the Tigris key is just an opaque blob ID

### CLI auth flow for headless nodes

- The SSO system uses browser cookies on `.axhxrx.com` — doesn't work for a CLI on a headless VM
- Options: (a) device-code-style flow, (b) `exfiltrator login` opens a browser/prints a URL, user approves, CLI gets a long-lived token, (c) generate an API key in the web UI and paste it into the CLI
- Option (c) is simplest for V1

### Transcript file locations

- Claude Code: somewhere under `~/.claude/` — exact path and structure TBD
- Gemini: location TBD
- OpenAI Codex: location TBD
- Custom agent logs: user-configured paths
- These locations may change over time as the AI tools evolve; the scan logic needs to be updatable
