# Frontend Engineering Standards & Operating Rules

> **Authoritative Code Review Reference:** Before auditing, refactoring, or reviewing any frontend file, read and execute the [Master Code Audit & Refactoring Protocol](../../docs/conventions/master-code-audit-and-refactoring-protocol.md). That document contains the 10-phase universal review prompt, the AI anti-pattern catalog (grounded in actual files), the frontend refactoring playbook (service layer split, Query Key Factory, error/loading/empty triad, component decomposition), the security and performance checklists, and the 4-tier priority matrix.

These rules are permanent across all sessions and coding tasks for `labOS-app`:

## 1. Senior Engineering Mindset & Code Quality
- Keep implementations clean, minimal, robust, and maintainable.
- Do NOT write extra, speculative, or unnecessary code.
- Keep components, hooks, and services small and focused on a single responsibility.
- Optimize for long-term maintainability, readability, and user experience rather than short-term speed or premature abstractions.
- **Explain Architectural Decisions:** When proposing changes, always provide clear rationale, highlight trade-offs, and explain why a particular design was chosen over alternatives.
- **Deep Research Before Coding:** When unsure about conventions, patterns, or requirements, **do not guess**. Thoroughly read and analyze relevant files in the codebase, and align with industry-leading production standards and MNC best practices before writing code.

## 2. Strict 1:1 Backend Parity (Inspect Feature Slices Directly)
- **Always inspect the backend feature slice directly (`labOS-service/src/modules/<module>/features/<feature>/` — checking its dedicated controller, DTO, and handler) before writing or updating any frontend code.**
- Never guess or assume frontend interfaces or request shapes. Update frontend types/schemas to match backend responses 1:1.
- Eliminate unnecessary fallback chains. Understand the backend response and map cleanly.

## 3. API Communication & Authentication
- Use an **Axios instance with interceptors** in `src/lib/api-client.ts`.
- **No manual `fetch` boilerplate**, no repetitive JSON parsing.
- **No `typeof window !== 'undefined'` or raw `localStorage` hacks.** Attach the Supabase auth token via official Supabase SDK session methods (`supabase.auth.getSession()` or cookies) in the Axios request interceptor.
- Use **`http-status-codes` (`StatusCodes`)** for all HTTP response checks (e.g. `StatusCodes.UNAUTHORIZED`, `StatusCodes.NOT_FOUND`). Zero magic status numbers (`200`, `204`, `401`, `500`).

## 4. Component Organization (Folder + index.tsx)
- Every reusable component lives in its own dedicated directory: `<component-name>/index.tsx`.
- Public/shared design system primitives live in `src/components/ui/`.
- Reusable form controls wired to **React Hook Form + Zod** live in `src/components/form/` (`text-field`, `number-field`, `select-field`, `textarea-field`).
- Private domain components live inside their respective feature slice (`src/features/<feature>/components/<component>/index.tsx`) or route-private `_components/`.

## 5. UI/UX & State Handling
- Every screen and query must properly handle:
  1. **Loading Skeletons** (never leave blank screens during data fetching).
  2. **Empty States** with clear call-to-actions.
  3. **Error Scenarios & Boundaries** with actionable messages.
  4. **Validation Edge Cases** with inline Zod errors.
- Ensure layouts are accessible, responsive, consistent, and visually hierarchical.
- Support fast keyboard navigation (`Tab`/`Enter`) for rapid medical data entry.
- **CRITICAL — No Demo Fallbacks in Query Hooks:** `catch { return DEMO_DATA }` is **strictly prohibited** in production query functions. It is a clinical safety risk — errors are silently swallowed and fictitious patient data is displayed to lab staff. Errors must propagate to TanStack Query's `error` state. The component must render an `<ErrorState>` with a retry action. Demo data is only permitted behind `NEXT_PUBLIC_DEMO_MODE=true` in a dedicated service adapter.
- **Mandatory Service Layer Split:** Every `features/<feature>/api/` must be split into three files: `<feature>.service.ts` (pure Axios calls — no hooks, no React), `use-<feature>.queries.ts` (TanStack `useQuery` hooks), and `use-<feature>.mutations.ts` (TanStack `useMutation` hooks). Mixing Axios calls, hooks, and mock data in a single file is prohibited.
- **Mandatory Query Key Factory:** Never use raw string array literals as query keys (e.g. `['finance']`). Create a `<feature>.keys.ts` file with a typed factory (`reportKeys.list(params)`). All `queryClient.invalidateQueries()` calls must reference the factory. Raw string key mismatches cause silent cache invalidation failures.


## 6. Clinical & Operational Guardrails
- **Timezone Standard:** Display timestamps strictly in Indian Standard Time (IST / `Asia/Kolkata`) (e.g. `10 Sep 2026, 01:15 PM`). Database timestamps remain UTC.
- **Currency Standard:** Format INR amounts using native `Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' })`. Never use manual string concatenation.
- **Tenant Security:** Never pass `labId` as a request parameter; backend strictly extracts it from the verified JWT.
- **Mutation Feedback:** All mutations must provide clean toast feedback (success message or actionable backend error). Never use optimistic updates on finalized medical reports.

## 7. Responsiveness, Adaptive Mobile/Desktop Ergonomics & Modern Styling
- **Full Viewport Responsiveness:** Every layout, table, modal, and drawer must adapt seamlessly across Mobile (<640px / 360–430px), Tablet (640px–1024px), Desktop (1024px–1440px), and Ultra-wide (>1440px).
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
  - Standard styling uses Tailwind CSS v4 design tokens and utility composition via `cn()`.
  - If a dedicated custom stylesheet is ever required beyond Tailwind utilities, create a `.scss` file using modern SCSS syntax, CSS variables, and modular/BEM naming. Never create ad-hoc unstructured global `.css` files.

