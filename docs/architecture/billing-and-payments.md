# Pathiq — Platform Billing, Subscriptions & Payment Architecture

**Status:** Accepted & Production Active  
**Last Updated:** 2026-10-15  
**Bounded Context:** `modules/platform-billing` & `modules/super-admin`  
**Related Documents:** `docs/product/prd.md`, `docs/architecture/system-design.md`, `docs/architecture/data-model.md`

---

## 1. Executive Summary & Foundational Architectural Notes

This document provides complete architectural notes, design decisions, and flow diagrams for the monetization, subscription lifecycle, transition engine, and payment processing infrastructure of **Pathiq**. The architecture follows Domain-Driven Design (DDD), Hexagonal Architecture (Ports & Adapters), and proven patterns from enterprise SaaS billing engines (Shopify, Stripe Billing, Netflix, AWS).

### Key Architectural Notes:

1. **Market Reality: Standalone Indian Diagnostic Labs**
   - **Target profile:** Standalone pathology labs with 1–3 staff processing 10–50 samples/day.
   - **Payment method:** **>95% operate on UPI (PhonePe, Google Pay, Paytm, BHIM) and Corporate Net Banking (SBI, HDFC, ICICI)**.
   - International credit cards are rarely used for operational healthcare software in India.
   - Stripe in India is restricted (invite-only, lacks native UPI intent deep-linking, and faces high failure rates under RBI recurring e-Mandate rules).
   - **Decision:** Razorpay is the primary gateway for India (`INR`). The domain is decoupled via Hexagonal Architecture (Strategy Pattern) so `StripeGatewayAdapter` can be plugged in for international expansion (`USD`/`EUR`) with zero changes to business logic or database schemas.

2. **Single Source of Truth: Webhook-Driven Fulfillment**
   - The frontend client (`labOS-app`) is **never trusted** to activate a subscription. Client-reported success is treated purely as an optimistic UI hint.
   - Cryptographically verified webhooks (`HMAC SHA-256`) from the payment gateway are the sole authority for state transitions in `tenant_subscriptions`.
   - Real-time client synchronization is achieved in under 200ms via our CloudEvents WebSocket infrastructure (`RealtimeChannelEnum.SUBSCRIPTION_ACTIVATED`) with fallback short-polling.

3. **Atomic Idempotency & Two-Speed Processing Engine**
   - Eliminates race conditions present in naive implementations (`findOne` → `process` → `create`) by enforcing an atomic pre-insert reservation on `webhook_events.event_id` with `ON CONFLICT DO NOTHING`.
   - Core subscription state changes execute synchronously in an atomic database transaction in **<15ms**, acknowledging Razorpay immediately with `200 OK`.
   - Heavy side-effects (PDF generation, storage bucket uploads, WhatsApp receipts) are decoupled via asynchronous message brokers (`pg-boss` background workers).
   - An automated Safety-Net Reconciliation Cron polls the gateway API every 30 minutes for stale pending payments, eliminating lost revenue due to dropped webhooks.

4. **Real-World B2B Transition Dynamics (Upgrades, Prorations & Scheduled Downgrades)**
   - **No Plan Stacking:** The active plan is disabled with a `Current Plan` badge; re-purchasing the same active tier is strictly disallowed (`SAME_PLAN` guard).
   - **Prorated Upgrades:** Upgrading to a higher tier applies daily proration credit for unused days of the current active cycle, automatically calculating 18% GST (CGST 9% + SGST 9% or IGST 18%) on the remaining net taxable balance.
   - **Non-Disruptive Scheduled Downgrades:** Downgrading to a lower tier or Yearly $\to$ Monthly is scheduled at `currentPeriodEnd` (`scheduled_at_period_end`) for **₹0 charged today**. This protects clinical technicians from mid-shift account lockout and sample quota freezing.
   - **1-Click Cancel Downgrade:** Lab owners can cancel a pending downgrade anytime before the renewal date with instant state restoration.

5. **Zero-Rework Environment-Driven Payment UI & Dev Sandbox**
   - When authentic Razorpay keys are configured in `.env`, the frontend automatically loads and launches the official Razorpay Checkout SDK.
   - When placeholder credentials (`rzp_test_placeholder`) are detected in dev/QA, the system generates an `order_sim_...` order and opens the interactive **Pathiq Dev Payment Simulator** modal without requiring any code changes, supporting UPI QR emulation, NetBanking emulation, and failure simulation.

6. **Enterprise B2B GST Rule 46 Tax Invoicing Engine (`pg-boss` + `pdfkit`)**
   - Every successful transaction triggers an asynchronous background job handled by `SubscriptionInvoicePdfWorker`.
   - `SubscriptionInvoicePdfGeneratorService` streams vector PDF invoices adhering to Indian GST Rule 46 (Pathiq Healthtech Private Limited, Gurugram, Haryana | GSTIN `06AAECP1234F1Z8` | SAC 998313 | 18% GST itemization).
   - PDFs are persisted to Supabase Storage (`lab-assets` bucket) and rendered in a self-serve **Billing History & GST Tax Invoices** table in `/settings?tab=billing`.

7. **Clinical Soft-Lock Guard & Patient Safety (NABL ISO 15189)**
   - `SubscriptionIntakeGuard` blocks write operations (`POST /reports`) when a lab's subscription is `LOCKED` or `SUSPENDED` (`403 Forbidden` with code `SUBSCRIPTION_INTAKE_LOCKED`).
   - All read and diagnostic query endpoints (`GET /reports`, `GET /patients`, `GET /accessions`) remain **100% accessible** so that ongoing patient care and historical verification are never compromised.

---

## 2. Master End-to-End Payment Architecture Blueprint & Reusability Guide

This section is the **single-source-of-truth master blueprint** for the entire payment, subscription lifecycle, and monetization engine. Any engineer, new developer, or AI agent building a payment flow—whether for **Pathiq** or adapting it for **another SaaS product (salon booking, clinic management, e-commerce, or B2B platforms)**—can use this visual blueprint as a complete reference.

