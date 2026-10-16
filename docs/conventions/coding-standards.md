# LabOS — Engineering Standards & Architectural Guide

**Status:** Accepted  
**Scope:** Full-Stack (NestJS Backend `labOS-service` & Next.js Frontend `labOS-app`)  
**Related:** `docs/conventions/master-code-audit-and-refactoring-protocol.md`, `docs/architecture/system-design.md`, `docs/architecture/data-model.md`, `docs/openapi.yaml`

This document is the definitive master standard for all full-stack development across LabOS. Every engineer, AI subagent, and code contributor must adhere to these standards without deviation.

---

## 0. Core Engineering Mindset & Research Discipline

1. **Explain Architectural Decisions:**
   - Whenever proposing or implementing structural changes, always explain the architectural rationale, highlight trade-offs, and compare against alternative patterns.
2. **Deep Research Before Coding:**
   - When unsure about project conventions, domain logic, or edge cases, **never guess or make assumptions**.
   - Thoroughly read and analyze relevant files across the codebase, cross-reference documentation, and research industry-leading MNC production standards before writing code.
3. **Clean Code & Single Responsibility:**
   - Keep implementations minimal, focused, and maintainable.
   - Every function, component, and module must own exactly one responsibility. Avoid speculative or unneeded abstractions.
4. **Phase-Wise Decomposition & Incremental Task Chunking:**
   - Never attempt large epics, broad phases, or complex components in a single monolithic block.
   - Break large phases into small, atomic, testable sub-tasks (e.g. types/contracts $\rightarrow$ API client/hooks $\rightarrow$ private UI components $\rightarrow$ page integration).
   - Execute one atomic sub-task at a time with intermediate verification before proceeding.
5. **Mandatory Pre-Task Check Protocol:**
   - BEFORE writing any code or modifying any file, you MUST:
     - Read `CONTEXT.md` to verify current project phase and previous session decisions.
     - Review `docs/conventions/coding-standards.md` to align with all design, naming, and architectural rules.
     - Inspect the relevant backend feature slice directly (`labOS-service/src/modules/<module>/features/<feature>/`) before writing frontend code. Never guess types.
6. **Mandatory Post-Task Documentation & Verification Protocol:**
   - AFTER completing any task or implementation step, you MUST:
     - Update `CONTEXT.md` with the session log, completed items, and next steps.
     - Update any relevant documentation (`coding-standards.md`, OpenAPI contracts, architecture docs, walkthrough).
     - Run automated verification (`npm run build` and `npm run lint`) to ensure zero errors and zero warnings before finishing.

---

## 1. Frontend Architecture & Engineering Standards (`labOS-app`)

### 1.1 Strict 1:1 Backend Parity
- **Inspect Feature Slices Directly:** Before writing or modifying any frontend code, **always inspect the corresponding backend feature slice directly** (`labOS-service/src/modules/<module>/features/<feature>/` — checking its dedicated controller, DTO, query/command, and handler).
- **Zero Type Guessing:** Frontend TypeScript interfaces and Zod schemas must match the backend request/response payloads 1:1.
- **Eliminate Unnecessary Fallback Chains:** Do not create nested fallback chains (`a?.b || c?.d || 'N/A'`) based on guessed data shapes. Understand the exact backend response structure and model it cleanly.

### 1.2 API Communication & Axios Interceptors (`src/lib/api-client.ts`)
- **Axios over Fetch:** All server communication must use the configured Axios instance (`src/lib/api-client.ts`).
- **No Manual Fetch Boilerplate:** Do not write manual `fetch` calls, custom JSON parsing, or repetitive header injection.
- **SSR-Safe Token Resolution (Zero `typeof window` Hacks):**
  - **Never** write `typeof window !== 'undefined'` to read raw strings from `localStorage`.
  - Attach the Supabase Bearer token in the Axios request interceptor using official Supabase SDK session resolution (`supabase.auth.getSession()` or `@supabase/ssr` cookies).
