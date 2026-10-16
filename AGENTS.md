# LabOS — Agent Instructions & Operating Rules

This workspace follows strict senior/principal engineering standards, Domain-Driven Design, and mandatory documentation discipline across both `labOS-service` (NestJS) and `labOS-app` (Next.js).

## Core Engineering Mandate

**Do not optimize for "making it work." Optimize for building the right system.**

Every technical decision prioritizes what is architecturally correct, maintainable, scalable, reliable, and sustainable for the long term — not the easiest or quickest implementation.

---

## Mandatory Pre-Task Protocol for All AI Sessions

Before planning, suggesting, or writing ANY code in this repository:
1. **Understand Startup & Product Reality (`docs/product/prd.md`)**:
   - LabOS is an operating system for India's independent diagnostic labs (small, standalone labs with 1–3 staff, 10–50 samples/day, previously running on paper and WhatsApp).
   - Target users are often DMLT/BMLT biochemists/technicians or standalone lab owners, not necessarily MD doctors. Never force Western hospital EHR assumptions or illegal doctor prefixes onto lab owners.
   - Realities: Thermal sticker barcode printers (50×25mm), offset pre-printed letterheads from local presses, cash/UPI counter payments, WhatsApp PDF delivery, and NMC/NABL ISO 15189 standards.
2. **Review Project History & Active Context (`CONTEXT.md`)**:
   - Mandatory read of `CONTEXT.md` before touching any code. Check the latest session log, active feature state, and architectural decisions. Never break or re-invent already verified systems.
3. **Review Architectural Rules (`.agents/rules/`)**:
   - `.agents/rules/senior-architecture-principles.md`: Evaluate at least 2 architectural approaches before any implementation. Compare trade-offs.
   - `.agents/rules/frontend-engineering-standards.md`: Axios interceptors, TanStack Query cache, React Hook Form + Zod, URL-based state (`useURLState`), responsive sheet ergonomics, zero magic numbers.
   - `.agents/rules/backend-engineering-standards.md`: Modular monolith with DDD Bounded Contexts, CQRS vertical slices, integer enums, repository pattern with dependency inversion.
   - `.agents/rules/workflow-and-documentation-discipline.md`: Break tasks into small atomic sub-tasks, explain architectural rationale, update `CONTEXT.md` after every task, and run verification (`npm run lint` and `npm run build`).

---

## Permanent Operating Rules
All permanent operating rules are defined in `.agents/rules/` and must be adhered to without exception:

1. **[Senior Engineering & Architecture Principles](.agents/rules/senior-architecture-principles.md)**:
   - **BEFORE ANY IMPLEMENTATION:** Evaluate at least two architectural approaches. Compare trade-offs. Choose the approach that yields the best long-term engineering outcome — not the most convenient one.
   - **CHROME VISUAL VERIFICATION (MANDATORY):** Before implementing any UI change, visually inspect all affected screens in Chrome. Take before/after screenshots. Never reason about layout bugs purely from JSX.
   - **CANONICAL CODEBASE REFERENCE:** All canonical architectural patterns (folder structure, naming conventions, `_components/` private sub-components, `.service.ts`/`.queries.ts`/`.mutations.ts` service layer split, `useURLState` pagination, `EllipsisCell`, `NetworkStatus`, and socket hooks) are fully established inside `src/` itself and codified in `docs/conventions/coding-standards.md`.
   - **REFACTORING IS NOT OPTIONAL:** Do not preserve a poor design because it already exists. Significant architectural refactoring is the correct choice when it yields a substantially better system.
   - **NON-NEGOTIABLES:** No `any` types. No magic numbers. No inline demo data in hooks. No component >200 lines without decomposition. No `window.print()` without `@media print` isolation. No React `useState` for URL-worthy data (`page`, `search`, `filter`).
   - **IDENTIFY TECHNICAL DEBT BEFORE ADDING TO IT:** Always document architectural smells, scalability risks, and missing patterns before building on top of them.

2. **[Workflow & Documentation Discipline](.agents/rules/workflow-and-documentation-discipline.md)**:
   - **BEFORE ANY TASK:** Mandatory read of `CONTEXT.md` and `docs/conventions/coding-standards.md`. For frontend tasks, inspect the corresponding backend feature slice directly (`labOS-service/src/modules/<module>/features/<feature>/`). Never guess types.
   - **PHASE DECOMPOSITION:** Break large features or phases into small, atomic, testable sub-tasks. Execute sequentially with intermediate verification.
   - **ARCHITECTURAL RATIONALE:** Always explain the architectural decisions, trade-offs, and "Why" behind proposed changes.
   - **AFTER EVERY TASK:** Update `CONTEXT.md`, synchronize related documentation, and run automated verification (`npm run build` and `npm run lint`).

3. **[Frontend Engineering Standards](.agents/rules/frontend-engineering-standards.md)**:
   - Axios with interceptors (Supabase session resolution, no raw `fetch`, zero `typeof window` hacks).
   - Strict `StatusCodes` from `http-status-codes` (zero magic numbers).
   - TanStack Query cache management + React Hook Form with Zod schemas.
   - Folder + `index.tsx` component structure (`src/components/ui/`, `src/components/form/`, `src/components/shared/`, private `_components/` for internal sub-components).
   - Mandatory UX states: loading skeletons, empty states, error boundaries, keyboard navigation.
   - **Responsive & Adaptive Ergonomics:** Mobile (<640px) drawer navigation, card conversions for wide tables, bottom sheets, 44px touch targets; desktop collapsible sidebar and dense data grids; CSS container queries (`@container`).
   - **Service Layer:** API calls in `.service.ts` (pure, no hooks). TanStack Query hooks in `.queries.ts`/`.mutations.ts`. Demo seed data in `src/lib/demo-data/`. No mixing.
   - **URL-State:** Filters, search, and pagination live in URL query params (`useURLState` hook). Never in `useState` alone.

4. **[Backend Engineering Standards](.agents/rules/backend-engineering-standards.md)**:
   - Modular Monolith with DDD Bounded Contexts.
   - CQRS + Vertical Slice Architecture with direct handler injection in slice controllers (100% compile-time type safety).
   - Repository Pattern with Dependency Inversion (domain interfaces, infrastructure implementations).
   - Integer enums via TypeORM ValueTransformers.
   - Domain Exceptions mapped by `AllExceptionsFilter`.
   - CloudEvents WebSocket gateway (`EventMessage<T>`) with room isolation (`RealtimeRoomBuilder`).
   - PostgreSQL background job queue (`pg-boss`).

5. **Clinical & Operational Guardrails**:
   - Timestamps: UTC in database (`timestamptz`), IST (`Asia/Kolkata`) on frontend.
   - Currency: INR formatting via `Intl.NumberFormat('en-IN')`.
   - Multi-tenancy: Strict JWT-only `labId` extraction on backend.

---

## Master Documentation References
- Master Tracking Log: `CONTEXT.md`
- Master Engineering Guide: `docs/conventions/coding-standards.md`
- Master Code Audit & Refactoring Protocol: `docs/conventions/master-code-audit-and-refactoring-protocol.md`
- System Architecture: `docs/architecture/system-design.md`
- Product PRD: `docs/product/prd.md`
- OpenAPI Contracts: `docs/openapi.yaml`

