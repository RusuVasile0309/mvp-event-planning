# Event Planning Platform — Architecture & Code-Style Guide

> This document describes how the codebase is structured, which patterns to follow, and the exact
> conventions to write code in. The platform is a **multi-tenant, white-label SaaS** with a
> **runtime permission matrix** and a **VIP-table canvas**.
> Companion docs: [`PRD.md`](./PRD.md) (features) · [`TECHNOLOGIES.md`](./TECHNOLOGIES.md) (the stack).

---

## 1. Philosophy & Guiding Principles

Every feature is a **vertical slice through the same fixed layers**. Learn one feature, you know them all.

1. **Strict layering.** Data flows through fixed layers and never skips one:
   `Page → Hook → API client → Next.js Route → Service → Database`.
2. **Type safety end-to-end.** Types are defined once (in `@app/shared` or as Zod schemas in the
   server) and reused across frontend and backend. Never redeclare a shape.
3. **Validate at the boundary.** Every API route parses its input with a **Zod** schema before
   touching a service. Services trust their inputs; routes do not.
4. **Tenant-safe by construction.** Every tenant-owned query is scoped by `company_id`, and Postgres
   RLS is the backstop. A service method that touches tenant data **always** takes `companyId` first.
5. **Permission-gated by default.** Access is decided by the runtime permission matrix — there are no
   hardcoded "admin-only" areas. No View → hidden from nav *and* URL blocked.
6. **Thin routes, fat services.** Routes do auth + tenant resolution + permission check + validation +
   response shaping. All business logic and SQL live in service classes.
7. **One responsibility per file.** `*.api.ts` = fetch calls · `*.hooks.ts` = React Query ·
   `*.service.ts` = business logic + SQL · `*.schema.ts` = Zod + types.
8. **Co-location.** Page-specific components live next to the page. Only truly shared primitives live
   in the global `components/` folder.

---

## 2. Monorepo Structure

An npm workspace with **three packages** and TypeScript project references:

```
event-platform/
├── package.json            # workspace root, shared scripts
├── tsconfig.json           # project references → shared, server, web
├── .prettierrc             # formatting rules (authoritative)
├── docs/                   # long-form documentation (this file, PRD, technologies)
└── packages/
    ├── shared/             # @app/shared — types, enums, constants, pure utils
    ├── server/             # @app/server — services, DB, schemas, integrations
    └── web/                # @app/web    — Next.js app (UI + API routes)
```

**Dependency direction (never violate):**

```
web  ──depends on──▶  server  ──depends on──▶  shared
 └──────────────────depends on──────────────────▶ shared
```

- `shared` depends on nothing internal — pure, isomorphic (runs on client and server).
- `server` depends on `shared`. It owns the database, RLS, integrations.
- `web` depends on both; it imports services directly inside API routes.

---

## 3. Package: `@app/shared`

**Purpose:** Isomorphic types, enums, constants, and *pure* business logic used by both sides.

```
shared/src/
├── types/
│   ├── auth.types.ts               # SessionUser, JwtPayload
│   ├── user.types.ts               # User, CompanyRole, SuperAdmin
│   ├── permission.types.ts         # Feature, Capability, PermissionMatrix
│   ├── database.types.ts
│   └── index.ts
├── constants/
│   ├── roles.ts                    # CompanyRole enum (EVENT_ADMIN, EVENT_MANAGER, …)
│   ├── features.ts                 # Feature enum (EVENTS, TASKS, VENDORS, CONTRACTS, FINANCE, PR, …)
│   ├── capabilities.ts             # Capability enum (VIEW, CREATE, EDIT, DELETE)
│   ├── permission-defaults.ts      # sensible per-role default matrix (the "standard ladder")
│   ├── themes.ts                   # the 6 predefined themes (palette + font pairing ids)
│   ├── table-types.ts              # VIP table capacities: 4, 6, 8, 10
│   ├── statuses.ts                 # TaskStatus, VendorEventStatus, ContractStatus, ExpenseStatus…
│   └── index.ts
├── utils/
│   ├── permissions.ts              # pure can(role, feature, capability) helpers over a matrix
│   ├── consumption.ts              # pure: capacity × zone price-per-seat
│   ├── money.ts                    # integer-minor-unit money helpers
│   └── index.ts
└── index.ts                        # export * from every barrel
```

