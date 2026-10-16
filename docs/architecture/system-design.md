# LabOS — System Architecture

**Status:** Accepted
**Last updated:** 2026-09-08
**Related:** `docs/product/prd.md`

---

## 1. Constraints Going In

- **Team:** solo developer (Shivam), using AI coding tools (Cursor/Antigravity/Claude Code) — architecture needs to be simple enough for one person to reason about and for AI tools to work within consistently.
- **Budget:** zero-to-minimal for Phase 1. Every choice below defaults to a genuine free tier, with a clear, predictable (not surprising) cost when scaling past it.
- **Scale target for Phase 1:** one pilot lab, growing to a small handful (5–10) of word-of-mouth labs in Phase 2. Not designing for hundreds of tenants yet — but not painting ourselves into a corner either.
- **Non-functional priorities:** low operational burden > raw performance. Correctness on medical data > premature optimization.
- **Note on scope:** LabOS is a new project, built from scratch. It shares no code, data, or architecture with any prior personal tool — any earlier prototype exploring similar ideas informed our thinking but is not a phase of this project and is not referenced further in this document.

---

## 2. Concepts Used In This Document (read this before the rest)

This document uses a few architectural terms that may be new. Explained here once, in plain language, so the rest of the doc can use them without re-explaining — and so any AI tool reading this repo has the same shared understanding we do.

### Domain-Driven Design (DDD)
The core idea: structure code around the real business concepts your friend already thinks in — a `Report`, a `TestPanel`, a `ReferringDoctor` — rather than around technical layers like "all database code" or "all API code." Related concepts get grouped together into a **bounded context**: a boundary around one area of the business (e.g. everything about reports lives together, everything about referrals lives together), with its own rules, not reaching into another boundary's internals. Our module list in Section 7 (`panels`, `reports`, `referrals`, `labs`, `notifications`, `billing`, `collections`) — each one *is* a bounded context.

### Aggregates & Aggregate Roots
An **Aggregate** is a cluster of associated domain objects that we treat as a single transactional unit for data changes. Every aggregate has an **Aggregate Root** (the gateway entity). Outside code can only hold references to the Aggregate Root, never directly to child entities inside the boundary. The Aggregate Root guarantees that all business invariants remain valid.
- In `panels`: `TestPanel` is the Aggregate Root. `PanelSection` and `PanelParameter` are internal child entities. `TestPackage` is a separate Aggregate Root encapsulating bundled panel items (`PackagePanel`).
- In `labs`: `Lab` is the Aggregate Root for the tenant organization; `Profile` is the user's membership entity bound to Supabase auth with role permissions.
- In `reports`: `Report` is the Aggregate Root controlling lifecycle states, encapsulating `ReportPanel`, `ReportValue`, and `ReportAmendment`. `Patient` is the patient demographic aggregate.

### Value Objects
A **Value Object** is an immutable object defined purely by its attributes, without a persistent conceptual identity (no unique primary key ID). If two value objects have the same attributes, they are structurally equal. Crucially, in a rich domain model, **Value Objects encapsulate their own validation and domain logic** rather than relying on external procedural evaluator functions.
- In `panels`: `NormalRange` is a rich Value Object defining physiological reference intervals (`numeric`, `gender_specific`, `text`). It contains its own `isOutOfRange(value, sex)` evaluation method, eliminating procedural utility bloat.

### Domain Services
A **Domain Service** performs domain-specific calculations or operations that do not naturally belong to a single entity or value object. It operates on domain concepts without maintaining mutable state.

### Repository Pattern with Dependency Inversion (DIP)
To keep the domain and application layers completely decoupled from database mechanics (TypeORM, SQL, drivers):
- **Domain Layer defines the contract:** Repository interfaces (`IPanelRepository`, `ILabRepository`, `IReportRepository`) live inside `domain/<aggregate>/interfaces/`.
- **Infrastructure Layer implements the contract:** Concrete repository implementations (`TypeOrmPanelRepository`, etc.) live inside `infrastructure/database/repositories/`.
- **Application Handlers depend only on the interface:** Use-case handlers receive the repository interface via dependency injection. They never write raw `dataSource.transaction()` or call TypeORM methods directly, ensuring 100% database-agnostic business logic that can be unit-tested without mocking database drivers.

