# Event Planning Platform — Product Requirements Document (PRD)

> **Status:** In progress — being defined collaboratively, module by module.
> **Scope of this doc:** Feature definition only (no tech/architecture yet).
> **Last updated:** 2026-07-03

---

## 1. Product Summary

A **multi-tenant SaaS platform** that event planning companies use to plan and run their events. The platform owner (Super Admin) onboards each event planning company as a **Company-Client**; each company gets its own **white-labeled, themed workspace** where its team plans events end-to-end.

Guiding principle: **keep it simple / don't overengineer.** Ship a focused MVP.

---

## 2. User Roles

### Platform level
- **Super Admin (platform owner)** — creates and manages Company-Clients, their subscriptions and access. Sits above all companies. *(This role is the platform owner only.)*

### Company level (5 roles)
1. **Event Admin** — company owner. Full control of the company: team, events, branding, settings, billing.
2. **Event Manager** — owns specific events end-to-end (tasks, timeline, vendors, guests, budget for their events). No company-wide settings/billing.
3. **Coordinator** — hands-on execution within assigned events (update tasks, upload files, message vendors, check-in). Cannot delete events, change budgets, or sign contracts.
4. **Finance / Accounts** — money across all events (budgets, quotes, invoices, payments). Not logistics.
5. **Marketing / PR Manager** — social + promotion (content calendar, posts/stories/reels, media, invitations/landing). No budgets/contracts/logistics.

> Future (external, not in this set): **Client** and **Vendor** login roles.

---

## 3. App Structure

### 🅰️ Platform level — Super Admin (you)
- Manage Company-Clients (create/edit/suspend)
- Subscriptions / access

### 🅱️ Company workspace — the 5 roles

**Global modules** (company-wide, each filterable by event):
- **Events** — list of events + each individual event
- **Team & Users** — the company's team (built before planning)
- **Tasks** — each task attached to one event or one custom tag · *per-event filter*
- **Vendors** — reusable supplier directory · *per-event filter*
- **Contracts** — central repository of all contracts · *per-event filter*
- **Finance / Budget** — contracts, payments, full money picture · *per-event filter*
- **PR & Social Media** — content calendar for scheduled posts/stories/reels · *per-event filter*

**Inside a single event:**
- **Details** — name, date (one day), start/end time, location, short description
- **Team assignment** — pick from the company's team
- **Tasks** — this event's tagged tasks
- **VIP Tables** — zones + drawing canvas with draggable table types (see §9)

---

## 4. Foundation — Auth & Onboarding  *(Locked)*

### Authentication
- Login
- Signup
- Reset Password
- **Invite User** (Event Admin invites teammate → email invite → set own password → lands in assigned role)

### Company-Client provisioning (by Super Admin)
- Creating a Company-Client **auto-creates**: the **company account (workspace/tenant)** + its **Event Admin account** (first user)
- Admin receives an invite / first-login link

### First-login Quick Setup wizard (Event Admin, one time)
- Upload **company logo**
- Pick a **theme** (colors + fonts) — white-labels the whole workspace
- Finish → enter the app

### Theming
- **Pick one of 6 predefined themes** (no custom colors in v1)
- Theme is **changeable later** in Settings
- Each theme = color palette + font pairing

**The 6 themes:**
| # | Theme | Vibe | Palette direction | Fonts (Heading / Body) |
|---|-------|------|-------------------|------------------------|
| 1 | Midnight | Luxe, elegant, dark | Deep navy + charcoal, gold accent | Playfair Display / Inter |
| 2 | Blossom | Soft, romantic, light | Blush, cream, sage green | Cormorant Garamond / Nunito Sans |
| 3 | Meridian | Clean, corporate | Slate blue + white, cool grays | Poppins / Inter |
| 4 | Festival | Bold, energetic | Magenta → orange, purple | Sora / DM Sans |
| 5 | Terra | Warm, natural, boho | Terracotta, olive, sand | Fraunces / Work Sans |
| 6 | Noir | Minimal, high-contrast | Black & white + one accent | Space Grotesk / IBM Plex Sans |