**Rules**

- No React, no Node-only APIs, no DB calls — must be safe to bundle into the browser.
- Business rules that must agree on both sides (permission checks, table consumption, money math)
  live here so client and server never disagree.
- Consumers import from the package root: `import { CompanyRole, Feature, can } from '@app/shared';`

---

## 4. Package: `@app/server`

**Purpose:** All backend logic — service classes, DB client, Zod schemas, RLS setup, integrations.
It is a library (not a running server); `web`'s API routes call into it.

```
server/src/
├── config/
│   └── env.ts                      # typed accessor over process.env (single source)
├── database/
│   ├── db.ts                       # Neon client (lazy) + per-request tenant context (RLS GUC)
│   ├── tenant.ts                   # withTenant(companyId) helper — sets app.company_id
│   ├── schemas/                    # Zod schemas + inferred types, one file per entity
│   │   ├── company.schema.ts
│   │   ├── user.schema.ts
│   │   ├── permission.schema.ts
│   │   ├── event.schema.ts
│   │   ├── task.schema.ts
│   │   ├── vendor.schema.ts
│   │   ├── contract.schema.ts
│   │   ├── expense.schema.ts
│   │   ├── zone.schema.ts          # VIP zone (name, price_per_seat)
│   │   ├── table.schema.ts         # placed table (type, x, y, label, holder, status…)
│   │   ├── content.schema.ts       # PR content item
│   │   └── index.ts
│   ├── migrations/                 # NNN_description.sql (tables + RLS policies), run in order
│   ├── migrate.ts / run-migrate.ts
│   └── index.ts
├── modules/                        # one folder per feature domain
│   ├── auth/           auth.service.ts          # JWT sign/verify, password hash, invites
│   ├── companies/      companies.service.ts     # provisioning, suspend/reactivate (Super Admin)
│   ├── permissions/    permissions.service.ts   # read/write the per-company matrix
│   ├── users/          users.service.ts         # team roster, invite, deactivate
│   ├── events/         events.service.ts
│   ├── tasks/          tasks.service.ts
│   ├── vendors/        vendors.service.ts
│   ├── contracts/      contracts.service.ts     # metadata; files via storage
│   ├── finance/        finance.service.ts       # expenses + partial payments
│   ├── vip/            zones.service.ts + tables.service.ts   # VIP zones & placed tables
│   ├── pr/             content.service.ts + media.service.ts  # calendar + media library
│   ├── storage/        storage.service.ts       # Cloudflare R2 (S3 SDK), presigned URLs
│   └── email/          email.service.ts         # Resend (invites, resets)
└── index.ts                        # barrel: re-exports services + schemas
```

### 4.1 Service layer conventions

- One **class per domain**, named `<Domain>Service`; methods are **`static`**.
- **Tenant rule:** every method that reads/writes tenant data takes **`companyId` as its first
  argument** and includes `WHERE company_id = ${companyId}` (belt); RLS is the suspenders.
- Methods take plain typed inputs and return schema-inferred types.
- Services own the SQL — import `sql` from `../../database/db` and use tagged templates.
- Cross-domain calls are allowed by importing another service (e.g. `FinanceService` reads a
  `ContractsService` record; `TablesService` reads its `ZonesService` price-per-seat).
- Domain constants (e.g. a `VALID_TRANSITIONS` map for status machines) live at the top of the file.
- **Super-Admin methods are separated** — cross-tenant queries live in `CompaniesService` and are
  never mixed with tenant-scoped methods.

