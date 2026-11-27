# Complete Product Feature Inventory & System Reconciliation

**Document Version:** 1.0.0  
**Date:** October 9, 2026  
**Audited Systems:** `docs/product/prd.md`, `labOS-app` (Frontend), `labOS-service` (Backend), `labOS-admin` (Super Admin), `labOS-web` (Marketing Website)  
**Governing Standard:** [.agents/rules/senior-architecture-principles.md](file:///home/navdish/Desktop/labOS/.agents/rules/senior-architecture-principles.md) & [prd.md](file:///home/navdish/Desktop/labOS/docs/product/prd.md)

---

## Executive Summary

This document reconciles all features across four authoritative sources:
1. **The Product PRD** (`docs/product/prd.md`)
2. **The Laboratory Frontend Workstation** (`labOS-app`)
3. **The NestJS Modular Monolith API** (`labOS-service`)
4. **The Public Marketing Website** (`labOS-web`)

Every feature is classified based on strict code inspection (not assumptions). This provides the definitive foundation for **purging mock data**, establishing **clean production empty states**, and planning the **Mission Control Dashboard** around real, verified laboratory workflows.

---

## 1. Master 4-Way Feature Reconciliation Matrix

| Feature Domain | Feature Description | PRD Ref | Frontend Route / Components | Backend Endpoint & CQRS Slice | Marketing Claim | Status | Evidence & Reality |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Auth & Tenancy** | Multi-tenant auth, JWT session extraction, role scoping | P0 (Sec 6) | `/login`, `/signup`, `src/features/auth` | `@UseGuards(JwtAuthGuard)`, `TenantInterceptor` | "14-Day Free Trial", "No Credit Card" | **Fully Implemented** | Supabase Auth sessions verified; JWT propagates `labId` automatically across all requests. |
| **Lab Onboarding** | 4-step wizard: Profile, letterhead branding, signatures, catalog seeding | Goal 1 | `/onboarding` (`features/onboarding/`) | `PATCH /labs/me`, `POST /panels/seed` | "Onboard in under 1 day" | **Fully Implemented** | Wizard collects lab phone, NABL ID, digital signature canvas, and seeds initial test panels. |
| **Patient Registration** | Walk-in patient demographic intake (Name, Age, Sex, Phone, MRN) | P0 (Sec 6) | `/reports/new` (`features/reports/`) | `POST /patients`, `POST /reports` | "Fast Patient Intake" | **Fully Implemented** | Atomically creates or matches patient by mobile number (`PT-XXXX`), creates accession. |
| **Barcode Generation & Tracking** | 50×25mm browser thermal sticker modal (`JsBarcode` Code-128 of `reportNumber`) | P2 in PRD (sticker print MVP, tracking deferred) | `features/reports/components/thermal-sticker-modal/`, `/accessions` | `reports.report_number` + `collection_samples.barcode` (home collection only); no multi-bench tracking endpoint | "50×25mm Thermal Tube Labels (Print at Intake)" | **Label Print Ready / Tracking Deferred** | `@page { size: 50mm 25mm; }` print CSS works via browser print dialog. `/accessions` filters only by `reportNumber`/name/phone/MRN; `sampleStatus` transitions `COLLECTED` → `COMPLETED` directly (`PROCESSING` unused); multi-bench custody tracking stays Phase 3 per `prd.md`. |
| **Accession Register** | Daily accession worklist, filter by Draft vs Finalized, search by name/ID | P0 (Sec 6) | `/accessions` (`features/reports/`) | `GET /reports?status=&patientId=` | "Live Daily Register" | **Fully Implemented** | Both `useReports` and `useReport` query backend directly with explicit error/empty states and zero demo data fallbacks. |
| **Result Entry Console** | Numerical result entry with biological reference ranges & panic highlights | P0 (Sec 6) | `/reports/[id]/entry` (`features/reports/`) | `PUT /reports/:id/values` | "Out-of-range panic alerts" | **Fully Implemented** | Highlights abnormal values in bold red with critical panic pulses; computes high/low flags live. |
| **Pathologist Verification** | Digital signature stamp, NABL disclaimer, report finalization | P1 (Sec 6) | `/reports/[id]/preview` (`features/reports/`) | `POST /reports/:id/finalize` | "Digital Sign-off" | **Fully Implemented** | Locks report against further edits; appends verification record & pathologist credentials. |
| **PDF & Print Engine** | Digital QR verification PDF (`pdfkit`) + calibrated pre-printed stationery layout (`ReportPdfGeneratorService` & `A4DocumentSheet`) | P0 (Sec 6) | `/reports/[id]/preview`, `/reports/[id]/pdf` | `ReportPdfGeneratorService` + `resolvePdfPrintLayout` | "In-Lab Stationery Margins & QR Reports" | **Fully Implemented** | Both browser print (`A4DocumentSheet`) and downloaded vector PDF (`ReportPdfGeneratorService`) enforce saved `lab.printSettings` (`PLAIN` / `PREPRINTED_HEADER` / `PREPRINTED_HEADER_AND_FOOTER`, `headerMarginMm`, `footerMarginMm`). |
| **Public QR Portal** | Patient report verification portal via share token | P0 (Sec 6) | `/v/[token]` (`src/app/v/[token]`) | `GET /reports/share/:token/pdf` | "Tamper-Proof QR" | **Fully Implemented** | Public endpoint returns verified report without requiring laboratory login; invalid/expired links show honest error cards cleanly. |
| **Outsourced Testing** | Dispatching send-out tests to reference labs (Lal PathLabs/SRL) | P0 (Goal 3) | `/referrals` (Tab 3: Outsourced) | `GET/POST /referrals/outsourced` | "Outsourced Lab Sync" | **Fully Implemented** | Tracks external waybill, lab partner, wholesale cost, and merges into patient's final PDF report with zero local mock mutations. |
| **Referring Doctor Ledger** | Clinician association, commission rule (Flat / %), running balance | P0 (Sec 6) | `/referrals` (`features/referrals/`) | `GET/POST /referrals/doctors`, `/commissions` | "Referral Doctor Ledger" | **Fully Implemented** | Queries live doctor/commission APIs directly (`DEMO_*` purged); settlement remains manual. |
| **Diagnostic Billing** | SAC 999316 GST-exempt cash receipt vouchers & invoice tracking | P0 (Sec 6) | `/billing` (`features/billing/`) | `GET/POST /billing/invoices` | "Cash Receipts & Till Drawers" | **Fully Implemented** | Invoices, expenses, and payments mutate backend directly with zero silent demo array mutations; print modal supports SAC 999316 receipts. |
| **Operational Expenses** | Expense outflow logger (Reagents, consumables, rent, AMC) | P1 (Sec 6) | `/billing` (`features/billing/`) | `GET/POST /billing/expenses` | "Expense Tracking" | **Fully Implemented** | Categorizes outflows; computes live aggregate cashflow and category breakdowns. |
| **Test Catalog Manager** | Custom test panels, parameters, reference ranges, fees (₹) | P0 (Sec 6) | `/panels` (`features/panels/`) | `GET/PUT/DELETE /panels` | "Custom Test Pricing" | **Fully Implemented** | Live API CRUD with explicit empty catalog prompt when zero panels exist yet. |
| **Home Sample Collection** | Phlebotomy booking, address, preferred time, runner dispatch | P0 (Sec 6) | `/collections` (`features/collections/`) | `GET/POST /collections`, `/assign` | "Home Phlebotomy Queue" | **Fully Implemented** | All collection queries and status/sample mutations run live against backend with explicit retry errors. |
| **WhatsApp Dispatch** | 1-click WhatsApp delivery with pre-filled message & report link | P0 (Sec 6) | `/reports/[id]/preview`, `/patients` | `https://wa.me/91...` click-to-chat | "1-Click WhatsApp Delivery" | **Client-Side Working** | Works via direct WhatsApp Web links. Headless background Meta Cloud API dispatch deferred. |
| **RBAC View Gating** | View gating for Owner, Pathologist, Technician, Phlebotomist | P1 (Sec 6) | `src/features/auth/hooks/use-rbac.ts` | Backend JWT role verification | "Role Simulator" | **Fully Implemented** | Gated navigation and NABL sign-off protection with live persona switcher. |
| **SaaS Subscription Guard** | In-app billing upgrade modal, 72-hr grace period, accession locking | P0 (Sec 7a) | `features/subscription/`, `/settings` | `modules/platform-billing/` | "Transparent Pricing" | **Fully Implemented** | 14-day trial countdown; accession intake locked when `LOCKED`; WebSocket optimistic unlock. |
| **Super Admin Platform** | Platform governance, tenant directory, offline 12-digit UTR approval | Admin PRD | `labOS-admin` (Port 3001) | `billing/admin/tenants`, `/manual-activation` | N/A (Internal) | **Fully Implemented** | 1-click offline bank UTR verification and new lab onboarding. |

| **Settings: Branding & Color** | Report header accents, lab logo upload, brand palette selection | P0 (Sec 6) | `/settings` (Tab: Branding) | `PATCH /labs/me` (`accentColor`, `logoUrl`) | "Custom Laboratory Branding" | **Fully Implemented** | Hex swatch picker + live header preview; applied directly to vector PDFs and web report views. |
| **Settings: Letterhead Calibration** | Plain A4 vs Pre-printed stationery millimeter spacers (`10-80mm` top, `10-50mm` bottom) | P0 (Sec 6) | `/settings` (Tab: Branding/Stationery), `/reports/[id]/preview` | `PATCH/PUT /labs/current` (`printSettings`) + `ReportPdfGeneratorService` | "Pre-Printed Stationery Margin Calibration" | **Fully Implemented** | Persists in PostgreSQL (`print_settings`), calibrates `A4DocumentSheet` browser print, and dynamically suppresses digital header/footer blocks in downloaded server PDFs. |
| **Billing & Operational Expenses** | Patient cash/UPI invoices (`SAC 999316`) + operating expense logger + lab aggregate summary | P0/P1 (Sec 6) | `/billing` (`features/billing/`) | `GET/POST /billing/invoices`, `/expenses`, `GET /billing/financial-summary` | "Revenue, Expenses & Collections Ledger" | **Implemented (Unit Test P&L Deferred)** | Aggregate `{ totalRevenue, totalPaid, totalExpenses, expensesByCategory }` drives `useDiagnosticPnL()` with zero mock salary/reagent numbers; `useTestProfitability()` returns `[]` with a clean Phase 2 banner. |
| **Settings: Pathologist Credentials** | MCI/NMC council registration, MD/DNB qualifications, digital signature | P1 (Sec 6) | `/settings` (Tab: Signatures) | `PATCH /labs/me/signatures`, `/staff` | "NABL Digital Sign-Off & Council Verification" | **Fully Implemented** | Upload high-res signature or draw on canvas; prints with council reg number per NABL clause 5.8. |
| **Settings: Team & RBAC Roles** | Staff governance (Owner, Pathologist, Sr Tech, Tech, Phlebotomist, Billing) | P1 (Sec 6) | `/settings` (Tab: Team) | `GET/POST /staff`, `/invite` | "Multi-User Role Permissions" | **Fully Implemented** | Role gating, CSV staff audit export, invitation workflow with sign-off scope tags. |
| **Settings: Platform Billing** | 14-day trial countdown, tiered plan upgrades (₹499 Starter / ₹999 Growth / ₹1,999 Scale), offline 12-digit UTR bank payment | P0 (Sec 7a) | `/settings` (Tab: Billing) | `GET/POST /billing/subscription`, `/utr` | "Transparent Pricing (₹499 – ₹1,999/mo)" | **Fully Implemented** | Trial status banner, UPI QR generator, 12-digit UTR payment submission with live WebSocket unlock. |
| **AI Clinical Assistant** | 1-click provisional clinical remarks and differential diagnosis generator for pathologists based on out-of-range parameters | P1 (AI Co-Pilot) | `/reports/[id]/entry` (`features/reports/`) | `POST /api/v1/reports/:id/ai-impression` (`GenerateClinicalRemarksController`) | "AI Clinical Impression Assistant" | **Fully Implemented** | Google Gemini 1.5 Flash + deterministic clinical fallback; complies with NMC Act 2019 (pathologist retains 100% final review and edit authority). |
| **Patient Smart Summary** | Bilingual (English & Hindi) layperson test explainer on patient public QR portal & WhatsApp redirection | P1 (Patient Explainer) | `/v/[token]` (`features/reports/`) | `GET /api/v1/reports/share/:token/ai-summary` (`GetPatientSmartSummaryController`) | "Bilingual WhatsApp Patient Explainer" | **Fully Implemented** | Public endpoint; translates complex abnormal findings into simple, reassuring layperson terms; strictly enforces NMC non-prescriptive doctor-consultation disclaimers. |

---

## 2. Granular Settings & Print Customization Analysis

### A. Frontend vs. Backend Split
* **Frontend-Driven Ergonomics**:
  * Real-time CSS layout simulation showing exactly how test tables flow when margins are adjusted.
  * Interactive canvas for signing with touch/stylus or dragging high-resolution PNG signatures.
  * Role switcher preview modal for testing technician vs. pathologist permissions.
* **Backend Enforcement**:
  * Immutable persistence of `printSettings` (`stationeryType: PLAIN | PREPRINTED_HEADER | PREPRINTED_HEADER_AND_FOOTER`, `headerMarginMm: number`, `footerMarginMm: number`) in the `labs` table.
  * Vector PDF engine (`ReportPdfGeneratorService` + `resolvePdfPrintLayout`) suppresses digital headers cleanly under `PREPRINTED_HEADER` / `PREPRINTED_HEADER_AND_FOOTER`, reserves clamped top/bottom stationery offsets (`PT_PER_MM`), and suppresses the bottom legal footer note when `PREPRINTED_HEADER_AND_FOOTER` is active.
  * Pathologist profile credentials and qualification metadata are rendered on final reports alongside report numbers (`ReportPdfGeneratorService`).

### B. Marketing Framing vs. Operational Reality

| Feature Capability | What to Show on Marketing (`labOS-web`) | What Remains Internal to Station (`labOS-app`) | Why This Distinction Matters |
| :--- | :--- | :--- | :--- |
| **Pre-Printed Letterheads** | **"Letterhead Alignment Engine"** — Highlight that labs can keep their existing offset paper pads from local printing presses without wasting expensive pre-printed stationery. | Sliders for 30–80mm header and 10–50mm footer margin calibration. | Lab owners fear switching software because it ruins their pre-printed paper stock. Pitching the *solution* converts them; technical slider knobs are just implementation. |
| **Pathologist Signature** | **"NABL ISO 15189 Digital Sign-Off"** — Highlight tamper-proof digital signatures and council registration credentials. | Canvas drawing pad, signature file dimension validator, and sign-off scope checkboxes. | Emphasizes legal compliance, trust, and audit readiness for doctors and patients. |
| **Barcode Thermal Labels** | **"50×25mm Thermal Tube Labels"** — Highlight Code-128 sticker printing via browser print setup for TVS/TSC/Zebra thermal printers (keep multi-bench barcode custody tracking deferred to Phase 3). | `@page { size: 50mm 25mm }` CSS + `JsBarcode`, Accession filter matching on `reportNumber`. | Reassures owners that LabOS avoids handwriting on tubes without overpromising automated analyzer/bench routing. |
| **Team Roles & Permissions** | **"Role-Based Access Control"** — Highlight that technicians cannot alter final results and receptionists only see billing. | Staff invite modals, CSV export buttons, internal role toggles. | Reassures lab directors that their clinical integrity and financial books are protected from unauthorized staff edits. |
| **Subscription & Billing** | **"Affordable Flat Pricing (₹499 – ₹1,999/mo)"** — Transparent pricing with 14-day free trial and UPI payment support. | 12-digit UTR entry modal, 72-hour grace period warning banners, and WebSocket lock triggers. | Keeps the marketing message simple and friction-free while maintaining strict subscription enforcement internally. |
| **AI Clinical Co-Pilot & Explainer** | **"AI Clinical Assistant & Bilingual WhatsApp Explainer"** — Provisional remark drafting under pathologist sign-off control and layperson Hindi/English QR summaries. | Prompt sanitization, deterministic rule-based clinical fallback when LLM is offline, strict non-prescription disclaimers, doctor consultation lock. | Demonstrates practical workflow assistance without unverified speed claims or NMC Act conflicts. |

---

## 3. Inconsistencies & Root Causes Identified

### 1. The "Silent Demo Fallback" Smell
* **Root Cause**: During frontend development (Phases 1–12), query hooks in `labOS-app` were wrapped in `try/catch` blocks that returned mock data arrays from `src/lib/demo-data/` on any fetch error or empty response.
* **Problem**: When a real user logs into a brand new lab account, if an endpoint returns an empty array `[]` or encounters an auth initialization delay, the screen populated fake patients ("Sunita Rao", "Ramesh Gupta") and fake doctors.
* **Remediation**: Purge all silent mock fallbacks. When an API returns `[]`, return `[]` so the application renders a clean, actionable **Empty State**.

### 2. The Abstract Telemetry / Margin Meter Gimmick
* **Root Cause**: An earlier prototype proposed a complex "telemetry bar" and profit margin gauges on the dashboard.
* **Reality**: Standalone lab owners running 10–50 samples/day on paper do not want abstract telemetry or machine-margin meters. They want to know:
  1. *How many patients came in today?*
  2. *Which test reports are pending entry?*
  3. *Which abnormal panic values must be reported immediately?*
  4. *How much cash/UPI was collected at the counter today?*
* **Remediation**: The Mission Control Dashboard must surface **real clinical action items**, not speculative gauges.

---

## 4. Grounded Mission Control Dashboard Plan

Based strictly on **verified backend capabilities** (`GET /reports/dashboard-stats`, `GET /labs/me`, and `GET /billing/subscription`), the redefined dashboard will contain the following four sections:

### Section 1: Facility Identity & Operational Status Bar
* **Purpose**: Instantly reassure the lab owner that their station is legally and technically operational.
* **Data Sources**: `GET /labs/me` and `GET /billing/subscription`.
* **Elements**:
  * **Lab Legal Title**: e.g., *"Apex Diagnostic Center & Advanced Pathology Lab"*.
  * **Station & Compliance Pill**: *"Station 01 • NABL ISO 15189 Aligned"*.
  * **Live Connection Beacon**: Green pulsing indicator powered by `RealtimeProvider` WebSocket status.
  * **Subscription Badge**: *"14-Day Free Trial (12 days left)"* or *"Active Plan (Growth Tier)"*.

### Section 2: Clinical Counter & Financial KPI Ribbon (4 Real Cards)
* **Purpose**: The 4 primary numbers an independent lab owner needs every single morning.
* **Data Source**: `GET /reports/dashboard-stats` (backed by real SQL queries against `reports` and `invoices`).
* **The 4 Cards**:
  1. **Today's Patients**: Total unique walk-in patients registered today (`todayPatientsCount`).
  2. **Pending Test Entry**: Reports waiting for technician numerical value entry (`pendingResultsCount`). Clicking navigates directly to `/accessions?status=DRAFT`.
  3. **Ready for Doctor Review**: Reports with results entered, awaiting pathologist sign-off (`readyForReviewCount`). Clicking navigates directly to `/accessions?status=READY_FOR_REVIEW`.
  4. **Counter Cash & UPI Collected**: Total realized payments collected at the reception counter today (`todayRevenue`). Formatted in Indian Rupee format (`₹X,XXX`).

### Section 3: Clinical Priority Action Queue (Actionable Exceptions)
* **Purpose**: Eliminates paper registers by highlighting urgent exceptions requiring immediate intervention.
* **Data Source**: Derived directly from `GET /reports/dashboard-stats`.
* **Tiles (Shown only when count > 0; displays "All Queues Cleared" when 0)**:
  * **Critical TAT Overdue Alert** (`overdueCount > 0`): Highlights reports that have exceeded standard turnaround time. Direct CTA: `[ Expedite Overdue Reports ]`.
  * **Awaiting Pathologist Sign-Off** (`readyForReviewCount > 0`): Highlights cases ready for digital signature and letterhead printing. Direct CTA: `[ Review & Sign ]`.
  * **Home Collection Pickups** (`pendingCollectionsCount > 0`): Highlights pending home sample dispatches. Direct CTA: `[ View Phlebotomy List ]`.

### Section 4: First-Day "Getting Started" Setup Guide
* **Purpose**: Guides first-time trial users through setting up their lab without cluttering the screen with fake patient data.
* **State**: Automatically computes progress from existing lab profile and sample count:
  * `[✓]` Step 1: Lab Legal Profile & Branding configured.
  * `[✓]` Step 2: Pathologist Digital Signature uploaded.
  * `[ ]` Step 3: Register your first patient sample (`[ + Register First Sample ]`).
  * `[ ]` Step 4: Calibrate 50×25mm thermal printer or letterhead.
* **Graceful Exit**: The entire guide collapses or dismisses automatically once the lab has registered its first 3 real samples.

---

## 5. Prioritized Recommendations & Outstanding Tasks

| Priority | Task | Target Component | Engineering Rationale |
| :--- | :--- | :--- | :--- |
| **Completed** | **Purge Mock Data Fallbacks** | `labOS-app/src/features/*/api/` | Deleted `src/lib/demo-data/` and eliminated all silent `DEMO_*` fallbacks across reports, panels, billing, profitability, collections, referrals, notifications, settings, patients, and subscription hooks. |
| **Completed** | **Sync `printSettings` Into Server PDF** | `labOS-service/.../report-pdf-generator.service.ts` | Added `resolvePdfPrintLayout()` so downloaded PDFs suppress digital headers/footers cleanly and paginate inside safe margins matching browser print. |
| **P0** | **Verify Empty States Across Live Pilot** | `/accessions`, `/patients`, `/referrals`, `/billing`, `/collections`, `/panels`, `/settings` | Manually exercise zero-record workflows in running apps against Supabase/Postgres. |
| **P1** | **Pre-Pilot Smoke Test** | All 4 Apps | Verify the complete live loop: Admin onboards lab $\rightarrow$ Owner logs in $\rightarrow$ Intakes sample $\rightarrow$ Finalizes report $\rightarrow$ Activates via 12-digit UTR. |
