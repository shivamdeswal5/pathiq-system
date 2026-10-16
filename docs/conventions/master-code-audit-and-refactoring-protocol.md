# LabOS — Master Code Audit, Refactoring & Quality Protocol

**Document Version:** 2.0.0  
**Target Scope:** Full-Stack — `labOS-app` (Next.js 15, React 19, TypeScript) & `labOS-service` (NestJS, TypeScript, TypeORM, PostgreSQL / Supabase)  
**Authority:** Derived from `docs/product/prd.md`, `.agents/rules/`, `docs/architecture/system-design.md`, SonarQube Clean Code Taxonomy, OWASP, React 19 / Next.js 15 best practices, NestJS production standards, and DDD / Clean Architecture first principles.  
**Purpose:** A comprehensive, executable, file-by-file code audit protocol for systematically eliminating AI-generated bloat, hardening TypeScript type-safety, enforcing DDD, identifying security vulnerabilities, and establishing permanent production-grade coding standards for LabOS.

> **IMPORTANT:** This is the **authoritative code review document** for every refactoring session. Before auditing any file, read and internalize all sections in this protocol. Do not rewrite everything. Prioritize correctness, security, and reliability above style.

---

## 0. Product & Operational Context (Read Before Any Review)

LabOS is a clinical Lab Information System for India's standalone diagnostic labs (1–3 staff, 10–50 samples/day, previously paper-based). Key operational realities that must inform every code decision:

| Domain Constraint | Code Implication |
|---|---|
| Standalone Indian lab owners (DMLT/BMLT), not MDs | Never force "Dr." prefix. Respect the literal `fullName` field. |
| Cash/UPI counter payments, WhatsApp PDF delivery | Receipts, report PDFs, and WhatsApp integrations are core, mission-critical flows. |
| Thermal barcode printers (50×25mm), offset letterheads | Print CSS isolation is not optional. `@media print` is a clinical accuracy requirement. |
| 10–50 reports/day peak concurrency | N+1 query bugs and memory-resident full-table joins are operational risks, not just smells. |
| Clinical stakes: patient diagnoses depend on correct results | Silent error swallowing and fake fallback data are patient safety issues, not just bad UX. |

---

## 1. Understanding the Architecture Before Judging It

Before raising any architectural finding, understand the established design decisions:

### 1.1 Backend — `labOS-service`
- **Modular Monolith + DDD Bounded Contexts** under `src/modules/` (`labs`, `panels`, `reports`, `referrals`, `billing`, `notifications`, `collections`, `websockets`, `shared`)
- **CQRS + Vertical Slice Architecture**: one use-case = one folder (`features/<feature>/<slice>/` containing `.controller.ts`, `.handler.ts`, `.dto.ts`, `.command.ts` / `.query.ts`, `.module.ts`)
- **Direct Handler Injection**: controllers inject handlers directly — no `CommandBus` indirection. This is intentional for 100% compile-time type safety. Do NOT flag this as a defect.
- **Repository Pattern + DIP**: handlers depend on domain repository interfaces (`@Inject(TOKEN)`), not TypeORM `DataSource` directly.
- **Integer Enums + ValueTransformer**: database stores integers (`0, 1, 2`), TypeScript uses string enums (`ReportStatusEnum.DRAFT`), bridged by `createEnumTransformer`.
- **Domain Exceptions → AllExceptionFilter**: `domain/exceptions/*.exception.ts` extend `DomainException`, which `AllExceptionFilter` maps to HTTP status codes at the boundary. Never throw NestJS HTTP exceptions inside handlers.
- **`BaseDomainEntity`**: all mutable root entities carry `id` (UUID), `createdAt` (timestamptz), `updatedAt` (timestamptz), and `version` (optimistic concurrency).

### 1.2 Frontend — `labOS-app`
- **Next.js App Router** (React Server Components + Client Components)
- **Supabase Auth** via `@supabase/ssr` for middleware-level session, and `supabase-browser` for client-side token attachment in Axios interceptor
- **Axios** (`src/lib/api-client.ts`) with request interceptor (attaches Bearer token) and response interceptor (unwraps payload, maps errors to `ApiError`)
- **TanStack Query** for all server state; mutations invalidate related query keys
- **React Hook Form + Zod** for all forms
- **`useURLState`** hook (`src/hooks/use-url-state.ts`) for URL-synchronized filter/search/pagination state
- **Component classification**: `src/components/ui/` (Radix / shadcn primitives), `src/components/shared/` (cross-feature widgets), `src/features/<feature>/components/` (domain-private), `_components/` (private sub-components)
- **Demo data**: extracted to `src/lib/demo-data/` — hooks must not contain inline mock arrays

---

## 2. What NOT to Do in This Review

> **CAUTION — Violating these constraints invalidates the review.**

1. **Do not recommend rewriting the entire application** because a theoretically cleaner architecture exists.
2. **Do not flag architecture patterns as bugs** if they are intentional, documented decisions (e.g., direct handler injection instead of `CommandBus`).
3. **Do not recommend newer APIs merely because they are newer** — verify the recommendation against the project's actual dependency versions.
4. **Do not recommend micro-optimizations** (`useMemo`, `useCallback`, `React.memo`) without a demonstrated re-render problem.
5. **Do not treat style preferences as bugs** — distinguish between correctness issues, maintainability issues, and personal style choices.

---

## 3. The AI-Generated Code Problem Catalog

Assume every file may carry these specific defects from iterative AI assistance:

### 3.1 Silent Error Swallowing with Demo Fallbacks (CRITICAL)

**This is the most dangerous class of defect in this codebase.**

