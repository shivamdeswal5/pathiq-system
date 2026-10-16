# LabOS — Data Model

**Status:** Accepted  
**Last updated:** 2026-10-16  
**Related:** `docs/architecture/erd.md`, `docs/architecture/system-design.md`, `docs/product/prd.md`  
**Interactive Diagrams:** [dbdiagram.io Live Schema](https://dbdiagram.io/d/PathIQ-6ad1b1b3064322c8696969c9) | [Eraser.io Architecture Canvas](https://app.eraser.io/workspace/hJc9cXMMs4DR3jCNpgR6?origin=share)

This is the schema specification underlying every module in LabOS. Each section below maps directly to one DDD bounded context. Written for PostgreSQL (Supabase / standard Postgres). Visual diagrams and complete DBML are available in [docs/architecture/erd.md](erd.md).

---

## 1. Conventions Applied to Every Table

- **Primary keys:** UUID (`gen_random_uuid()`), not auto-incrementing integers — avoids leaking record counts, integrates cleanly with Supabase Auth UUIDs, and supports offline-first writes.
- **Tenant isolation:** Every tenant-scoped table has a `lab_id` column (FK to `labs.id`), enforced by Postgres Row Level Security (RLS) and repository query scopes.
- **Optimistic concurrency:** Mutable entity tables carry a `version integer default 1` column (via `BaseDomainEntity`). Every successful update increments it.
- **Timestamps:** `created_at timestamptz default now()` on every table; `updated_at timestamptz default now()` on mutable tables.
- **Soft delete:** Medical, patient, and catalog records (`test_panels`, `patients`, `reports`) use a `deleted_at timestamptz null` column to prevent accidental catastrophic data loss.
- **Enum storage:** Stored as compact `smallint` integers in database columns via TypeORM value transformers (`createEnumTransformer`), with descriptive string enums in TypeScript.

---

## 2. `labs` — Tenancy & Staff Profiles (bounded context: `labs`)

### `labs` (Aggregate Root)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| name | text | Diagnostic center name |
| address | text | Physical center address |
| phone_numbers | text[] | Array of contact phone numbers |
| logo_url | text, null | Public URL in Supabase Storage |
| accent_color | varchar(10) | Hex branding color (e.g. `#0f172a`) |
| tagline | text, null | Lab tagline on letterhead |
| footer_note | text, null | e.g. "NOT VALID FOR MEDICO LEGAL PURPOSE" |
| report_language | varchar(10) | Preferred report language (default `'en'`) |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `profiles` (Entity / Member Aggregate)
Links Supabase `auth.users` to a lab tenant and role.
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | Matches Supabase `auth.users.id` |
| lab_id | uuid, FK → labs.id | Tenant association (ON DELETE CASCADE) |
| full_name | text | Staff member name |
| role | smallint | `0` = OWNER, `1` = TECHNICIAN, `2` = PATHOLOGIST |
| signature_url | text, null | Pathologist digital signature in Supabase Storage |
| qualification | text, null | e.g. "MD (Pathology), DCP" |
| created_at / updated_at | timestamptz | |

---

## 3. `panels` — Test Catalog, Templates & Packages (bounded context: `panels`)

Hierarchy: `TestPanel` (Aggregate Root) → `PanelSection` → `PanelParameter`.

### `test_panels` (Aggregate Root)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| lab_id | uuid, FK → labs.id | Tenant scoped |
| name | text | e.g. "Complete Blood Count (CBC)" |
| category | text | e.g. "Haematology", "Biochemistry" |
| price | decimal(10,2) | Base price for the test panel |
| sort_order | integer | Display ordering in catalog |
| deleted_at | timestamptz, null | Soft delete |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `panel_sections` (Child Entity)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| panel_id | uuid, FK → test_panels.id | Cascade on delete |
| name | text | e.g. "Haemogram", "Differential Count" |
| sort_order | integer | Section ordering within panel |

### `panel_parameters` (Child Entity)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| section_id | uuid, FK → panel_sections.id | Cascade on delete |
| name | text | English name, e.g. "Hemoglobin" |
| name_local | text, null | Regional language name, e.g. "हीमोग्लोबिन" |
| unit | text, null | e.g. "g/dL", "mg/dL" |
| input_type | smallint | `0` = NUMBER, `1` = TEXT, `2` = DROPDOWN, `3` = GRID |
| options | jsonb, null | Dropdown choices, e.g. `["Positive", "Negative"]` |
| method | text, null | Test methodology, e.g. "Cyanmethemoglobin" |
| normal_range | jsonb, null | Structured JSONB (`type`, `min`, `max`, `male`, `female`, `text`) |
| sort_order | integer | Parameter ordering within section |

### `panel_templates` (Catalog Seed Aggregate)
System-wide pre-seeded templates available during lab onboarding (not tenant-scoped).
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| name | text | e.g. "Lipid Profile" |
| category | text | Clinical discipline |
| description | text, null | Diagnostic description |
| default_price | decimal(10,2) | Suggested retail price |
| template_data | jsonb | Full template hierarchy: `{ sections: [ { parameters: [...] } ] }` |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `test_packages` (Aggregate Root)
Bundled panels offered at a combined package price.
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| lab_id | uuid, FK → labs.id | Tenant scoped |
| name | text | e.g. "Executive Health Checkup" |
| description | text, null | Package marketing summary |
| price | decimal(10,2) | Discounted bundled price |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `package_panels` (Join Entity)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| package_id | uuid, FK → test_packages.id | Cascade on delete |
| panel_id | uuid, FK → test_panels.id | Cascade on delete |
| sort_order | integer | Ordering of panels inside package |

---

## 4. `reports` — Patients, Test Results & Amendments (bounded context: `reports`)

### `patients` (Aggregate Root)
Separate entity to preserve longitudinal patient test history across repeat visits.
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| lab_id | uuid, FK → labs.id | Tenant scoped |
| patient_number | varchar(50) | Sequential human-readable ID (e.g. `P-2026-001`) |
| name | text | Full patient name |
| age | text, null | e.g. "34" or "34Y 2M" |
| date_of_birth | date, null | Optional exact DOB for precise pediatric/geriatric normal ranges |
| sex | smallint | `0` = MALE, `1` = FEMALE, `2` = OTHER |
| phone | varchar(20), null | Patient mobile for WhatsApp report delivery |
| address | text, null | Physical address |
| deleted_at | timestamptz, null | Soft delete |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `reports` (Aggregate Root)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | `BaseDomainEntity` |
| lab_id | uuid, FK → labs.id | Tenant scoped |
| patient_id | uuid, FK → patients.id | Restrict on delete |
| report_number | varchar(50) | Accession number (unique per lab, e.g. `R-260908-001`) |
| ref_by_doctor_id | uuid, FK → referring_doctors.id, null | Referring physician |
| status | smallint | `0` = DRAFT, `1` = FINALIZED |
| sample_status | smallint | `0` = COLLECTED, `1` = PROCESSING, `2` = COMPLETED, `3` = REJECTED |
| rejection_reason | text, null | e.g. "Hemolyzed specimen", "Insufficient volume" |
| sample_collected_at | timestamptz, null | Turnaround Time (TAT) tracking |
| results_entered_at | timestamptz, null | TAT tracking |
| finalized_at | timestamptz, null | TAT tracking |
| delivered_at | timestamptz, null | TAT tracking |
| remarks | text, null | Pathologist clinical remarks / impression |
| share_token | varchar(64) | Secure random token for online report viewing |
| share_expires_at | timestamptz, null | Expiration timestamp for share link |
| pdf_url | text, null | Generated signed PDF in Supabase Storage |
| deleted_at | timestamptz, null | Soft delete |
| created_at / updated_at / version | | `BaseDomainEntity` |

### `report_panels` (Join Entity)
Composite association between a report and test panels included in the visit.
| Column | Type | Notes |
|---|---|---|
| report_id | uuid, PK, FK → reports.id | Cascade on delete |
| panel_id | uuid, PK, FK → test_panels.id | Restrict on delete |

### `report_values` (Child Entity)
The recorded clinical parameter results.
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| report_id | uuid, FK → reports.id | Cascade on delete |
| parameter_id | uuid, FK → panel_parameters.id | Restrict on delete |
| value | text | Stored as string, evaluated based on input type |
| is_out_of_range | boolean | Computed automatically against `NormalRange` Value Object |
| remarks | text, null | Parameter-specific clinical note |

### `report_amendments` (Audit Child Entity)
Formal audit record created when a finalized report is corrected or amended.
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| report_id | uuid, FK → reports.id | Cascade on delete |
| amended_by | uuid | Staff profile ID |
| reason | text | Medical or typographical reason for amendment |
| previous_data | jsonb | Snapshot of values prior to amendment |
| created_at | timestamptz | Immutable timestamp |

---

## 5. Supporting Bounded Contexts

### `referrals` (bounded context: `referrals`)
- **`referring_doctors`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `name` (text, e.g. "Dr. R. K. Sharma")
  - `clinic_name` (text, null)
  - `phone` (varchar(20), null)
  - `email` (text, null)
  - `commission_type` (smallint: `0` = NONE, `1` = PERCENTAGE, `2` = FLAT)
  - `commission_value` (decimal(10,2), default 0)
  - `notes` (text, null)
- **`doctor_commission_ledger`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `doctor_id` (uuid, FK → referring_doctors.id)
  - `report_id` (uuid, FK → reports.id, null)
  - `amount` (decimal(10,2))
  - `status` (smallint: `0` = PENDING, `1` = SETTLED, `2` = CANCELLED)
  - `settled_at` (timestamptz, null)
  - `notes` (text, null)
- **`outsourced_tests`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `report_id` (uuid, FK → reports.id)
  - `panel_id` (uuid, FK → test_panels.id)
  - `target_lab_name` (text, e.g. "Thyrocare Technologies")
  - `courier_tracking_number` (text, null)
  - `status` (smallint: `0` = DISPATCHED, `1` = IN_TRANSIT, `2` = RESULT_RECEIVED, `3` = CANCELLED)
  - `dispatched_at` / `result_received_at` (timestamptz, null)
  - `cost` (decimal(10,2), default 0)
  - `notes` (text, null)

### `billing` (bounded context: `billing`)
- **`invoices`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `patient_id` (uuid, FK → patients.id)
  - `report_id` (uuid, FK → reports.id, null)
  - `invoice_number` (varchar(50), e.g. `INV-2026-0001`)
  - `subtotal` (decimal(10,2))
  - `discount_amount` (decimal(10,2), default 0)
  - `discount_reason` (text, null)
  - `total_amount` (decimal(10,2))
  - `paid_amount` (decimal(10,2), default 0)
  - `payment_status` (smallint: `0` = UNPAID, `1` = PARTIALLY_PAID, `2` = PAID, `3` = REFUNDED)
  - `payment_method` (smallint: `0` = CASH, `1` = UPI, `2` = CARD, `3` = NET_BANKING, `4` = OTHER)
  - `paid_at` (timestamptz, null)
  - `notes` (text, null)
- **`invoice_items`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `invoice_id` (uuid, FK → invoices.id)
  - `description` (text)
  - `quantity` (integer, default 1)
  - `unit_price` (decimal(10,2))
  - `total_price` (decimal(10,2))
- **`expenses`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `category` (smallint: `0` = REAGENTS, `1` = CONSUMABLES, `2` = EQUIPMENT, `3` = RENT, `4` = UTILITIES, `5` = SALARIES, `6` = MAINTENANCE, `7` = OTHER)
  - `description` (text)
  - `amount` (decimal(10,2))
  - `expense_date` (date)
  - `vendor` (text, null)
  - `invoice_number` (text, null)
  - `paid_via` (smallint: `0` = CASH, `1` = UPI, `2` = CARD, `3` = BANK_TRANSFER)

### `notifications` (bounded context: `notifications`)
- **`notification_logs`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `recipient_type` (smallint: `0` = PATIENT, `1` = DOCTOR, `2` = STAFF)
  - `recipient_name` (text)
  - `destination` (text, phone or email)
  - `channel` (smallint: `0` = WHATSAPP, `1` = SMS, `2` = EMAIL)
  - `notification_type` (smallint: `0` = REPORT_READY, `1` = CRITICAL_ALERT, `2` = PAYMENT_RECEIPT, `3` = CUSTOM)
  - `status` (smallint: `0` = PENDING, `1` = SENT, `2` = DELIVERED, `3` = FAILED)
  - `message_content` (text)
  - `payload` (jsonb, template values and report IDs)
  - `provider` (text, e.g. "whatsapp-cloud", "mock-provider")
  - `provider_message_id` (text, null)
  - `sent_at` (timestamptz, null)
  - `failure_reason` (text, null)

### `collections` (bounded context: `collections`)
- **`collection_requests`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `lab_id` (uuid, FK → labs.id)
  - `patient_id` (uuid, FK → patients.id, null)
  - `request_number` (varchar(50), e.g. `COL-2026-0001`)
  - `patient_name` (text)
  - `patient_phone` (varchar(20))
  - `patient_age` (text, null)
  - `patient_sex` (smallint, null)
  - `address` (text)
  - `preferred_date` (date)
  - `time_slot` (varchar(50), e.g. "07:00 AM - 09:00 AM")
  - `is_fasting_required` (boolean, default false)
  - `test_names` (text[], null)
  - `status` (smallint: `0` = REQUESTED, `1` = ASSIGNED, `2` = IN_TRANSIT, `3` = SAMPLE_COLLECTED, `4` = DELIVERED_TO_LAB, `5` = CANCELLED)
  - `assigned_phlebotomist_id` (uuid, null)
  - `assigned_phlebotomist_name` (text, null)
  - `collected_at` / `delivered_to_lab_at` (timestamptz, null)
  - `cancellation_reason` (text, null)
  - `report_id` (uuid, null)
- **`collection_samples`**:
  - `id` (uuid, PK, BaseDomainEntity)
  - `collection_id` (uuid, FK → collection_requests.id)
  - `tube_type` (smallint: `0` = EDTA, `1` = SERUM, `2` = FLUORIDE, `3` = CITRATE, `4` = URINE, `5` = OTHER)
  - `barcode` (varchar(100), optional pre-printed barcode scanned at collection)

### `websockets` (bounded context: `websockets`)
- **Architecture**: Ephemeral, state-free in-memory real-time router.
- **Database Tables**: **None** (zero persistent DB footprint).
- **Transient State**: Socket client handles, room subscriptions (`lab:{labId}`, `lab:{labId}:doctors`, `user:{userId}`), and CloudEvents message framing are managed completely in-memory via the Socket.IO adapter.

---

## 6. Entity Relationship Summary

```
labs ──< profiles
labs ──< test_panels ──< panel_sections ──< panel_parameters
labs ──< test_packages ──< package_panels >── test_panels
labs ──< patients ──< reports ──< report_values >── panel_parameters
│         │               │  ├──< report_panels >── test_panels
│         │               │  ├──< report_amendments
│         │               │  ├──< outsourced_tests
│         │               │  └──< invoices ──< invoice_items
│         │               │
│         └──< collection_requests ──< collection_samples
│
labs ──< referring_doctors ──< doctor_commission_ledger >── reports
labs ──< expenses
labs ──< notification_logs
```

---

## 7. Row Level Security (RLS) Multi-Tenant Pattern

```sql
alter table reports enable row level security;

create policy "tenant_isolation_policy" on reports
  using (lab_id = (select lab_id from profiles where id = auth.uid()));
```
Applied uniformly to every table carrying `lab_id` to guarantee tenant data isolation at the PostgreSQL storage engine level.

---

## 8. Single-Table Modular Migrations Inventory (22 Migrations)

Every table has a dedicated, sequentially timestamped migration matching the `residency-backend` standard:

| Timestamp Prefix | Module | Migration Name | Table Created |
|---|---|---|---|
| `1710000001001` | `labs` | `create-labs-table` | `labs` |
| `1710000001002` | `labs` | `create-profiles-table` | `profiles` |
| `1710000002001` | `panels` | `create-panel-templates-table` | `panel_templates` |
| `1710000002002` | `panels` | `create-test-panels-table` | `test_panels` |
| `1710000002003` | `panels` | `create-panel-sections-table` | `panel_sections` |
| `1710000002004` | `panels` | `create-panel-parameters-table` | `panel_parameters` |
| `1710000002005` | `panels` | `create-test-packages-table` | `test_packages` |
| `1710000002006` | `panels` | `create-package-panels-table` | `package_panels` |
| `1710000003001` | `reports` | `create-patients-table` | `patients` |
| `1710000003002` | `reports` | `create-reports-table` | `reports` |
| `1710000003003` | `reports` | `create-report-panels-table` | `report_panels` |
| `1710000003004` | `reports` | `create-report-values-table` | `report_values` |
| `1710000003005` | `reports` | `create-report-amendments-table` | `report_amendments` |
| `1710000004001` | `referrals` | `create-referring-doctors-table` | `referring_doctors` |
| `1710000004002` | `referrals` | `create-doctor-commission-ledger-table` | `doctor_commission_ledger` |
| `1710000004003` | `referrals` | `create-outsourced-tests-table` | `outsourced_tests` |
| `1710000005001` | `billing` | `create-invoices-table` | `invoices` |
| `1710000005002` | `billing` | `create-invoice-items-table` | `invoice_items` |
| `1710000005003` | `billing` | `create-expenses-table` | `expenses` |
| `1710000006001` | `notifications` | `create-notification-logs-table` | `notification_logs` |
| `1710000007001` | `collections` | `create-collection-requests-table` | `collection_requests` |
| `1710000007002` | `collections` | `create-collection-samples-table` | `collection_samples` |