### Master Architecture Canvas (Single-Pane-of-Glass)

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               MASTER END-TO-END PAYMENT ARCHITECTURE BLUEPRINT                                         │
│     (In-App Checkout + Prorated Upgrades + Two-Speed Webhook + Reconciliation + Offline UTR + Scheduled Downgrades)    │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ 1. FRONTENDS & ACTORS ]       [ 2. API & PORTS LAYER ]           [ 3. DB & STATE MACHINE ]      [ 4. REALTIME & WORKERS ]

  ════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  FLOW A: STANDARD ONLINE CHECKOUT & PRORATED UPGRADES (HAPPY PATH)
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Tenant App (labOS-app)
       │
       │ 1. GET /billing/subscriptions/preview-transition
       ├────────────────────────────► NestJS API (CQRS Handler)
       │  { planSlug, cycle }          │
       │                               │ 2. Calculates:
       │                               │    • Transition Type (UPGRADE, CYCLE_CHANGE, etc.)
       │                               │    • Daily Proration Credit for unused active days
       │                               │    • 18% GST (CGST 9% + SGST 9%) on remaining balance
       │ 3. Displays Checkout Dialog   │
       │◄──────────────────────────────┤
       │ (Itemized base, credit, GST)
       │
       │ 4. POST /billing/orders
       ├────────────────────────────► CreateSubscriptionOrderHandler
       │  { planSlug, cycle, gstin }   │
       │                               │ 5. Check 15m Intent Window & JIT Gateway Check:
       │                               ├──────────────────────────► platform_payments
       │                               │                            (Checks recent pending order)
       │                               │ 5a. IF already captured on Gateway (delayed webhook):
       │                               │     Fulfills subscription atomically on the spot!
       │ 6a. { alreadyFulfilled: true }│     Returns order payload with alreadyFulfilled=true
       │◄──────────────────────────────┤
       │ (Frontend skips SDK modal,    │ 5b. IF failed on Gateway:
       │  invalidates cache, unlocks!) │     UPDATE status='EXPIRED', generates fresh order
       │                               │
       │                               │ 5c. IF still pending on Gateway:
       │                               │     Reuses existing order_id (no double-charging)
       │                               │
       │                               │ 5d. IF no pending order:
       │                               │     IPaymentGatewayPort.createOrder()
       │                               ├───────────────────────────────────────────────────────► Razorpay Orders API
       │                               │                                                               │
       │                               │ 6. INSERT status='PENDING'                                    │ Returns order_id
       │                               ├──────────────────────────► platform_payments                  │
       │ 7. Returns order details      │                            (Stores order_id & lab_id)          │
       │◄──────────────────────────────┤                                                               │
       │                                                                                               │
       │ 8. Opens Checkout SDK (Live) OR DevPaymentSimulatorModal (Sandbox)                            │
       ├──────────────────────────────────────────────────────────────────────────────────────────────►│
       │                                                                                               │
       │ 9. Customer Authorizes Payment via PhonePe / GPay UPI / NetBanking                            │
       └──────────────────────────────────────────────────────────────────────────────────────────────►│
                                                                                                       │
  ════════════════════════════════════════════════════════════════════════════════════════             │
  FLOW B: TWO-SPEED SYNCHRONOUS WEBHOOK ACTIVATION (<15ms)                                             │
  ────────────────────────────────────────────────────────────────────────────────────────             │
                                  POST /billing/webhooks/razorpay                                      │
                                  (rawBody + HMAC SHA-256 header)                                      │
                                  ◄────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ 1. Verify HMAC with crypto.timingSafeEqual (<1ms)
                                  │
                                  │ 2. Pre-Insert Lock:
                                  │    INSERT INTO webhook_events
                                  │    ON CONFLICT (event_id) DO NOTHING;
                                  ├──────────────────────────► webhook_events
                                  │                            (Abort immediately if duplicate)
                                  │
                                  │ 3. Managed DB Transaction:
                                  │    • payment.status = 'SUCCESS'
                                  │    • subscription.status = 'ACTIVE'
                                  │    • subscription.period_end extended (30d / 365d)
                                  │    • create subscription_invoices record
                                  │    • clear pending downgrade (scheduledPlanId = null)
                                  ├──────────────────────────► tenant_subscriptions
                                  │                            subscription_invoices
                                  │
                                  │ 4. Broadcast Realtime Event
                                  ├───────────────────────────────────────────────────────► Realtime Gateway
                                  │                                                         (Socket.IO CloudEvents)
       5. [ Instant Unlock &      │                                                                │
            Confetti Screen! ]    │                                                                │ Broadcasts to
       ◄──────────────────────────┼────────────────────────────────────────────────────────────────┘ room "lab:${labId}"
                                  │
                                  │ 6. Return 200 OK (<15ms total!)
                                  │    (Acknowledges Razorpay immediately)
                                  │
                                  │ 7. Enqueue Async PDF Generation Job
                                  └───────────────────────────────────────────────────────► pg-boss Job Queue
                                                                                            (SubscriptionInvoicePdfWorker
                                                                                             generates Rule 46 PDF &
                                                                                             uploads to Supabase Storage)

  ════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  FLOW C: SAFETY-NET RECONCILIATION CRON (SELF-HEALS DROPPED WEBHOOKS & NETWORK DROPS)
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                                  PaymentReconciliationCron
                                  (Automated 30-Minute Schedule)
                                  │
                                  │ 1. Query PENDING payments
                                  │    (15m < created_at < 24h)
                                  ├──────────────────────────► platform_payments
                                  │
                                  │ 2. GET /orders/{id}/payments
                                  ├───────────────────────────────────────────────────────► Gateway Remote API
                                  │                                                         (Did user pay?)
                                  │ 3. Evaluate Remote Gateway State:
                                  │    ├── CAPTURED ──► Execute Self-Healing Activation (Same DB Transaction!)
                                  │    └── FAILED/EXP ─► UPDATE payment status='EXPIRED'

  ════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  FLOW D: TWO-WAY OFFLINE CORPORATE BANK TRANSFER (NEFT / RTGS) & UTR LEDGER
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Tenant App (labOS-app)
       │
       │ 1. Lab selects "Bank Transfer" tab & copies Pathiq HDFC Bank Account Details
       │ 2. Transfers funds via NetBanking and gets 12-digit UTR
       │ 3. POST /billing/payments/offline-utr
       ├────────────────────────────► SubmitOfflinePaymentHandler
       │  { orderId, utrNumber }       │
       │                               │ 4. Anti-Fraud Check: Verify UTR uniqueness
       │                               │ 5. Sets payment status='PENDING', method='MANUAL_UTR',
       │                               │    offlineStatus='PENDING_VERIFICATION'
       │                               ├──────────────────────────► platform_payments
       │ 6. Shows "Submitted for      │
       │    Manual Verification"       │
       │◄──────────────────────────────┤
       │
  Super Admin (labOS-admin)
       │
       │ 7. Opens Payment Ledger -> Sees "PENDING_VERIFICATION" badge
       │ 8. Clicks "[ Verify & Activate ]"
       ├────────────────────────────► AdminManualActivationHandler
       │  { labId, planSlug, utr }     │
       │                               │ 9. Managed DB Transaction:
       │                               │    • payment.status = 'SUCCESS'
       │                               │    • subscription.status = 'ACTIVE' (+30d/+365d)
       │                               │    • create subscription_invoices record
       │                               ├──────────────────────────► tenant_subscriptions
       │                               │                            platform_payments
       │ 10. Displays Success Toast    │ 11. Broadcast Realtime Unlock
       │◄──────────────────────────────┼──────────────────────────────────────────────────► Realtime Gateway
                                       │                                                          │
  Tenant App (labOS-app)               │                                                          ▼ Instant Sync!
       ◄───────────────────────────────┴──────────────────────────────────────────────────────────┘

  ════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════
  FLOW E: NON-DISRUPTIVE SCHEDULED DOWNGRADES & HOURLY LIFECYCLE CRON
  ────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Tenant App (labOS-app)
       │
       │ 1. POST /billing/subscriptions/schedule-downgrade { planSlug: 'STARTER' }
       ├────────────────────────────► ScheduleSubscriptionDowngradeHandler
       │                               │
       │                               │ 2. Sets scheduledPlanId='...', scheduledAt=currentPeriodEnd
       │                               │    (No plan change or charge today: ₹0)
       │                               ├──────────────────────────► tenant_subscriptions
       │ 3. Displays Amber Banner      │ 4. Emits WebSocket SUBSCRIPTION_UPDATED
       │    "Downgrade Scheduled"      ├──────────────────────────────────────────────────► Realtime Gateway
       │◄──────────────────────────────┤                                                          │
       │                                                                                          ▼
       │ 5. Optional: DELETE /cancel-downgrade (1-click restore before renewal date)               Update UI
       │
  Hourly Background Engine (SubscriptionLifecycleCron)
       │
       │ 6. Evaluates expired subscriptions every 1 hour:
       │    ├── IF TRIALING expired ──► Move to 72h GRACE_PERIOD
       │    ├── IF ACTIVE expired with scheduledPlanId:
       │    │   ├── Atomically switch planId = scheduledPlanId, clear scheduled fields
       │    │   └── Move to 72h GRACE_PERIOD for renewal
       │    ├── IF ACTIVE expired without scheduledPlan:
       │    │   └── Move to 72h GRACE_PERIOD
       │    └── IF GRACE_PERIOD expired ──► Transition to LOCKED (Intake blocked via Guard)
       └─────────────────────────────► tenant_subscriptions