```typescript
// CRITICAL ANTI-PATTERN — Active in: use-reports.ts, use-report.ts, use-billing.ts, use-panels.ts
queryFn: async () => {
  try {
    const data = await api.get<Invoice[]>('/invoices');
    if (Array.isArray(data) && data.length > 0) return data;
    return DEMO_INVOICES; // returns fake data if API returns empty array (new lab has 0 records!)
  } catch {
    return DEMO_INVOICES; // swallows all errors — hides backend outages, 401s, 500s
  }
}
```

**Why it is critical in LabOS:**
- A backend timeout or 401 expiry causes the app to display **fictitious patient records** to lab staff.
- `if (Array.isArray(data) && data.length > 0)` silently replaces real empty states (new lab with zero records) with fake seed data.
- The `catch` block with no argument discards the error type completely.

**Correct production pattern:**
```typescript
// report.service.ts — pure Axios call, no hooks, no fallbacks
export const reportService = {
  list: (params?: { status?: ReportStatus; patientId?: string }) =>
    api.get<DetailedReport[]>('/reports', { params }),
};

// use-reports.queries.ts — TanStack Query orchestration
export function useReports(params?: { status?: ReportStatus; patientId?: string }) {
  return useQuery({
    queryKey: reportKeys.list(params),
    queryFn: () => reportService.list(params),
    staleTime: 15_000,
  });
}
```

TanStack Query's `error` state propagates to the UI which renders an `<ErrorState>` component with a retry button.

### 3.2 Defensive Fallback Pyramids (HIGH)

```typescript
// AI ANTI-PATTERN — Guessing data shapes speculatively
const name = data?.patient?.name ?? data?.patientName ?? data?.name ?? 'Unknown';
const phone = report?.patient?.phone ?? report?.phone ?? 'N/A';

// CLEAN — Guaranteed contract mapping (inspect backend DTO first)
const { patient, refByDoctor } = report;
const doctorName = refByDoctor?.name ?? 'Direct Walk-in'; // nullable by contract, one coalesce
```

**Rule:** If the backend contract guarantees a field is non-null, type it as non-optional and access it directly. If a field is legitimately nullable, handle it with one deliberate nullish coalesce or a conditional render.

### 3.3 Manual URL Query Parameter Construction (MEDIUM)

```typescript
// AI ANTI-PATTERN — Active in use-billing.ts, use-reports.ts
const queryParams = new URLSearchParams();
if (filter?.status && filter.status !== 'ALL') queryParams.append('status', filter.status);
const queryString = queryParams.toString() ? `?${queryParams.toString()}` : '';
const data = await api.get<Invoice[]>(`/invoices${queryString}`);

// CLEAN — Axios handles params serialization natively
const data = await api.get<Invoice[]>('/invoices', { params: { status: filter?.status } });
```

### 3.4 Inline Mock Data in API Hook Files (HIGH)

Active violations: `use-reports.ts` (~400 lines of `DEMO_ACCESSIONS`), `use-report.ts` (`DEMO_REPORT` complete fixture).

**Rule:** `src/lib/demo-data/` is the correct home for all mock fixtures. Hooks must never contain inline data objects.

### 3.5 Loose Typing and Unsafe Casts (HIGH)

```typescript
// ANTI-PATTERN — Active in list-reports.handler.ts
(report as unknown as { invoice: unknown }).invoice = invoiceByReportId.get(report.id) ?? null;

// ANTI-PATTERN — In all-exception.filter.ts
const resObj = res as Record<string, any>; // 'any' contradicts the zero-any rule

// ANTI-PATTERN — In api-client.ts
apiClient.get(url, config).then((res) => res as unknown as T) // double cast is a smell
```

### 3.6 In-Memory Full-Table Joins (CRITICAL for Scalability)

```typescript
// ANTI-PATTERN — Active in list-reports.handler.ts
const [invoices, doctors] = await Promise.all([
  this.invoiceRepository.findByLabId(query.labId), // Fetches ALL invoices for the lab
  this.doctorRepository.findByLabId(query.labId),  // Fetches ALL doctors for the lab
]);
// Then manually stitches associations in memory with new Map()
```

A lab doing 50 reports/day for a year has ~18,000 invoices. Loading all into Node.js heap on every list call is a scalability bomb.

### 3.7 Deprecated/Outdated Patterns (MEDIUM)

- `onSuccess`, `onError`, `onSettled` inside `useQuery` options: deprecated in TanStack Query v5. Break in concurrent rendering.
- `window.location.href = '/onboarding'` in `auth-provider.tsx` and `api-client.ts`: should be `useRouter().push()`. **8 lint warnings confirm this.**
- `<img>` tags instead of `<Image />` from `next/image`: **confirmed by 4 lint warnings** in `a4-document-sheet.tsx`, `report-letterhead-preview.tsx`, `branding-letterhead-section.tsx`.
- Client-side filtering in `use-billing.ts`: `filterInvoices()` function duplicates what the backend query should handle.

### 3.8 Anemic Domain Models (HIGH)

Domain logic scattered in procedural helpers instead of encapsulated in Value Objects:
- Normal range out-of-range evaluation should live in a `NormalRange` Value Object with `isOutOfRange(value, sex)`.
- Report status transition validation (can only finalize a DRAFT) should be an entity method, not a handler `if` check.

### 3.9 Magic Query Keys / Invalidation Mismatches (MEDIUM)

```typescript
// ANTI-PATTERN — use-create-report.ts invalidates 'finance' but key is 'billing/financial-summary'
queryClient.invalidateQueries({ queryKey: ['finance'] }); // does NOTHING — wrong key

// CLEAN — Query Key Factory prevents typos
export const billingKeys = {
  financialSummary: () => ['billing', 'financial-summary'] as const,
};
queryClient.invalidateQueries({ queryKey: billingKeys.financialSummary() });
```

---

## 4. The Universal Review Prompt (Copy & Paste for Each File/Slice)

