# LabOS — Product Requirements Document

**Status:** Finalized — v1
**Owner:** Shivam
**Last updated:** 2026-09-03

---

## 1. Problem Statement

India's diagnostic lab market is roughly 85% unorganized — dominated by small, standalone labs run by 1–3 people, still operating on paper registers or basic Excel sheets. These labs want to digitize but are systematically priced out: every existing LIS/LIMS product (CrelioHealth, MocDoc, LabWare, and global players like Orchard) is built and priced for labs doing 100+ samples/day across multiple branches, with real year-one costs of ₹1L–3L+ once onboarding, analyzer integration, and hidden fees are included. This mismatch — not lack of demand — is why the majority of India's labs remain undigitized.

The cost of not solving this: these labs continue losing time to manual report writing, have no way to track referral-doctor relationships or outsourced tests systematically, can't see their own profitability per test, and remain vulnerable to the compliance pressure that's increasingly being enforced (Clinical Establishments Act, NABL push) without any affordable path to get compliant.

---

## 2. Goals

1. Enable a small, standalone lab (1–3 people, 10–50 samples/day) to fully replace paper-based reporting with a digital workflow, with onboarding achievable in under a day without vendor hand-holding.
2. Give lab owners visibility into their own business — cost per test, referral-doctor revenue, basic profitability — that they currently have no tooling for at all.
3. Support the outsourced/referral testing workflow (sending select tests to reference labs like Lal PathLabs/SRL/Metropolis and merging results back) that's common at this segment but unaddressed by existing products.
4. Be the first LIS in this market segment with transparent, published, flat pricing — no quote-only sales process.
5. Build an architecture that lets a lab grow from single-location to multi-branch on the *same* product, avoiding the painful platform migration that currently happens when small labs outgrow cheap tools.

*(Note: these are directional goals. Specific measurable targets — e.g. "X% of pilot labs fully stop using paper within Y weeks" — belong in Section 6, and need real numbers agreed before launch, not invented here.)*

---

## 3. Non-Goals