---

## 5. Team & Users  *(Locked)*

The company builds its roster **before** planning events. Events later just *assign* from this pool.

**Team roster**
- List of all company users: name, avatar, role, email/phone, status (**Active** / **Invited** / **Deactivated**)
- Search + filter by role

**Invite a member** (Admin)
- Form: **First name, Last name, Email, Role** (Admin fills these — he knows who he's inviting)
- Invitee receives invite → sets their own password → becomes Active
- Re-send or revoke a pending invite

**Manage members**
- Edit a user's **role**
- **Deactivate** (revokes access, keeps their history on past events) vs. **Remove**
- **No job title** — the role is the descriptor
- **One role per user**

**Roles & Permissions**
- **Access is entirely permission-driven — there are no hardcoded "admin-only" areas.** Every section is gated by the permissions matrix.
- **Event Admin** holds **all permissions by default** (a superset) and is the one who assigns permissions to every role.
- **Permissions screen:** a matrix of **Roles × Features**; per feature, toggle **View / Create / Edit / Delete** for each role.
- Ships with **sensible per-role defaults** (the standard ladder); Admin tweaks from there.
- Capabilities are **independent per feature** — e.g., Marketing/PR can be **View-only on Contracts** but **Edit on Media/PR** at the same time.

**Permission enforcement rules**
- **View is the gate.** If a role has **no View** on a section: the section is **hidden from navigation** *and* its **URL is inaccessible** (direct navigation blocked).
- **View without Create/Edit/Delete** → the user sees the data **read-only**.
- **Invariant:** Create / Edit / Delete **cannot** be enabled without View. The Permissions screen enforces this dependency (enabling any of them requires View; removing View clears the rest).
- **Decision:** the Permissions screen is itself permission-gated like any other feature (no special-casing) — granting a role access to it is the Admin's explicit choice.

**Not in v1 (future ideas)**
- Team-size caps tied to subscription plan *(good future monetization lever)*
- Departments / team groupings

---

## 6. Events + Details  *(Locked)*

The event object is intentionally minimal. This module is about creating, finding, and opening events.

**Events list** (global home)
- All events as **cards**: name, date, start–end time, location, **countdown to event**, **# VIP tables booked**, **# team assigned**
- **Search** by name; **filter** by date range and by assigned team member
- Split into **Upcoming** vs **Past** (past events remain accessible as an archive); sort by date

**Create / Edit event**
- Fields: **name, date (one day), start time, end time, location, short description**
- **Location = name + coordinates**; tapping the location **opens Google Maps** at those coordinates
- Governed by the permissions matrix (who can create/edit)

**Event detail page** (the hub)
- Opens to **Details**, with the event's sections alongside: Team assignment, Tasks, VIP Tables, plus the *event-filtered* Vendors / Contracts / Finance / PR
- **Edit** and **Delete** the event

**Not in v1 (future improvements)**
- Duplicate / template an event (copy setup & tasks as a starting point)
- Structured address + in-app map pin picker

---

## 7. Tasks  *(Locked)*

The company's unified task system. **Global module with a per-event filter.** Planning can span days/weeks/months.

**A task has:**
- Title
- Description
- **Assignee** (a team member)
- **Created by** (shown on every task)
- **Start date** + **Due date**
- **Attached to exactly one:** an **Event** *or* a **Custom tag** (a company-defined label for non-event / general work, e.g. "Internal," "Marketing")
- **Status** — To Do / In Progress / Done / **Blocked** / **Cancelled** *(abandoned)*
- **Priority** — Low / Med / High
- **No attachments**; **no time-of-day** (no hour-based scheduling)

**Views (v1)** (same tasks, different lenses):
- **By team member** — primary view (tasks are defined per person)
- **Board (kanban)** — columns by status, drag to move

**Actions**
- **Anyone can create tasks** (and assign them); each task records its creator
- Filters: by **event / tag**, member, status, priority, date

**Not in v1**
- Timeline / calendar view
- Subtasks / checklists (tasks stay flat)

---

## 8. Home / My Dashboard — Notification Center  *(Locked)*

Every user's **landing page after login** — their personal command center.

- Aggregates **all of the user's tasks across every event** they're assigned to
- **Grouped & sorted by urgency:** Overdue → Due today → This week → Later
- Sort logic: **due date first, priority as tiebreaker**
- Click a task to open it or change its status inline

**Future**
- Surface other notifications too (invites, mentions, event changes, etc.)

---

## 9. Guests → VIP Tables  *(v1 = VIP Tables only)*

> **v1 scope:** ONLY the **VIP Tables** below. The broader guest system (guest list & types, ticketing, invitations) is documented as **planned/future structure** but is **not built in this stage.**

### v1 — VIP Table Setter  *(Locked — the 100%-need part)*

**Zones (VIP / backstage sections)**
- An event can have **multiple zones** (e.g., 4 VIP spaces), each kept **separate** in the app
- Each zone is built on its own **drawing canvas** where tables are placed
- Zones are named and can be added/edited
- **Each zone has a `price per seat` field** — set by the user, applied to every table in that zone

**Table types (predefined, draggable onto the canvas)**
- **4 predefined types by capacity:** table for **4**, **6**, **8**, **10** persons
- A table's **minimum consumption is derived** = capacity × **the zone's price per seat** (no manual entry)
- The user **drags a table type onto the canvas** to add it to the zone

**Each placed table carries:**
- Table type / capacity (persons) — from the type
- **Derived total consumption** (capacity × zone's price per seat)
- **Name / label**
- **Reservation holder** (booker — name + phone)
- **Status** — Available / Reserved (so staff can see which tables are still open)
- Optional **guest list** (names of people at the table)

**Two views per zone:**
- **Visual view (canvas)** — drag table types to build & arrange the layout; the primary working view. Intentionally simple — just for organizing & visualizing.
- **Table (list) view** — all tables in the zone with their details (holder, capacity, consumption, status) in a grid/list

**Currency**
- Money amounts use the **company currency** (e.g., RON), set once at company level

### Planned / Future — full Guest system  *(documented, NOT in this stage)*
- **Guest list & guest types** — guest records (name, contact, type, RSVP status, notes); predefined types + per-event custom; search/filter; CSV import
- **Tickets** — ticket types (price, capacity); 1 guest ⇆ 1 ticket; public sales page; checkout + payment processing; emailed QR tickets; QR-scan check-in *(effectively its own product)*
- **Invitations** — generate & send invitations, track RSVPs

---

## 10. Vendors  *(Locked)*

A **reusable company directory** of suppliers, attached to events as needed. **Global module with a per-event filter.**

**Vendor record** (company-level)
- **Name**
- **Category** — predefined list (Catering, Venue, DJ, Florist, AV, Security, Photography…) **+ option to add a custom category**
- **Contact person**
- **Phone**
- **Email**
- **Notes**
- *(no website field)*

**Directory features**
- Search + filter by category
- Add once, attach to many events

**Attaching vendors to an event**
- Pick vendors from the directory for a given event
- **Per-event booking status:** Contacted / Booked / Confirmed (a vendor can be at a different status for each event)
- The event's Vendors view shows only that event's attached suppliers (per-event filter)
- Cost/fees live in **Finance / Contracts** (not duplicated on the vendor)

---

## 11. Contracts  *(Locked)*

A **central document repository** for all the company's contracts (upload-only — not an in-app contract builder). **Global module with a per-event filter.**

**Contract record**
- **Title**
- **Party** — who it's with (a client or a vendor)
- **Uploaded file** (PDF/doc)
- **Value / amount**
- **Status** — **Pending / Signed**
- **Key dates** — signed date, expiry date
- **Notes**
- Optional **link to an event** and/or a **vendor** from the directory

**Actions**
- **Upload** a contract file, download it, replace/version it
- Search + filter by status, party, event
- Per-event filter (an event shows its own contracts)

**Not in v1**
- E-signature
- In-app contract authoring / templates

---

## 12. Finance / Budget  *(Locked — v1 = Expenses only)*

> **v1 scope:** the **Expenses** side only. The **Income** side (ticket-sales revenue, VIP-table consumption as expected revenue, profit/net) is deferred to **Future**, alongside ticketing.

**Expenses** (per event) — **Global module with a per-event filter**
- Line items: **description**, **category**, **amount** (actual amount — no planned-vs-actual budgeting in v1), **status** (paid / partially paid / unpaid)
- **Category:** a few **predefined tags** (Venue, Catering, Staff, Equipment/AV, Decor, Transport, Marketing, Security, Miscellaneous…) **+ option to add a custom tag**
- Each expense can **link to a Contract and/or a Vendor** (reference only — contracts are never generated in-app)

**Payments**
- **Partial payments** supported: record multiple payments over time (amount + date) against an expense
- App shows **paid vs. outstanding**; when an expense references a contract, the contract reflects how much is paid vs. still outstanding

**Summary**
- Per event: total expenses, total paid, total outstanding; **totals by category**
- Global roll-up: expenses across all events

**Future**
- Income side: ticket-sales revenue, VIP-table consumption as expected revenue, net/profit
- Planned-vs-actual budgeting

## 13. PR & Social Media  *(Locked — v1 = planning calendar only)*

A dedicated area where the **Marketing / PR** role plans event promotion. **Global module with a per-event filter.**

**Content calendar**
- **Calendar view** (month/week) + **list view** with filters
- Plan and schedule content; no automatic publishing in v1

**A content item:**
- **Platform** — from a **predefined list** (Instagram, Facebook, TikTok, LinkedIn, X, YouTube); no custom platforms
- **Content type** — Post / Story / Reel
- **Caption / text**
- **Media** — attached image(s) / video
- **Scheduled date & time**
- **Status** — **Draft / Scheduled / Published** (no approval flow — kept simple)
- **Linked to an event or a custom tag** (same tagging pattern as Tasks)

**Media library**
- Upload and reuse images/videos for content (event media, brand assets)

**v1 behavior:** the app is a **planning/scheduling calendar**. Posting is done **manually** by the team on each platform, then marked **Published** here. No connection to live social accounts.

---

### 13a. Would-be-great-to-have — Auto-publishing (the app as a scheduler)  *(needs further investigation)*

> **Not committed.** A nice-to-have upgrade to the planning calendar. Documented so we can investigate before scoping. **Platforms of interest: Instagram, Facebook, TikTok.**

**The concept:** the app stays a **pure scheduler** — nothing more. Each company **connects their own social account(s)**, and the app publishes the scheduled posts to those accounts on their behalf. We don't host or resell posting; we just push to the user's connected account.

**Cost — it's effectively free at normal volumes.** The official publishing APIs charge **no per-post / per-account fee**; posting is free **within each platform's per-account allowance**:

| Platform | API fee | Free per-account posting allowance |
|---|---|---|
| **Instagram** (Graph API) | **Free** | **50 API-published posts / 24h** per account |
| **Facebook** (Graph API) | **Free** | Rate-limited per account (generous for normal use) |
| **TikTok** (Content Posting API) | **Free** | **25 videos / account / day** |

**What still needs investigation (the catch isn't money, it's setup):**
- **Account prerequisites:** e.g., Instagram must be a **Business/Creator** account linked to a **Facebook Page** for API publishing.
- **App review/audit:** Meta requires **App Review** (~1–4 weeks/round); TikTok requires a separate **audit** (~2–4 weeks); until TikTok is audited, posts are private (SELF_ONLY) and only 5 accounts can authorize.
- **Build effort:** implementing each platform's OAuth + publishing flow is non-trivial (~weeks for the first platform).
- **Shortcut to investigate:** an aggregator (e.g., **Ayrshare**) offers one unified API across IG/FB/TikTok and absorbs the per-platform review burden, but it's **billed per connected profile** (~$2.49–$8.99/profile/mo on business plans) — a recurring cost to weigh against building direct connections ourselves.

**Status:** would be great to have; **revisit and investigate further before committing.**

---

## 14. Platform / Super Admin  *(Locked)*

The platform owner's control panel, above all companies. **Super Admin only.**

**Company-Clients management**
- **List of all companies:** name, status (Active / Suspended), **plan** (label only), # users, # events, created date
- **Create a company-client** → auto-provisions the company workspace + its Event Admin account (per §4)
- **Edit** company details
- **Suspend / Reactivate** a company (blocks/restores access)
- **Delete** a company
- Assign a **plan/tier** — a **label only** in v1 (no enforced limits yet)

**Platform overview**
- High-level metrics: total companies, active vs. suspended, total events across the platform

**Not in v1 (future)**
- **Real in-app billing** (Stripe, invoices, subscriptions) — currently plan status is **tracked manually**
- **Log in as / impersonate** a company for support
- **Enforced plan limits** (e.g., team-size caps, feature gating by tier)

---

## 15. Cross-cutting & conventions  *(reference)*

Patterns that recur across modules, captured once:
- **Multi-tenant isolation:** every company sees only its own data; Super Admin sees across all.
- **Global module + per-event filter:** Tasks, Vendors, Contracts, Finance, PR each have a company-wide home and an event-scoped view.
- **Permission-gated navigation:** no View → hidden from nav + URL blocked (see §5).
- **Predefined + custom tags:** the reusable pattern for vendor categories, expense categories, guest/table types, task custom tags.
- **Company currency:** one currency per company (e.g., RON), set at company level.
- **White-label theming:** one of 6 themes + logo per company (see §4).

---

## 16. Deferred to Future (master list)

Consolidated from the modules above:
- **Full ticketing & guest system** — guest list/types, ticket types, public sales page, checkout + payments, emailed QR tickets, QR check-in, invitations *(effectively its own product)*
- **Finance Income side** — ticket-sales revenue, VIP-table consumption as expected revenue, net/profit; planned-vs-actual budgeting
- **Social auto-publishing** — app-as-scheduler to connected IG/FB/TikTok accounts *(needs investigation, see §13a)*
- **Real in-app billing** — Stripe/subscriptions; enforced plan limits; team-size caps
- **Log in as / impersonate** for support
- **Tasks:** timeline/calendar view; subtasks/checklists
- **Events:** duplicate/template an event; structured address + map pin picker
- **E-signature & in-app contract authoring**
- **Custom permission granularity** beyond the role matrix; departments/team groupings
- **Dashboard:** broader notifications (invites, mentions, event changes)

---

## 17. Competitor Landscape  *(market research — 2026)*

**Headline:** No single competitor combines what this product does. The market splits into separate camps, and our mix — a **white-label, multi-tenant workspace for event-planning companies** *plus* a **nightlife-style VIP table/zone canvas with per-seat consumption** — sits in the gap between them.

### The market splits into 5 camps

**1. Enterprise event management (conferences / corporate)**
- **Cvent, Bizzabo, Whova, Eventtia, InEvent**
- Strong at registration, ticketing, attendee management, badges, analytics. Expensive, conference-oriented.
- **Gap they leave:** weak on physical floor/VIP layout (Cvent notably has no map-based canvas / capacity tooling); not built for a planning *agency's* internal ops.

**2. Event-planner / agency CRM (weddings & social)**
- **Planning Pod, HoneyBook, Aisle Planner**
- Closest to our "agency workspace" side: client CRM, budgets, vendors, timelines, basic seating.
- **Gap:** no nightlife VIP-table / bottle-service canvas, no per-seat consumption, generally single-tenant (for one agency, not a platform reselling to many).

**3. Nightlife / VIP table & bottle-service reservations**  ← *overlaps our signature feature*
- **SevenRooms, TablelistPro, Firetable, Scenetech, RESV, Mr. Black**
- Drag-and-drop VIP floor plans, table booking, bottle service, guest lists. Firetable/Tablelist do the interactive table canvas well.
- **Gap:** these are built for **venues that own the night** (clubs/restaurants), not for **planning agencies** running events across venues; no agency toolkit (tasks, contracts, expenses, PR calendar); not white-label multi-tenant for reselling.
- **Pricing signal:** SevenRooms ≈ **$499–$2,000+/mo per venue** (custom, not public).

**4. Site / floor-plan tools**
- **OnePlan, Social Tables (Cvent)**
- Map-based, scaled drag-and-drop site layouts (stages, tents, crowd flow).
- **Gap:** layout-only; no reservations, consumption, or business ops around it.

**5. White-label / multi-tenant platforms**  ← *overlaps our business model*
- **Eventtia, Younivent, Accelevents, EventsX, Ticketsauce**
- Rebrandable, multi-tenant, sell-under-your-own-brand — validates the model.
- **Gap:** almost all are **ticketing/registration-first**, not an operational agency workspace, and none carry the nightlife VIP-table canvas.

### Romania specifically
- **Veluna** — events/weddings platform: locations, suppliers, e-invitations, guest lists (consumer/marketplace lean)
- **Venner** — venue & organizer booking/calendar management
- **PYNBOOKING** — event management inside a hospitality PMS
- **EasyWeek** — booking/scheduling for event organizers
- **Eventbook** — ticketing & event management for organizers
- **ePlaceEvent (FivePlus)** — HoReca table-booking with role-based access
- **Clubbable** — consumer VIP-club guest-list/table app ("WhatsApp for going out")
- **MICE Romania** — a corporate-event *agency* (a potential customer, not a competitor)
- **Takeaway:** the Romanian market is **fragmented point solutions** — no dominant all-in-one for event *agencies*, and nothing pairing the agency workspace with a VIP-table canvas.

### Adjacent / complementary — NOT direct competitors
- **Nightz** (`nightz.app`, by MWM) — *"Nightlife's Social Network."* A **consumer-facing** discovery + social + in-app ticketing app: users get location-based venue/event recommendations, buy tickets, and coordinate nights out with friends (plus a "Dayz" mode for daytime/networking events).
  - **For businesses:** venues/organizers can create a business account to **promote their events and sell tickets** to Nightz's audience — i.e., a **marketing & ticketing channel** (demand generation).
  - **Why it's complementary, not competing:** Nightz helps a venue get *discovered and sell tickets*; our platform helps an event *company run the event* (tasks, VIP tables, vendors, contracts, finance, PR). No operational back-office overlap — no VIP-table canvas, team roles, contracts, or expenses. Could even be a **future ticketing/promotion channel to integrate with**, not a rival.

### Where we win (differentiation)
- **The combination is the moat:** agency ops workspace **+** nightlife VIP-table canvas **+** white-label multi-tenant, in one product.
- **Built for the agency, not the venue** — the planning company is the customer/tenant, running many events across venues.
- **VIP zones with per-seat consumption & Available/Reserved status** — a nightlife-grade feature that the wedding/corporate CRMs lack and the venue tools don't package for agencies.
- **Per-company white-label theming** (6 themes + logo) — each agency gets a branded instance.

### Watch-outs
- If we later add **ticketing/registration** and **social auto-publishing**, we step onto the turf of much larger, well-funded players — differentiate on the agency+nightlife niche first.
- The nightlife reservation incumbents (SevenRooms/Tablelist) could move "up" into agency workflows; speed and niche focus are our edge.

> **Sources:** OnePlan/Boompop/Capterra (event mgmt); Firetable, TablelistPro, SevenRooms, Scenetech, RESV, Mr. Black (nightlife/VIP); Eventtia, Younivent, Accelevents (white-label); Softlead, Veluna, Venner, PYNBOOKING, Eventbook, FivePlus/ePlaceEvent, Clubbable, MICE.ro (Romania). Full links shared in chat.