When instructed to review any file or vertical slice, execute the following prompt verbatim:

---

```
LABOS CODE AUDIT — FILE-BY-FILE REVIEW

Target: [INSERT FILE PATH]
Context: [Frontend feature / Backend slice / Shared infrastructure / Provider / Middleware]
Backend Contract Reference: [INSERT BACKEND SLICE PATH if frontend file]

You are acting as a Principal Engineer and SonarQube Quality Auditor for LabOS.
Your mandate is to find real bugs, security vulnerabilities, maintainability issues,
and architectural violations — NOT to rewrite for stylistic preference.

Execute in this exact sequence:

PHASE 1: UNDERSTAND BEFORE JUDGING
- Read the full file. Understand its purpose and context.
- For frontend files: immediately inspect the corresponding backend slice
  (labOS-service/src/modules/<module>/features/<feature>/<slice>/)
  to understand the actual API contract before commenting on types or fallbacks.
- For backend files: check the TypeORM entity, migration, and domain interface
  to verify schema matches handler behavior.
- Identify what this file is supposed to do vs what it actually does.
- Do NOT raise findings until Phase 1 is complete.

PHASE 2: CORRECTNESS & CONTRACT VERIFICATION
- Do frontend TypeScript types match the backend response shape 1:1?
  - Flag any field typed as optional that the backend guarantees as non-null.
  - Flag any field assumed present that the backend marks as nullable.
- Are all API request payloads matching the backend DTO exactly?
- Are query key invalidations actually invalidating the right keys?
- Are enum values on the frontend matching backend enum names exactly?

PHASE 3: CRITICAL BUGS & SECURITY
- Error swallowing: Does any catch block silently return fake data or swallow errors?
- Auth bypass: Is any route or action accessible without authentication?
- Authorization: Does any handler/controller miss a tenant isolation check
  (i.e., does it query data without enforcing labId from @CurrentUser)?
- Sensitive data: Is any secret, token, or PII logged or exposed in error messages?
- Input validation: Are all user inputs validated via class-validator (backend)
  or Zod (frontend) before being used in queries or responses?
- SQL/injection risks: Are raw queries constructed with user-supplied strings?
- XSS: Is user-supplied HTML rendered unsafely (dangerouslySetInnerHTML)?
- Client-side authorization: Is any security check done only on the frontend
  without a corresponding backend guard?

PHASE 4: REACT / NEXT.JS REVIEW (Frontend files only)
- Server vs Client Component: Is 'use client' present only where actually needed?
  Could this be a Server Component or at least have its interactive parts split out?
- Data fetching strategy: Is data fetched in a Server Component where possible?
  TanStack Query hooks belong in Client Components only.
- Effect correctness: Are all useEffect dependency arrays complete and correct?
  Are there stale closure bugs?
- Controlled/uncontrolled: Are form inputs properly controlled with React Hook Form?
- Loading states: Is there a skeleton or meaningful loading indicator?
- Empty states: Is there a meaningful empty state when the query returns []?
- Error states: Does the component handle the query.error case?
- Unnecessary state: Is there useState for data that should be derived from
  query results or from URL params via useURLState?
- Unnecessary memoization: Are useMemo/useCallback used without a demonstrated
  performance problem?
- Component size: Is this component over ~200 lines? Which sub-responsibilities
  should be extracted into _components/?

PHASE 5: BACKEND REVIEW (Backend files only)
- Layer purity: Does this file import from a layer it should not depend on?
  (Domain imports nothing framework-specific; Application imports domain only;
   Presentation imports application and framework)
- Handler responsibility: Does this handler do exactly ONE use-case?
- Repository usage: Does the handler inject repository interfaces, not DataSource?
- Read model: For queries returning multi-entity associations, is a dedicated Read DTO
  used instead of mutating TypeORM entities with dynamic properties?
- Domain exceptions: Does the handler throw only DomainException subclasses?
  (Never NestJS HttpException inside handlers)
- DTO validation: Is the request DTO decorated with class-validator decorators?
  Is ValidationPipe configured with whitelist: true, forbidNonWhitelisted: true?
- N+1 queries: Does the handler make database calls inside a loop?
  Use Promise.all for parallel fetches, or a single query with WHERE IN.
- Full-table scans: Does any findAll() or findByLabId() fetch unbounded data?
- Transaction boundaries: Are multi-step mutations wrapped in a TypeORM transaction?
- Tenant isolation: Does every database query include a labId WHERE clause?

PHASE 6: API & INTEGRATION REVIEW
- HTTP method correctness: GET reads, POST creates, PATCH partial updates,
  PUT full replacements, DELETE deletions.
- Response shape consistency: Does the response match frontend type definitions?
- Pagination: For list endpoints with potentially large result sets, is cursor-based
  pagination (created_at + id) implemented?
- Error response consistency: Are all errors returned via AllExceptionFilter
  in the standard ErrorResponse shape?
- Status code accuracy: Is 204 returned for void operations? 201 for creates?
  Are StatusCodes from http-status-codes used on the frontend?

PHASE 7: SONARQUBE CLEAN CODE
- Cognitive Complexity: Are any functions above complexity 15?
  (Nested if/else, ternary chains in map callbacks, multi-level try-catch)
  Fix with guard clauses and early returns.
- Dead code: Are there unused imports, unreachable branches, commented-out blocks?
- Duplication: Is any logic copy-pasted from another file?
- Magic numbers/strings: Are raw HTTP status codes, timeout ms, or limit numbers
  used without a named constant?
- Long functions: Is any function over ~40 lines? Extract sub-responsibilities.
- Poor naming: Do names communicate intent clearly?
  Are domain terms from the PRD used consistently?

PHASE 8: PERFORMANCE (Only flag real problems)
- Is there a demonstrable N+1 query pattern (loop with DB call inside)?
- Are list endpoints returning unbounded datasets that will grow with usage?
- Are images using <img> instead of next/image (CLS, LCP regression)?
- Is there any synchronous heavy computation blocking the render thread?

PHASE 9: TESTING GAPS
- Is this file covered by unit tests?
- If it contains domain logic (Value Objects, entity methods), is it unit-testable
  without spinning up a full database?
- If it contains a critical security path (auth guard, tenant isolation),
  is there an integration test?
- Flag specific untested scenarios for each critical function.

PHASE 10: PRODUCE STRUCTURED OUTPUT

## Review: [File Path]

### Executive Summary
[1-3 sentences: overall condition, major risk, and primary action required]

### Findings

| # | Category | Severity | Line(s) | Issue | Recommended Fix |
|---|---|---|---|---|---|
| 1 | [Bug/Security/Architecture/TypeScript/React/Backend/API/Performance/SonarQube/Deprecated/Duplication/Testing] | [Critical/High/Medium/Low] | [L10-20] | [What is wrong] | [What to do instead] |

### Before / After (for Critical & High findings only)

**Finding #N — [Short Title]**

BEFORE: [current code with problem highlighted]
AFTER: [clean replacement]
WHY: [one-sentence rationale grounded in correctness or safety, not preference]

### Refactoring Priority
- Critical (fix before any production use): [list]
- High (fix in next refactoring session): [list]
- Medium (address during feature work): [list]
- Low (optional cleanup): [list]

### Line Count
Before: N lines — After (estimated): M lines (~X% reduction)
```