```

---

### The 12-Step Universal Execution Map

1. **Transition Preview & Proration:** When a lab owner clicks "Upgrade" or "Downgrade", the client calls `GET /api/v1/billing/subscriptions/preview-transition?planSlug=...&billingCycle=...`. The backend dynamically computes transition type (`NEW_ACTIVATION`, `SAME_PLAN`, `UPGRADE`, `DOWNGRADE`, `CYCLE_CHANGE`), proration credit for remaining active days, and 18% GST (CGST 9% + SGST 9%).
2. **User Confirmation in Checkout Review Dialog:** The client displays an itemized receipt showing base tier fee, proration credit deduction, GST calculation, and an optional Lab GSTIN input field.
3. **Idempotency Defense & Dual-Channel JIT Settlement:** 
   - **In-Flight Coalescing:** An in-process `inFlightOrders` Map coalesces concurrent checkout requests for `${labId}:${planId}:${cycle}`, attaching concurrent clicks to the identical Promise without holding database connections open.
   - **15-Minute Intent Window & Gateway Verification:** Application verifies if an open `PENDING` payment exists within the 15-minute intent window. If found, it queries Razorpay `fetchPaymentStatus`. If already captured (webhook delayed), it fulfills immediately and returns `{ alreadyFulfilled: true }` (frontend skips the checkout modal). If failed, it expires and re-creates. If pending, it reuses the existing order, preventing double charging and eliminating the *"This order has already been paid"* Razorpay error.
4. **Gateway Strategy Abstraction:** Core calls outbound port `IPaymentGatewayPort.createOrder()`. The active adapter (`RazorpayGatewayAdapter`) requests order initialization from Razorpay, or generates a sandbox order (`order_sim_...`) if placeholder keys are detected.
5. **Pending Ledger Record:** A `platform_payments` row is inserted with status `PENDING` and unique `provider_order_id`.
6. **Client Payment Execution:**
   - **Production:** Frontend dynamically loads `checkout.js` and opens the official Razorpay Checkout SDK with prefilled contact data (`contact: +91 85688 84848`, `email`, `name`).
   - **Development/QA:** Opens `DevPaymentSimulatorModal` (UPI QR scan emulation, NetBanking emulation, and test response triggers).
   - **Corporate Bank Transfer:** Lab owner copies Pathiq bank account details, transfers funds, and submits the 12-digit UTR via `SubmitOfflinePaymentHandler`.
7. **Cryptographic Webhook Handshake:** Gateway sends `POST /api/v1/billing/webhooks/razorpay`. Raw binary body is validated against webhook secret via constant-time `crypto.timingSafeEqual` in <1ms.
8. **Atomic Idempotency Lock:** Controller attempts `INSERT INTO webhook_events (...) ON CONFLICT (event_id) DO NOTHING`. Duplicates receive `200 OK` immediately without re-processing.
9. **Synchronous <15ms Database Activation:** Atomic transaction (`dataSource.transaction`) acquires a `pessimistic_write` row lock on `platform_payments`. If already `SUCCESS` or `DUPLICATE_UNFULFILLED`, it exits immediately. Otherwise:
   - For legitimate upgrades or cycle changes (`planId` or `billingCycle` change), the new subscription tier activates immediately starting from `now` with `fulfillment_status = 'FULFILLED'`.
   - For accidental duplicate payments on the exact same plan, `fulfillment_status = 'DUPLICATE_UNFULFILLED'` is recorded for manual refund review instead of stacking unintended subscription days.
   - In both cases, a collision-free statutory B2B GST tax invoice is generated via PostgreSQL sequence `subscription_invoice_seq` (`PATHIQ-INV-YYYY-XXXXXX`), protected by unique constraint `UQ_subscription_invoices_payment_id`.
   - Clears any pending scheduled downgrades, and updates the webhook event to `PROCESSED`.
10. **Real-Time Client Broadcast:** CloudEvents WebSocket emits `RealtimeChannelEnum.SUBSCRIPTION_ACTIVATED` to room `lab:${labId}`, unlocking the user's screen in real time with celebration feedback.
11. **Decoupled Background GST Invoicing (`pg-boss` + `pdfkit`):** Asynchronous `SubscriptionInvoicePdfWorker` generates statutory B2B GST Rule 46 PDF invoice with SAC 998313 breakdown, uploads to Supabase Storage, and updates `subscription_invoices.pdf_url`.
12. **Self-Healing Reconciliation & Lifecycle Crons:**
    - `PaymentReconciliationCron` polls every 30 minutes to self-heal unconfirmed transactions where webhooks were delayed or dropped.
    - `SubscriptionLifecycleCron` runs hourly to transition expired trials, apply scheduled downgrades, manage 72-hour grace periods, and lock overdue accounts.

---

### How to Reuse This Architecture in Other SaaS Applications

This payment architecture was designed as a **universal, enterprise-grade template**. To adapt this system for another application (such as salon booking, medical clinics, gym management, or B2B SaaS):

#### What You Keep (Domain-Agnostic Core — 100% Reusable):
- **`IPaymentGatewayPort` & Hexagonal Adapters:** Strategy Pattern allowing plug-in of `RazorpayGatewayAdapter` for domestic Indian payments (UPI Intent/QR) or `StripeGatewayAdapter` for international cards without altering business logic.
- **Proration Engine:** Daily fractional math for upgrades that calculates exact credit and adjusts GST automatically.
- **Scheduled Downgrades:** Protecting operations by postponing lower tier migrations to period end.
- **Atomic Webhook Lock (`ON CONFLICT DO NOTHING`):** The exact pattern that prevents double billing and handles duplicate webhook retries.
- **Two-Speed Architecture:** Synchronous database transaction (<15ms) + asynchronous event decoupling via `pg-boss` for PDF generation and messaging.
- **Zero-Rework Dev Simulator:** Complete UI and API sandbox allowing full team testing without live payment keys.
- **Safety-Net Reconciliation Cron:** Self-heals payments when customers close their phone browser or when gateway webhooks drop.
- **15-Minute Idempotency Window:** Prevents impatient users from creating duplicate gateway orders via rapid multi-clicks.
- **Offline Bank Transfer (NEFT/RTGS) & UTR Ledger:** Allows collecting offline bank transfers with anti-fraud duplicate detection.

#### What You Customize (Application-Specific):
1. **Plan Slugs & Entitlements:** Adjust `STARTER`, `GROWTH`, `SCALE` slugs and JSON entitlement fields in `platform_plans` to match your app's quota metrics (e.g., appointments/month instead of samples/month).
2. **Pricing & Currency:** Update monthly and yearly prices in INR paise or target currency.
3. **Tenant Scoping:** Replace `lab_id` with your tenant identifier (`tenant_id`, `organization_id`, or `store_id`).
4. **GST / Tax Rules:** Customize invoice tax rates (e.g., VAT, Sales Tax) in `SubscriptionInvoicePdfGeneratorService`.

---

## 3. System Blueprint & Implementation Status

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      IMPLEMENTATION PIPELINE & STATUS MAP                   │
└─────────────────────────────────────────────────────────────────────────────┘

   STEP 1: CORE ENGINE (labOS-service) [100% COMPLETED & VERIFIED]
   ┌────────────────────────────────────────────────────────────────────────┐
   │ • 7 PostgreSQL Migrations (plans, subscriptions, payments, webhooks,   │
   │   invoices, default plan seeds, and scheduled transitions)             │
   │ • Domain Repositories (DIP) & Domain Exceptions (404/400/401/409)      │
   │ • IPaymentGatewayPort + RazorpayGatewayAdapter (Strategy Pattern)      │
   │ • 21 CQRS Vertical Slices (orders, webhooks, proration, transitions,   │
   │   offline UTR, invoices, dev simulation, admin management)             │
   │ • pg-boss Async Message Broker for B2B GST Rule 46 PDF Generation      │
   │ • 30-Minute Safety-Net Reconciliation Cron + Hourly Lifecycle Cron     │
   └───────────────────────────────────┬────────────────────────────────────┘
                                       │ Real API Contracts & DB Schemas Ready
                                       ▼
   STEP 2: SUPER ADMIN PANEL (labOS-admin) [100% COMPLETED & VERIFIED]
   ┌────────────────────────────────────────────────────────────────────────┐
   │ • Tenant Directory (Active labs, trial countdown, read-only locks)     │
   │ • Offline UTR Verification & Manual Activation Tool (Payment Ledger)   │
   │ • Plan Management (Pricing toggles, annual discounts, tenant overrides)│
   │ • Data Purge Utility ("Purge Demo Records" for test cleanups)          │
   └───────────────────────────────────┬────────────────────────────────────┘
                                       │ Admin Verification Ready
                                       ▼
   STEP 3: TENANT IN-APP BILLING (labOS-app) [100% COMPLETED & VERIFIED]
   ┌────────────────────────────────────────────────────────────────────────┐
   │ • In-App Plan Upgrade Modal + Razorpay Standard Checkout SDK           │
   │ • CheckoutReviewDialog with daily proration & 18% GST breakdown        │
   │ • Zero-Rework DevPaymentSimulatorModal (UPI QR + NetBanking emulators) │
   │ • ScheduleDowngradeDialog (Non-disruptive, ₹0 charged today)           │
   │ • Billing History & GST Invoices Table with PDF viewer & print engine  │
   │ • 14-Day Trial Header Chip, Grace Period Banner & Soft-Lock Guard      │
   └───────────────────────────────────┬────────────────────────────────────┘
                                       │ In-App Monetization Complete
                                       ▼
   STEP 4: MARKETING WEBSITE (labOS-web) [100% COMPLETED & VERIFIED]
   ┌────────────────────────────────────────────────────────────────────────┐
   │ • Public Landing Page & Feature Showcase (Independent Next.js SSG/ISR) │
   │ • Dynamic Pricing Table reading plans live from Backend API            │
   │ • Marketing Acquisition Handshake with URL fallback safety             │
   │ • High-Performance SEO Suite (Schema.org JSON-LD, Metadata)           │
   └────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Hexagonal Architecture: Ports & Adapters Design (Strategy Pattern)

The domain core depends strictly on domain contracts, completely decoupled from vendor SDKs (Razorpay, Stripe) or database mechanics.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HEXAGONAL PORTS & ADAPTERS ARCHITECTURE                  │
└─────────────────────────────────────────────────────────────────────────────┘

  APPLICATION / CQRS FEATURES (21 Vertical Slices)
  ├── GetPublicPlansHandler
  ├── GetTenantSubscriptionHandler
  ├── CreateSubscriptionOrderHandler
  ├── PreviewSubscriptionTransitionHandler
  ├── ScheduleSubscriptionDowngradeHandler
  ├── CancelScheduledDowngradeHandler
  ├── HandleRazorpayWebhookHandler
  ├── SimulatePaymentCaptureHandler
  ├── SubmitOfflinePaymentHandler
  ├── GetTenantInvoicesHandler
  ├── ReconcileStalePaymentsHandler
  └── Admin CQRS Handlers (ManualActivation, ExtendTrial, Purge, etc.)
                                       │
                                       ▼
  OUTBOUND DOMAIN PORTS        ┌──────────────────────────────┐
                               │     IPaymentGatewayPort      │
                               │  --------------------------  │
                               │  + createOrder()             │
                               │  + verifyWebhookSignature()  │
                               │  + parseWebhook()            │
                               │  + fetchPaymentStatus()      │
                               └──────────────┬───────────────┘
                                              │
        ┌─────────────────────────────────────┼─────────────────────────────────────┐
        │                                     │                                     │
        ▼                                     ▼                                     ▼
   [ Razorpay Adapter ]              [ Stripe Adapter ]                   [ Manual Adapter ]
  (Active: India / INR)            (Pluggable: USD / EUR)                 (Offline UTR / Admin)
┌───────────────────────┐        ┌───────────────────────┐             ┌───────────────────────┐
│ RazorpayGatewayAdapter│        │  StripeGatewayAdapter │             │  ManualPaymentAdapter │
│ • razorpay.orders     │        │ • stripe.paymentIntent│             │ • Super Admin UTR     │
│ • timingSafeEqual HMAC│        │ • constructEvent HMAC │             │ • Zero Webhook Secret │
│ • UPI / QR / NetBank  │        │ • International Cards │             │ • Direct Activation   │
│ • Dev Sandbox Fallback│        └───────────────────────┘             └───────────────────────┘
└───────────────────────┘
```

