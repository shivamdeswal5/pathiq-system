# LabOS — Backend Engineering, Architecture & Review Standard

> **Core Philosophy:** Write clean, production-ready, idiomatic TypeScript and NestJS. Prioritize long-term maintainability, readability, type-safety, and simplicity over cleverness or unnecessary abstractions.

---

## 1. Operating Instructions for Engineers & AI Assistants

- **Act as a senior software engineer.**
- **Always analyze existing code before making changes.**
- **Follow the existing architecture, folder structure, file naming conventions, coding style, and project patterns exactly.**
- **Do not introduce new folder structures, naming conventions, component patterns, state management approaches, or architectural patterns if similar implementations already exist in the codebase.**
- **When unsure about project conventions, read and analyze more relevant files before writing code.**
- **For state management, Redux Toolkit, async thunks, slices, selectors, services, and related patterns, refer to existing implementations in the project and follow the same approach consistently.**
- **Create proper TypeScript types for all new code. Use enums where appropriate instead of hardcoded string values. If types, enums, constants, or utilities are shared across multiple features or components, place them in the appropriate common/shared location following existing project conventions.**
- **Prefer simple, maintainable solutions over complex abstractions.**
- **Do not introduce new libraries unless there is a clear and justified benefit.**
- **Keep functions, components, and modules small and focused on a single responsibility.**
- **Reuse existing utilities, hooks, services, components, and helpers whenever possible.**
- **Avoid code duplication.**
- **Write production-ready TypeScript.**
- **Handle edge cases, validation, loading states, empty states, and error scenarios appropriately.**
- **Consider performance, scalability, maintainability, accessibility, and type safety when implementing solutions.**
- **Always build responsive interfaces that work seamlessly across mobile, tablet, laptop, and large desktop screens.**
- **Prioritize usability, accessibility, consistency, visual hierarchy, and smooth interactions.**
- **Avoid cluttered layouts and ensure components adapt gracefully to different screen sizes.**
- **Explain architectural decisions when proposing changes.**
- **When multiple approaches exist, recommend the most maintainable option and explain the trade-offs.**
- **Optimize for long-term maintainability, consistency, readability, and user experience rather than short-term speed or unnecessary abstractions.**

---

## 2. Architecture: DDD & Onion Architecture Boundaries

LabOS follows a **Modular Monolith** organized into **DDD Bounded Contexts** (Modules), with **CQRS Vertical Slices** and **Onion Architecture** layers:

```
┌──────────────────────────────────────────────────────────┐
│                   Presentation Layer                     │
│  (Per-feature Controllers, Guards, Interceptors, Filters)│
└────────────────────────────┬─────────────────────────────┘
                             │ calls
┌────────────────────────────▼─────────────────────────────┐
│          Application Layer: features/ Directory          │
│  (Vertical Feature Slices: Commands, Queries, Handlers)  │
└────────────────────────────┬─────────────────────────────┘
                             │ depends on
┌────────────────────────────▼─────────────────────────────┐
│                     Domain Layer                         │
│  (Entities, Value Objects, Domain Events, Enums,         │
│   Repository Interfaces - Core Business Logic)           │
└────────────────────────────▲─────────────────────────────┘
                             │ implemented by
┌────────────────────────────┴─────────────────────────────┐
│                 Infrastructure Layer                     │
│  (Categorized: database/seeders, repositories, clients)  │
└──────────────────────────────────────────────────────────┘
```

### Domain-Driven Design (DDD) Taxonomy for LabOS