### Direct Handler Injection in Slice Controllers
For in-process Modular Monolith controllers:
- Each dedicated slice controller (`create-panel.controller.ts`) directly injects its specific handler (`CreatePanelHandler`).
- **Why?** It guarantees 100% compile-time type safety for the return type (unlike `commandBus.execute()` which returns `Promise<any>`), provides zero reflection lookup overhead, and allows instant IDE navigation (`Cmd+Click` goes directly to the handler).
- An internal Event Bus (`EventEmitter2`) is reserved exclusively for cross-boundary **Domain Events** (`ReportFinalizedEvent`), where multiple decoupled subscribers need to react asynchronously.

### Modular Database Migrations
Rather than using `synchronize: true` (which risks data loss and drift), each bounded context manages its own database migrations inside `infrastructure/database/migrations/`. Npm scripts provide isolated migration runs per module.

### Modular Monolith
One deployed application (not many separate services talking over a network), but internally organized into the strict, independent modules described above. You get the simplicity of one codebase and one deployment, while still keeping a clean boundary between concerns — so if a module ever genuinely needs to become its own service later, it's a clean extraction, not a rewrite. The opposite extreme, **microservices** (many independently deployed services), adds real operational cost — network calls between services, distributed transactions, deployment orchestration — that is not justified at our scale and is explicitly rejected here.

### CQRS (Command Query Responsibility Segregation)
Separates code that **changes** state (a **Command**, e.g. "finalize this report") from code that **reads** state (a **Query**, e.g. "get this patient's history"), rather than mixing both into one general-purpose service class. We use it in its **lightweight, in-process form**: one Postgres database, no event sourcing, no eventual consistency — just clean separation and an internal event system (see next).

### Domain Events
When something meaningful happens (e.g. a report is finalized), the module where it happened publishes an **event** (`ReportFinalizedEvent`) instead of directly calling other modules. Other modules that care (e.g. `notifications`, to send a WhatsApp/email) **listen** for that event. Neither module needs to know the other exists — this is what keeps the Modular Monolith's boundaries real instead of quietly collapsing into one tangled mass over time.

### Vertical Slice Architecture
The traditional way to organize backend code is **by technical layer**:
```
/controllers   ← every controller
/services      ← every business logic class
/repositories  ← every database access class
```
Understanding "what happens when a report is finalized" means hunting across three unrelated folders. **Vertical Slice Architecture organizes by feature/use-case instead** — everything needed for one specific action lives together in one folder:
```
/finalize-report
  finalize-report.command.ts   ← the request shape
  finalize-report.handler.ts   ← the logic
  finalize-report.dto.ts       ← validation class (class-validator)
  finalize-report.controller.ts← dedicated route endpoint
  finalize-report.module.ts    ← slice DI module
```
This pairs naturally with CQRS: **each Command or Query IS a vertical slice.** We are not layering a separate architecture on top of CQRS — Vertical Slice is simply how we organize the commands/queries we already decided to use, instead of the older by-layer style. See Section 8 for how this looks in our actual folder structure.

---

## 3. High-Level Shape

```
┌─────────────────────┐        ┌──────────────────────┐
│   Next.js Frontend   │  HTTPS │   NestJS Backend API  │
│   (Vercel, free)     │ ─────► │   (hosting: TBD below)│
└─────────────────────┘        └──────────┬────────────┘
                                            │
                                 ┌──────────▼────────────┐
                                 │   Supabase             │
                                 │   - Postgres (data)     │
                                 │   - Auth (users/roles)  │
                                 │   - Storage (logos/PDFs)│
                                 └────────────────────────┘
```