### Architectural Notes on Ports & Adapters:
- **`IPaymentGatewayPort` (Outbound Port):** The core domain only knows this interface. It provides order creation, signature verification, webhook normalization, and remote payment querying.
- **`RazorpayGatewayAdapter` (Active India Adapter):** Integrates Razorpay Orders API, UPI QR, intent flows, and validates cryptographic signatures using `crypto.timingSafeEqual` in constant time to prevent timing attacks. Includes graceful sandbox simulation fallback when credentials are placeholder.
- **`StripeGatewayAdapter` (International Ready):** Plugs in without touching subscription entities or handlers when expanding to Middle East, UK, or US markets.
- **`ManualPaymentAdapter` (Pilot Offline Adapter):** Handles corporate NEFT/RTGS bank transfers where customers submit a 12-digit UTR and administrators verify it in the Super Admin panel.

---

## 5. Subscription Plan Modeling & Entitlement Rules

In enterprise SaaS, plans are **never** arbitrary rows in a database table where an admin can freely type any name. Features and entitlements are hardcoded into application permissions.

### Canonical Plan Entitlement Matrix:

| Plan Slug | Target Diagnostic Lab | Monthly INR | Yearly INR | Staff Limit | Samples/Mo | Key Entitlements |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`STARTER`** | Small 1–2 technician standalone lab | **₹499** | **₹4,990** | 2 Users | 500 | Barcode labels, A4 millimeter letterhead calibration, WhatsApp sharing, Referral ledger. |
| **`GROWTH`** | Established lab with doctor network | **₹999** | **₹9,990** | 6 Users | 2,000 | All Starter + NABL 5.8 Reference Lab Send-Outs, Barcode scanner sync, Diagnostic P&L. |
| **`SCALE`** | High-volume center with home riders | **₹1,999** | **₹19,990** | 20 Users | 10,000 | All Growth + Home Collection GPS Dispatch, Multi-counter ledger, Priority SLA support. |

### Plan Rules:
1. **Slugs are Immutable Code Constants:** `STARTER`, `GROWTH`, `SCALE` are seeded via TypeORM migration `1710000008006`.
2. **Super Admin Controls:**
   - Update display prices (`monthly_price_inr`, `yearly_price_inr`) or tax rates.
   - Deactivate plans (`is_active = false`) to grandfather existing labs while hiding them from new signups.
   - Set custom tenant overrides (`tenant_subscriptions.custom_entitlement_overrides`) for high-volume enterprise labs.
3. **Immutability Invariant:** Super Admin cannot delete a plan with active subscribers, preventing database foreign key corruption.

---

## 6. End-to-End Payment & Transition Flows

---

### Path 1: Standard In-App Checkout, Upgrade & Proration Engine

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                   PATH 1: IN-APP CHECKOUT, PRORATED UPGRADE & TAX CALCULATION                          │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Lab Owner ]         [ Next.js App ]        [ NestJS API ]         [ PostgreSQL ]         [ Razorpay ]
        │                      │                      │                      │                    │
     1. │ Clicks Upgrade       │                      │                      │                    │
        ├─────────────────────►│                      │                      │                    │
     2. │                      │ GET /preview-transit.│                      │                    │
        │                      ├─────────────────────►│                      │                    │
     3. │                      │                      │ Calculates proration │                    │
        │                      │                      │ & 18% GST            │                    │
     4. │                      │ Returns preview DTO  │                      │                    │
        │                      │◄─────────────────────┤                      │                    │
     5. │ Opens CheckoutReview │                      │                      │                    │
        │ Dialog (Base-Credit) │                      │                      │                    │
     6. │ Confirms & Pays      │                      │                      │                    │
        ├─────────────────────►│ POST /billing/orders │                      │                    │
     7. │                      ├─────────────────────►│ Check 15m Idempotency│                    │
        │                      │                      ├─────────────────────►│                    │
     8. │                      │                      │ orders.create()      │                    │
        │                      │                      ├──────────────────────────────────────────►│
     9. │                      │                      │                      │ Returns order_id   │
        │                      │                      │◄──────────────────────────────────────────┤
    10. │                      │                      │ INSERT PENDING pymt  │                    │
        │                      │                      ├─────────────────────►│                    │
    11. │                      │ Returns orderId, key │                      │                    │
        │                      │◄─────────────────────┤                      │                    │
    12. │                      │ Opens Checkout SDK   │                      │                    │
        │                      │ (Prefilled Contact)  │                      │                    │
        │                      ├─────────────────────────────────────────────────────────────────►│
    13. │ Pays via PhonePe/UPI │                      │                      │                    │
        ├────────────────────────────────────────────────────────────────────────────────────────►│
    14. │                      │ Returns payment_id   │                      │                    │
        │                      │◄─────────────────────────────────────────────────────────────────┤
    15. │ Shows "Syncing..."   │                      │                      │                    │
        │ (Awaits WebSocket)   │                      │                      │                    │