| DDD Concept | Definition & Role in LabOS | Concrete LabOS Examples |
|---|---|---|
| **Core Domain** | High-value, competitive differentiator of the business. Heart of clinical operations. | `panels` (Test catalog & parameter ranges), `reports` (Diagnostic results, out-of-range evaluation, verification & sign-off) |
| **Supporting Subdomain** | Necessary business capabilities tailored to the lab domain. | `labs` (Multi-tenant registration, branding & team membership), `collections` (Phlebotomy home collection), `billing` (Invoices & payments), `referrals` (Doctor commissions & ledger) |
| **Generic Subdomain** | Standard capabilities that are not unique to diagnostic pathology. | `notifications` (WhatsApp / SMS / Email), `websockets` (Real-time bidirectional event gateway & room routing), `shared/infrastructure` (Supabase Auth, Logging, Health) |
| **Aggregate Root** | Gateway entity for a cluster of associated objects treated as a single transactional unit. External objects only reference the Root. | `TestPanel` (owns sections & parameters), `TestPackage` (owns package items), `Lab` (owns lab tenant & branding), `Profile` (owns user lab membership), `Report` (owns test values & amendments), `Patient` (owns patient demographic history), `CollectionRequest` (owns samples), `Invoice` (owns invoice items) |
| **Child Entity** | Object with local identity within an aggregate boundary; cannot exist or be mutated independently of its root. | `PanelSection`, `PanelParameter`, `PackagePanel`, `ReportPanel`, `ReportValue`, `ReportAmendment`, `CollectionSample`, `InvoiceItem` |
| **Value Object** | Immutable, identity-free concept defined entirely by its attributes with structural equality. **Rich Value Objects encapsulate their own validation and domain logic.** | `NormalRange` (numeric min/max, gender-specific ranges, text ranges) with `isOutOfRange(value, sex)` method; no external procedural evaluator needed. |
| **Domain Event** | Record of a significant business occurrence that happened in the domain. | `ReportFinalizedEvent`, `CollectionCreatedEvent`, `CollectionAssignedEvent`, `CollectionStatusUpdatedEvent` (triggers billing, notification dispatch, doctor commissions, and real-time socket updates) |


### Canonical Module Layout (from `residency-backend`)

Every bounded context under `src/modules/<module-name>/` MUST follow this exact structure:

```
modules/<module-name>/
├── domain/                                     # Layer 1: Core Domain (Zero framework dependencies)
│   ├── <aggregate-or-entity>/
│   │   ├── <entity>.entity.ts                  # TypeORM entity or Domain Entity
│   │   ├── enums/                              # String enums & numeric DB mappers
│   │   ├── events/                             # Domain Events emitted by this aggregate
│   │   └── interfaces/                         # Repository interfaces (Dependency Inversion)
│
├── features/                                   # Layer 2 & 3: Application & Presentation Slices
│   ├── <feature-group>/                        # e.g., panel, package, reservation
│   │   ├── <use-case-name>/                    # e.g., create-panel, list-panels
│   │   │   ├── <use-case>.command.ts           # (or .query.ts if read-only)
│   │   │   ├── <use-case>.handler.ts           # CQRS handler implementing ICommandHandler / IQueryHandler
│   │   │   ├── <use-case>.dto.ts               # Request validation class using class-validator
│   │   │   ├── <use-case>.controller.ts        # Dedicated endpoint controller for this slice
│   │   │   └── <use-case>.module.ts            # NestJS module wiring controller, handler, and entities
│   │   └── <feature-group>.module.ts           # Aggregates all slice modules in this feature group
│
├── infrastructure/                             # Layer 4: External Concerns (Categorized into subfolders)
│   ├── database/
│   │   ├── seeders/                            # Database seeders (e.g. panel-templates.seed.ts)
│   │   └── repositories/                       # Concrete repository implementations of domain interfaces
│   └── http/
│       └── clients/                            # External HTTP / 3rd party integration clients
│
└── <module-name>.module.ts                     # Root bounded context module (imports all feature group modules)
```

### How to Create a New Feature or Module (Step-by-Step)

When adding any new feature, command, query, or module, follow this exact sequence:

1. **Step 1: Define Domain Model (`domain/`)**
   - Create entity in `domain/<entity>/<entity>.entity.ts`.
   - Define string enums in `domain/<entity>/enums/<name>.enum.ts` and integer DB mappers.
   - Define domain events in `domain/<entity>/events/<name>.event.ts` if other modules need to react.
   - If a custom repository is needed, define `interfaces/<name>.repository.interface.ts`.

2. **Step 2: Create Vertical Feature Slice (`features/<group>/<use-case>/`)**
   - **DTO:** Create `<use-case>.dto.ts` with `class-validator` decorators.
   - **Command / Query:** Create `<use-case>.command.ts` (state mutation) or `<use-case>.query.ts` (read-only).
   - **Handler:** Create `<use-case>.handler.ts` implementing `handle(commandOrQuery)` and injecting domain repository interfaces.
   - **Controller:** Create `<use-case>.controller.ts` directly injecting the handler for 100% compile-time type safety.
   - **Slice Module:** Create `<use-case>.module.ts` wiring the controller, handler, and repository token.

3. **Step 3: Wire into Feature Group & Root Module**
   - Register the slice module into `<feature-group>.module.ts`.
   - Register `<feature-group>.module.ts` into `<module-name>.module.ts`.
   - Register `<module-name>.module.ts` into `src/app.module.ts`.

