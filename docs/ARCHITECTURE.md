# Snap — Architecture

AI-guided low-code platform: users architect data schemas, roles, and dashboards through Google-Forms-like wizards powered by LLMs. The platform *generates real Postgres tables* (codegen), enforces org/row/field permissions, and serves auto-generated CRUD UI — with sandboxed JS plugin extensibility. Deployable anywhere via Docker/Podman, all services open-source.

## Guiding principles

1. **LLM emits structured metadata only** — all SQL/codegen is deterministic, testable code
2. **Strict Ports & Adapters layering** — app logic never touches infra libraries
3. **Platform-agnostic** — same services locally and in prod (Postgres, Valkey, MinIO, Ollama); no Neon/Upstash-specific code
4. **Production-safe schema evolution** — human-reviewed diffs, additive-first migrations
5. **Accessibility by default** — shadcn/ui (Radix primitives) + semantic tokens

## Core abstraction

```
Organization
└── App (e.g., "Car Maintenance Tracker")
    ├── Objects (entity types, e.g., Vehicle, Service Record)
    │   └── Fields (name, type, required, options, relations)
    ├── Roles & Permissions (per object: CRUD, field-level, row-level)
    ├── UI/UX (auto-generated forms, lists, views from schema)
    ├── Dashboards (chart/widget specs)
    └── Plugins (sandboxed JS hooks & custom endpoints)
```

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Runtime | Next.js 15 (App Router) + TypeScript | self-hostable, Vercel-deployable |
| ORM/DB | Drizzle ORM + Postgres 16 | typed, SQL-first, works with raw RLS/DDL |
| Auth | Better Auth (self-hosted) | OSS, session-based, extendable |
| Queue | BullMQ on Valkey 8 | OSS, mature, retries/scheduling |
| Cache | Valkey (ioredis) | Redis drop-in, OSS |
| Storage | MinIO (S3-compatible) | OSS object storage, presigned uploads |
| LLM | Vercel AI SDK (MIT) via OpenAI-compatible `LlmPort` | same adapter for Ollama (dev) and OpenAI/any (prod) |
| Sandbox | `isolated-vm` | hard memory/time limits |
| DI | Hand-rolled `Container` | zero deps, full control |
| Validation | Zod (shared: UI ↔ API ↔ codegen ↔ AI) | single source of truth for schemas |
| UI | shadcn/ui (Radix + Tailwind CSS v4) | accessible primitives, source-owned components, semantic tokens |
| Server state | TanStack Query | caching, mutations, optimistic updates, pagination |
| Data tables | TanStack Table | virtualized, sort/filter/column-visibility |
| Forms | TanStack Form | typed, async validation, field-level state |
| Charts | shadcn Chart (wraps Recharts) | dashboard widgets |
| Icons | lucide-react | standard shadcn icon set |
| Logging/Obs | pino + OpenTelemetry | structured, traceable |
| Tests | Vitest + Playwright + Testcontainers (Podman) | unit/integration/e2e |
| Containers | Podman + Compose (ships Dockerfiles) | machine-native, any-host deploy |

## Repository layout (Ports & Adapters)