---

## 5. Documentation & Rule Audit

Evaluate existing rules. If a rule conflicts with real-world best practices, flag it:

| Rule | Location | Current Behavior | Concern | Recommended Change | Decision |
|---|---|---|---|---|---|
| `<component>/index.tsx` naming | `senior-architecture-principles.md` | Every component file named `index.tsx` | IDE "Index Tab Hell" — 15 tabs titled `index.tsx`; ambiguous Sentry stack traces | Use `<component>/<component>.tsx` for implementation + `index.ts` for one-line re-export | **Confirm — low risk, high DX gain** |
| Silent demo fallback in hooks | Rules (no explicit prohibition, but tolerated) | `catch { return DEMO_DATA }` in production hooks | Clinical safety risk — hides errors, shows fake patient data | Prohibit catch-fallbacks; use `NEXT_PUBLIC_DEMO_MODE` env flag + MSW | **Confirm — critical safety fix** |
| `any` in `AllExceptionFilter` | `backend-engineering-standards.md` (no-any rule) | `as Record<string, any>` in shared error filter | Contradicts the "no any" rule in the same ruleset | Use `as Record<string, unknown>` with explicit narrowing | **Fix immediately** |
| `window.location.href` navigation | `frontend-engineering-standards.md` | Used in `auth-provider.tsx` and `api-client.ts` | 8 lint warnings; bypasses Next.js router; breaks back/forward navigation | Use `useRouter().push()` in Client Component event handlers | **Address in next session** |
| No Query Key Factory rule | Missing — not codified anywhere | String literals scattered across files | `['finance']` vs `['billing', 'financial-summary']` mismatch silently breaks invalidation | Mandate centralized Query Key Factory (`<feature>.keys.ts`) per feature slice | **Establish as new mandatory rule** |
| Client-side filtering in hooks | Missing prohibition | `filterInvoices()` inside `use-billing.ts` | Duplicates what the backend query should handle; runs on potentially large arrays in-browser | Move filtering to server-side query params; remove client-side filter functions | **Fix — remove frontend filter functions** |

---

## 6. Architecture Review

### 6.1 Backend — What Is Working Well

- Direct handler injection (no CommandBus) — compile-time type safety ✅
- Integer enum + ValueTransformer bridge — clean DB/TS enum parity ✅
- DomainException hierarchy mapped by AllExceptionFilter ✅
- CloudEvents WebSocket envelope with tenant room isolation ✅
- Single-table modular migrations per bounded context ✅
- `BaseDomainEntity` with optimistic concurrency `version` column ✅
- `getUser()` in middleware (server-validates JWT, not just cookie) ✅

### 6.2 Backend — What Needs Improvement

| Issue | Location | Severity | Fix |
|---|---|---|---|
| In-memory full-table joins | `list-reports.handler.ts` | Critical | Use `findByReportIds(ids)` + `findByIds(ids)` with `WHERE IN` |
| `as unknown as` entity mutation | `list-reports.handler.ts` | High | Introduce `ReportListItemDto` with a static `from()` factory |
| Manual UUID validation in handler | `list-reports.handler.ts` | Medium | Move to DTO with `@IsUUID(4)` decorator |
| Generic `Error` throws in handlers | Various handlers | High | Replace with specific `DomainException` subclasses |
| `any` cast in exception filter | `all-exception.filter.ts` | Medium | Replace with `unknown` + type narrowing |

### 6.3 Frontend — What Is Working Well

- `api-client.ts` interceptor pattern: Supabase token without `typeof window` hacks ✅
- `useURLState` for filter/search/pagination ✅
- `formatters.ts` with `Intl.NumberFormat('en-IN')` for INR and `Asia/Kolkata` IST ✅
- `demo-data/` directory exists and is correctly separate ✅
- CloudEvents event handling in `RealtimeProvider` ✅
- Correct middleware: `getUser()` not just `getSession()` ✅

### 6.4 Frontend — What Needs Improvement