4. **Step 4: Place Infrastructure in Categorized Subfolders**
   - Seeders MUST go into `infrastructure/database/seeders/`.
   - Repositories MUST go into `infrastructure/database/repositories/`.
   - Migrations MUST go into `infrastructure/database/migrations/`.
   - **RULE:** NEVER place loose files directly inside `infrastructure/`. Always use dedicated subdirectories.

5. **Presentation Layer:**
   - Dedicated slice controllers receive HTTP requests, parse DTOs, invoke the injected handler, and return results.
   - Zero business logic inside controllers!

---

## 3. SOLID Principles — Applied Concretely in NestJS

| Principle | Concrete Rule in this Codebase | Anti-Pattern to Reject |
|---|---|---|
| **Single Responsibility (SRP)** | Every Command/Query handler does exactly one business action. Controllers only route HTTP to the injected handler. Entities only manage internal invariants. One entity per file. | A 500-line service handling lab CRUD, user invitations, and email dispatch all in one class. Multiple `@Entity()` classes in one file. |
| **Open/Closed (OCP)** | Extend functionality by creating new command handlers, domain event listeners, or implementing repository interfaces without mutating existing tested flows. | Modifying a core `CreateReportHandler` to add email notification code instead of publishing a `ReportFinalizedEvent`. |
| **Liskov Substitution (LSP)** | Repository implementations and services must strictly conform to their domain contracts so implementations can be swapped without side effects. | An in-memory mock repository returning a different shape than the Postgres TypeORM repository. |
| **Interface Segregation (ISP)** | Keep repository interfaces lean and specific to the bounded context's requirements. | A bloated `IGenericRepository<T>` with 30 unused CRUD methods forced onto every entity. |
| **Dependency Inversion (DIP)** | Handlers depend strictly on domain repository interfaces (`IPanelRepository`), never directly on `DataSource`, raw SQL, or TypeORM drivers. | Calling `this.dataSource.transaction()` or `@InjectRepository(TestPanel)` directly in a CQRS use-case handler. |


---

## 4. NestJS Standards, DTOs & Validation

### DTOs & Validation: Class-Validator over Zod
- **Standard DTO Classes:** All request payloads must be standard TypeScript classes decorated with `class-validator` and transformed with `class-transformer`.
- **Built-in `ValidationPipe`:** Validated globally via NestJS `ValidationPipe({ whitelist: true, transform: true })`.
- **No Zod on Backend:** Do not use Zod schemas, `z.infer`, or custom `ZodValidationPipe` in the NestJS service. It introduces dual-definition overhead and conflicts with standard NestJS decorators and OpenAPI tools.

#### Example: Standard DTO
```typescript
import { IsNotEmpty, IsOptional, IsString, IsUUID } from 'class-validator';

export class CreateLabDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsString()
  @IsNotEmpty()
  address: string;

  @IsOptional()
  @IsString()
  tagline?: string;
}
```

---

## 5. Metadata Keys, Constants & Magic Strings

- **No Magic Strings:** Never scatter raw string literals across decorators, guards, and handlers (e.g. `'isPublic'`, `'roles'`, `'lab_id'`).
- **Centralized Constants:** Place reflection keys and system constants in dedicated constants files under `src/shared/constants/` or module-specific constants:
  - `src/modules/shared/constants/metadata.constants.ts` (e.g., `METADATA_KEYS = { IS_PUBLIC: 'isPublic', ROLES: 'roles' }`)
- **Enums for Domain Values:** Always use TypeScript enums for status, roles, categories, and channels (e.g., `Role`, `ReportStatus`, `SampleStatus`).

---

## 6. Code Quality & Anti-Duplication ("Do Not Write Extra Code")

1. **Reuse Existing Utilities & Components:**
   - Always search `src/modules/shared/` and related modules before writing a new helper, guard, decorator, or transformer.
   - If a utility or base entity (`BaseDomainEntity`, `createEnumTransformer`) exists, extend or consume it directly.
2. **Zero Unnecessary Code & No Redundant Comments:**
   - Code must be clear, expressive, and self-documenting through strong types and precise domain naming.
   - Do not write redundant JSDoc comments or narrative inline explanations for obvious code.
   - Keep files small, focused, and free of commented-out code, redundant wrappers, or unused imports.