```typescript
// server/src/modules/events/events.service.ts
import { sql } from '../../database/db';
import { EventSchema, EventCreateInput } from '../../database/schemas';

export class EventsService {
  static async list(companyId: string): Promise<EventSchema[]> {
    return (await sql`
      SELECT * FROM events WHERE company_id = ${companyId} ORDER BY event_date DESC
    `) as EventSchema[];
  }

  static async getById(companyId: string, eventId: string): Promise<EventSchema | undefined> {
    const [event] = (await sql`
      SELECT * FROM events WHERE id = ${eventId} AND company_id = ${companyId}
    `) as EventSchema[];
    return event; // undefined if it belongs to another company → route returns 404
  }

  static async create(companyId: string, input: EventCreateInput): Promise<EventSchema> {
    const [event] = (await sql`
      INSERT INTO events (company_id, name, event_date, start_time, end_time, location_name, location_lat, location_lng, description)
      VALUES (${companyId}, ${input.name}, ${input.eventDate}, ${input.startTime}, ${input.endTime},
              ${input.locationName}, ${input.locationLat}, ${input.locationLng}, ${input.description ?? null})
      RETURNING *
    `) as EventSchema[];
    return event;
  }
}
```

### 4.2 Database schemas (Zod-first)

Each entity file defines **three tiers** and exports inferred types:

1. **Database schema** — the row shape from the DB (`eventSchema`), including `companyId`.
2. **Input schemas** — what goes *into* the API, with validation messages (`eventCreateSchema`).
3. **Inferred types** — `export type EventSchema = z.infer<typeof eventSchema>`.

```typescript
export const eventSchema = z.object({
  id: z.string().uuid(),
  companyId: z.string().uuid(),
  name: z.string(),
  eventDate: z.string(),          // one day (date-only)
  startTime: z.string(),          // HH:mm
  endTime: z.string(),
  locationName: z.string(),
  locationLat: z.number(),
  locationLng: z.number(),
  description: z.string().nullable(),
  createdAt: z.date(),
  updatedAt: z.date(),
});
export type EventSchema = z.infer<typeof eventSchema>;
```

### 4.3 Migrations & RLS

- Plain `.sql` files in `database/migrations/`, ordered (`001_…`, `002_…`), run in filename order.
- Schema changes are **always** a new migration file — never edit an applied one.
- **Every tenant table's migration also enables RLS and adds the tenant policy:**

```sql
CREATE TABLE events (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  company_id  uuid NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
  name        text NOT NULL,
  event_date  date NOT NULL,
  -- …
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_events_company ON events(company_id);

ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON events
  USING (company_id = current_setting('app.company_id', true)::uuid);
```

### 4.4 Database client & tenant context

- `db.ts` exposes a lazily-initialised Neon `sql` tagged-template client.
- `tenant.ts` exposes `withTenant(companyId, fn)` which runs
  `SELECT set_config('app.company_id', ${companyId}, true)` at the start of the request so RLS is
  armed for every subsequent query. Tenant-scoped API routes wrap their service calls in it.
- Super-Admin (cross-tenant) routes deliberately **do not** set a tenant and use a role that bypasses
  RLS for the platform-overview tables.

### 4.5 Environment access

All env vars are read once, in `config/env.ts`, into a typed `env` object. Application code imports
`env`, never `process.env` directly.

```typescript
export const env = {
  DATABASE_URL: process.env.DATABASE_URL || '',
  JWT_SECRET: process.env.JWT_SECRET || '',
  SESSION_MAX_AGE: Number(process.env.SESSION_MAX_AGE ?? 60 * 60 * 24 * 7),
  RESEND_API_KEY: process.env.RESEND_API_KEY || '',
  R2_ACCOUNT_ID: process.env.R2_ACCOUNT_ID || '',
  GEOAPIFY_API_KEY: process.env.GEOAPIFY_API_KEY || '',
  // …
};
```

---

## 5. Package: `@app/web`

**Purpose:** The Next.js 15 App-Router application — pages, layouts, UI, the network layer, and the
API routes that expose server services over HTTP. **No `[locale]` segment** (English-only).