| Issue | Location | Severity | Fix |
|---|---|---|---|
| Silent error swallowing + demo fallbacks | `use-reports.ts`, `use-report.ts`, `use-billing.ts`, `use-panels.ts` | Critical | Remove catch blocks; let errors propagate to Query's `error` state |
| Inline demo data in hook files | `use-reports.ts` (400 lines), `use-report.ts` | High | Move to `src/lib/demo-data/` (already exists) |
| Mixed Axios + hooks + data in one file | All feature `api/` files | Medium | Split into `.service.ts`, `.queries.ts`, `.mutations.ts` |
| `window.location.href` navigation | `auth-provider.tsx`, `api-client.ts` | High | Use `useRouter().push()` |
| `<img>` instead of `<Image />` | `a4-document-sheet.tsx`, `report-letterhead-preview.tsx`, `branding-letterhead-section.tsx` | High | Replace with `next/image` |
| Query key string mismatches | `use-create-report.ts` (invalidates `['finance']`) | High | Centralize in Query Key Factory |
| Client-side `filterInvoices()` | `use-billing.ts` | Medium | Remove; use backend filter via `params` |
| Large component files | `realtime-provider.tsx` (375 lines) | Medium | Decompose socket setup, toast rendering, event list into `_components/` |

---

## 7. Frontend Refactoring Playbook

### 7.1 Mandatory Service Layer Split

Every `features/<feature>/api/` directory must follow this 3-file structure:

```
features/<feature>/api/
├── <feature>.service.ts        # Pure Axios calls — no hooks, no React, pure Promise-returning functions
├── use-<feature>.queries.ts    # TanStack useQuery hooks — caching, stale time, query key factory
└── use-<feature>.mutations.ts  # TanStack useMutation hooks — cache invalidation, toast feedback
```

### 7.2 Query Key Factory Pattern (Mandatory New Rule)

Create a `<feature>.keys.ts` alongside each service:

```typescript
// features/reports/api/report.keys.ts
export const reportKeys = {
  all: ['reports'] as const,
  lists: () => [...reportKeys.all, 'list'] as const,
  list: (params?: ListReportsParams) => [...reportKeys.lists(), params] as const,
  details: () => [...reportKeys.all, 'detail'] as const,
  detail: (id: string) => [...reportKeys.details(), id] as const,
  pdf: (id: string) => [...reportKeys.all, 'pdf', id] as const,
} as const;
```

All invalidation calls use the factory — never raw strings:
```typescript
queryClient.invalidateQueries({ queryKey: reportKeys.lists() });
```

### 7.3 Eliminating Fallback Pyramids

Inspect the handler's return type first:

```typescript
// BEFORE — speculative optional chaining on every property
const reportNumber = report?.reportNumber ?? report?.number ?? 'N/A';
const age = patient?.age ?? patient?.ageText ?? '—';

// AFTER — type the interface correctly from the backend DTO
// DetailedReport.reportNumber: string  (non-null per backend entity)
// Patient.age: string | null           (nullable per backend entity)
const { reportNumber } = report;        // direct access — no optional chaining needed
const age = patient.age ?? '—';         // one deliberate nullish coalesce
```

### 7.4 Error State, Loading State, Empty State Triad (Mandatory)

Every query-backed component must handle all three states:

```tsx
function ReportList() {
  const { data, isLoading, isError, error, refetch } = useReports();

  if (isLoading) return <TableSkeleton rows={10} />;
  if (isError) return <ErrorState error={error} onRetry={refetch} />;
  if (!data || data.length === 0) return <EmptyState title="No reports yet" action="Create First Report" />;
  return <DataTable data={data} />;
}
```

### 7.5 Component Decomposition Threshold

A component is due for decomposition when ANY is true:
- Over ~200 lines
- Managing more than 3 pieces of local state
- Contains more than one distinct user interaction flow
- Contains both data-fetching logic AND complex rendering logic

Extract to `_components/` (private to parent) or `components/shared/` (reusable across features).

### 7.6 Client vs Server Component Decision Tree

```
Is this component interactive? (onClick, onChange, useState, useEffect)
├── YES → Must be 'use client'
└── NO → Can be a Server Component
    ├── Fetches data from backend? → Fetch in Server Component (no TanStack Query needed)
    └── Receives all data as props? → Server Component, no special handling
```

For clinical data requiring real-time invalidation → TanStack Query in Client Component is correct.
For static content (lab profile display) → Server Component fetch is preferred.

---

## 8. Backend Refactoring Playbook

### 8.1 Read DTOs Instead of Entity Mutation (CRITICAL)

```typescript
// BEFORE — Unsafe entity mutation (list-reports.handler.ts)
for (const report of reports) {
  (report as unknown as { invoice: unknown }).invoice = invoiceByReportId.get(report.id) ?? null;
  (report as unknown as { refByDoctor: unknown }).refByDoctor = doctorMap.get(report.refByDoctorId) ?? null;
}
return reports; // Returns mutated TypeORM entities — breaks encapsulation, destroys TS safety

// AFTER — Typed Read DTO projection
export class ReportListItemDto {
  id: string;
  reportNumber: string;
  status: ReportStatusEnum;
  sampleStatus: SampleStatusEnum;
  createdAt: string;
  finalizedAt: string | null;
  patient: { id: string; name: string; age: string | null; sex: SexEnum; phone: string | null };
  refByDoctor: { id: string; name: string; clinic: string | null } | null;
  invoice: { id: string; totalAmount: number; paymentStatus: InvoicePaymentStatusEnum } | null;

  static from(report: Report, doctor: Doctor | undefined, invoice: Invoice | undefined): ReportListItemDto {
    return {
      id: report.id,
      reportNumber: report.reportNumber,
      status: report.status,
      sampleStatus: report.sampleStatus,
      createdAt: report.createdAt.toISOString(),
      finalizedAt: report.finalizedAt?.toISOString() ?? null,
      patient: {
        id: report.patient.id,
        name: report.patient.name,
        age: report.patient.age ?? null,
        sex: report.patient.sex,
        phone: report.patient.phone ?? null,
      },
      refByDoctor: doctor ? { id: doctor.id, name: doctor.name, clinic: doctor.clinic ?? null } : null,
      invoice: invoice
        ? { id: invoice.id, totalAmount: invoice.totalAmount, paymentStatus: invoice.paymentStatus }
        : null,
    };
  }
}
```