```

#### Step-by-Step Notes (Checkout & Proration):
1. **Transition Preview (`PreviewSubscriptionTransitionHandler`):**
   - Calculates daily fractional proration credit for unused days of the current plan:
     $$\text{fraction} = \frac{\text{remainingMs}}{\text{totalPeriodMs}}$$
     $$\text{prorationCredit} = \min(\text{targetPrice}, \text{fraction} \times \text{currentPrice})$$
   - Computes 18% GST (CGST 9% + SGST 9% or IGST 18%) on the remaining gross amount.
2. **Checkout Review Dialog:** Displays transparent price breakdown (Base Tier Fee $-$ Unused Plan Credit $+$ 18% GST $=$ Net Payable). Allows lab to input their 15-character Lab GSTIN.
3. **Idempotency Check:** Reuses open `PENDING` payment created within the last 15 minutes to prevent duplicate gateway orders.
4. **Prefill Ergonomics:** Prefills lab contact phone (`+91 85688 84848`), administrator email, and lab name directly into Razorpay options to eliminate repetitive phone-entry screens.

---

### Path 2: Two-Speed Webhook Ingestion & Asynchronous GST Invoicing (<15ms)

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│             PATH 2: TWO-SPEED WEBHOOK INGESTION & ASYNC TAX INVOICING (<15ms)                          │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Razorpay Gateway ]      [ Webhook Controller ]      [ PostgreSQL DB ]     [ Realtime WS ]     [ pg-boss Worker ]
           │                         │                         │                     │                   │
        1. │ POST /webhooks/razorpay │                         │                     │                   │
           ├────────────────────────►│                         │                     │                   │
        2. │                         │ Verify HMAC SHA-256     │                     │                   │
           │                         │ (rawBody in <1ms)       │                     │                   │
        3. │                         │ Pre-insert lock:        │                     │                   │
           │                         │ INSERT INTO webhooks    │                     │                   │
           │                         │ ON CONFLICT DO NOTHING  │                     │                   │
           │                         ├────────────────────────►│                     │                   │
           │                         │                         │                     │                   │
           │                 ┌───────┴───────┐                 │                     │                   │
           │                 │ Is Duplicate? │                 │                     │                   │
           │                 └───────┬───────┘                 │                     │                   │
           │                         │                         │                     │                   │
           │            [YES]        │       [NO]              │                     │                   │
           │     ┌───────────────────┘──────────────┐          │                     │                   │
           │     │ Return 200 OK                    │          │                     │                   │
           │◄────┤ (Idempotent ACK)                 ▼          │                     │                   │
           │     │                     Managed Transaction:    │                     │                   │
           │     │                     • Row-lock subscription │                     │                   │
           │     │                     • payment = SUCCESS     │                     │                   │
           │     │                     • Extend subscription   │                     │                   │
           │     │                     • Clear pending downgr. │                     │                   │
           │     │                     • Generate Tax Invoice  │                     │                   │
           │     │                     • webhook = PROCESSED   │                     │                   │
           │     │                     ├──────────────────────►│                     │                   │
        4. │     │                     │ Emit WS Notification  │                     │                   │
           │     │                     ├────────────────────────────────────────────►│                   │
        5. │     │ Return 200 OK       │                                             │ Broadcast to lab  │
           │◄────┴─────────────────────┤ Total Time: <15ms!                          ├──────────────────►│
        6. │                           │                                             │                   │
           │                           │ Enqueue 'generate-subscription-invoice-pdf' │ [ Instant Unlock! ]
           │                           ├────────────────────────────────────────────────────────────────►│
           │                           │                                             │                   │ 7. Generates
           │                           │                                             │                   │    Rule 46 PDF
           │                           │                                             │                   │    & uploads to
           │                           │                                             │                   │    Supabase
```

#### Step-by-Step Notes (Two-Speed Webhook & Invoicing):
1. **Cryptographic Validation:** `rawBody` is validated against `RAZORPAY_WEBHOOK_SECRET` using `crypto.timingSafeEqual` in <1ms.
2. **Atomic Pre-Insert Reservation:** `INSERT INTO webhook_events (...) ON CONFLICT (event_id) DO NOTHING`. Duplicates receive `200 OK` immediately.
3. **Synchronous <15ms Transaction:** Row-locks subscription, updates payment to `SUCCESS`, extends period by 30 or 365 days, generates invoice record, and clears pending downgrades.
4. **Real-Time WebSocket Broadcast:** Emits `SUBSCRIPTION_ACTIVATED` to room `lab:${labId}` in <100ms.
5. **Gateway ACK:** Returns `200 OK` to Razorpay in under 15ms.
6. **Decoupled PDF Worker (`pg-boss`):** `SubscriptionInvoicePdfWorker` consumes the job, calls `SubscriptionInvoicePdfGeneratorService` to stream vector PDF adhering to Indian GST Rule 46, uploads to Supabase Storage `lab-assets`, and updates `subscription_invoices.pdf_url`.

---

### Path 3: Safety-Net Reconciliation Cron Flow

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                        PATH 3: SAFETY-NET RECONCILIATION CRON (SELF-HEALING)                           │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Scheduled Cron ]        [ NestJS Handler ]        [ PostgreSQL DB ]       [ Razorpay Gateway API ]
          │                          │                         │                         │
       1. │ Triggers Every 30 Min    │                         │                         │
          ├─────────────────────────►│                         │                         │
       2. │                          │ Query PENDING payments  │                         │
          │                          │ (15m < age < 24h)       │                         │
          │                          ├────────────────────────►│                         │
          │                          │ Returns stale list      │                         │
          │                          │◄────────────────────────┤                         │
       3. │                          │                                                   │
          │                          │ Loop: For each stale payment:                     │
          │                          │   GET /orders/{id}/payments                       │
          │                          ├──────────────────────────────────────────────────►│
          │                          │   Returns [{ id, status: 'captured' }]            │
          │                          │◄──────────────────────────────────────────────────┤
          │                          │                                                   │
          │                  ┌───────┴────────┐                                          │
          │                  │ Gateway State? │                                          │
          │                  └───────┬────────┘                                          │
          │                          │                                                   │
          │            [CAPTURED]    │       [FAILED / EXPIRED]                          │
          │      ┌───────────────────┘────────────────┐                                  │
          │      ▼                                    ▼                                  │
       4. │ Execute Self-Healing Activation    UPDATE payment SET status='EXPIRED'       │
          │ (Atomic Transaction)               ├────────────────────────►│               │
          │ ├─────────────────────────────────►│                         │               │
          │ │ Log Audit: "Auto-reconciled"     │                         │               │
```

#### Step-by-Step Notes (Reconciliation Cron):
1. **Schedule:** `PaymentReconciliationCron` triggers every 30 minutes.
2. **Query Stale Payments:** Searches `platform_payments` for records where `status = 'PENDING'`, `created_at < NOW() - 15m`, and `created_at > NOW() - 24h`.
3. **Gateway Verification:** Calls `IPaymentGatewayPort.fetchPaymentStatus(providerOrderId)`.
4. **Resolution:** If captured, executes identical atomic activation transaction and enqueues invoice generation. If failed, marks status `EXPIRED`.

---

### Path 4: Two-Way Offline Corporate Bank Transfer (NEFT/RTGS) & UTR Ledger

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│               PATH 4: TWO-WAY OFFLINE BANK TRANSFER (NEFT/RTGS) & UTR APPROVAL                         │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Lab Owner ]           [ Next.js App ]          [ NestJS API ]         [ Super Admin ]      [ PostgreSQL ]
        │                        │                       │                      │                     │
     1. │ Selects "Bank Transfer"│                       │                      │                     │
     2. │ Transfers via NEFT/RTGS│                       │                      │                     │
     3. │ Submits 12-digit UTR   │                       │                      │                     │
        ├───────────────────────►│ POST /offline-utr     │                      │                     │
        │                        ├──────────────────────►│ Verify UTR uniqueness│                     │
        │                        │                       ├───────────────────────────────────────────►│
        │                        │                       │ Set PENDING_VERIFIC. │                     │
        │ 4. "Submitted for      │ Returns success       ├───────────────────────────────────────────►│
        │    Verification" toast │◄──────────────────────┤                      │                     │
        │◄───────────────────────┤                       │                      │                     │
        │                        │                       │ 5. Opens Admin Ledger│                     │
        │                        │                       │    Sees UTR row      │                     │
        │                        │                       │◄─────────────────────┤                     │
        │                        │                       │ 6. Clicks [Verify]   │                     │
        │                        │                       │    POST /activate    │                     │
        │                        │                       ├──────────────────────┤                     │
        │                        │                       │ Managed Transaction: │                     │
        │                        │                       │ • status = 'SUCCESS' │                     │
        │                        │                       │ • extend +30d / +365d│                     │
        │                        │                       │ • create Tax Invoice │                     │
        │                        │                       ├───────────────────────────────────────────►│
        │                        │ 7. WS Unlock Broadcast│                      │                     │
        │ 8. Screen Unlocks!     │◄──────────────────────┴──────────────────────┤                     │
        │◄───────────────────────┤                                                                    │
```