```
web/
├── middleware.ts                   # JWT session check + route protection (no locale rewrite)
└── app/
    ├── (auth)/                     # login, signup (invite-accept), reset-password
    ├── (platform)/                 # Super-Admin console (companies, platform overview)
    ├── (workspace)/                # the company workspace (all 5 roles)
    │   ├── dashboard/              # Home / My Dashboard — notification center
    │   ├── events/                 # events list + [eventId]/ detail hub
    │   │   └── [eventId]/          # details, team, tasks, vip-tables (zones + canvas)
    │   ├── tasks/                  # global tasks (by-member + kanban board)
    │   ├── vendors/  contracts/  finance/  pr/
    │   ├── team/                   # Team & Users roster + invites
    │   ├── permissions/            # the Roles × Features matrix screen
    │   ├── settings/               # theme, logo, currency
    │   └── components/             # components shared across workspace pages
    ├── api/                        # Next.js route handlers (the HTTP layer)
    │   ├── auth/                   # login, logout, refresh, accept-invite, reset
    │   ├── companies/              # Super-Admin: provision, suspend, list (cross-tenant)
    │   ├── permissions/            # read/update the matrix
    │   ├── events/  tasks/  vendors/  contracts/  finance/  vip/  pr/  users/
    │   └── ...
    ├── network/                    # ⭐ client-side API layer (see §6)
    │   └── <feature>/  <feature>.api.ts + <feature>.hooks.ts + index.ts
    ├── components/                 # ⭐ GLOBAL shared UI only (shadcn/Radix primitives)
    ├── stores/                     # Zustand stores (client UI state)
    ├── hooks/                      # cross-page hooks (useIsMobile, usePermissions, useSession)
    ├── helpers/                    # pure client helpers (dates, maps, money)
    ├── lib/                        # providers.tsx (TanStack Query), utils.ts (cn)
    ├── theme/                      # ThemeProvider — sets CSS variables from company theme id
    └── seo/                        # metadata helpers
```

### 5.1 Routing model

- **No locale prefix** — English-only, so routes are plain (`/events`, `/tasks/…`).
- **Route groups** `(auth)`, `(platform)`, `(workspace)` organise pages by audience without adding
  URL segments, each with its own `layout.tsx`.
- **API routes** live under `app/api/` and are the HTTP boundary (see §7).

### 5.2 Component organisation

- **Page-specific components** → a `components/` folder next to the page/route group.
- **Global shared components** → `app/components/` — reusable primitives (Button, Dialog, Select,
  Input, Calendar, Toast, Badges, skeletons) wrapping shadcn/Radix.
- Promote a component to `app/components/` **only** once a second area needs it.

---

## 6. The Network Layer (frontend data access)

Every backend feature has a matching folder in `app/network/<feature>/` with **two files + a barrel**:

```
network/events/
├── events.api.ts     # fetch functions + request/response TS interfaces
├── events.hooks.ts   # TanStack Query hooks that wrap the api functions
└── index.ts          # re-exports both
```

### 6.1 `*.api.ts` — typed fetch functions

- Define `Payload` / `Response` interfaces alongside the functions.
- Never call `fetch` directly — go through the shared `apiRequest<T>()` helper
  (`network/utils/request.ts`), which sets JSON headers, resolves URLs, and shows a `sonner` toast on
  error (suppressible via `{ suppressToast: true }`).

```typescript
import { apiRequest } from '../utils/request';

export interface CreateEventPayload { name: string; eventDate: string; /* … */ }
export interface EventResponse { id: string; name: string; /* … */ }

export async function createEvent(payload: CreateEventPayload): Promise<EventResponse> {
  return apiRequest<EventResponse>('/api/events', { method: 'POST', body: JSON.stringify(payload) });
}
```

### 6.2 `*.hooks.ts` — TanStack Query wrappers

- Marked `'use client'`.
- `useQuery` for reads, `useMutation` for writes; keep client-side derivation inside the hook.
- **Query-key convention:** stable array, feature-first, general → specific:
  `['events']`, `['events', 'upcoming']`, `['events', id]`, `['tasks', { eventId }]`.