### 8.2 Database-Level Associations (N+1 Elimination)

```typescript
// BEFORE — Full-table memory join
const invoices = await this.invoiceRepository.findByLabId(query.labId); // fetches ALL invoices
const doctors = await this.doctorRepository.findByLabId(query.labId);   // fetches ALL doctors

// AFTER — Targeted batch lookup by exact IDs
const reportIds = reports.map((r) => r.id);
const doctorIds = reports.flatMap((r) => r.refByDoctorId ? [r.refByDoctorId] : []);

const [invoices, doctors] = await Promise.all([
  this.invoiceRepository.findByReportIds(reportIds),  // WHERE report_id IN (...)
  this.doctorRepository.findByIds(doctorIds),          // WHERE id IN (...)
]);
```

Add `findByReportIds(ids: string[]): Promise<Invoice[]>` and `findByIds(ids: string[]): Promise<Doctor[]>` to the respective repository interfaces.

### 8.3 Domain Exception Completeness

```typescript
// CORRECT PATTERN — Specific domain exceptions, not generic Error
if (!report) throw new EntityNotFoundException('Report', reportId);
if (report.status === ReportStatusEnum.FINALIZED) throw new ReportAlreadyFinalizedException(reportId);
if (report.labId !== query.labId) throw new DomainForbiddenException('Access denied');
```

Never throw `new Error('...')` in handlers — it produces 500s instead of appropriate 404/409/403.

### 8.4 Input Validation Hardening

Move UUID validation from handler logic to the DTO presentation layer:

```typescript
// BEFORE — UUID_REGEX manual validation in business logic (list-reports.handler.ts)
const UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
if (query.patientId && !UUID_REGEX.test(query.patientId)) return [];

// AFTER — Validated at the boundary via DTO
export class ListReportsQueryDto {
  @IsOptional()
  @IsEnum(ReportStatusEnum, { message: 'status must be DRAFT or FINALIZED' })
  status?: ReportStatusEnum;

  @IsOptional()
  @IsUUID(4, { message: 'patientId must be a valid UUID v4' })
  patientId?: string;
}
```

---

## 9. Security Review Checklist

Evaluate for every file reviewed:

### 9.1 Authentication & Authorization
- [ ] Every protected endpoint has a guard (`@UseGuards(SupabaseAuthGuard)` or equivalent)
- [ ] Every handler extracts `labId` from `@CurrentUser()`, never from request body or query params
- [ ] Frontend route protection in `middleware.ts` uses `getUser()`, not just `getSession()` (**confirmed ✅**)
- [ ] The `x-lab-id` cookie is a convenience redirect signal only — not an auth token (**confirmed ✅**)
- [ ] No sensitive data (`labId`, `userId`, token) passed as URL query params in GET requests

### 9.2 Input Validation
- [ ] All user inputs validated via `class-validator` DTOs with `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })`
- [ ] UUID path/query parameters validated by `@IsUUID(4)` on the DTO (not manually in handlers)
- [ ] All form submissions validated against a Zod schema before calling mutations

### 9.3 Sensitive Data Exposure
- [ ] Stack traces/internal errors not exposed in production responses (`AllExceptionFilter` uses `NODE_ENV === 'production'` check — **confirmed ✅**)
- [ ] No API keys or secrets in client-side code or `NEXT_PUBLIC_` vars
- [ ] PII (patient name, phone, address) not logged at INFO/DEBUG level

### 9.4 Infrastructure
- [ ] NestJS CORS configured to allow only the production frontend origin, not `*`
- [ ] Rate limiting on auth endpoints (`/auth/*`) to prevent brute-force
- [ ] Supabase RLS policies verified if any direct DB access bypasses the NestJS API layer

---

## 10. Performance Review Checklist

Only flag demonstrated problems:

### 10.1 Backend (Confirmed Issues)
- **N+1 confirmed — CRITICAL**: `list-reports.handler.ts` fetches ALL invoices and ALL doctors for a lab on every list-reports request. A lab with 6 months of data will have this call degrade measurably.
- **Unbounded list endpoints**: verify every `findByLabId()` has `LIMIT` or cursor-based pagination. A growing lab's register will exceed manageable page sizes.

### 10.2 Frontend (Confirmed Issues)
- **`<img>` vs `<Image />`**: 4 confirmed lint warnings. `next/image` provides automatic format optimization (WebP), lazy loading, and eliminates Cumulative Layout Shift.
- **`refetchInterval: 30_000`** in `useReports`: polling is acceptable for a live lab register. TanStack Query pauses polling on hidden tabs — this is correct behavior.
- **Demo data in production bundles**: if `DEMO_ACCESSIONS` (400-line objects) are bundled in production pages, they add significant bundle weight for users who will never see them. Gate behind env flag.

---

## 11. Testing Gap Identification

### 11.1 Unit Testing Priorities (by Risk)