- **Automatic Response Unwrapping & Global Error Handling:**
  - Axios response interceptors unwrap `response.data` so calling hooks receive data directly.
  - Global error interceptors catch HTTP failures and format them using backend's `ApiErrorResponse`.
- **Zero Magic Numbers (`http-status-codes`):**
  - **Never** use raw status numbers (`200`, `201`, `204`, `400`, `401`, `404`, `500`) in code.
  - Always import and use `StatusCodes` from `http-status-codes` (e.g. `StatusCodes.UNAUTHORIZED`, `StatusCodes.NOT_FOUND`, `StatusCodes.NO_CONTENT`).

### 1.3 State Management Strategy
- **Server State (`@tanstack/react-query`):**
  - All server-side data (reports, panels, patients, invoices) is managed exclusively via TanStack Query.
  - One dedicated `useQuery` / `useMutation` hook per feature slice, co-located within that feature's directory (`src/features/<feature>/api/`).
  - **Cache Invalidation:** Always invalidate related query keys on mutation success (e.g., `queryClient.invalidateQueries({ queryKey: ['reports'] })` upon finalizing a report).
- **Client-Only State:**
  - Standard React local state (`useState`, `useReducer`) for UI toggles, modal open/close, and local tabs.
  - If cross-cutting client-only state is required later, reach for **Zustand** (small, zero Provider ceremony). **No Redux**.

### 1.4 Form Management & Validation
- **React Hook Form + Zod (`@hookform/resolvers/zod`):**
  - All forms must use React Hook Form for performance and non-rendering keystroke efficiency.
  - All form validation schemas must be defined using **Zod** in dedicated files (`src/features/<feature>/schemas/<slice>.schema.ts`).
- **Pre-Wired Form Controls (`src/components/form/`):**
  - Use reusable, accessible form components bound to React Hook Form:
    - `text-field/index.tsx` — Text input with label, helper text, and inline Zod error.
    - `number-field/index.tsx` — Numeric input with clinical unit adornment (e.g. `g/dL`, `mg/dL`, `10^3/µL`).
    - `select-field/index.tsx` — Accessible controlled dropdown select.
    - `textarea-field/index.tsx` — Multiline text for pathologist remarks and clinical notes.
    - `switch-field/index.tsx` — Toggle for booleans.

### 1.5 Component Architecture (Folder + `index.tsx` Convention)
- **Folder + `index.tsx` Everywhere:**
  Every component resides in its own folder with an `index.tsx` entry point (e.g., `button/index.tsx`, `stat-card/index.tsx`).
- **Component Classification:**
  1. `src/components/ui/` — Atomic design system primitives based on Radix UI / shadcn/ui (`button`, `card`, `input`, `badge`, `dialog`, `table`).
  2. `src/components/form/` — RHF-connected form inputs.
  3. `src/components/shared/` — Cross-feature layout widgets (`status-badge`, `stat-card`, `empty-state`).
  4. `src/features/<feature>/components/` — Domain-private components (e.g. `src/features/reports/components/result-entry-grid/index.tsx`).

### 1.6 Mandatory UX & Interaction Standards
- **Loading Skeletons:** Never show a blank screen or a generic spinner during initial data fetching. Provide content-shaped pulse skeletons (`SkeletonCard`, `SkeletonTable`).
- **Empty States:** When queries return empty arrays, display an `EmptyState` component with a helpful icon, descriptive explanation, and a primary action button (e.g. "No reports found — Create your first report").
- **Error Boundaries & Retries:** Show user-friendly error banners with a "Retry" button rather than unhandled white screens.
- **Fast Keyboard Navigation:** Medical lab technicians enter dozens of test results in sequence. Diagnostic result entry tables must support `Tab` and `Enter` key progression without requiring mouse clicks.