- Mutations `invalidateQueries` the affected keys on success.

```typescript
'use client';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { getEvents, createEvent } from './events.api';

export function useEvents() {
  return useQuery({ queryKey: ['events'], queryFn: getEvents });
}
export function useCreateEvent() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: createEvent,
    onSuccess: () => qc.invalidateQueries({ queryKey: ['events'] }),
  });
}
```

### 6.3 Provider & global error handling

`app/lib/providers.tsx` configures the `QueryClient` (staleTime 60s, no refetch-on-focus) and a
`MutationCache` that toasts **every** mutation error. Combined with `apiRequest`'s toast on failed
reads, you rarely handle errors manually — return an error from the API and the UI surfaces it.

---

## 7. The API Route Layer (HTTP boundary)

Route handlers in `app/api/**/route.ts` are **thin**. For a tenant-scoped route the job is:

1. **Authenticate** — verify the JWT session cookie, resolve the user + their `companyId` + role.
2. **Resolve tenant** — arm RLS via `withTenant(companyId, …)`.
3. **Authorize** — check the permission matrix: does this role have the required `Capability` on this
   `Feature`? (e.g. `EVENTS` + `CREATE`.)
4. **Validate** — `schema.safeParse` the body/query; return `400` with `error.flatten().fieldErrors`.
5. **Delegate** — call the service method (passing `companyId` first).
6. **Respond** — correct status code, JSON body.

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { EventsService, eventCreateSchema } from '@app/server';
import { Feature, Capability } from '@app/shared';
import { getSession, requirePermission, withTenant } from '@/lib/auth';