3. **Clean Domain Error Handling:**
   - Handlers and domain models must throw **Domain Exceptions** (`LabAlreadyExistsException`, `PanelNotFoundException`, `ReportFinalizedException`).
   - The presentation layer (`AllExceptionsFilter`) maps domain exceptions to HTTP status codes (`400`, `404`, `409`). Never throw HTTP exceptions inside domain or application handlers.

---

## 7. AI Code Review & Refactoring Checklist

Before accepting or merging any backend code, verify each item on this checklist:

- [ ] **Architecture Check:** Does the code respect Domain -> Application -> Infrastructure -> Presentation separation?
- [ ] **DTO vs Value Object Separation:** Are DTOs strictly in features/presentation, and Value Objects strictly in domain with zero validation decorators?
- [ ] **First-Class Enums:** Are all categorical domain values declared as first-class enums without raw string arrays in validation or string equality in business logic?
- [ ] **One Concept Per File:** Is each entity, enum (+ mapper), and major DTO in its own isolated file?
- [ ] **No Proxy or Barrel Files:** Are all entities and models directly imported without proxy files or domain-level index barrels?
- [ ] **No Cross-Slice Feature Coupling:** Does each slice avoid reaching into peer slice folders for DTOs or internals?
- [ ] **Repository Pattern Check:** Do handlers depend on domain repository interfaces (`@Inject(TOKEN)`), rather than raw `DataSource` or TypeORM repositories?
- [ ] **Direct Handler Injection Check:** Does the slice controller directly inject its dedicated use-case handler with strict compile-time return types?
- [ ] **One Entity Per File:** Is every entity defined in its own file?
- [ ] **CQRS Check:** Are commands mutating state and queries only reading data? Is there one handler per file?
- [ ] **DTO & Validation Check:** Are DTOs clean classes using `class-validator` and `class-transformer`? Are all inputs validated and whitelisted?
- [ ] **Domain Exceptions Check:** Do application handlers throw Domain Exceptions instead of NestJS HTTP exceptions?
- [ ] **No Magic Strings:** Are metadata keys, constants, and enums imported from centralized definitions instead of raw strings?
- [ ] **No Redundant Comments:** Is the code self-documenting without redundant JSDoc or narrative filler?
- [ ] **Path Aliases & No `.js`:** Are imports using clean `src/*` path aliases with zero `.js` extensions?
- [ ] **Tenant Isolation:** Is `lab_id` properly scoped in repository queries and enforced via authentication context?
- [ ] **WebSockets & Real-Time Check:** Do WebSocket broadcasts adhere to CloudEvents `EventMessage<T>`? Do they inherit from `WebSocketEvent<T>`? Are rooms scoped via `RealtimeRoomBuilder` and channels strictly typed via `RealtimeChannelEnum`? Is event bridging done in-process via `@OnEvent` without HTTP self-calls?
- [ ] **SOLID Compliance:** Does the class/function have a single responsibility? Are dependencies injected via interfaces?
- [ ] **Zero Bloat Check:** Is there any unused dependency, redundant helper, or dead code?
- [ ] **Type Safety:** Is strict TypeScript adhered to without `any` casts?
- [ ] **Build & Lint:** Does `npm run build` and `npm run lint` pass with 0 errors?

---

## 8. Database Migrations & Seeder Standards (Aligned with `residency-backend`)

LabOS follows the modular database migration and seeder conventions established in `residency-backend`:

### Universal Lifecycle Commands
- `npm run migration:up`: Runs all pending database migrations across all DDD modules.
- `npm run migration:undo`: Reverts the latest executed migration batch.
- `npm run seed:run`: Runs master database seeders (e.g. Master Clinical Panel Catalog).

### Bounded-Context Module Migration Commands
Each DDD module maintains its own isolated migrations in `src/modules/<module>/infrastructure/database/migrations/`:
- **Labs:** `npm run migration:up:labs` / `npm run migration:undo:labs`
- **Reports:** `npm run migration:up:reports` / `npm run migration:undo:reports`
- **Panels:** `npm run migration:up:panels` / `npm run migration:undo:panels` / `npm run seed:run:panels`
- **Referrals:** `npm run migration:up:referrals` / `npm run migration:undo:referrals`
- **Billing:** `npm run migration:up:billing` / `npm run migration:undo:billing`
- **Collections:** `npm run migration:up:collections` / `npm run migration:undo:collections`
- **Notifications:** `npm run migration:up:notifications` / `npm run migration:undo:notifications`