### 1.7 Clinical & Operational Guardrails
- **Date & Timezone Standard:**
  - All database timestamps are strictly stored in **UTC (ISO-8601)** (`timestamptz`).
  - Frontend displays dates strictly in **Indian Standard Time (IST / Asia/Kolkata)** (e.g. `10 Sep 2026, 01:15 PM`). Never store or pass pre-formatted date strings.
- **Indian Currency & Number Formatting (INR):**
  - Never concatenate raw strings for currency (`'₹' + amount`).
  - Use native `Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' })`.
  - Store monetary amounts as `decimal(10,2)` on backend.
- **Tenant Security (JWT-Only Resolution):**
  - Frontend **never** passes `labId` as a request parameter for tenant-scoped operations.
  - Backend strictly resolves `user.labId` from the verified Supabase JWT via `@CurrentUser()`.
- **Standardized Mutation Feedback:**
  - Every mutation (saving CBC values, patient creation, finalizing report) must provide immediate toast feedback (success toast with entity identifier, or actionable error toast with backend error message).
  - Never perform optimistic updates on finalized medical reports.

### 1.8 Responsiveness, Mobile/Desktop Ergonomics & Modern Styling
- **Full Viewport Support:**
  - Layouts must be tested and responsive across:
    - **Mobile (<640px / 360–430px):** Single-column stacked layouts, touch-friendly interfaces.
    - **Tablet (640px–1024px):** 2-column adaptive views, scrollable tabular regions.
    - **Desktop / Laptop (1024px–1440px):** Multi-column dashboard with fixed/collapsible navigation.
    - **Ultra-Wide (>1440px):** Constrained max-width containers (`max-w-7xl` or fluid grid) preventing visual stretching.
- **Sidebar & Navigation Ergonomics:**
  - **Desktop:** Persistent, collapsible sidebar with expanded (icon + label) and compact (icon + tooltip) states.
  - **Mobile:** Hamburger toggle opening an accessible slide-over drawer/sheet (`Sheet` from Radix UI) with overlay backdrop, auto-closing upon route navigation.
- **Mobile vs Desktop UX Differentiation:**
  - **Tables vs Cards:** Render dense tabular data grids with sticky headers on desktop; seamlessly adapt to structured summary card lists on mobile to eliminate clumsy horizontal viewport overflow.
  - **Modals vs Bottom Sheets:** Centered dialog modals on desktop; fluid slide-up bottom sheets on mobile for thumb-zone ergonomics.
  - **Touch Targets & Virtual Keyboards:** Minimum 44x44px touch targets on mobile (WCAG AAA). Use `inputMode="decimal"` or `inputMode="numeric"` for clinical parameters, prices, and phone numbers to automatically summon the mobile numeric keypad.
- **CSS Container Queries (`@container`):**
  - Employ CSS container queries (`@container (min-width: ...)`) for modular cards, widgets, and stat blocks so components respond fluidly to their parent container's dimensions rather than relying solely on global viewport breakpoints.
- **SCSS / Custom Stylesheet Standard:**
  - Primary styling uses Tailwind CSS v4 design tokens and utility composition via `cn()`.
  - If a dedicated custom stylesheet is ever required beyond Tailwind utilities, create a `.scss` file using modern SCSS syntax, CSS variables, and modular/BEM naming. Never create ad-hoc unstructured global `.css` files.

### 1.9 Print Subsystem & Modal Isolation Architecture
- **Global `@media print` Engine (`globals.css`):**
  - Never scatter ad-hoc `print:hidden` classes across individual components.
  - The central `@media print` block in `globals.css` governs all print behavior:
    - Automatically suppresses all shell chrome: `aside, header, nav, .live-activity-drawer, .realtime-toast-stack, [data-radix-portal]:not(.printable-modal), button:not(.print-keep)`.
    - Forces `@page { size: A4 portrait; margin: 10mm; }` with zero header/footer margin clipping.
    - Preserves background colors and barcodes via `print-color-adjust: exact` and `-webkit-print-color-adjust: exact`.