- **Frontend:** Next.js (App Router), deployed on Vercel free tier.
- **Backend:** NestJS (TypeScript) — same language as the frontend, reducing context-switching for a solo dev and for AI tools working across the codebase.
- **Data/Auth/Storage:** Supabase (see ADR-0001 below for the reasoning).

---

## 4. Architectural Pattern & Design Principles — Summary

(Full explanations of each term are in Section 2 above; this is the concrete decision list.)

- **Pattern:** Modular Monolith with DDD bounded contexts as modules. Not microservices.
- **CQRS:** lightweight, in-process, via direct handler injection (no CQRS bus indirection). Not distributed/event-sourced.
- **Code organization within each module:** Vertical Slices — one folder per command/query, not one folder per technical layer.
- **SOLID principles**, applied concretely:
  - *Single Responsibility* — each module owns one domain concern; each slice owns one use case.
  - *Dependency Inversion* — the storage-abstraction-layer principle (no component talks to Supabase directly) is this principle in practice; same applies to `notifications` (abstracted behind an interface so swapping WhatsApp providers later doesn't touch calling code).
  - Full SOLID discipline (Open/Closed, Liskov, Interface Segregation) belongs in `docs/conventions/coding-standards.md` as day-to-day coding practice — noted here so it isn't lost.
- **API style:** REST, versioned (`/api/v1/...`) — not GraphQL. GraphQL's flexibility doesn't pay for its schema/resolver complexity at our scale, and REST is simpler for AI coding tools to work with predictably.
- **Frontend component layer:** shadcn/ui on top of Tailwind — Radix UI primitives (accessible, unstyled) delivered as code you own in the repo, not an opaque dependency. Fits into `/components/ui`.

---

## 5. Multi-Tenancy Approach

**Decision: shared schema, `lab_id` column on every tenant-scoped table** — not schema-per-tenant.

Reasoning: schema-per-tenant (a separate Postgres schema, or separate database, per lab) adds real operational complexity — migrations run per-tenant, provisioning a new lab means provisioning new infrastructure. At our current and near-term scale (single digits to low tens of labs), this complexity buys us nothing. A `lab_id` foreign key on every relevant table, enforced via Postgres **Row Level Security (RLS)** policies (a Supabase-native feature — a direct benefit of the Supabase choice), gives us real tenant isolation without the operational overhead.

If/when we reach a scale where noisy-neighbor performance or very large lab data volumes become a real problem, that's the point to revisit schema-per-tenant — not before.

---

## 6. Auth & Roles

- Supabase Auth handles user accounts (email/password to start; social login not needed for this user base).
- Every user belongs to exactly one lab (`lab_id` on the user record) — no cross-lab access, enforced by RLS.
- Roles (P1 requirement): `owner`, `technician`, `pathologist` — a simple role column, checked both in RLS policies (defense in depth) and in NestJS route guards (business-logic-level checks, e.g. "only pathologist role can sign off a report").

---

## 7. Core Backend Modules (DDD bounded contexts, maps to PRD requirements)

- **`panels`** — config-driven test panel/section/parameter engine, packages, and onboarding templates
- **`reports`** — patient info, filled results, print/PDF generation trigger, history/search, amendments, public share token
- **`referrals`** — outsourced test tracking (send-out status, merge into final report) and referring-doctor tracking (commission ledger & settlement)
- **`labs`** — tenant/lab profile, branding, team membership, credentials
- **`notifications`** — WhatsApp/SMS/email report delivery (abstracted behind an interface with WhatsApp Cloud API and mock fallback)
- **`billing`** — patient invoicing, line items breakdown, payment recording, operational expenses, financial analytics
- **`collections`** — phlebotomy home sample collection requests, technician dispatch, sample tube barcodes
- **`websockets`** — real-time bidirectional WebSocket gateway, CloudEvents envelope contracts, multi-tenant room routing, in-process domain event bridging

Each module owns its own commands, queries, domain logic, and repository — never reaches into another module's internals directly. Cross-module reactions happen via Domain Events (Section 2) or in-process listeners, not direct cross-boundary calls.

---

## 8. Folder Structure — DDD Modules + Vertical Slices + CQRS, together

This is what Sections 2 and 4 look like as an actual repo layout (aligned with `residency-backend`):

```
/src
  /modules
    /reports                          ← DDD bounded context
      /domain                         ← Core business logic, zero framework imports
        /report                       ← Report Aggregate Root boundary
          report.entity.ts
          report-panel.entity.ts
          report-value.entity.ts
          report-amendment.entity.ts
          /enums                      ← One enum per file (e.g. report-status.enum.ts)
            report-status.enum.ts
            sample-status.enum.ts
          /interfaces                 ← Repository contract (report.repository.interface.ts)
        /patient                      ← Patient Aggregate Root boundary
          patient.entity.ts
          /interfaces
            patient.repository.interface.ts
      /features                       ← Vertical Slices (Commands AND Queries)
        /report                       ← Feature group
          /create-report              ← Use case slice
            create-report.command.ts
            create-report.handler.ts
            create-report.dto.ts
            create-report.controller.ts
            create-report.module.ts
          /finalize-report
            finalize-report.command.ts
            finalize-report.handler.ts
            finalize-report.dto.ts
            finalize-report.controller.ts
            finalize-report.module.ts
          report-feature.module.ts     ← Bundles all report slice modules
        /patient
          /get-patient-history
            get-patient-history.query.ts
            get-patient-history.handler.ts
            get-patient-history.dto.ts
            get-patient-history.controller.ts
            get-patient-history.module.ts
          patient-feature.module.ts
      /infrastructure                 ← Categorized external concerns
        /database
          /repositories               ← Concrete TypeORM implementations (strictly <entity>.repository.ts)
            report.repository.ts
            patient.repository.ts
          /migrations                 ← Dedicated single-table migrations
            1710000003001-create-patients-table.ts
            1710000003002-create-reports-table.ts
            1710000003003-create-report-panels-table.ts
            1710000003004-create-report-values-table.ts
            1710000003005-create-report-amendments-table.ts
      /events
        report-finalized.event.ts
        report-finalized.websocket-event.ts
      reports.module.ts               ← Root bounded context module
    /panels                           (same shape: features/panel/, features/package/, features/template/)
    /labs                             (same shape: features/lab/, features/profile/)
    /referrals                        (same shape: features/doctor/, features/commission/, features/outsourced/)
    /billing                          (same shape: features/invoice/, features/expense/, features/finance/)
    /notifications                    (same shape: features/send-notification/, features/resend-notification/)
    /collections                      (same shape: features/create-collection/, features/assign-phlebotomist/)
    /websockets                       ← Real-time gateway bounded context
      /contracts                      ← CloudEvents envelope interfaces & payloads
        event-message.interface.ts
        realtime-room.builder.ts
        /enums/realtime-channel.enum.ts
        /payloads/
      /infrastructure
        /gateways                     ← Authenticated Socket.IO gateway
          events.gateway.ts
          events-gateway.module.ts
      /listeners                      ← In-process @OnEvent listeners bridging domain events to socket channels
        domain-events-bridge.listener.ts
      /features
        /publish-event                ← Fallback REST endpoint POST /events/publish
          publish-event.command.ts
          publish-event.handler.ts
          publish-event.dto.ts
          publish-event.controller.ts
          publish-event.module.ts
      websockets.module.ts
    /shared                           ← Unified shared module (under src/modules/shared)
      /domain                         ← BaseDomainEntity, WebSocketEvent<T>, shared enums (SexEnum), domain exceptions
      /constants                      ← Centralized metadata & system constants
      /guards                         ← Auth & role guards
      /decorators                     ← CurrentUser, Roles, Public decorators
      /filters                        ← Global exception filter (maps domain exceptions to HTTP)
      /infrastructure                 ← Database config, Pino logger, health, Supabase

Each command/query folder is a **vertical slice** — self-contained, easy for a human or an AI tool to open and fully understand without hunting across the codebase. Each top-level module under `/modules` is a **DDD bounded context**. The `/domain` + `/infrastructure` split inside each module is where **Dependency Inversion** lives concretely: domain logic depends on an interface (`report.repository.interface.ts`), and the Supabase-specific implementation is swappable without touching the domain logic.

---

## 9. Offline Sync Design (P1/P2)

As designed in our earlier discussion — formalize as its own ADR before implementation (see Action Items). Summary: optimistic concurrency via a `version` integer per record, field-level auto-merge for non-overlapping edits, human-resolved flag for genuine same-field conflicts. Not needed until multi-device/multi-user usage begins (Phase 2), but the `version` column should be added to relevant tables from the start of Phase 1 so it's not a painful retrofit.

---

## 10. WhatsApp/Notification Delivery — needs a decision

Not locked yet. Options, roughly in cost/complexity order:
- **Meta WhatsApp Cloud API directly** — free tier available (a number of free conversations/month), more setup (Meta Business verification), most control.
- **Aggregator (Twilio, MSG91, Gupshup)** — easier integration, per-message cost, faster to get running.
- **Defer WhatsApp, ship email/SMS first** — lowest effort initially, add WhatsApp once the core product is proven with the pilot lab.

Given zero-budget-for-now and that this is a genuine differentiator worth getting right rather than rushed, my suggestion: **defer WhatsApp to early in Phase 1, ship with email delivery first**, since email is free and trivial, and revisit WhatsApp once you're ready to invest the Meta Business verification effort. Flagging this as open rather than deciding it here — want your call before it's locked into requirements.

---

## 11. Storage Abstraction

No component talks to Supabase directly. Domain logic depends on repository interfaces (Section 8); the Supabase-specific implementation is an interchangeable detail behind that interface. This is Dependency Inversion applied concretely, and it's what makes swapping providers later (if ever needed) touch one layer, not the whole codebase.

---

## 12. Hosting (Phase 1)

- **Frontend:** Vercel free tier.
- **Backend (NestJS):** needs a decision — Render free tier (cold starts, but acceptable for this workflow since report-filling doesn't need to round-trip the backend on every keystroke) vs. a small always-on VM (e.g. Oracle Always Free) if cold starts prove annoying in practice. **Suggest starting with Render free tier for simplicity, and only move to a VM if cold starts become a real problem for the pilot lab** — don't solve a problem we don't have yet.
- **Database/Auth/Storage:** Supabase free tier, as decided below.

---

## 13. WebSockets & Real-Time Gateway Architecture

Real-time capabilities (live report status, home phlebotomy dispatch updates, critical abnormal alerts) are provided by the `websockets` bounded context using **Socket.IO** (`@nestjs/platform-socket.io`).

### 1. Events over Commands
Pushing updates over WebSockets represents an immutable fact that already happened and was committed to PostgreSQL. Therefore, WebSocket notifications are modeled strictly as **Events** inheriting from `WebSocketEvent<T>`. Commands are reserved exclusively for requests that change state.

### 2. Standardized CloudEvents Envelope (`EventMessage<T>`)
Every WebSocket message conforms strictly to the CloudEvents envelope standard:
```typescript
export interface EventMessage<T = any> {
  channels: string[];
  event: RealtimeChannelEnum;
  timestamp: string;
  traceId: string;
  payload: T;
}
```
- **Top-level attributes:** Transport metadata (target channels, event name, ISO timestamp, unique trace UUID).
- **`payload: T`:** Strongly typed clinical or operational data (no untyped `any`).

### 3. Multi-Tenant Room Isolation (`RealtimeRoomBuilder`)
Sockets never broadcast globally without tenant scoping. Room names are constructed strictly via `RealtimeRoomBuilder`:
- `lab:{labId}` — Lab-wide broadcast (new reports, status updates)
- `lab:{labId}:doctors` — Pathologist-only room (urgent critical alerts)
- `lab:{labId}:phlebotomists` — Phlebotomy staff room (pickup dispatch requests)
- `user:{userId}` — Direct user channel (personal assignment notifications)

### 4. Connection Handshake Authentication
When a client connects to the WebSocket gateway:
1. Sockets supply their Supabase JWT in `handshake.auth.token` or `handshake.headers.authorization`.
2. The gateway verifies the token with `SupabaseService.getUserFromToken(token)`.
3. The gateway fetches the user's `Profile` via `IProfileRepository.findById(userId)`.
4. Client joins `lab:{labId}` and `user:{userId}`. If `role === PATHOLOGIST`, client also joins `lab:{labId}:doctors`. Unauthenticated connections are disconnected immediately.

### 5. In-Process Domain Event Bridging (Zero Latency)
To avoid the anti-pattern of an internal service making an HTTP round-trip back to itself, domain events are bridged in memory:
1. Feature handlers emit domain events via NestJS `EventEmitter2` (e.g. `'report.finalized'`, `'collection.assigned'`).
2. `DomainEventsBridgeListener` handles these events in-process.
3. The listener instantiates the appropriate `WebSocketEvent` subclass and calls `EventsGateway.publishToClients(event.getBroadcastPayload())` with microsecond latency.

### 6. REST Publish Fallback (`POST /events/publish`)
For external integrations or administrative triggers, `PublishEventModule` exposes a guarded REST endpoint that validates channels, event enums, applies tenant-scoping, and broadcasts through the gateway.

---

## ADR-0001: Database & Backend-as-a-Service Choice

**Status:** Accepted
**Deciders:** Shivam

### Context
LabOS Phase 1 needs a real multi-tenant backend with persistent storage: user accounts, role-based access, and file storage (lab logos, generated PDFs) are all Phase 1 requirements from the PRD.

### Decision
Use **Supabase** (managed Postgres + Auth + Storage) rather than a plain self-managed Postgres instance or Aiven.

### Options Considered

**Plain self-managed PostgreSQL (e.g. on a free VM)**
| Dimension | Assessment |
|---|---|
| Complexity | High — self-managed backups, auth, connection pooling |
| Cost | Free (compute) but real ops-time cost |
| Scalability | Fine, but manual effort to scale |
| Team familiarity | Good (Node/TS background) but auth still has to be hand-built |

**Aiven (managed Postgres only)**
| Dimension | Assessment |
|---|---|
| Complexity | Medium — managed DB, but no bundled auth/storage |
| Cost | Free-tier status unclear/inconsistent as of research; paid tiers start ~$19–75/month |
| Scalability | Good, standard managed Postgres |
| Team familiarity | Neutral — just Postgres, nothing unusual |

**Supabase (Postgres + Auth + Storage bundled)**
| Dimension | Assessment |
|---|---|
| Complexity | Low — auth and storage come free, less to build |
| Cost | Confirmed free tier: unlimited API requests, 50,000 MAU (auth), 500MB database (shared CPU/RAM), 5GB egress + 5GB cached egress, 1GB file storage, community support; predictable $25/mo Pro tier when scaling past it |
| Scalability | Good — standard Postgres underneath, no proprietary lock-in on the data itself |
| Team familiarity | New tool, but thin/well-documented API surface |

### Trade-off Analysis
The deciding factor isn't raw database capability — plain Postgres or Aiven would work fine as a database. It's that **Auth is now a real Phase 1 requirement** (role-based access), and building that ourselves is avoidable, real effort. Supabase removes that work for free, with no meaningful lock-in risk since it's standard Postgres underneath — we can always migrate the data out later if we ever outgrow Supabase's layer.

### Consequences
- **Easier:** Auth, file storage, and RLS-based tenant isolation all come essentially free.
- **Harder:** One more third-party dependency to learn; free-tier project pausing after 7 days inactivity needs to be understood (non-issue for active daily use, but worth knowing).
- **Revisit when:** database size or usage patterns exceed what Supabase's pricing ladder makes sense for — unlikely before Phase 3+.

### Testing Strategy (brief note, expand in `docs/conventions/` later)
- Unit tests per module (service/repository layers) — standard NestJS testing tools (Jest, NestJS's default).
- Integration tests for the CQRS command/query handlers, since correctness of report data is genuinely high-stakes here.
- End-to-end test for the critical path (fill report → print/PDF output matches expected letterhead) — the flow the pilot lab will judge the product on, worth protecting with a real test rather than manual checking alone.

---

## 13. Frontend Architecture: Layout Engine, Print Subsystem & Component Standard

To match enterprise clinical engineering standards, `labOS-app` follows these core architectural patterns:

### 1. Polymorphic AppShell Architecture
The root application shell (`AppShell`) governs navigation, header controls, command palette (`Ctrl+K`), and live WebSockets telemetry drawer across all 13 routes. It implements three layout variants:
- `contained`: Max-width constrained (`max-w-5xl`) container for focused, linear data-entry forms (e.g. `/reports/new`, `/settings`).
- `full-bleed`: Unconstrained fluid grid (`w-full px-4 sm:px-6 lg:px-8`) designed for data-dense clinical registers and ledgers (`/accessions`, `/billing`, `/patients`, `/referrals`, `/collections`). Eliminates artificial width bottlenecks and prevents unnecessary horizontal scrollbars on desktop viewports.
- `workspace`: Height-locked split console (`h-[calc(100vh-3.5rem)] overflow-hidden`) with independent column scrolling (`overflow-y-auto`) for high-throughput diagnostic consoles (`/reports/[id]/entry`).

### 2. Enterprise `@media print` Subsystem
Print is implemented via a central CSS engine in `globals.css` rather than fragile ad-hoc utility classes:
- Universal chrome suppression: strips `aside`, `header`, `nav`, floating drawers, toasts, and portal overlays.
- A4 standardization: enforces `@page { size: A4 portrait; margin: 10mm; }` with exact background color preservation.
- Modal print isolation: dialogs rendering printable vouchers (e.g. invoice receipts, TDS settlement sheets) utilize `.printable-modal-backdrop` and `.printable-modal-content` to suppress background pages and grey backdrops during browser print dialog execution.

### 3. URL-Synchronized State & Next.js App Router Static Analysis
Filter tabs, search terms, and pagination parameters are synchronized directly to URL query parameters via `useURLState()`. To prevent Next.js static analysis bailout during `next build`, consumers of `useSearchParams()` (such as `TablePagination`) wrap inner search parameter readers in `<React.Suspense>`.

### 4. Component Hierarchy & Demo Data Isolation
- Component structure strictly enforces single-responsibility decomposition: files exceeding 200 lines are factored into private sub-components located in co-located `_components/` directories.
- Mock fixtures and demo seed data (`DEMO_DOCTORS`, `DEMO_COLLECTIONS`, `DEMO_INVOICES`) are strictly isolated within `src/lib/demo-data/`, ensuring API hooks remain pure queries and mutations with clean fallback capabilities.

---

### Action Items
1. [ ] Set up Supabase project for Phase 1 development
2. [ ] Define RLS policies for `lab_id`-scoped tenant isolation
3. [ ] Decide WhatsApp vs. email-first for notification delivery (Section 10 — open)
4. [ ] Decide backend hosting (Render free vs. VM) once Phase 1 is closer to real usage (Section 12)
5. [ ] Write a dedicated ADR for the offline-sync design before Phase 2 implementation begins