| Component | Test Priority | Reason | Recommended Test Scenarios |
|---|---|---|---|
| Out-of-range evaluation logic | **Critical** | Clinical accuracy — wrong flag = missed diagnosis | Numeric min/max boundary; gender-specific ranges; text equality; null/undefined value |
| `AllExceptionFilter` | High | Maps every error a user sees | DomainException → correct HTTP status; HttpException passthrough; generic Error → 500 in prod |
| `createEnumTransformer` | High | Integer/enum bridge used everywhere | Round-trip: DB integer → TS enum → DB integer; invalid integer → error |
| `formatCurrency()` | High | INR amounts shown to lab owners | Lakhs/crores compact; negative; zero; null/undefined input |
| `formatDate()` / `formatIST()` | High | Timestamps shown in IST | UTC input → IST output; date-only format; null input |
| `useURLState` | Medium | URL state used for all filters/pagination | Set param; delete on null; multiple params; scroll option |
| `listReports` tenant isolation | **Critical** | Multi-tenancy security | Lab A cannot see Lab B's reports |

### 11.2 Integration Testing Priorities

| Scenario | Why Critical |
|---|---|
| Create report → invoice auto-generated → query key invalidated on frontend | Multi-step mutation with cross-module side effects |
| Finalize already-finalized report → `ReportAlreadyFinalizedException` → 409 response | Business invariant enforcement |
| List reports with wrong `labId` → empty result | Multi-tenant data isolation |
| Auth middleware: unauthenticated → /login redirect | Core route guard |
| Onboarded user → no /onboarding redirect | Cookie + middleware interaction |

---

## 12. Naming & Organization Audit

### 12.1 File Naming Conventions (Established + Additions)

| Type | Convention | Example |
|---|---|---|
| Service file (pure API calls) | `<feature>.service.ts` | `report.service.ts` |
| Query hooks | `use-<feature>.queries.ts` | `use-report.queries.ts` |
| Mutation hooks | `use-<feature>.mutations.ts` | `use-report.mutations.ts` |
| Query key factory | `<feature>.keys.ts` | `report.keys.ts` |
| Zod schema | `<slice>.schema.ts` | `create-report.schema.ts` |
| Types file | `<feature>.types.ts` | `report.types.ts` |
| Backend Read DTO | `<slice>.read-dto.ts` | `report-list-item.read-dto.ts` |
| Backend query DTO | `<slice>.dto.ts` | `list-reports.dto.ts` |
| Private sub-component | `_components/<name>/index.tsx` | `_components/report-card/index.tsx` |

### 12.2 Domain Terminology (from PRD — enforce consistency)

| Preferred Term | Avoid |
|---|---|
| `Report` | `Test`, `Lab Result`, `Document` |
| `Panel` | `Test Type`, `Service`, `Item` |
| `Parameter` | `Field`, `Test Item`, `Measurement` |
| `Accession` | `Record`, `Entry`, `Submission` |
| `Patient` | `Client`, `Customer`, `User` |
| `Doctor` / `Referring Doctor` | `Referrer`, `Provider` |
| `Lab` | `Organization`, `Tenant`, `Account` |
| `Outsourced test` | `External test`, `Reference lab` |

---

## 13. Production Readiness Checklist

### 13.1 Observability
- [ ] Pino structured JSON logging enabled in production
- [ ] Error tracking (Sentry or equivalent) integrated for both frontend and backend
- [ ] `AllExceptionFilter` logs unhandled errors with full stack traces
- [ ] Health check endpoint (`/health` or `/api/v1/health`) available

### 13.2 Configuration
- [ ] All secrets in environment variables, not source code
- [ ] `NODE_ENV=production` set in production (affects error verbosity)
- [ ] CORS `origin` explicitly set — not `*`
- [ ] Supabase keys scoped correctly (anon key on frontend, service key on backend if needed)

### 13.3 Database
- [ ] All pending migrations run against production database
- [ ] Indexes confirmed: `reports.(lab_id, status)`, `reports.(lab_id, patient_id)`, `invoices.report_id`
- [ ] Supabase RLS policies reviewed for direct database access paths

### 13.4 Clinical Accuracy
- [ ] IST timezone formatting verified in production environment
- [ ] INR formatting verified with real amounts
- [ ] Out-of-range flag logic tested with boundary values
- [ ] Report PDF letterhead margins tested against Deswal's actual offset letterhead

---

## 14. Refactoring Priority Matrix

### Critical — Fix Before Any Production Use
Issues causing data loss, security breaches, incorrect clinical results, or complete feature failure:
- Silent error swallowing in API hooks (fake patient data shown on auth failure)
- Missing tenant isolation (`labId`) in any backend query
- `as unknown as` type casts to bypass TypeScript on entity mutations
- `catch` blocks swallowing errors without logging or re-throwing
- Query key mismatch `['finance']` that silently disables cache invalidation

### High — Fix in the Next Refactoring Session
Issues causing real problems as usage grows, or significant correctness risks:
- In-memory full-table joins in `list-reports.handler.ts` (N+1 scalability bomb)
- Missing Read DTO for list-reports response
- `window.location.href` navigation (8 lint warnings, broken browser history)
- `<img>` instead of `next/image` (CLS, 4 lint warnings)
- Generic `Error` throws in handlers (produces 500 instead of 404/409/403)
- UUID validation in handler business logic (move to DTO)

### Medium — Address During Routine Feature Work
Issues reducing maintainability without immediate risk:
- Inline demo data in `use-reports.ts` and `use-report.ts` (move to `demo-data/`)
- Missing service layer split (`.service.ts` + `.queries.ts` + `.mutations.ts`)
- Defensive fallback pyramids on guaranteed fields
- Missing Query Key Factory centralization
- Manual `URLSearchParams` string construction (replace with Axios `params`)
- Client-side `filterInvoices()` function (belongs in backend query)
- `any` cast in `AllExceptionFilter` response parsing
- Large files needing decomposition (`realtime-provider.tsx` at 375 lines)

### Low — Optional Cleanup
Style improvements, minor clarity enhancements:
- Renaming `index.tsx` to `<component>.tsx` inside folders (DX improvement)
- Additional JSDoc on public service functions
- Named constants for timeout values