- **Modal Print Isolation Pattern:**
  - When printing from a modal dialog (e.g. invoice receipt, TDS settlement voucher), mark the modal with `.printable-modal`.
  - The modal backdrop and parent page must be hidden via `.printable-modal-backdrop { display: none !important; }` and `.printable-modal-content` must expand to 100% width with transparent background to prevent grey overlay captures.

### 1.10 Polymorphic Shell Layouts & URL-Synchronized State
- **Polymorphic `AppShell` Variants:**
  - Every route must use `<AppShell variant="...">`:
    - `contained` (default): Constrained `max-w-5xl` for linear forms and settings.
    - `full-bleed`: Full-width container (`w-full px-4 sm:px-6 lg:px-8`) for data-dense worklists (`/accessions`, `/billing`, `/patients`, `/referrals`, `/collections`). Never choke tables in `max-w-7xl`.
    - `workspace`: Height-locked `h-[calc(100vh-3.5rem)] overflow-hidden` with independent column scrolling for split consoles (e.g. `/reports/[id]/entry`).
- **URL-Synchronized Query State (`useURLState`):**
  - Table pagination (`page`, `limit`), search terms, and filter tabs must sync to URL query parameters via `useURLState`. Never store pagination in React `useState` alone.
  - To support Next.js App Router static pre-rendering, any component invoking `useSearchParams()` (like `TablePagination`) must wrap its inner reader in `<React.Suspense>`.
- **Shared Primitives Benchmark:**
  - Every page must use standard primitives:
    - `<PageHeader>` for breadcrumbs, title, clinical subtitle, back navigation, and action slots.
    - `<EmptyState>` for empty query results with contextual icons and CTA buttons.
    - `<EllipsisCell>` with ResizeObserver tooltip for table cells with variable text lengths.
    - `<TableSkeleton>` for pulse loading rows.
    - `<Modal>` for accessible focus-trapped dialogs with Escape key listeners.

### 1.11 Demo Data Separation Standard
- **No Inline Fixtures in API Hooks:**
  - Hardcoded seed arrays (`DEMO_DOCTORS`, `DEMO_COLLECTIONS`, `DEMO_INVOICES`, `DEMO_PANELS`) must never live inside API hook files (`src/features/*/api/`).
  - All mock fixtures must reside cleanly in `src/lib/demo-data/` with dedicated domain modules (`doctors.ts`, `panels.ts`, `collections.ts`, `billing.ts`, `index.ts`).
  - API hooks remain pure queries/mutations and import seed fallbacks from `@/lib/demo-data`.

---

## 2. Backend Architecture & Engineering Standards (`labOS-service`)

### 2.1 Modular Monolith with DDD Bounded Contexts
- The codebase is structured around real medical business domains, not technical layers:
  - `labs` — Tenancy, lab profile, branding, credentials, staff roles
  - `panels` — Test panels, parameters, reference intervals, packages, templates
  - `reports` — Patients, accession orders, result values, amendments, vector PDF
  - `referrals` — Referring doctors, commission ledger & settlement, outsourced tests
  - `billing` — Invoices, line items, payments, operational expenses, profit analytics
  - `notifications` — Pluggable WhatsApp/Email dispatch, delivery logs
  - `collections` — Phlebotomy home collection requests, technician dispatch, tube barcodes
  - `websockets` — Real-time Socket.IO gateway, CloudEvents envelopes, room routing
  - `shared` — Base entities, guards, decorators, database configs, exception filters

### 2.2 CQRS & Vertical Slice Architecture
- **One Use Case = One Vertical Slice:**
  Feature code is organized by use-case folder (`features/<feature>/<slice>/`), containing:
  - `<slice>.command.ts` or `<slice>.query.ts` — Typed request intent
  - `<slice>.dto.ts` — Input validation using `class-validator` and `class-transformer`
  - `<slice>.handler.ts` — Business execution logic
  - `<slice>.controller.ts` — Dedicated REST endpoint controller
  - `<slice>.module.ts` — Feature dependency injection container