```
snap/
├── docker-compose.yml            # dev + prod services (podman compose up)
├── Dockerfile                    # multi-stage, dev & prod targets
├── package.json / tsconfig.json / drizzle.config.ts
├── src/
│   ├── core/                     # PURE app logic. No infra imports. Ever.
│   │   ├── auth/                 #   orgs, memberships, app access use cases
│   │   ├── apps/                 #   app CRUD, publish flow
│   │   ├── schema/               #   field model, zod schemas, validation, diffing
│   │   ├── codegen/              #   DDL + RLS + registry generators (pure fns)
│   │   ├── permissions/          #   role matrix → row/field rule compilation
│   │   ├── runtime/              #   query builder, record CRUD orchestration
│   │   ├── ai/                   #   the 4 wizard orchestrators (LLM-driven)
│   │   ├── dashboards/           #   widget spec model + renderer mapping
│   │   └── plugins/              #   hook registry, lifecycle, contracts
│   ├── ports/                    # INTERFACES ONLY (the seams)
│   │   ├── DatabasePort.ts       #   ConnectionPort, TransactionPort, QueryPort
│   │   ├── ObjectStoragePort.ts
│   │   ├── QueuePort.ts          #   enqueue/worker/retry/schedule
│   │   ├── CachePort.ts
│   │   ├── LlmPort.ts            #   complete<T>(zod) + stream
│   │   ├── SandboxPort.ts        #   execute(code, ctx) with limits
│   │   ├── AuthPort.ts
│   │   └── EmailPort.ts          #   transactional mail (SMTP/Resend)
│   ├── adapters/                 # IMPLEMENTATIONS, swappable
│   │   ├── postgres/             #   Drizzle+pg, SET LOCAL session, migrations
│   │   ├── minio/                #   S3 SDK
│   │   ├── bullmq/               #   Valkey-backed queue worker
│   │   ├── valkey/               #   cache
│   │   ├── openai-compatible/    #   Ollama & OpenAI & any OpenAI-compatible
│   │   ├── isolated-vm/          #   plugin sandbox
│   │   └── better-auth/          #   session/auth
│   ├── composition/              # DI ROOT
│   │   ├── container.ts          #   hand-rolled Container
│   │   ├── tokens.ts             #   TOKENS constants
│   │   ├── env.ts                #   env var config (zod-validated)
│   │   ├── bindings.prod.ts / bindings.test.ts
│   │   └── server.ts             #   createContainer(): request-scoped root
│   ├── app/                      # Next.js entry (THIN ROUTES ONLY)
│   │   ├── (marketing)/          #   landing, docs
│   │   ├── orgs/[org]/           #   apps list, builder UI (wizards/editor)
│   │   ├── a/[appSlug]/          #   runtime UI (generated forms/lists)
│   │   └── api/                  #   thin handlers → core services
│   └── generated/                # per-app typed registry (gitignored)
├── worker/                       # BullMQ consumer entry (codegen, migrations,
│                                #   plugin host, scheduled jobs)
└── packages/contracts/           # shared zod schemas (UI + API + codegen + AI)
```

**Enforcement rules:** `core/` + `ports/` may not import `adapters/`. ESLint rule (`no-restricted-imports`) makes this a build failure, not a convention.

## Hand-rolled DI container

```ts
// composition/container.ts — ~60 lines, zero deps
type Factory<T> = (c: Container) => T;

export class Container {
  #bindings = new Map<symbol | string, { factory: Factory<unknown>; singleton: boolean }>();
  #singletons = new Map<symbol | string, unknown>();

  register<T>(token: Token, factory: Factory<T>, opts?: { singleton?: boolean }): void;
  resolve<T>(token: Token): T;            // throws descriptive error on missing binding
  createChild(): Container;               // request-scoped overlay
}
```

- **Tokens** (`composition/tokens.ts`): symbols — `TDatabasePort`, `TQueuePort`, `TLlmPort`, `TSandboxPort`, `TCurrentUser`, `TCurrentOrg`, etc.
- **Request scoping:** each HTTP request resolves from a child container where `TCurrentUser`/`TCurrentOrg`/`TRequestContext` are bound fresh — this is how RLS session context and permission decisions stay per-request with zero global state.
- **Lifecycle:** per-environment binding sets (`bindings.prod.ts` vs `bindings.test.ts`), each mapping every port to an adapter or fake. If `resolve` finds an unbound port, startup fails loudly.
- **Why hand-rolled:** ~60 lines, no decorators/magic, trivially auditable; swappable for tsyringe behind the same tokens.

## Dev infrastructure (Podman + Compose)

```yaml
# docker-compose.yml (single source of truth for dev AND prod)
services:
  postgres:   image: postgres:16   # + healthcheck; volume for data
  valkey:     image: valkey/valkey:8
  minio:      image: minio/minio   # console :9001, storage :9000
  ollama:     image: ollama/ollama # optional, for offline LLM dev
  worker:     build: . target: prod   command: node worker/index.js  # BullMQ consumer
  app:        build: . target: dev    # next dev (hot reload)
```

Setup (Podman on macOS):
```
podman machine init && podman machine start   # one-time
podman compose up -d postgres valkey minio ollama
```