1. **Enterprise/chain labs (Lal PathLabs, Metropolis, Dr. Lal, SRL, etc.) as direct customers.** Different buyer, different sales motion (procurement/RFP cycles vs. self-serve signup), and they're already served by mature enterprise LIS products. Revisit only after the core segment is proven and profitable.
2. **Developed-market foreign expansion (US/UK/EU).** These markets carry HIPAA/CLIA/CAP-equivalent compliance burdens and entrenched, well-funded incumbents. Foreign expansion, if pursued, should target markets with a similar unorganized-small-lab dynamic to India (parts of Southeast Asia, Africa, Middle East) — not the US/UK as a first move.
3. **Analyzer/instrument HL7-ASTM integration in v1.** Genuinely complex and expensive to build well; belongs in a later phase once there's real demand signal from active customers, not built speculatively.
4. **NABL-accreditation-grade compliance tooling in v1.** Relevant for labs pursuing accreditation, but premature before we have paying customers at that stage of growth.
5. **Full accounting/bookkeeping replacement.** Basic expense tracking and cost-per-test visibility are in scope (Goal 2); replacing a lab's actual accounting software/CA relationship is not.
6. **Leading go-to-market with mid-sized chains (3–10 branches, 100–500 samples/day).** This was considered and explicitly rejected as a Phase 1 strategy. The reasoning: we already have a live design partner in the small-standalone-lab segment (Goal 1's persona), that segment is dramatically less served, and mid-size buyers expect features that are themselves Non-Goals right now (analyzer integration, multi-branch, NABL tooling) and a sales motion (procurement cycles, demos, account management) we're not built for. Revisit only after the small-lab segment is proven.

---

## 4. Target Users / Personas

### Primary: Independent Lab Owner ("Deswal" persona)
A pathologist or lab technician who owns and runs a single-location diagnostic lab in a Tier 2/3 town, doing 10–50 samples/day. Often doubles as phlebotomist, report-filler, and business owner. Low-to-moderate tech comfort. Currently on paper. No dedicated IT or procurement function — buys and adopts tools himself, based on trust and word-of-mouth from other lab owners.

### Secondary: Lab Technician/Assistant
Works under the owner, may handle sample collection and data entry but typically doesn't make purchasing decisions or configure the system.

### Secondary: Referring Doctor
Sends patients to the lab and receives reports back — not a direct user of the system initially, but a stakeholder in report delivery (WhatsApp/email) and, longer-term, potentially a portal user.
Design a minimalist, intellectually clever tech logo for "Pathiq", a software platform digitizing pathology labs and medical diagnostics.

Conceptual Direction:
Explore intelligent visual metaphors around:
- The moment of clarity when a diagnostic result is discovered
- Dynamic flow and seamless data transformation
- Clever use of negative space or continuous line geometry that subtly suggests vision, analysis, or biological harmony

Visual Execution:
- A distinct, iconic abstract glyph alongside the lowercase modern typography "pathiq"
- Balanced, mathematically precise layout with sophisticated proportions
- Sophisticated dual-tone palette: deep midnight blue and intelligent vibrant teal/aquamarine
- Modern, tech-native, enterprise-grade feel suitable for international healthcare investors and lab technicians alike
- Crisp vector logo on pure white background, zero drop shadows, no 3D textures, no stock clipart tropes

### Secondary: Patient
Receives the report — not a system user in v1 beyond receiving a WhatsApp/email/printed report, but a future patient-portal user in later phases.

---

## 5. User Stories

**Owner — Core reporting**
- As a lab owner, I want to fill in test results using a form that matches my existing paper sheets, so that I don't have to relearn a new report structure.
- As a lab owner, I want to print or send a report exactly formatted with my lab's letterhead, so that it looks professional to patients and referring doctors.
- As a lab owner, I want out-of-range results automatically flagged, so that abnormal values are never missed by a distracted staff member.

**Owner — Referral/outsourced testing**
- As a lab owner, I want to mark a test as "sent to [reference lab]" and track its status, so that I know which results are still pending from outside my lab.
- As a lab owner, I want the outsourced result to merge into the final patient report alongside my in-house results, so that the patient gets one consolidated report, not two.
- As a lab owner, I want to track which referring doctors send me patients and any commission arrangement (flat fee, percentage, or none), and see a clear ledger of what's owed, so that I don't need a separate register — with actual payout/settlement remaining a manual step I control, not an automated payment.
- Same tracking model applies to outsourced-test commissions/margins from reference labs.

> **Flag, not a decision:** doctor referral commissions ("cut practice") sit in a genuinely sensitive area of Indian medical ethics/regulation (Medical Council of India guidelines restrict commission-for-referral in various contexts). This product should support neutral *record-keeping* of referral relationships and revenue, not market itself as a "commission automation" tool. Before finalizing this feature's framing, have a direct conversation with the pilot lab owner about how this is actually handled in practice — he'll know the real norms better than either of us.

**Owner — Business management**
- As a lab owner, I want to see my cost and profit per test, so that I know which tests are actually worth running in-house versus outsourcing.
- As a lab owner, I want to track lab expenses (reagents, consumables, equipment purchases), so that I have one place for my operational costs instead of a separate notebook.

**Owner — Setup/customization**
- As a lab owner, I want to configure my own test panels and parameters, so that I'm not stuck with a generic template that doesn't match what I actually test for.
- As a lab owner, I want to set up my lab's branding (name, address, logo) once, so that every report and print output uses it automatically.

**Owner — Home sample collection**
- As a lab owner, I want to flag a booking as "home collection" with the patient's address and preferred time, so I can track it alongside walk-in samples.
- As a lab owner, I want a simple daily list of today's home-collection bookings, so I know where to go without needing route-optimization tooling built for multi-phlebotomist operations (out of scope for our 1–3 person ICP; revisit if/when serving larger teams).

**Edge cases to cover**
- As a lab owner, I want to reprint a past report without retyping all values, so that a patient who lost their copy can get a new one quickly.
- As a lab owner, I want the system to keep working (or clearly tell me it's offline) if my internet drops mid-entry, so that I don't lose work in low-connectivity conditions.

---

## 6. Requirements

### Must-Have (P0) — the product isn't viable without these
- Config-driven test panel/parameter system (add/edit/remove panels, sections, parameters — no hardcoded test lists)
- Lab branding/letterhead configuration
- Patient info capture + report filling matching paper-sheet structure
- Print/PDF output matching lab's actual letterhead
- Multi-tenant backend with real accounts and persistent storage
- Patient/report history and search (reprint without retyping)
- Out-of-range auto-flagging on results with a defined normal range
- Referral/outsourced test tracking (send-out status, merge result into final report)
- Referring-doctor tracking: name, association, commission arrangement (flat/percentage/none), running ledger of what's owed — settlement itself stays manual (see Section 5 flag on referral commissions)
- WhatsApp and/or email report delivery
- Transparent, published, flat subscription pricing (not quote-only) — see Section 7a, Pricing Model
- Simple home-collection booking + daily list (address, time, status) — not route optimization (see P2)

### Nice-to-Have (P1) — significantly better, not launch-blocking
- Basic expense tracking (reagents, consumables, purchases)
- Cost-per-test / profitability visibility
- Role-based access (owner vs. technician vs. pathologist sign-off)
- Offline-tolerant local-first data entry with background sync (design: optimistic concurrency with version numbers + field-level merge; see `docs/architecture/decisions/` once written)
- Multi-branch support under one account
- "Lab Health" dashboard — a short, prioritized action list (e.g. "14 reports overdue," "reagent running low") rather than raw charts; inspired by a genuine gap found in every competitor reviewed

### Future Considerations (P2) — explicitly deferred, but shouldn't be architected out
- Analyzer/instrument integration (HL7/ASTM)
- Barcode-based sample tracking
- NABL-audit-readiness tooling (audit trails, QC checkpoints)
- Patient self-service portal
- Home-collection route optimization (Maps API, multi-phlebotomist dispatch) — for teams beyond our 1–3 person ICP
- API access for hospital/EHR integration
- Predictive inventory / AI-assisted operations — promising but genuinely later-stage; avoid pulling forward prematurely

---

## 7a. Pricing Model

Researched against real India LIS pricing data (not just the mid-market players): a genuine low-end segment already exists (LabSmart ~₹417/mo, AyusLab ~₹1,179/mo, Qmarksoft ~₹1,500) sitting well below CrelioHealth/Drlogy's ₹1,700–25,000+/month. This validates that small labs will pay *something* — our pricing needs to sit at or below that low end while being clearly more usable.

- **Free trial:** 14–21 days, full-feature, self-serve, no card required.
- **Starter tier:** ₹499–₹799/month (or ~₹5,000/year annual) — core reporting, patient history, WhatsApp delivery.
- **Growth tier:** ₹1,500–₹2,500/month — adds referral/outsourced tracking, expense/business visibility, home collection.
- **Scale tier:** custom/quote-based — multi-branch, analyzer integration, NABL tooling (the one tier where quote pricing is acceptable, since these buyers expect it).
- **Both monthly and annual billing offered**, with a discount (~15–20%) for annual. Monthly matters for a first-time software adopter's commitment anxiety; annual matters for our cash flow and churn.
- **Foreign pricing** (deferred, not urgent): ~$50/month anchors us as the affordable alternative internationally too, given foreign incumbents (Orchard, CGM) start at $500–2,000/month. Only relevant once foreign expansion is actually pursued.

**Trial vs. pilot, kept distinct:**
- *Standard free trial* — anyone, self-serve, 14–21 days.
- *Founding partner deal* — a separate, personally-negotiated arrangement for the pilot lab and a handful (5–10 max) of early hand-picked labs: longer free access (e.g. 3–6 months) in exchange for real feedback, testimonial, and referrals. Capped in count, with an explicit (even if informal) point at which they roll onto a paid plan.

---

## 7. Success Metrics

*(Flagging upfront: this section needs real discussion — the targets below are placeholders showing the shape of what we should define, not agreed numbers.)*

**Leading indicators (early signal, days–weeks post-launch)**
- % of reports generated digitally vs. still on paper, per pilot lab
- Time to complete a full report (digital vs. lab's own paper baseline)
- Onboarding time: how long from signup to first real report generated

**Lagging indicators (weeks–months)**
- Retention: % of pilot labs still actively using the product after 3 months
- Word-of-mouth signal: number of new labs onboarded via referral from an existing lab, not direct outreach
- Revenue: number of paying labs, MRR

**Open decision:** what specific targets count as "success" for the pilot phase (e.g. "your friend's lab fully off paper within 2 weeks") needs to be agreed explicitly, not assumed.

---

## 8. Decisions Log

Previously open questions, now resolved:

- **Pricing:** Tiered ₹499–₹2,500/month depending on tier, both monthly and annual offered. See Section 7a.
- **Pilot/trial structure:** Standard self-serve trial (14–21 days) kept distinct from a capped set of founding-partner deals (longer free access, negotiated individually). See Section 7a.
- **Offline sync conflict resolution:** Optimistic concurrency — version numbers per record, field-level auto-merge when edits don't overlap, human-resolved flag when they do. Not needed until Phase 2 (multi-device/multi-user). To be formalized as an ADR before Phase 2 implementation.
- **Referral-doctor commission scope:** Configurable tracking + calculated ledger, no automated settlement/payout in v1. See the ethics/regulatory flag in Section 5 — needs a direct conversation with the pilot lab owner before finalizing framing.
- **Timeline:** None — personal project, self-directed pacing, no external deadline.

**Still genuinely open:** the specific numeric success-metric targets in Section 7 (retention %, TAT targets, etc.) — these should be set once the pilot lab has real baseline data to compare against, not guessed now.

---

## 9. Phasing / Timeline Considerations

*(Note: LabOS is a new, standalone project — it does not build on or depend on any prior personal tool. Phasing below starts fresh.)*

- **Phase 1:** Real backend, multi-tenancy, patient history, referral/outsourced test tracking, referring-doctor tracking, WhatsApp delivery, basic billing, simple home-collection booking/list. Target: pilot lab (Deswal) fully migrated off paper.
- **Phase 2:** Expense tracking, cost/profitability visibility, role-based access, offline-tolerant sync (write the sync ADR before starting this), Lab Health dashboard. Onboard a small number of additional labs via word-of-mouth from Phase 1.
- **Phase 3:** Multi-branch support, barcode tracking, analyzer integration — only once real customer demand signals this is needed.
- **Phase 4:** NABL-readiness tooling, patient portal, home-collection workflow — for labs actively scaling toward accreditation/growth.

No hard external deadlines currently — pacing is self-directed. Worth revisiting once the pilot lab is live and real usage data starts coming in.