#### Step-by-Step Notes (Two-Way Offline Bank Transfer):
1. **Customer Submission (`SubmitOfflinePaymentHandler`):** Lab owner inputs their 12-digit UTR in `CheckoutReviewDialog`. The system verifies UTR uniqueness against prior payments, links it to the pending order, and marks `offlineStatus = 'PENDING_VERIFICATION'`.
2. **Super Admin Ledger (`PaymentLedger` on `labOS-admin`):** Displays pending transfers with UTR, timestamp, lab name, and amount.
3. **1-Click Manual Activation (`AdminManualActivationHandler`):** Administrator verifies the bank credit and clicks `[ Verify & Activate ]`. Atomically extends the subscription, creates a `SUCCESS` payment and GST invoice, and broadcasts real-time WebSocket activation.

---

### Path 5: Non-Disruptive Scheduled Downgrades & Hourly Lifecycle Cron

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│               PATH 5: NON-DISRUPTIVE SCHEDULED DOWNGRADES & LIFECYCLE CRON                             │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Lab Owner ]         [ Next.js App ]        [ NestJS API ]         [ PostgreSQL ]     [ Lifecycle Cron ]
        │                      │                      │                      │                    │
     1. │ Clicks Downgrade     │                      │                      │                    │
        ├─────────────────────►│                      │                      │                    │
     2. │                      │ POST /schedule-down. │                      │                    │
        │                      ├─────────────────────►│                      │                    │
     3. │                      │                      │ UPDATE subscription: │                    │
        │                      │                      │ • scheduledPlanId=...│                    │
        │                      │                      │ • scheduledAt=period │                    │
        │                      │                      ├─────────────────────►│                    │
     4. │ Displays amber banner│ Emits WS update      │                      │                    │
        │ "Downgrade on [date]"│◄─────────────────────┤                      │                    │
        │ (₹0 charged today)   │                      │                      │                    │
        │                      │                      │                      │                    │
        │                      │                      │                      │ 5. Hourly scan:    │
        │                      │                      │                      │    periodEnd <= now│
        │                      │                      │                      │◄───────────────────┤
        │                      │                      │                      │ 6. Applies change: │
        │                      │                      │                      │    plan = scheduled│
        │                      │                      │                      │    grace = 72h     │
        │                      │                      │                      ├───────────────────►│
        │                      │ WebSocket broadcast  │                      │                    │
        │                      │◄─────────────────────────────────────────────────────────────────┤
        │ 7. Seamlessly        │                      │                      │                    │
        │    transitions tier  │                      │                      │                    │
```

#### Step-by-Step Notes (Scheduled Downgrades):
1. **Zero Today Charge:** Downgrading to a lower tier or Yearly $\to$ Monthly charges ₹0 today. The current active features remain 100% unlocked until `currentPeriodEnd`.
2. **Scheduled Record:** Sets `scheduled_plan_id`, `scheduled_billing_cycle`, and `scheduled_at = currentPeriodEnd`.
3. **1-Click Cancellation:** Lab owners can click `Cancel Downgrade` anytime before the period ends via `DELETE /api/v1/billing/subscriptions/cancel-downgrade`.
4. **Hourly Cron Application (`SubscriptionLifecycleCron`):** Checks matured subscriptions every hour. When `currentPeriodEnd` passes, it atomically switches `planId = scheduledPlanId`, clears the scheduled columns, enters the 72-hour grace period for renewal, and notifies the client via WebSockets.

---

### Path 6: Zero-Rework Dev Payment Simulator Sandbox

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                 PATH 6: ZERO-REWORK DEV PAYMENT SIMULATOR SANDBOX                                      │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

  [ Developer / QA ]        [ Next.js App ]             [ NestJS API ]             [ PostgreSQL DB ]
          │                         │                          │                          │
       1. │ Clicks Upgrade (Dev)    │                          │                          │
          ├────────────────────────►│                          │                          │
       2. │                         │ POST /billing/orders     │                          │
          │                         ├─────────────────────────►│ Detects placeholder keys │
          │                         │                          │ Generates order_sim_...  │
          │                         │ Returns simulation order │                          │
          │                         │◄─────────────────────────┤                          │
       3. │                         │                          │                          │
          │ Opens DevPayment-       │                          │                          │
          │ SimulatorModal (UPI QR) │                          │                          │
       4. │ Clicks [Simulate Success│                          │                          │
          ├────────────────────────►│ POST /simulate-capture   │                          │
          │                         ├─────────────────────────►│ Managed Transaction:     │
          │                         │                          │ • payment = SUCCESS      │
          │                         │                          │ • subscription = ACTIVE  │
          │                         │                          │ • create Tax Invoice     │
          │                         │                          ├─────────────────────────►│
          │                         │                          │ Enqueue pg-boss PDF job  │
          │                         │ WebSocket Broadcast      │                          │
          │                         │◄─────────────────────────┤                          │
       5. │ Immediate Confetti &    │                          │                          │
          │ "Subscription Activated"│                          │                          │
```

#### Step-by-Step Notes (Dev Simulator):
1. **Zero Code Changes:** Enabled automatically when `RAZORPAY_KEY_ID = 'rzp_test_placeholder'`.
2. **Authentic Emulation:** Renders PhonePe, Google Pay, Paytm, and BHIM QR scan emulation and bank selector.
3. **Test Controls:** Supports `[ Simulate Success ]`, `[ Bank Decline ]`, and `[ Network Delay / Timeout ]`.
4. **Instant Fulfillment:** Direct call to `POST /api/v1/billing/payments/simulate-capture` triggers the full production activation pipeline in <100ms.

---

## 7. All Edge Cases & Failure Recovery Flows

---

### Edge Case 1: Webhook Arrives BEFORE Client Browser Redirects (Race Condition)
- **Problem:** On high-speed gateway networks, the webhook arrives and processes before the user's mobile browser redirects back to the lab application.
- **Mitigation:** Because the webhook executes synchronously in <15ms, by the time the browser completes its HTTP redirect 500ms later, the database is already marked `ACTIVE`. The client reads the activated state immediately with zero delay.

---

### Edge Case 2: Customer Closes Browser Tab or Drops Network on UPI App
- **Problem:** Lab owner authorizes payment in PhonePe/GPay, then switches apps or closes the browser tab before returning to LabOS.
- **Mitigation:** The frontend is never trusted for activation. The webhook independently fulfills the order in PostgreSQL. When the user logs back in, their account is already active.

---

### Edge Case 3: Gateway Network Drop / Dropped Webhook (Dual-Channel JIT Settlement & Cron Healing)
- **Problem:** Temporary network disruption causes the gateway webhook to fail delivery or experience severe queue delay.
- **Mitigation (Dual-Channel Settlement):**
  1. **Synchronous Pull (Just-In-Time Verification):** When the user returns to LabOS and clicks "Upgrade" again, `CreateSubscriptionOrderHandler` checks the active pending order against Razorpay's API (`fetchPaymentStatus`). If Razorpay confirms the order was captured, the backend **activates the subscription on the spot**, returns `{ alreadyFulfilled: true }`, and the client closes the modal with a success toast—completely eliminating the jarring `"This order has already been paid"` gateway error.
  2. **Asynchronous Pull (Safety-Net Cron):** The automated `PaymentReconciliationCron` runs every 30 minutes, identifies pending payments older than 15 minutes, queries Razorpay's API, and automatically activates the subscription.

---

### Edge Case 4: Duplicate Webhook Retries / Replay Attacks (Two-Tier Atomic Idempotency)
- **Problem:** Gateway retries can send duplicate webhook events simultaneously across parallel threads (or Razorpay sends both `payment.captured` and `order.paid` for the same payment), risking duplicate subscription extensions and duplicate tax invoices.
- **Mitigation:**
  1. **Event Reservation Layer (L1):** Atomic pre-insert on `webhook_events.event_id` with `ON CONFLICT DO NOTHING`. Duplicate event IDs abort immediately and return `200 OK`.
  2. **Payment Row Lock Layer (L2):** In `activateSubscriptionFromPayment`, an atomic database transaction (`dataSource.transaction`) acquires a `pessimistic_write` row lock on `platform_payments`. If `status === 'SUCCESS'` or `fulfillmentStatus === 'DUPLICATE_UNFULFILLED'`, execution exits immediately without duplicate subscription mutation or invoice creation.