- **Ports/adapters read env** (`composition/env.ts`, zod-validated): `DATABASE_URL`, `REDIS_URL`, `S3_*`, `LLM_BASE_URL`, `AUTH_SECRET`, etc. Local and prod configs are the same variables — only values change. No Neon/Upstash-specific code anywhere.
- **Migrations:** platform tables via Drizzle (dev: push/generate; prod: applied by worker on boot). App-generated tables are created by the codegen pipeline, not by Drizzle migrations.

## Platform data model (Postgres, Drizzle-managed)

```
organizations (id, name, slug uq, created_at)
users (id, email uq, name, ...)              ← managed by Better Auth tables
memberships (id, org_id fk, user_id fk, role: owner|admin|member, uq(org,user))

apps (id, org_id fk, slug uq(org), name, description, status: draft|live,
      current_version_id fk, created_at, updated_at)

schema_versions (id, app_id fk, version, schema jsonb,   -- canonical snapshot
      status: pending|applied|failed|superseded, applied_at, created_by, uq(app,version))

objects (id, app_id fk, version_id fk, name, label, table_name,   -- runtime lookup
      uq(app,name))

roles (id, app_id fk, name, description, is_system)              -- owner/admin/member seeded
role_permissions (id, role_id fk, object_id fk, can_create, can_read,
      can_update, can_delete, field_rules jsonb, row_rules jsonb, uq(role,object))
app_role_assignments (id, app_id fk, user_id fk, role_id fk, uq(app,user))

migrations (id, app_id fk, version_id fk, sql text, status: pending|running|applied|failed,
      error, applied_at)

dashboards (id, app_id fk, name, layout jsonb, created_by)
dashboard_widgets (id, dashboard_id fk, type: chart|metric|table|list, spec jsonb, config jsonb)

plugins (id, app_id fk, name, description, hooks jsonb, code text,
      status: draft|active|disabled|failed, version, created_by)
```

**Canonical source of truth:** the `schema_versions.schema` JSON snapshot (objects → fields → relations). `objects` + `table_name` are denormalized for fast runtime lookup. Diffing is done against prior versions' JSON.

**Field type model** (shared zod):
```
text(short|long) · number(int|float) · money · date · datetime · boolean ·
single_select · multi_select · file · relation(1:n) · relation(n:m) · computed · identifier
```
Each carries: `required, unique, default, options, helpText, validation, index`.

## Codegen engine (core/codegen — pure, deterministic, no LLM)

Input: `schema_versions.schema` + app slug → Output: DDL SQL + RLS SQL + registry rows.

**Generated table shape** (object `service_record` in app `carbuddy`):
```sql
CREATE TABLE carbuddy_service_record (
  id uuid PK default gen_random_uuid(),
  org_id uuid NOT NULL, created_by uuid NOT NULL, updated_by uuid,
  created_at timestamptz DEFAULT now(), updated_at timestamptz DEFAULT now(),
  deleted_at timestamptz,                     -- soft delete
  _version integer DEFAULT 1,                 -- optimistic lock
  service_date date, odometer integer, cost numeric(12,2),
  service_type text, notes text, vehicle_id uuid  -- 1:n FK
);
CREATE INDEX ON carbuddy_service_record (org_id, created_at);
```
Many-to-many → `carbuddy_<objA>_<objB>` join tables.

**Generated RLS** (per table, compiled from role_permissions + org):
```sql
ALTER TABLE carbuddy_service_record ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation ON carbuddy_service_record
  USING (org_id = current_setting('app.org_id')::uuid)
  WITH CHECK (org_id = current_setting('app.org_id')::uuid);
CREATE POLICY role_read_own ON carbuddy_service_record FOR SELECT
  USING (created_by = current_setting('app.user_id')::uuid);   -- from row_rules
```

**Pipeline:** `SchemaSqlGenerator` (pure: version → DDL diff) → `MigrationService` (applies via dedicated connection in a transaction; updates `migrations`/`schema_versions`/`apps.current_version_id`) → `RegistryWriter` (syncs `objects`, `table_name`, field metadata for runtime).