export async function POST(request: NextRequest) {
  const session = await getSession();                                    // 1. authenticate
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const denied = await requirePermission(session, Feature.EVENTS, Capability.CREATE); // 3. authorize
  if (denied) return denied;                                             // 403 if not allowed

  const validated = eventCreateSchema.safeParse(await request.json());   // 4. validate
  if (!validated.success) {
    return NextResponse.json(
      { error: 'Validation failed', details: validated.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  const event = await withTenant(session.companyId, () =>               // 2 + 5. tenant + delegate
    EventsService.create(session.companyId, validated.data)
  );
  return NextResponse.json(event, { status: 201 });                      // 6. respond
}
```

**Status-code conventions:** `401` unauthenticated · `403` forbidden (no permission) · `400`
validation · `404` not found (incl. wrong-tenant rows) · `409` conflict · `500` unexpected.
Error bodies are always `{ error: string, details?: … }`.

---

## 8. Authentication & Authorization

### 8.1 Auth — one custom JWT system for everyone (decision #1)

All users (Super Admin + the 5 company roles) authenticate the same way:

1. **Provisioning / invites** create a user with no password and email a single-use token (Resend):
   - Super Admin provisions a company → auto-creates the company + its Event Admin → first-login link.
   - Event Admin invites a teammate (first name, last name, email, role) → invite email.
2. The invitee **accepts the invite**, sets a password (hashed with **bcryptjs**), becomes **Active**.
3. **Login** verifies the password and issues a session JWT signed with **`jose`**, set as an
   HTTP-only cookie; `SESSION_MAX_AGE` controls expiry. Refresh/logout are separate `/api/auth/*` routes.
4. The JWT payload carries `userId`, `companyId` (null for Super Admin), and `role`.

`middleware.ts` verifies the session on protected routes: **API** → `401` on failure (never a
redirect); **pages** → redirect unauthenticated users to `/login`. A matcher allow-lists public
routes (`/login`, `/accept-invite`, `/reset-password`).

### 8.2 Authorization — the runtime permission matrix

Access is **entirely permission-driven** — no hardcoded admin-only areas (PRD §5).

- **Model:** per company, a matrix of **Roles × Features**, each cell toggling **View / Create /
  Edit / Delete**. Stored in Postgres (`role_permissions(company_id, role, feature, capability)`),
  seeded from `permission-defaults.ts` at company creation, editable by the Event Admin.
- **Event Admin** holds all permissions by default (a superset) and assigns the rest.
- **Invariant:** Create/Edit/Delete cannot be enabled without View; removing View clears the rest.
  Enforced both in the Permissions screen UI and server-side in `PermissionsService`.
- **View is the gate:** no View on a feature → the section is **hidden from nav** *and* its **API/URL
  is blocked** (`requirePermission` returns `403`; the client `usePermissions` hook hides nav + guards
  pages).
- The Permissions screen is itself a permission-gated feature (no special-casing).

```typescript
// pure check lives in @app/shared so client & server agree
export function can(matrix: PermissionMatrix, role: CompanyRole, feature: Feature, cap: Capability): boolean;
```

- **Server:** `requirePermission(session, feature, capability)` loads the company's matrix (cached)
  and calls `can(...)`; returns a `403` response or `null`.
- **Client:** `usePermissions()` exposes `can(feature, capability)` to hide nav items and disable
  actions; **the server check is the real gate** — the client check is UX only.

### 8.3 Super Admin

The platform owner sits **above** all tenants. Distinguished by a platform flag (no `companyId`),
routed under `(platform)` / `/api/companies/*`, and uses **cross-tenant** service methods that
deliberately bypass the tenant matrix and RLS. Never mixed with workspace routes.

---

## 9. Multi-Tenancy (decision #2, in depth)

Two layers of defence — app-level scoping (belt) + Postgres RLS (suspenders):

| Layer | Mechanism |
|---|---|
| **App-level scoping** | Every tenant table has `company_id`; every service method takes `companyId` first and filters on it. Wrong-tenant reads return `undefined` → route `404`. |
| **RLS backstop** | Every tenant table has RLS enabled + a `tenant_isolation` policy. Each request arms it via `withTenant(companyId)` → `set_config('app.company_id', …)`. A forgotten filter returns zero rows, never another tenant's data. |
| **Super-Admin path** | Separate cross-tenant service methods + a DB role that bypasses RLS, used only by `(platform)` routes. |

**Rule:** if you write a query against a tenant table without `WHERE company_id = ${companyId}`, that's
a bug — even though RLS would save you. Both layers, always.

---

## 10. State Management

Two distinct tools, never mixed:

| Kind of state | Tool | Example |
|---|---|---|
| **Server state** (anything in the DB) | **TanStack Query** via `network/*` hooks | events, tasks, vendors, contracts, expenses, zones/tables |
| **Client/UI state** (ephemeral, UI-only) | **Zustand** stores in `stores/` | canvas selection & tool, active tab, kanban drag state, multi-step form draft |

**Rule:** never cache server data in Zustand. If it comes from an endpoint, it belongs in a Query hook.
The VIP canvas edits table positions locally (Zustand/local state) and **persists via a mutation** —
the saved positions are server state.

---

## 11. White-Label Theming

- **6 predefined themes** (PRD §4), each a palette + font pairing; a company picks one at Quick Setup
  and can change it in Settings. No custom colours in v1.
- Themes are defined once in `@app/shared/constants/themes.ts` (ids + token values).
- `app/theme/ThemeProvider` reads the logged-in company's theme id and sets **CSS custom properties**
  (`--color-primary`, `--font-heading`, …) on the root; Tailwind consumes those variables so the whole
  workspace re-skins with no per-component branching. Fonts are loaded per theme via `next/font`.
- The company **logo** and **currency** are company-level settings applied the same way.

---

## 12. The VIP-Table Canvas (decision #3)

The signature feature (PRD §9). Kept **intentionally simple** — organise & visualise, nothing more.

- **Zones** belong to an event; each has a name and a `price_per_seat`. Each zone is its own canvas.
- **Canvas** built with **react-konva**: a palette of the 4 predefined table types (4/6/8/10 seats);
  the user drags a type onto the Konva stage to add a table; tables are draggable to reposition.
- **Each placed `table` row** stores: `zone_id`, `capacity` (from type), `x`, `y`, `label`,
  `holder_name`, `holder_phone`, `status` (Available/Reserved), optional `guests` (names).
- **Derived consumption** = `capacity × zone.price_per_seat` — computed via the pure helper in
  `@app/shared`, never stored as an editable field.
- **Two views per zone:** the **visual canvas** (primary) and a **table/list view** (grid of the same
  tables with holder, capacity, consumption, status). Both read the same `tables` query.
- **Persistence:** canvas edits mutate the `tables` rows (position + details) through the network
  layer; positions saved on drag-end. Money uses the **company currency**.

> **dnd-kit** is used separately for the **Tasks kanban board** (status columns) — a DOM sortable, not
> a canvas. Don't conflate the two.

---

## 13. Integrations (server modules) — v1

| Module | Integration | Responsibility |
|---|---|---|
| `email/` | **Resend** | Invite emails, company-provisioning links, password resets. |
| `storage/` | **Cloudflare R2** (S3 SDK) | Uploaded contract files + PR media library; presigned up/download URLs. |
| `geocoding/` | **Geoapify** | Address autocomplete + coordinates for event locations. |

Deferred (not in v1): Stripe billing, Twilio WhatsApp/Verify, pdf-lib contract generation, social
auto-publishing. See `TECHNOLOGIES.md §7` and the PRD's deferred list.

---

## 14. Conventions & Code Style

### Formatting (`.prettierrc` — authoritative)

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

`prettier-plugin-tailwindcss` auto-sorts Tailwind classes. Run `npm run format`.

### Naming

| Thing | Convention | Example |
|---|---|---|
| Service class | `PascalCase` + `Service`, static methods | `EventsService.create()` |
| Service / network / schema files | `kebab-case.<role>.ts` | `events.service.ts`, `events.api.ts`, `event.schema.ts` |
| React components | `PascalCase.tsx` | `EventCard.tsx`, `PermissionMatrix.tsx` |
| Hooks | `useX` | `useEvents`, `usePermissions` |
| Zustand stores | `useXStore` | `useCanvasStore` |
| Zod schema / inferred type | `xSchema` / `XSchema` | `eventSchema` → `type EventSchema` |
| DB columns / TS fields | `snake_case` / `camelCase` | `price_per_seat` ⇄ `pricePerSeat` |
| Enums | `PascalCase` name, `UPPER_SNAKE` members | `TaskStatus.IN_PROGRESS`, `CompanyRole.EVENT_ADMIN` |
| Query keys | array, feature-first | `['tasks', { eventId }]` |

### General rules

- Import shared code from package roots (`@app/shared`, `@app/server`), web-internal code via `@/`.
- Comments explain **why**; section large files with `// ==== Types ====` banners.
- Prefer `static` service methods and plain typed objects over stateful classes.
- **Money** is handled in integer minor units (bani for RON); the display currency is company-level.
- **Tenant safety** and **permission checks** are not optional — every tenant route does both.

---

## 15. End-to-End Recipe — Adding a Feature (`vendors`)

The canonical checklist. Follow the layers top-to-bottom; note the tenant + permission steps.

**1. Shared** (constants/types needed on both sides)
```typescript
// packages/shared/src/constants/features.ts  → add VENDORS to the Feature enum + defaults matrix
```

**2. Server: Zod schema + migration (with RLS)**
```typescript
// packages/server/src/database/schemas/vendor.schema.ts
export const vendorSchema = z.object({
  id: z.string().uuid(),
  companyId: z.string().uuid(),
  name: z.string(),
  category: z.string(),
  contactPerson: z.string().nullable(),
  phone: z.string().nullable(),
  email: z.string().email().nullable(),
  notes: z.string().nullable(),
  createdAt: z.date(),
});
export type VendorSchema = z.infer<typeof vendorSchema>;

export const vendorCreateSchema = z.object({
  name: z.string().min(1),
  category: z.string().min(1),
  contactPerson: z.string().optional(),
  phone: z.string().optional(),
  email: z.string().email().optional(),
  notes: z.string().optional(),
});
export type VendorCreateInput = z.infer<typeof vendorCreateSchema>;
```
Add migration `0NN_create_vendors.sql` — table with `company_id`, index, `ENABLE ROW LEVEL SECURITY`,
and the `tenant_isolation` policy.

**3. Server: service (tenant-scoped)**
```typescript
export class VendorsService {
  static async list(companyId: string): Promise<VendorSchema[]> {
    return (await sql`SELECT * FROM vendors WHERE company_id = ${companyId} ORDER BY name`) as VendorSchema[];
  }
  static async create(companyId: string, input: VendorCreateInput): Promise<VendorSchema> {
    const [vendor] = (await sql`
      INSERT INTO vendors (company_id, name, category, contact_person, phone, email, notes)
      VALUES (${companyId}, ${input.name}, ${input.category}, ${input.contactPerson ?? null},
              ${input.phone ?? null}, ${input.email ?? null}, ${input.notes ?? null})
      RETURNING *
    `) as VendorSchema[];
    return vendor;
  }
}
```
Re-export from `server/src/index.ts`.

**4. Web: API route (auth → permission → validate → tenant → delegate)**
```typescript
export async function POST(request: NextRequest) {
  const session = await getSession();
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const denied = await requirePermission(session, Feature.VENDORS, Capability.CREATE);
  if (denied) return denied;

  const validated = vendorCreateSchema.safeParse(await request.json());
  if (!validated.success) {
    return NextResponse.json(
      { error: 'Validation failed', details: validated.error.flatten().fieldErrors },
      { status: 400 }
    );
  }

  const vendor = await withTenant(session.companyId, () =>
    VendorsService.create(session.companyId, validated.data)
  );
  return NextResponse.json(vendor, { status: 201 });
}
```

**5. Web: network api + hooks**
```typescript
// vendors.api.ts
export async function createVendor(payload: CreateVendorPayload): Promise<VendorResponse> {
  return apiRequest<VendorResponse>('/api/vendors', { method: 'POST', body: JSON.stringify(payload) });
}
// vendors.hooks.ts  ('use client')
export function useCreateVendor() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: createVendor,
    onSuccess: () => qc.invalidateQueries({ queryKey: ['vendors'] }),
  });
}
```

**6. Web: use it in a page** — gate the UI with `usePermissions`
```typescript
'use client';
import { useCreateVendor } from '@/network';
import { usePermissions } from '@/hooks';
import { Feature, Capability } from '@app/shared';

export function AddVendorButton() {
  const { can } = usePermissions();
  const { mutate, isPending } = useCreateVendor();
  if (!can(Feature.VENDORS, Capability.CREATE)) return null;   // UX gate; server is the real gate
  return <button disabled={isPending} onClick={() => mutate(/* … */)}>Add vendor</button>;
}
```

---

## 16. Key Principles (Checklist)

1. **Layered flow** — `Page → Hook → api.ts → route.ts → Service → DB`; never skip a layer.
2. **Types once** — shared types in `@app/shared`; DB shapes as Zod schemas in `server`.
3. **Validate at the edge** — every route `safeParse`s its input before delegating.
4. **Tenant-safe always** — `company_id` filter in every tenant query **and** RLS armed via `withTenant`.
5. **Permission-gated always** — `requirePermission` on the server; `usePermissions` for UX. No View → hidden + blocked.
6. **Thin routes, fat services** — routes do auth/tenant/permission/validation/response; services do logic + SQL.
7. **Two-file network layer** — `*.api.ts` (fetch) + `*.hooks.ts` (React Query), one per feature.
8. **TanStack Query for server state, Zustand for UI state** — never blur the two.
9. **Co-locate components** — page-specific next to the page; global primitives in `app/components/`.
10. **Errors surface themselves** — `apiRequest` + mutation cache toast automatically.
11. **Super Admin is separate** — cross-tenant methods and routes, never mixed with workspace/tenant code.
12. **English-only** — no i18n, no locale segment; company-level currency instead.
