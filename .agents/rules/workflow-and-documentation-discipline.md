# Workflow, Context & Documentation Discipline

These rules govern the exact operating procedure for the agent before, during, and after every task in this workspace:

## 1. Mandatory Pre-Task Check (BEFORE Writing Any Code or Starting Any Task)
Before writing any code or modifying any file, the agent MUST:
1. **Read `CONTEXT.md`**: Review current phase, master tracking log, and previous decisions. Do not reopen settled architectural decisions without explicit user instruction.
2. **Review `docs/conventions/coding-standards.md`**: Refresh all architecture, naming, state management, and component conventions to avoid introducing non-standard patterns.
3. **Inspect Feature Slices Directly**: For any frontend work, directly inspect the actual backend feature slice files (`labOS-service/src/modules/<module>/features/<feature>/`) — inspecting its controller, DTO, command/query, and handler. Never guess response types or interfaces.
4. **Deep Analysis Over Guesswork**: If any convention or domain requirement is unclear, read and analyze more relevant files across the codebase and reference industry-leading MNC production standards before writing code.

## 2. Phase-Wise Decomposition & Incremental Task Chunking
1. **Deconstruct Large Tasks**: Never attempt to implement large features or full phases in a single monolithic block.
2. **Break into Atomic Sub-Tasks**: If a task or phase is large, break it down into small, atomic, and testable sub-tasks (e.g., Step 1: Types & API client, Step 2: Query Hook, Step 3: UI Component, Step 4: Page Integration).
3. **Execute Incrementally**: Complete one atomic sub-task at a time. Verify it, record it, and align before moving to the next.

## 3. Execution Discipline (DURING Code Implementation)
1. **Explain Architectural Decisions**: Whenever proposing or structuring changes, provide senior-level rationale, highlight trade-offs, and explain why the approach was selected over alternatives.
2. **Clean Code & Single Responsibility**: Keep functions, components, and modules small and focused. No speculative, unused, or bloated code.
3. **Strict Type Safety & Zero Magic Values**: Use `StatusCodes` from `http-status-codes`. Map backend responses 1:1. Use Axios request/response interceptors with official Supabase auth sessions (zero `typeof window` or raw `localStorage` hacks).
4. **Mandatory UX States**: Every frontend view must handle loading skeletons, empty states, and validation edge cases.

## 4. Mandatory Post-Task Protocol (AFTER Completing Any Task)
After completing any task, vertical slice, or implementation phase, the agent MUST:
1. **Update `CONTEXT.md`**: Record the session log, completed files/features, and clear next steps.
2. **Update Related Documentation**: Keep `coding-standards.md`, `docs/openapi.yaml`, architecture docs, and walkthroughs completely synchronized with reality.
3. **Automated Verification**: Run `npm run build` and `npm run lint` across the affected package (`labOS-service` or `labOS-app`) to verify 0 compiler errors and 0 linter warnings before declaring completion.