**Diffing & safety:**
- Diff = object/field added/removed/modified, computed as JSON path comparison
- **Additive-only by default** (add table/column, widen numeric/text, add index)
- Destructive ops (drop, narrow type, remove relation) are **flagged**, require explicit "confirm destructive" and are implemented as *soft* (column renamed `_old_*`, table kept, data preserved)
- LLM proposal → user sees a **human-readable diff** → confirm → version applied. A failed migration never un-lives the previous version.

**Typed registry:** persisted in metadata tables, cached in-memory in `SchemaRegistry` (keyed by app slug). At "export code" time, generates a standalone Drizzle + Next.js project.

## Permission engine & runtime

**Resolution flow per request:**
```
session (AuthPort) → user → org from app membership → app role
→ load role_permissions → build role context
```

**Three enforcement layers:**
| Scope | Enforcement |
|---|---|
| Org isolation | RLS policy on every table (`org_id`) |
| Row-level | RLS `USING`/`WITH CHECK` from `row_rules` predicates |
| Field-level | Query builder: SELECT column allow-list; INSERT/UPDATE column filtering |

**QueryBuilder** (pure, tested): takes `object, roleContext, {filters, sort, page}` → parameterized SQL with allow-listed columns, org scoping, soft-delete filtering. Executed in a transaction that first runs:
```sql
SET LOCAL app.org_id = $1; SET LOCAL app.user_id = $2;
```

**Runtime API** (thin routes → core service):
```
POST   /api/a/{app}/objects/{object}            create
GET    /api/a/{app}/objects/{object}            list  (filter/sort/page)
GET    /api/a/{app}/objects/{object}/{id}       read
PUT    /api/a/{app}/objects/{object}/{id}       update (optimistic lock on _version)
DELETE /api/a/{app}/objects/{object}/{id}       delete (soft)
GET    /api/a/{app}/objects/{object}/aggregate  sum/avg/count/groupBy  (dashboards)
```

## LLM layer

```ts
interface LlmPort {
  complete<T>(o: { messages; schema: ZodSchema<T>; model? }): Promise<T>;  // structured output
  stream(o): AsyncIterable<string>;
}
```
Adapter `OpenAiCompatibleAdapter` — base URL + model from env. Same code drives **Ollama (dev)** and **OpenAI/any (prod)**. Structured outputs enforced via zod → JSON schema; nonconforming responses are rejected and retried.

**Four wizards** (each = `use case` + `draft provider` + `review UI`):
1. **SchemaWizard** — NL → `{ objects: [{name, label, fields[]}] }` → rendered as a Google-Forms-like field editor → confirm → diff → apply. Multi-turn refinement.
2. **PermissionWizard** — NL → proposed role × object CRUD matrix + row/field rules → review table → save.
3. **DashboardWizard** — NL → `WidgetSpec[]` (source, measure, agg, groupBy, filter, chartType) → live preview → persist.
4. **PluginWizard** — NL → plugin code scaffold against the typed hook contract → user edits → sandbox-test → activate.

**Safety invariants:**
- LLM produces *metadata/specs only*. DDL, RLS, SQL are generated by deterministic code.
- Every AI output passes zod validation before any side effect.
- Every schema change requires human diff approval.
- AI features are logged (prompt, model, latency, rejection) via pino + OTel spans.

## Plugin system

```ts
type PluginContext = { db: TypedDbPort; user; app; config; logger };
type Plugin = {
  hooks: {
    beforeCreate?(input, ctx); afterCreate?(rec, ctx); beforeUpdate?; afterUpdate?;
    beforeDelete?; afterDelete?; validate?(input, ctx) → errors[];
    computedFields?: Record<string, (rec, ctx) => Promise<unknown>>;
    endpoints?: [{ path, method, handler(ctx) }];
  };
  config?: { allowedDomains?: string[]; network?: boolean };
};
```

**Sandbox (`IsolatedVmAdapter`):**
- `isolated-vm` with memory cap (128 MB), execution timeout (5 s), explicit import allowlist injected into the VM, network disabled unless declared (`network: true` → proxy fetch with domain allowlist)
- Plugins load from `plugins.code`, versioned, cached compiled
- Hook execution runs in the **worker** process (Node, not Vercel edge — `isolated-vm` is a native module); lightweight hooks may run in-process with hard timeout via a fast-path flag