- **Direct Handler Injection (Superior to Dynamic CommandBus):**
  - Slice controllers directly inject their use-case handler:
    `constructor(private readonly handler: CreateReportHandler) {}`
  - **Why?** Guarantees 100% compile-time type safety for return types, avoids runtime reflection overhead, and allows instant IDE code navigation (`Cmd+Click` directly opens the handler).

### 2.3 Domain Layer Purity & Rich Value Objects
- **Zero Framework Imports in Domain:**
  Domain entities (`domain/<entity>/<entity>.entity.ts`) and value objects must never import NestJS framework decorators (`@Injectable()`, `@Controller()`), HTTP exceptions, or presentation libraries.
- **Rich Value Objects:**
  Domain logic belongs inside entities and value objects, not procedural helper files. For example, `NormalRange` encapsulates reference interval logic and evaluates its own out-of-range status (`isOutOfRange(value, sex)`).
- **Optimistic Concurrency Locking:**
  All mutable root entities extend `BaseDomainEntity`, which provides a `version` integer column automatically incremented on updates to prevent concurrent overwrite bugs.

### 2.4 Repository Pattern with Dependency Inversion (DIP)
- **Domain defines the contract:** Repository interfaces live in `domain/<entity>/interfaces/<entity>.repository.interface.ts` (e.g. `IReportRepository`).
- **Infrastructure implements the contract:** Concrete TypeORM repositories live in `infrastructure/database/repositories/<entity>.repository.ts` (e.g. `ReportRepository`).
- Application handlers inject the repository via token (`@Inject(REPORT_REPOSITORY_TOKEN)`), ensuring business logic is 100% decoupled from database drivers.

### 2.5 Enum Storage via ValueTransformers
- **Integers in Database, Strings in TypeScript:**
  Enums are stored as compact `smallint` integers in PostgreSQL and exposed as descriptive string enums in TypeScript via `createEnumTransformer(Mapper, Enum)`.
  This preserves database storage efficiency and indexing speed while maintaining full developer readability.

### 2.6 Background Job Processing (`pg-boss`)
- **Zero-Infra Queueing:**
  Heavy, non-blocking tasks (vector PDF generation, batch WhatsApp notifications) are offloaded to `pg-boss` via `QueueModule` and `PgBossService`.
  This leverages our existing PostgreSQL instance using `SKIP LOCKED` row-level locks, eliminating the need to host, manage, and pay for Redis or RabbitMQ clusters.

### 2.7 Real-Time WebSockets Architecture
- **Standardized CloudEvents Envelope (`EventMessage<T>`):**
  All WebSocket frames follow a uniform structure: `{ channels, event, timestamp, traceId, payload }`.
- **Multi-Tenant Room Isolation:**
  Sockets are partitioned strictly by tenant rooms via `RealtimeRoomBuilder`:
  - `lab:{labId}` — Lab-wide notifications
  - `lab:{labId}:doctors` — Pathologist-only alerts
  - `lab:{labId}:phlebotomists` — Field pickup alerts
  - `user:{userId}` — Direct personal messages
- **In-Process Domain Event Bridging:**
  Domain events emitted via `EventEmitter2` (e.g. `'report.finalized'`) are caught by in-process listeners and broadcast to WebSocket rooms in microseconds without HTTP loopback calls.

---

## 3. Improvements Over Legacy / Reference Architectures

Key architectural improvements established in LabOS compared to reference codebases (e.g. `residency-backend`):