---

### Edge Case 5: Payment Declined / User Cancels on UPI App
- **Problem:** Insufficient funds, wrong UPI PIN, or bank server timeout causes payment failure.
- **Mitigation:** The gateway fires `payment.failed`. The webhook handler updates `platform_payments.status = 'FAILED'`, stores the failure reason, and the client displays a clear error message with a retry button.

---

### Edge Case 6: Concurrent Multi-Tab Checkout (In-Flight Promise Coalescing & 15m Intent Window)
- **Problem:** User clicks "Upgrade" multiple times or opens multiple tabs, potentially generating multiple open orders on Razorpay.
- **Mitigation:**
  1. **In-Flight Coalescing (L1):** `inFlightOrders` Map coalesces concurrent checkout requests for `${labId}:${planId}:${cycle}` within the process, attaching concurrent requests to the identical execution Promise without holding database connections open across external HTTP calls.
  2. **Intent Window Lookup (L2):** `CreateSubscriptionOrderHandler` queries `paymentRepo.findRecentPending` (15-minute window). If an active pending order exists, it performs JIT verification: if already paid, fulfills immediately; if failed, expires and creates fresh; if pending, reuses the order ID.

---

### Edge Case 7: Plan Stacking Attempt (Re-purchasing Active Plan)
- **Problem:** An active subscriber clicks "Upgrade" for their current plan, which would double-bill or corrupt dates.
- **Mitigation:** The active plan card displays a disabled `Current Plan` button. In addition, the backend `PreviewSubscriptionTransitionHandler` returns `allowed: false` with `SAME_PLAN` transition type, blocking order generation.

---

### Edge Case 8: Duplicate UTR Submission (Anti-Fraud)
- **Problem:** A lab owner attempts to reuse a previously approved bank transfer UTR or submits an identical UTR across two orders.
- **Mitigation:** `SubmitOfflinePaymentHandler` and `AdminManualActivationHandler` perform a uniqueness query for `providerPaymentId = cleanUtr`. Any duplicate throws `DuplicateUtrException` (HTTP 409 Conflict).

---

### Edge Case 9: Razorpay Contact Gating
- **Problem:** Razorpay checkout modal prompts the user to manually enter an email and 10-digit phone number before revealing UPI payment options.
- **Mitigation:** `checkout-review-dialog.tsx` automatically injects `prefill: { contact: '+91 85688 84848', email: user.email, name: labProfile.name }` directly into the SDK config, bypassing the gating screen entirely.

---

### Edge Case 10: Dropped Invoice PDF Queue Job (Durable Cron Recovery)
- **Problem:** Worker server restarts, queue connection drops, or transient storage failure occurs while generating the GST invoice PDF.
- **Mitigation:** In addition to `pg-boss` exponential retries, the 30-minute `PaymentReconciliationCron` (`ReconcileStalePaymentsHandler`) queries `findInvoicesMissingPdf` for invoices where `pdf_url IS NULL` (between 5 minutes and 24 hours old) and automatically re-enqueues `GENERATE_SUBSCRIPTION_INVOICE_PDF`.

---

### Edge Case 11: Accidental Duplicate Payments for the Same Cycle (Case B Hold Policy)
- **Problem:** User opens two checkout tabs, generates Order 1 and Order 2, and pays both orders via UPI/card for the exact same billing cycle.
- **Mitigation:** 
  1. **Upgrades / Cycle Changes:** A plan tier upgrade (e.g. Starter $\to$ Growth) or cycle change (Monthly $\to$ Yearly) is recognized by `planId` or `billingCycle` difference and is **always fulfilled immediately** starting from `now`.
  2. **Same-Plan Duplicate:** If the payment is for the **exact same plan and cycle** while an unexpired active period exists, it does NOT auto-extend the subscription by an arbitrary 60 days. Instead, `payment.status` is set to `SUCCESS`, statutory GST Tax Invoice is generated from sequence (every rupee captured requires a tax invoice under Indian law), and `fulfillmentStatus` is set to `DUPLICATE_UNFULFILLED` (`failureReason = 'Accidental duplicate payment; held for refund review'`).

---

### Edge Case 12: Late Capture of an Expired Order
- **Problem:** An order was marked `EXPIRED` by the 15-minute reconciliation cron, but the customer's bank approved the transaction late.
- **Mitigation:** When Razorpay's webhook delivers a valid `payment.captured` for a payment with `status = 'EXPIRED'`, the webhook handler safely recovers the payment from `EXPIRED` $\to$ `SUCCESS` and fulfills the subscription.

---

## 8. The Subscription Lifecycle State Machine & Clinical Soft-Lock Guard

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                             SUBSCRIPTION LIFECYCLE STATE MACHINE                                       │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────────────────────────────┐
                    │                   TRIALING                   │
                    │          (14 Days Full Feature Access)       │
                    └───────┬──────────────────────────────┬───────┘
                            │                              │
              Payment Made  │                              │ Trial Expires (Day 14)
             (₹499/999/1999)│                              │
                            ▼                              ▼
                    ┌───────────────┐              ┌───────────────┐
                    │    ACTIVE     │◄─────────────┤ GRACE_PERIOD  │
                    │  (Full Access)│ Payment Made │ (72 Hours)    │
                    └───┬───────▲───┘ within Grace └───────┬───────┘
                        │       │                          │
          Renewal Fails │       │ Renewal Cleared          │ Grace Period Ends
          (After 30d)   │       │                          │ (No Payment)
                        ▼       │                          ▼
                    ┌───────────┴───┐              ┌───────────────┐
                    │ GRACE_PERIOD  │              │    LOCKED     │
                    │ (72h Warning) │─────────────►│  (Read-Only   │
                    └───────────────┘ Grace Closes │  Historical)  │
                                                   └───┬───────────┘
                                                       │
                                       Payment Cleared │
                                       at any time     ▼
                                                   [ ACTIVE ]