**Lifecycle:** draft → (sandbox unit-test in editor) → active → disabled/failed on repeated errors. A hook failure rolls back the surrounding transaction.

## Runtime UI & dashboards

- **Forms/lists/detail views** auto-generated from object metadata: field type → component mapping (`single_select` → `RadioGroup`/`ToggleGroup`, `text_long` → `Textarea`, `file` → upload w/ presigned URL, `relation` → `Combobox`, `boolean` → `Switch`).
- **Role-aware rendering:** field allow-list and row filters applied server-side; UI hides unreadable fields.
- **Dashboards:** `WidgetSpec` → renderer mapping (chart → shadcn Chart/Recharts; metric; table/list). Aggregates via the `/aggregate` endpoint. Drag-drop layout in builder.
- **Routing:** builder at `/orgs/[org]/apps/[app]/*`, generated runtime at `/a/[appSlug]/*` with the same session/auth.

### shadcn conventions

- Components in `src/app/components/ui/*` (shadcn-managed) + `src/app/components/features/*` (builder, wizard, runtime views)
- Form layout via `FieldGroup` + `Field`; validation via `data-invalid` on `Field` + `aria-invalid` on controls
- Semantic tokens (`bg-background`, `text-muted-foreground`); `gap-*` over `space-y-*`; `cn()` for conditionals
- `Dialog`/`Sheet`/`Drawer` always have titles; `sonner` for toasts; `Alert`, `Empty`, `Skeleton`, `Badge`, `Separator` over custom markup
- Icons via `data-icon`; no manual sizing classes on component icons

## Frontend architecture

- **TanStack Query** wraps all `/api/*` calls; runtime CRUD views use it for list pagination, optimistic create/update/delete, and dashboard aggregate queries
- **TanStack Table** powers: record list views (sort/filter/page), role-permission matrix editor, schema field lists
- **TanStack Form** powers: schema editor fields, wizard review screens, generated runtime forms (field type → mapped form component per the metadata model)
- RSC for server components, `"use client"` on interactive components

## Testing strategy

| Layer | Tool | Covers |
|---|---|---|
| Unit | Vitest | core modules with fake adapters (fake DB/LLM/queue) |
| Contract | Vitest, same suite both impls | port behavior parity fake ↔ real adapter |
| Integration | Testcontainers (Podman) + real Postgres | migrations, RLS cross-org denial, diff/rollback |
| E2E | Playwright | wizard flows, generated UI, sandbox plugin |
| Determinism | golden-file tests | DDL/RLS generation output snapshots |

## Milestones

- **M0** Dev stack + skeleton: compose, Dockerfile, scaffold, Container/tokens/env, ports + fakes, shadcn init + TanStack setup, Drizzle + platform schema
- **M1** Foundations: Better Auth, orgs, memberships, app CRUD, field model, schema editor (no LLM)
- **M2** Codegen engine: DDL/RLS generators, diff, migration runner + worker, registry, query builder, runtime CRUD API
- **M3** Wizards 1 & 2: LlmPort + Ollama, SchemaWizard, roles/permissions wizards
- **M4** Runtime UI: generated forms/lists/details, role-aware rendering, uploads
- **M5** Dashboards: widgets, aggregate endpoint, DashboardWizard
- **M6** Plugins: isolated-vm sandbox, hook registry, plugin editor + wizard, export-code
- **M7** Polish: templates gallery, tenant limits, security audit, backup/restore, docs, load tests

## Key risks & mitigations

- **`isolated-vm` vs Vercel edge** → plugin host is the worker container (Node), never edge
- **RLS + connection pooling** → `SET LOCAL` inside each transaction; integration tests assert isolation
- **Long DDL in serverless** → migrations run in worker with dedicated connection, not request handlers
- **LLM hallucinated schemas** → structured outputs + zod + human diff approval before any DDL
- **Sandbox escape / DoS** → hard caps, allowlist imports, network gated by declaration
- **Schema migration at scale** → additive-first, soft-destructive, version rollback, golden tests