---

## 15. Before / After Reference Examples

### Example A: Billing Hook — Full Service Layer Refactoring

**File:** `features/billing/api/use-billing.ts` (261 lines, all concerns mixed)

**Before:**
```typescript
export function useInvoices(filter?: InvoiceFilter) {
  return useQuery<Invoice[]>({
    queryKey: INVOICES_QUERY_KEY(filter),
    queryFn: async () => {
      try {
        const queryParams = new URLSearchParams();
        if (filter?.status && filter.status !== 'ALL') {
          queryParams.append('status', filter.status === 'PAID' ? 'PAID' : 'UNPAID');
        }
        const queryString = queryParams.toString() ? `?${queryParams.toString()}` : '';
        const data = await api.get<Invoice[]>(`/invoices${queryString}`);
        if (Array.isArray(data) && data.length > 0) return data;
        return filterInvoices(DEMO_INVOICES, filter); // fake data on empty
      } catch {
        return filterInvoices(DEMO_INVOICES, filter); // fake data on error
      }
    },
    staleTime: 30 * 1000,
  });
}
```

**After:**
```typescript
// billing.service.ts
export const billingService = {
  listInvoices: (params?: InvoiceQueryParams): Promise<Invoice[]> =>
    api.get('/invoices', { params }),
};

// billing.keys.ts
export const billingKeys = {
  invoices: (params?: InvoiceQueryParams) => ['billing', 'invoices', params] as const,
  expenses: (params?: ExpenseQueryParams) => ['billing', 'expenses', params] as const,
  financialSummary: () => ['billing', 'financial-summary'] as const,
};

// use-billing.queries.ts
export function useInvoices(params?: InvoiceQueryParams) {
  return useQuery({
    queryKey: billingKeys.invoices(params),
    queryFn: () => billingService.listInvoices(params),
    staleTime: 30_000,
  });
}
```

**Why:** Correctness — errors propagate to React Query's error state. Clarity — 3 lines replace 20. Testability — `billingService.listInvoices` is independently testable. Type safety — no casting, no fallbacks needed.

### Example B: Backend Read DTO

See Section 8.1 for the full `ReportListItemDto` before/after — eliminates the `as unknown as` cast pattern in `list-reports.handler.ts`.

### Example C: Query Key Invalidation Bug Fix

**File:** `features/reports/api/use-create-report.ts` (L50-53)

**Before:**
```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ['reports'] });
  queryClient.invalidateQueries({ queryKey: ['invoices'] });       // wrong key — billing uses 'billing', 'invoices'
  queryClient.invalidateQueries({ queryKey: ['finance'] });         // WRONG — key is 'billing', 'financial-summary'
  queryClient.invalidateQueries({ queryKey: DASHBOARD_STATS_QUERY_KEY });
  queryClient.invalidateQueries({ queryKey: ['patients'] });
},
```

**After:**
```typescript
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: reportKeys.lists() });
  queryClient.invalidateQueries({ queryKey: billingKeys.invoices() });
  queryClient.invalidateQueries({ queryKey: billingKeys.financialSummary() });
  queryClient.invalidateQueries({ queryKey: DASHBOARD_STATS_QUERY_KEY });
  queryClient.invalidateQueries({ queryKey: patientKeys.lists() });
},
```

**Why:** The current `['finance']` invalidation silently does nothing. Creating a Report is an Order-to-Cash flow — it generates an invoice, affects the financial summary, and affects the patient's report history. All four caches must actually be invalidated.

---

## 16. Duplicate Logic Audit

| Logic | Current Locations | Recommended Consolidation |
|---|---|---|
| `filterInvoices()` client-side filter | `use-billing.ts` | Remove entirely — filtering belongs in backend query via `params` |
| UUID validation regex | `list-reports.handler.ts` inline `UUID_REGEX` | Replace with `@IsUUID(4)` on DTO |
| `window.location.href` navigation | `auth-provider.tsx` (L66, L80), `api-client.ts` (L73), `onboarding/step-confirmation/index.tsx` (L146) | Route all navigation through `useRouter().push()` |
| INR currency formatting | `formatters.ts` (canonical) | Audit all components for `'₹' + amount` or `.toFixed(2)` and replace with `formatCurrency()` |
| IST date formatting | `formatters.ts` (canonical) | Audit all components for `new Date(...).toLocaleDateString()` and replace with `formatDate()` |
| Demo fallback pattern | `use-reports.ts`, `use-report.ts`, `use-billing.ts`, `use-panels.ts` | Gate all demo data behind `NEXT_PUBLIC_DEMO_MODE=true` env flag — remove from all hooks |

---

## 17. Automated Verification Gate

Every refactoring session must end with zero warnings and zero errors:

```bash
# Backend verification
cd labOS-service && npm run build && npm run lint

# Frontend verification
cd labOS-app && npm run lint && npm run build
```

**Current known lint warnings (target: 0):**

| File | Line | Issue | Fix |
|---|---|---|---|
| `auth-provider.tsx` | 66, 80 | `window.location.href` navigation | `useRouter().push()` |
| `api-client.ts` | 73 | `window.location.href` navigation | Centralize redirect in auth flow |
| `onboarding/step-confirmation/index.tsx` | 146 | `window.location.href` navigation | `useRouter().push()` |
| `a4-document-sheet.tsx` | 516, 537 | `<img>` instead of `<Image />` | `next/image` with `unoptimized` if needed for print |
| `report-letterhead-preview.tsx` | 252 | `<img>` instead of `<Image />` | `next/image` |
| `branding-letterhead-section.tsx` | 66 | `<img>` instead of `<Image />` | `next/image` |

**All 8 are documented High-priority defects. Target: 0 warnings, 0 errors.**