| Area | Reference Pattern | LabOS Senior Standard | Benefit |
|---|---|---|---|
| **API Docs** | Heavy runtime `@ApiProperty()` decorators on every DTO | Clean, modular OpenAPI 3.0 contracts (`.openapi.yaml`) per module | Zero code pollution, zero runtime overhead, 100% accurate API contracts |
| **Handler Invocation** | Untyped CQRS Bus (`commandBus.execute()` returning `any`) | Direct Handler Injection in Slice Controllers | 100% compile-time type safety, zero reflection lag, instant IDE jump |
| **Real-Time Gateway** | Ad-hoc socket emit strings (`socket.emit('data', ... )`) | CloudEvents envelope (`EventMessage<T>`) + `RealtimeRoomBuilder` | Zero magic strings, room-isolated multi-tenancy, structured tracing |
| **Queue Architecture** | Redis / BullMQ requiring external services | Postgres-backed `pg-boss` queue engine | Zero extra servers, zero cost, atomic transactional job safety |
| **HTTP Errors** | Handlers throwing HTTP exceptions (`NotFoundException`) | Handlers throw typed Domain Exceptions; mapped by `AllExceptionsFilter` | Handlers are presentation-agnostic and reusable in CLI, cron, & queues |
| **Frontend API Client** | Ad-hoc `fetch` with manual `localStorage` and `typeof window` | Clean Axios instance with Supabase session interceptor & `StatusCodes` | SSR-safe, token-refresh aware, typed status checks, zero boilerplate |

---

## 4. SOLID Principles Applied Concretely

1. **Single Responsibility (SRP):**
   - Each module owns exactly one bounded context.
   - Each vertical slice owns exactly one command or query.
   - Each component owns exactly one visual or interaction responsibility.
2. **Open/Closed (OCP):**
   - Adding new diagnostic parameters, test panels, or notification providers is done via configuration or new implementations of existing interfaces, without modifying existing domain handlers.
3. **Liskov Substitution (LSP):**
   - Notification providers (`WhatsAppCloudProvider`, `MockNotificationProvider`) implement `INotificationProvider` and can be swapped with zero changes to caller code.
4. **Interface Segregation (ISP):**
   - Repository interfaces define only the operations needed by their specific aggregate root, avoiding bloated monolithic database interfaces.
5. **Dependency Inversion (DIP):**
   - High-level domain and application logic depends exclusively on domain interfaces, never on concrete TypeORM entities, Supabase SDKs, or external network drivers.

---

## 5. Naming Conventions Cheat Sheet

| Layer | Concept | Suffix / Pattern | Example |
|---|---|---|---|
| **Backend** | Aggregate Entity | `.entity.ts` | `report.entity.ts` |
| **Backend** | Use-Case Controller | `.controller.ts` | `finalize-report.controller.ts` |
| **Backend** | Command (State Change) | `.command.ts` | `finalize-report.command.ts` |
| **Backend** | Query (Data Read) | `.query.ts` | `get-dashboard-stats.query.ts` |
| **Backend** | Handler | `.handler.ts` | `finalize-report.handler.ts` |
| **Backend** | Request DTO | `.dto.ts` | `finalize-report.dto.ts` |
| **Backend** | Slice Module | `.module.ts` | `finalize-report.module.ts` |
| **Backend** | Repository Contract | `.repository.interface.ts` | `report.repository.interface.ts` |
| **Backend** | Repository Implementation | `.repository.ts` | `report.repository.ts` |
| **Backend** | Domain Event | `.event.ts` | `report-finalized.event.ts` |
| **Backend** | Background Worker | `.worker.ts` | `report-pdf.worker.ts` |
| **Frontend** | Component (Folder + index) | `<name>/index.tsx` | `src/components/form/text-field/index.tsx` |
| **Frontend** | Query/Mutation Hook | `use-<name>.hook.ts` | `use-finalize-report.hook.ts` |
| **Frontend** | API Service Call | `<name>.service.ts` | `reports.service.ts` |
| **Frontend** | Zod Form Schema | `<name>.schema.ts` | `create-report.schema.ts` |
| **Frontend** | Domain Type Definition | `<name>.types.ts` | `reports.types.ts` |