```

### Operational Behavior Matrix by State:

| Subscription State | Duration | Clinical Operations (Worklist, Results, Reports, Print) | Accession Intake (New Patients) | User Experience & UI Banners |
| :--- | :--- | :--- | :--- | :--- |
| **`TRIALING`** | 14 Days from Registration | **Unlocked (Full Access)** | **Unlocked** | Green badge: *"Trial: X days remaining"* |
| **`ACTIVE`** | 30 Days (or 365 Days) | **Unlocked (Full Access)** | **Unlocked** | Clean UI (Active Plan Name displayed) |
| **`GRACE_PERIOD`** | 72 Hours after Due Date | **Unlocked (Zero Patient Interruption)** | **Unlocked** | Amber warning banner: *"Payment due. 72h grace active."* |
| **`LOCKED`** | Indefinite until Payment | **Read-Only (Can view/print past records)** | **BLOCKED (Requires Plan Purchase)** | Red banner & modal: *"Subscription expired. Upgrade to resume intake."* |

### Clinical Soft-Lock Guard (`SubscriptionIntakeGuard`):
```typescript
@Injectable()
export class SubscriptionIntakeGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const labId = request.user?.labId;
    const subscription = await this.subRepo.findByLabId(labId);

    if (subscription?.status === SubscriptionStatusEnum.LOCKED || subscription?.status === SubscriptionStatusEnum.SUSPENDED) {
      throw new ForbiddenException({
        code: 'SUBSCRIPTION_INTAKE_LOCKED',
        message: 'Laboratory subscription is locked. Please activate your plan to register new patient accessions.',
      });
    }
    return true;
  }
}
```
- **NABL ISO 15189 Clinical Rule:** Diagnostic labs cannot halt ongoing patient testing or prevent doctor verification because a subscription expired at midnight. The 72-hour `GRACE_PERIOD` guarantees zero patient disruption. Even when `LOCKED`, all historical test results and reports remain 100% accessible and printable.

---

## 9. Summary Reference Matrix: All Edge Cases & Protections

| # | Production Scenario | Risk / Failure Mode | Detection Mechanism | Automated Architectural Mitigation |
| :- | :--- | :--- | :--- | :--- |
| **1** | **Customer pays in UPI, closes browser before redirect** | Client-side redirect never fires; user fears lost payment. | Webhook independent delivery | **Webhook Single Source of Truth:** Webhook activates database in <15ms regardless of client state. |
| **2** | **Duplicate webhooks sent concurrently** | Gateway network retries cause race condition; double subscription extension. | Pre-insert collision on `event_id` | **Atomic Pre-Insert Lock:** `ON CONFLICT (event_id) DO NOTHING` aborts duplicate threads immediately with 200 OK. |
| **3** | **Webhook arrives before client browser redirect** | Client queries status and sees stale data. | Order lookup timing | **Optimistic Read-Through:** <15ms webhook execution guarantees DB is active before 500ms browser redirect. |
| **4** | **Payment declined / UPI timeout** | Lab owner abandons checkout or bank server fails. | `payment.failed` webhook | **Failure Tracking:** Updates status to `FAILED`, logs gateway error reason, displays retry CTA. |
| **5** | **Double-click / Rapid checkout retries** | User clicks "Upgrade" multiple times across tabs. | Lookup for active pending orders | **15-Minute Idempotency Reuse:** Reuses existing `order_id` for identical lab and plan within 15 minutes. |
| **6** | **Dropped webhook during server deploy** | Payment captured at gateway, but webhook dropped. | Scheduled cron query (`age > 15m`) | **Safety-Net Reconciliation Cron:** Runs every 30m, polls gateway API, and self-heals unconfirmed transactions. |
| **7** | **Invalid webhook signature / Spoofing** | Malicious actor sends fake activation webhook. | HMAC SHA-256 verification | **Constant-Time Verification:** `crypto.timingSafeEqual` rejects spoofed requests with 401 Unauthorized. |
| **8** | **Two-way offline bank transfer (NEFT/RTGS)** | Lab pays via direct bank transfer and submits 12-digit UTR. | Anti-fraud UTR uniqueness query | **ManualPaymentAdapter & Ledger:** Records as `PENDING_VERIFICATION`; Super Admin activates with 1 click. |
| **9** | **Prorated Tier Upgrades** | Mid-cycle upgrade risks double billing for unused days. | Transition preview calculation | **Daily Proration Engine:** Deducts unused fractional credit and calculates 18% GST on remaining balance. |
| **10** | **Non-Disruptive Downgrades** | Downgrading immediately risks technician lockout & quota cut. | Transition classifier | **Scheduled at Period End:** Sets `scheduled_plan_id` for ₹0 charged today; applied on period end by hourly cron. |
| **11** | **Delayed Webhook + Order Reuse ("This order has already been paid")** | User pays via UPI, webhook is slow, user clicks Upgrade again; Razorpay throws error if order reused. | Gateway API query on pending order reuse | **Dual-Channel JIT Settlement:** Queries `fetchPaymentStatus` before returning cached order; if captured, fulfills immediately and returns `alreadyFulfilled: true` (skips SDK). |
| **12** | **Accidental Same-Plan Duplicate Payment (Parallel Tabs)** | User pays twice for the exact same plan tier in separate browser tabs. | Compare plan ID and billing cycle with unexpired active subscription | **Case B Hold Policy:** Captures payment, issues sequential GST tax invoice, but records `fulfillment_status = 'DUPLICATE_UNFULFILLED'` for admin refund review instead of corrupting subscription dates. |
| **13** | **Late Capture of Expired Payment** | Bank approves payment hours after reconciliation marked order EXPIRED. | Webhook `payment.captured` on `status === 'EXPIRED'` | **Late Capture Recovery:** Webhook handler transitions `EXPIRED` $\to$ `SUCCESS` and fulfills subscription safely. |
| **14** | **Dropped Invoice PDF Background Job** | Worker restart or transient storage failure loses PDF generation task. | Cron scan for `pdf_url IS NULL` (5m < age < 24h) | **Self-Healing PDF Recovery:** Reconciliation cron re-enqueues `GENERATE_SUBSCRIPTION_INVOICE_PDF` job durably. |

---

## 10. Database Schema Specification (TypeORM Entities)

### 1. `platform_plans` (Migration `1710000008001`)
```sql
CREATE TABLE platform_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(50) UNIQUE NOT NULL, -- 'STARTER', 'GROWTH', 'SCALE'
    name VARCHAR(100) NOT NULL,
    description TEXT,
    monthly_price_inr INTEGER NOT NULL, -- 499, 999, 1999
    yearly_price_inr INTEGER NOT NULL,  -- 4990, 9990, 19990
    feature_entitlements JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_public BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    version INTEGER NOT NULL DEFAULT 1
);
```

### 2. `tenant_subscriptions` (Migrations `1710000008002` & `1710000008007`)
```sql
CREATE TABLE tenant_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    plan_id UUID NOT NULL REFERENCES platform_plans(id) ON DELETE RESTRICT,
    status VARCHAR(30) NOT NULL DEFAULT 'TRIALING',
    billing_cycle VARCHAR(20) NOT NULL DEFAULT 'MONTHLY',
    trial_start TIMESTAMPTZ NOT NULL DEFAULT now(),
    trial_end TIMESTAMPTZ NOT NULL,
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    grace_period_end TIMESTAMPTZ,
    custom_entitlement_overrides JSONB DEFAULT '{}',
    canceled_at TIMESTAMPTZ,
    scheduled_plan_id UUID REFERENCES platform_plans(id) ON DELETE SET NULL,
    scheduled_billing_cycle VARCHAR(20),
    scheduled_at TIMESTAMPTZ,
    customer_gstin VARCHAR(15),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    version INTEGER NOT NULL DEFAULT 1,
    CONSTRAINT uq_tenant_subscription_lab UNIQUE (lab_id)
);

CREATE INDEX IF NOT EXISTS "IDX_tenant_subscriptions_scheduled_at"
ON "tenant_subscriptions" ("scheduled_at");
```

### 3. `platform_payments` (Migrations `1710000008003` & `1710000008008`)
```sql
CREATE TABLE platform_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES tenant_subscriptions(id) ON DELETE SET NULL,
    provider VARCHAR(30) NOT NULL, -- 'RAZORPAY', 'STRIPE', 'MANUAL'
    provider_order_id VARCHAR(100),
    provider_payment_id VARCHAR(100),
    amount_in_paise INTEGER NOT NULL,
    currency VARCHAR(10) NOT NULL DEFAULT 'INR',
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    payment_method VARCHAR(50), -- 'UPI', 'NETBANKING', 'CARD', 'MANUAL_UTR'
    payment_details JSONB DEFAULT '{}',
    idempotency_key VARCHAR(100) UNIQUE,
    fulfillment_status VARCHAR(30), -- 'FULFILLED', 'DUPLICATE_UNFULFILLED' (NULL for pending/failed/expired)
    failure_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    version INTEGER NOT NULL DEFAULT 1
);
```

### 4. `webhook_events` (Migration `1710000008004`)
```sql
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id VARCHAR(100) UNIQUE NOT NULL,
    provider VARCHAR(30) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    error_message TEXT,
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5. `subscription_invoices` (Migrations `1710000008005` & `1710000008008`)
```sql
-- PostgreSQL Sequence for Collision-Free Sequential Invoicing
CREATE SEQUENCE IF NOT EXISTS subscription_invoice_seq START WITH 100001 INCREMENT BY 1;

CREATE TABLE subscription_invoices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_number VARCHAR(50) UNIQUE NOT NULL, -- e.g. 'PATHIQ-INV-2026-100001'
    lab_id UUID NOT NULL REFERENCES labs(id) ON DELETE CASCADE,
    payment_id UUID NOT NULL REFERENCES platform_payments(id) ON DELETE RESTRICT,
    amount_inr NUMERIC(10,2) NOT NULL,
    tax_amount_inr NUMERIC(10,2) NOT NULL DEFAULT 0.00, -- 18% GST (SAC 998313)
    subtotal_inr NUMERIC(10,2) NOT NULL,
    customer_gstin VARCHAR(20),
    billing_period_start TIMESTAMPTZ NOT NULL,
    billing_period_end TIMESTAMPTZ NOT NULL,
    pdf_url TEXT,
    status VARCHAR(30) NOT NULL DEFAULT 'PAID',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT UQ_subscription_invoices_payment_id UNIQUE (payment_id)
);
```
