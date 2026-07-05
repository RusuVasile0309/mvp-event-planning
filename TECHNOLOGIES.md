# Event Planning Platform — Technology Stack

> **Companion docs:** [`PRD.md`](./PRD.md) (what we're building) · [`ARCHITECTURE.md`](./ARCHITECTURE.md) (how the code is organised).
> **Guiding principle (from the PRD):** *keep it simple / don't overengineer.* Ship a focused MVP.

---

## 1. Locked foundational decisions

Four decisions were made up front because they shape every table, service, and route:

| # | Decision | Choice | Why |
|---|----------|--------|-----|
| 1 | **Auth** | **Custom JWT** — `jose` sessions + `bcryptjs` passwords, all users in our Postgres | We must build the runtime permission matrix in our DB regardless; a hosted auth SaaS (Clerk) wouldn't save that work, would bill per-tenant on our core growth metric, and fits the Super-Admin-above-tenants model poorly. |
| 2 | **Tenant isolation** | **`company_id` scoping in every query + Postgres RLS as a DB-level backstop** | App-level scoping is the everyday pattern; RLS is cheap insurance against the one query that forgets the filter. A cross-tenant leak is existential for a resale platform. |
| 3 | **VIP-table canvas** | **react-konva** (zoomable canvas of draggable tables) + **dnd-kit** (the Tasks kanban board) | Konva is purpose-built for positioned shapes on a canvas (MIT, no watermark); dnd-kit is the accessible DOM drag tool for the kanban board. Two tools, each for its own job. |
| 4 | **i18n** | **None** — English-only UI | The PRD targets one market with a per-company **currency** (e.g. RON), not multi-language. We drop `next-intl` entirely. |

---

## 2. Core

| Technology | Version | Used for |
|---|---|---|
| **TypeScript** | ^5 | Entire codebase; strict types shared across packages via project references. |
| **Next.js** | 15 (App Router) | Web framework — pages, layouts, server components, and API routes in one app. |
| **React** | 19 | UI rendering; heavy use of client components (`'use client'`) for interactive views. |
| **Node / npm workspaces** | — | Monorepo tooling. Three packages under `packages/*` linked by the workspace protocol. |

---

## 3. Frontend

| Technology | Used for |
|---|---|
| **TanStack Query** (`@tanstack/react-query`) | Server-state: fetching, caching, refetching, mutations. The default data layer. |
| **Zustand** | Local/global **client** UI state (canvas selection, active tab, multi-step form drafts). Never for server data. |
| **Tailwind CSS** (v4, `@tailwindcss/postcss`) | All styling. Utility-first; **theme colours/fonts driven by CSS variables** so the 6 white-label themes swap at runtime. |
| **shadcn/ui + Radix UI** | Accessible base UI primitives (`Dialog`, `Select`, `Label`, `Tabs`, `Slot`) in `app/components/`. |
| **react-hook-form + `@hookform/resolvers` + Zod** | Forms and client-side validation (the same Zod schemas as the API where possible). |
| **sonner** | Toast notifications. The single global error/success channel. |
| **lucide-react** | Icon set. |
| **recharts** | Charts — Finance summaries (totals by category, paid vs. outstanding), Super-Admin platform metrics. |
| **react-day-picker** | Date selection — event dates, task start/due dates, PR scheduled dates. |
| **maplibre-gl + `@geoapify/*`** | Map rendering + address autocomplete/geocoding for **event locations** (name + coordinates → open in Google Maps). |
| **react-konva** | The **VIP-table canvas** — drag predefined table types onto a per-zone canvas, position/arrange, persist x/y. |
| **`@dnd-kit/core` + `@dnd-kit/sortable`** | The **Tasks kanban board** — drag tasks between status columns. |
| **@fullcalendar/react** *(or react-big-calendar)* | The **PR & Social** content calendar (month/week views). Final pick during PR-module build. |

> **Deliberately excluded:** i18n (`next-intl`), phone OTP (`input-otp`), bot-protection on public
> forms (`@marsidev/react-turnstile` — the app is entirely behind login in v1), and PWA/offline
> (`@serwist/next` — not needed for an internal back-office tool).

---

## 4. Authentication

Decision #1 — **one custom auth system for everyone** (Super Admin + all 5 company roles).

| Technology | Used for |
|---|---|
| **`jose`** | Sign & verify the session JWT (HTTP-only cookie). |
| **`bcryptjs`** | Hash & verify passwords. |
| **Crypto invite tokens** | Single-use, expiring tokens for **Invite User** (Event Admin → teammate) and **company provisioning** (Super Admin → Event Admin first-login link). |
| **Resend** | Delivers invite / password-reset emails (see §7). |

**Who logs in:** the **Super Admin** (platform owner, above all tenants) and the **5 company roles**
(Event Admin, Event Manager, Coordinator, Finance/Accounts, Marketing/PR). All are internal users —
there is no public/guest/vendor login in v1. Every identity is a row in our `users` table; Super Admin
is distinguished by a platform-level flag, not a company membership.

---

## 5. Backend / Data

| Technology | Used for |
|---|---|
| **Neon** (`@neondatabase/serverless`) | Serverless PostgreSQL, accessed via the `sql` tagged-template driver — **no ORM**. |
| **Postgres Row-Level Security (RLS)** | DB-level tenant backstop — policies constrain every tenant table to `current_setting('app.company_id')`. |
| **Zod** | Database row schemas + API input validation. The single source of truth for shapes. |
| **Raw SQL migrations** | Versioned, ordered `NNN_description.sql` files run in filename order. Schema evolution is explicit and reviewable. RLS policies live in migrations too. |

**Multi-tenancy in the data layer (decision #2):**
- Every tenant-owned table carries a `company_id NOT NULL REFERENCES companies(id)`, indexed.
- Every service method takes `companyId` as its **first argument** and filters on it.
- RLS policies are the backstop: even a query that forgets the filter returns zero rows, not another tenant's data.
- The Super Admin operates through a **separate, explicitly cross-tenant path** (its own service methods / bypass), never through tenant-scoped methods.

---

## 6. Multi-tenant, permissions & theming

These three concerns are core to the platform; the architecture doc details each.

| Concern | Approach | Tech |
|---|---|---|
| **Tenant isolation** | `company_id` on every table + always-scoped services + RLS backstop | Neon + Postgres RLS |
| **Runtime permission matrix** | Roles × Features × {View/Create/Edit/Delete}, stored per-company, editable by the Event Admin; enforced in routes and used to hide nav / block URLs | Postgres tables + a `PermissionsService` + a route guard + a client `usePermissions` hook |
| **White-label theming** | One of **6 predefined themes** (palette + font pairing) + company logo, swappable in Settings | CSS variables set from the company's theme id; fonts loaded per theme (`next/font`) |

---

## 7. Integrations (server modules) — v1 scope

Only the integrations the MVP actually needs. Everything payment/messaging-related is out of v1.

| Module | Integration | Responsibility |
|---|---|---|
| `email/` | **Resend** | Transactional email — user invites, company-provisioning links, password resets. |
| `storage/` | **Cloudflare R2** (`@aws-sdk/client-s3`, S3-compatible) | File storage — uploaded **contract** files (PDF/doc) and the **PR media library** (images/video); presigned URLs for up/downloads. |
| `geocoding/` | **Geoapify** | Address autocomplete + coordinates for event locations (pairs with maplibre on the client). |

> **Deliberately excluded from v1 (all deferred in the PRD):**
> **Stripe** (real billing is deferred; Super-Admin plans are labels only) · **Twilio WhatsApp/Verify**
> (no notifications/OTP) · **pdf-lib** (we *upload* contracts, we don't *generate* them) ·
> **svix** (no inbound third-party webhooks). These are documented here so it's clear they were
> considered and consciously left out — they return if/when ticketing, billing, or auto-publishing land.

---

## 8. Tooling

| Technology | Used for |
|---|---|
| **Prettier** (+ `prettier-plugin-tailwindcss`) | Formatting + Tailwind class sorting. Config is authoritative (see `ARCHITECTURE.md §Conventions`). |
| **ESLint** (`eslint-config-next`) | Linting the web package. |
| **tsx** | Running server-side TS scripts (migrations, seeds) without a build step. |
| **env-cmd / dotenv** | Loading `.env.local` for the dev server. |
