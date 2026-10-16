# Senior Engineering & Architecture Principles

These principles govern every technical decision made in this workspace, without exception. They override any temptation toward convenience, speed, or short-term workarounds.

---

## Core Mandate

**Do not optimize for "making it work." Optimize for building the right system.**

The goal is software that is clean, robust, testable, scalable, loosely coupled, highly cohesive, extensible, and maintainable for years. If achieving that requires substantial refactoring, restructuring, or challenging an existing design, do it — do not compromise the architecture for short-term convenience.

---

## 1. Architectural Decision-Making Process

Before implementing any non-trivial change, the agent MUST:

1. **Identify at least two architectural approaches** — never accept the first solution that appears correct.
2. **Evaluate trade-offs explicitly**: scalability, maintainability, reliability, performance, complexity, extensibility, operational impact, and testability.
3. **Reference patterns from mature engineering organizations** — large-scale, battle-tested systems from major technology companies where applicable.
4. **Choose the approach that provides the best overall engineering outcome**, not the one that is easiest to type.
5. **Explain the decision** — state which approach was chosen, why alternatives were rejected, and what trade-offs were accepted.

---

## 2. Architecture Quality Standards

### Separation of Concerns
- Every module, component, and function has one clearly defined responsibility.
- Data fetching, business logic, and UI rendering are strictly separated — never mixed in a single file.
- No "God Components" — a component that fetches data, manages 8+ state variables, renders complex JSX, and orchestrates multiple modals is always wrong.

### Coupling & Cohesion
- **High cohesion**: Things that change together are grouped together.
- **Low coupling**: Changes in one module do not cascade into unrelated modules.
- Private implementation details are never exported publicly. Use `_components/` (underscore prefix) for sub-components that are internal to a parent component.
- Shared abstractions are only created when the same logic genuinely recurs across multiple features — not speculatively.

### Clean Abstractions
- Abstractions must be simple, well-defined, and honest about what they do.
- A leaky abstraction (one that forces callers to understand its internals) is worse than no abstraction at all.
- Do not create abstractions that merely rename existing concepts without adding value.

### Naming Conventions (LabOS-specific, derived from residency-frontend standard)

| Concern | Convention |
|---|---|
| Feature slice folders | `verb-noun/` kebab-case (e.g., `get-report/`, `create-invoice/`) |
| Service files | `<use-case>.service.ts` — pure API call, no hooks |
| Query hooks | `use-<domain>.queries.ts` |
| Mutation hooks | `use-<domain>.mutations.ts` |
| Type files | `<use-case>.types.ts` co-located with the feature |
| Constants files | `<name>.constants.ts` |
| Private sub-components | `_components/` inside the parent component directory |
| Component directories | `<component-name>/index.tsx` always — never a bare `.tsx` in a components folder |

---

## 3. Pre-Implementation Mandatory Research Protocol

**Before writing a single line of implementation code, the agent MUST:**

1. **Read `CONTEXT.md`** — confirm current phase, settled decisions, and what has already been built.
2. **Read the relevant rule files** in `.agents/rules/` for the domain being worked on.
3. **Inspect all existing code paths** in the affected area — not just the one file being changed.
4. **Visually verify the current UI in Chrome** for all screens affected by the change. Do not reason about layout/UX issues purely from code — always look at the actual rendered output. Take screenshots before and after.
5. **Study the residency-frontend reference** for any pattern that doesn't already exist in this codebase — it is the primary reference for code quality, folder structure, naming conventions, private components, service layer patterns, socket hooks, URL-state patterns, pagination, and shared component design. Path: `/home/navdish/Desktop/residency/residency-frontend/src/`.
6. **Identify and document technical debt** in the current implementation before adding to it.

---

## 4. Technical Debt Identification

Always explicitly call out:
- **Architectural smells**: God components, leaky abstractions, tight coupling between unrelated features.
- **Naming violations**: Files or exports that don't match established conventions.
- **Missing patterns**: Areas where an established pattern (URL-state pagination, service layer, private components) should be applied but isn't.
- **Scalability risks**: Client-side filtering of unbounded datasets, unbound state in React context, synchronous operations on large data.
- **Reliability risks**: Missing error boundaries, missing loading states, `window.print()` without print CSS isolation, socket connections without reconnect/cleanup.

Do not add new code on top of architectural debt without first documenting it and proposing a remediation path.

---

## 5. Refactoring Is Not Optional

- Do not preserve a poor design merely because it already exists.
- If a component has grown beyond its single responsibility, split it — even if it is currently working.
- If the service layer is mixing API calls with UI state hooks, separate them.
- If the folder structure violates conventions, restructure it.
- **Refactoring is part of the work, not an optional extra.**

---

## 6. Trade-Off Evaluation Format

When multiple valid approaches exist, present a trade-off table:

| Approach | Pros | Cons | Verdict |
|---|---|---|---|
| A — ... | ... | ... | **Choose / Reject** |
| B — ... | ... | ... | **Choose / Reject** |

Always explain why the winning approach was chosen and what was accepted as a trade-off.

---

## 7. Non-Negotiables (Zero Exceptions)

- No magic numbers — use named constants for all HTTP codes, debounce delays, z-index values, pagination limits.
- No `any` types — use `unknown` and narrow explicitly.
- No inline demo data inside hook or component files — it belongs in `src/lib/demo-data/`.
- No component exceeding ~200 lines without documented justification and a decomposition plan.
- No `window.print()` without a corresponding `@media print` CSS isolation strategy.
- No modal that calls `window.print()` without isolated print context.
- No unhandled promise rejections in mutations — always handle in `onError`.
- No React state for data that belongs in the URL (`page`, `search`, `status` filter) — use URL query params.
- No `overflow-x-auto` on a table that should expand to fill available width.

---

## 8. Chrome Visual Verification (Mandatory Before & After)

**Before implementation**: Visually inspect every screen and flow related to the change via browser tool.
**After implementation**: Screenshot all affected screens before declaring completion.

Mandatory viewport checks for any layout change:
- Desktop (1440px+) — sidebar expanded
- Desktop (1440px+) — sidebar collapsed
- Tablet (768px)
- Mobile (390px) — hamburger menu open and closed
- Print dialog (Ctrl+P) for any page with a print action

---

## 9. Documentation Discipline

Every meaningful decision must be captured in:
1. **`CONTEXT.md`** — session log with what was built and why
2. **`docs/conventions/coding-standards.md`** — if a new pattern or convention was introduced
3. **Inline code comments** — only where the "why" is non-obvious
4. **ADRs in `docs/architecture/decisions/`** — for decisions affecting overall system design
