# LabOS — Entity Relationship Diagram (ERD) & Database Specification

**Status:** Accepted & Production-Verified  
**Last Updated:** 2026-10-16  
**Governing Standard:** `docs/architecture/system-design.md`, `docs/architecture/data-model.md`, `labOS-service` TypeORM Entities & Migrations  
**Target Database:** PostgreSQL 15+ (Supabase / Container Postgres)  
**Total Entities:** 26 Tables across 8 Domain-Driven Design (DDD) Bounded Contexts  

### 🌐 Interactive Live Diagram Workspaces
- **dbdiagram.io Live Schema (Interactive & Draggable):** [PathIQ on dbdiagram.io](https://dbdiagram.io/d/PathIQ-6ad1b1b3064322c8696969c9)
- **Eraser.io Architecture Workspace:** [PathIQ on Eraser.io](https://app.eraser.io/workspace/hJc9cXMMs4DR3jCNpgR6?origin=share)

---

## 1. Visual Entity Relationship Diagram (Mermaid)

```mermaid
erDiagram
    %% ==========================================
    %% 1. LABS & PROFILES (Tenancy)
    %% ==========================================
    labs ||--o{ profiles : "has members"
    labs ||--o{ test_panels : "owns"
    labs ||--o{ test_packages : "offers"
    labs ||--o{ patients : "registers"
    labs ||--o{ reports : "accessions"
    labs ||--o{ referring_doctors : "tracks"
    labs ||--o{ doctor_commission_ledger : "accrues"
    labs ||--o{ outsourced_tests : "dispatches"
    labs ||--o{ invoices : "bills"
    labs ||--o{ expenses : "incurs"
    labs ||--o{ collection_requests : "dispatches"
    labs ||--o{ notification_logs : "audits"
    labs ||--|| tenant_subscriptions : "subscribes"
    labs ||--o{ platform_payments : "transacts"
    labs ||--o{ subscription_invoices : "billed to"

    %% ==========================================
    %% 2. PANELS & TEST CATALOG
    %% ==========================================
    test_panels ||--o{ panel_sections : "divided into"
    panel_sections ||--o{ panel_parameters : "defines"
    test_packages ||--o{ package_panels : "bundles"
    test_panels ||--o{ package_panels : "included in"

    %% ==========================================
    %% 3. PATIENTS & CLINICAL REPORTS
    %% ==========================================
    patients ||--o{ reports : "undergoes"
    reports ||--o{ report_panels : "contains"
    test_panels ||--o{ report_panels : "tested in"
    reports ||--o{ report_values : "records"
    panel_parameters ||--o{ report_values : "measured for"
    reports ||--o{ report_amendments : "corrected by"
    profiles ||--o{ report_amendments : "authorized by"

    %% ==========================================
    %% 4. REFERRALS & OUTSOURCING
    %% ==========================================
    referring_doctors ||--o{ reports : "refers"
    referring_doctors ||--o{ doctor_commission_ledger : "credited to"
    reports |o--o{ doctor_commission_ledger : "generates"
    reports ||--o{ outsourced_tests : "outsources"

    %% ==========================================
    %% 5. BILLING & EXPENSES
    %% ==========================================
    patients ||--o{ invoices : "billed to"
    reports |o--o{ invoices : "associated with"
    invoices ||--o{ invoice_items : "itemizes"

    %% ==========================================
    %% 6. HOME COLLECTIONS
    %% ==========================================
    patients |o--o{ collection_requests : "requests"
    reports |o--o{ collection_requests : "fulfills"
    collection_requests ||--o{ collection_samples : "gathers"

    %% ==========================================
    %% 7. PLATFORM BILLING (SaaS)
    %% ==========================================
    platform_plans ||--o{ tenant_subscriptions : "tier"
    platform_plans |o--o{ tenant_subscriptions : "scheduled transition"
    tenant_subscriptions |o--o{ platform_payments : "renews"
    platform_payments ||--|| subscription_invoices : "settles"

    %% ==========================================
    %% ENTITY DEFINITIONS & CORE ATTRIBUTES
    %% ==========================================

    labs {
        uuid id PK
        text name
        text address
        text_array phone_numbers
        varchar accent_color
        text tagline
        text footer_note
        varchar report_language
        jsonb print_settings
        timestamptz created_at
        integer version
    }

    profiles {
        uuid id PK "auth.users.id"
        uuid lab_id FK
        text full_name
        smallint role "0=OWNER 1=TECH 2=PATH"
        text signature_url
        text qualification
        timestamptz created_at
    }

    test_panels {
        uuid id PK
        uuid lab_id FK
        text name
        text category
        decimal price
        integer sort_order
        timestamptz deleted_at
    }

    panel_sections {
        uuid id PK
        uuid panel_id FK
        text name
        integer sort_order
    }

    panel_parameters {
        uuid id PK
        uuid section_id FK
        text name
        text name_local
        text unit
        smallint input_type "0=NUM 1=TXT 2=DROP 3=GRID"
        jsonb options
        text method
        jsonb normal_range
        integer sort_order
    }

    patients {
        uuid id PK
        uuid lab_id FK
        varchar patient_number UK
        text name
        text age
        date date_of_birth
        smallint sex "0=M 1=F 2=O"
        varchar phone
        text address
        timestamptz deleted_at
    }

    reports {
        uuid id PK
        uuid lab_id FK
        uuid patient_id FK
        uuid ref_by_doctor_id FK
        varchar report_number UK
        smallint status "0=DRAFT 1=FINALIZED"
        smallint sample_status "0=COLL 1=PROC 2=DONE 3=REJ"
        varchar share_token UK
        text pdf_url
        timestamptz sample_collected_at
        timestamptz results_entered_at
        timestamptz finalized_at
    }

    report_values {
        uuid id PK
        uuid report_id FK
        uuid parameter_id FK
        text value
        boolean is_out_of_range
        text remarks
    }

    invoices {
        uuid id PK
        uuid lab_id FK
        uuid patient_id FK
        uuid report_id FK
        varchar invoice_number UK
        decimal subtotal
        decimal discount
        decimal total_amount
        decimal paid_amount
        smallint payment_status "0=UNPAID 1=PARTIAL 2=PAID 3=REF"
        smallint payment_method "0=CASH 1=UPI 2=CARD 3=NEFT"
        timestamptz paid_at
    }

    tenant_subscriptions {
        uuid id PK
        uuid lab_id UK_FK
        uuid plan_id FK
        varchar status "TRIALING ACTIVE PAST_DUE LOCKED"
        varchar billing_cycle "MONTHLY YEARLY"
        timestamptz current_period_end
        uuid scheduled_plan_id FK
        varchar customer_gstin
    }

    platform_payments {
        uuid id PK
        uuid lab_id FK
        uuid subscription_id FK
        varchar provider "RAZORPAY STRIPE MANUAL_UTR"
        varchar provider_order_id UK
        integer amount_in_paise
        varchar status "PENDING SUCCESS FAILED EXPIRED"
        varchar fulfillment_status "FULFILLED DUPLICATE_UNFULFILLED"
        varchar idempotency_key UK
    }

    subscription_invoices {
        uuid id PK
        varchar invoice_number UK
        uuid lab_id FK
        uuid payment_id UK_FK
        numeric amount_inr
        numeric tax_amount_inr
        numeric subtotal_inr
        varchar customer_gstin
        text pdf_url
        varchar status "PAID"
    }
```

---

## 2. Universal DBML Specification (for dbdiagram.io / Eraser.io)

You can explore and edit the live interactive diagram directly at:
- **[dbdiagram.io Live Workspace](https://dbdiagram.io/d/PathIQ-6ad1b1b3064322c8696969c9)**
- **[Eraser.io Architecture Canvas](https://app.eraser.io/workspace/hJc9cXMMs4DR3jCNpgR6?origin=share)**

Alternatively, copy and paste the DBML schema below directly into any DBML-compatible visual modeling tool:

```dbml
// ============================================================================
// LabOS Master Production ERD (100% Code-Verified)
// ============================================================================

Project LabOS {
  database_type: 'PostgreSQL'
  Note: 'LabOS - Multi-tenant diagnostic laboratory operating system'
}

// ----------------------------------------------------------------------------
// 1. TENANCY & PROFILES
// ----------------------------------------------------------------------------

Table labs {
  id uuid [pk]
  name text [not null]
  address text [not null]
  phone_numbers text[] [default: '{}']
  logo_url text
  accent_color varchar(10) [default: '#0f172a']
  tagline text
  footer_note text [default: 'NOT VALID FOR MEDICO LEGAL PURPOSE']
  report_language varchar(10) [default: 'en']
  print_settings jsonb [default: '{}']
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]
}

Table profiles {
  id uuid [pk, note: 'Foreign Key matching Supabase auth.users.id']
  lab_id uuid [not null, ref: > labs.id]
  full_name text [not null]
  role smallint [default: 1, note: '0=OWNER, 1=TECHNICIAN, 2=PATHOLOGIST']
  signature_url text
  qualification text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
}

// ----------------------------------------------------------------------------
// 2. PANELS & TEST CATALOG
// ----------------------------------------------------------------------------

Table test_panels {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  name text [not null]
  category text [not null]
  price decimal(10,2) [default: 0]
  sort_order integer [default: 0]
  deleted_at timestamptz
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, category)
    (lab_id, name)
  }
}

Table panel_sections {
  id uuid [pk]
  panel_id uuid [not null, ref: > test_panels.id]
  name text [not null]
  sort_order integer [default: 0]
}

Table panel_parameters {
  id uuid [pk]
  section_id uuid [not null, ref: > panel_sections.id]
  name text [not null]
  name_local text
  unit text
  input_type smallint [default: 1, note: '0=NUMBER, 1=TEXT, 2=DROPDOWN, 3=GRID']
  options jsonb
  method text
  normal_range jsonb
  sort_order integer [default: 0]
}

Table panel_templates {
  id uuid [pk]
  name text [unique, not null]
  category text [not null]
  description text
  default_price decimal(10,2) [default: 0]
  template_data jsonb [not null]
}

Table test_packages {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  name text [not null]
  description text
  price decimal(10,2) [default: 0]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]
}

Table package_panels {
  id uuid [pk]
  package_id uuid [not null, ref: > test_packages.id]
  panel_id uuid [not null, ref: > test_panels.id]
  sort_order integer [default: 0]
}

// ----------------------------------------------------------------------------
// 3. PATIENTS & CLINICAL REPORTS
// ----------------------------------------------------------------------------

Table patients {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  patient_number varchar(50) [not null]
  name text [not null]
  age text
  date_of_birth date
  sex smallint [default: 0, note: '0=MALE, 1=FEMALE, 2=OTHER']
  phone varchar(20)
  address text
  deleted_at timestamptz
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, patient_number) [unique]
    (lab_id, phone)
    (lab_id, name)
  }
}

Table reports {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  patient_id uuid [not null, ref: > patients.id]
  report_number varchar(50) [not null]
  ref_by_doctor_id uuid [ref: > referring_doctors.id]
  status smallint [default: 0, note: '0=DRAFT, 1=FINALIZED']
  sample_status smallint [default: 0, note: '0=COLLECTED, 1=PROCESSING, 2=COMPLETED, 3=REJECTED']
  rejection_reason text
  sample_collected_at timestamptz
  results_entered_at timestamptz
  finalized_at timestamptz
  delivered_at timestamptz
  remarks text
  share_token varchar(64) [unique, not null]
  share_expires_at timestamptz
  pdf_url text
  deleted_at timestamptz
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, report_number) [unique]
    (lab_id, patient_id)
    (lab_id, status)
    share_token [unique]
  }
}

Table report_panels {
  report_id uuid [not null, ref: > reports.id]
  panel_id uuid [not null, ref: > test_panels.id]

  Indexes {
    (report_id, panel_id) [pk]
  }
}

Table report_values {
  id uuid [pk]
  report_id uuid [not null, ref: > reports.id]
  parameter_id uuid [not null, ref: > panel_parameters.id]
  value text [not null]
  is_out_of_range boolean [default: false]
  remarks text

  Indexes {
    (report_id, parameter_id) [unique]
  }
}

Table report_amendments {
  id uuid [pk]
  report_id uuid [not null, ref: > reports.id]
  amended_by uuid [not null, ref: > profiles.id]
  reason text [not null]
  previous_data jsonb [not null]
  created_at timestamptz [default: `now()`]
}

// ----------------------------------------------------------------------------
// 4. REFERRALS & OUTSOURCING
// ----------------------------------------------------------------------------

Table referring_doctors {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  name text [not null]
  clinic text
  phone varchar(20)
  email varchar(255)
  commission_type smallint [default: 0, note: '0=NONE, 1=PERCENTAGE, 2=FLAT']
  commission_value decimal(10,2) [default: 0]
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, name)
    (lab_id, phone)
  }
}

Table doctor_commission_ledger {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  doctor_id uuid [not null, ref: > referring_doctors.id]
  report_id uuid [ref: > reports.id]
  amount decimal(10,2) [default: 0]
  status smallint [default: 0, note: '0=PENDING, 1=SETTLED, 2=CANCELLED']
  settled_at timestamptz
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, doctor_id)
    (lab_id, status)
    (lab_id, report_id)
  }
}

Table outsourced_tests {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  report_id uuid [not null, ref: > reports.id]
  test_name text [not null]
  reference_lab_name text [not null]
  status smallint [default: 0, note: '0=DISPATCHED, 1=IN_TRANSIT, 2=RESULT_RECEIVED, 3=CANCELLED']
  cost decimal(10,2)
  sent_at timestamptz
  received_at timestamptz
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, report_id)
    (lab_id, status)
    (lab_id, reference_lab_name)
  }
}

// ----------------------------------------------------------------------------
// 5. BILLING & EXPENSES
// ----------------------------------------------------------------------------

Table invoices {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  patient_id uuid [not null, ref: > patients.id]
  report_id uuid [ref: > reports.id]
  invoice_number varchar(50) [not null]
  subtotal decimal(10,2) [default: 0]
  discount decimal(10,2) [default: 0]
  total_amount decimal(10,2) [default: 0]
  paid_amount decimal(10,2) [default: 0]
  payment_status smallint [default: 0, note: '0=UNPAID, 1=PARTIALLY_PAID, 2=PAID, 3=REFUNDED']
  payment_method smallint [note: '0=CASH, 1=UPI, 2=CARD, 3=NET_BANKING, 4=OTHER']
  paid_at timestamptz
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, invoice_number) [unique]
    (lab_id, patient_id)
    (lab_id, report_id)
    (lab_id, payment_status)
  }
}

Table invoice_items {
  id uuid [pk]
  invoice_id uuid [not null, ref: > invoices.id]
  description text [not null]
  unit_price decimal(10,2) [default: 0]
  quantity integer [default: 1]
  total decimal(10,2) [default: 0]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    invoice_id
  }
}

Table expenses {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  category smallint [not null, note: '0=REAGENTS, 1=CONSUMABLES, 2=EQUIPMENT, 3=RENT, 4=UTILITIES, 5=SALARIES, 6=MAINTENANCE, 7=OTHER']
  title text [not null]
  amount decimal(10,2) [default: 0]
  expense_date date [not null]
  payment_method smallint [default: 0, note: '0=CASH, 1=UPI, 2=CARD, 3=BANK_TRANSFER']
  vendor text
  vendor_invoice_number varchar(100)
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, expense_date)
    (lab_id, category)
  }
}

// ----------------------------------------------------------------------------
// 6. PHLEBOTOMY & HOME COLLECTIONS
// ----------------------------------------------------------------------------

Table collection_requests {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  request_number varchar(50) [not null]
  patient_id uuid [ref: > patients.id]
  patient_name text [not null]
  patient_phone varchar(20) [not null]
  patient_age text
  patient_sex smallint [default: 0]
  address text [not null]
  preferred_date date [not null]
  time_slot varchar(50) [not null]
  status smallint [default: 0, note: '0=REQUESTED, 1=ASSIGNED, 2=IN_TRANSIT, 3=SAMPLE_COLLECTED, 4=DELIVERED_TO_LAB, 5=CANCELLED']
  assigned_phlebotomist_id uuid
  assigned_phlebotomist_name text
  is_fasting_required boolean [default: false]
  test_names jsonb [default: '[]']
  special_instructions text
  cancellation_reason text
  collected_at timestamptz
  delivered_to_lab_at timestamptz
  report_id uuid [ref: > reports.id]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, request_number) [unique]
    (lab_id, preferred_date)
    (lab_id, status)
    (lab_id, assigned_phlebotomist_id)
  }
}

Table collection_samples {
  id uuid [pk]
  collection_request_id uuid [not null, ref: > collection_requests.id]
  tube_type varchar(50) [default: 'EDTA', note: 'EDTA, SERUM, FLUORIDE, CITRATE, URINE, OTHER']
  barcode varchar(100) [not null]
  notes text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    collection_request_id
    barcode
  }
}

// ----------------------------------------------------------------------------
// 7. PLATFORM BILLING & SAAS SUBSCRIPTIONS
// ----------------------------------------------------------------------------

Table platform_plans {
  id uuid [pk]
  slug varchar(50) [unique, not null, note: 'STARTER, GROWTH, SCALE']
  name varchar(100) [not null]
  description text
  monthly_price_inr integer [not null]
  yearly_price_inr integer [not null]
  feature_entitlements jsonb [default: '{}']
  is_active boolean [default: true]
  is_public boolean [default: true]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]
}

Table tenant_subscriptions {
  id uuid [pk]
  lab_id uuid [unique, not null, ref: > labs.id]
  plan_id uuid [not null, ref: > platform_plans.id]
  status varchar(30) [default: 'TRIALING', note: 'TRIALING, ACTIVE, PAST_DUE, LOCKED, CANCELLED']
  billing_cycle varchar(20) [default: 'MONTHLY', note: 'MONTHLY, YEARLY']
  trial_start timestamptz [not null]
  trial_end timestamptz [not null]
  current_period_start timestamptz
  current_period_end timestamptz
  grace_period_end timestamptz
  custom_entitlement_overrides jsonb [default: '{}']
  canceled_at timestamptz
  scheduled_plan_id uuid [ref: > platform_plans.id]
  scheduled_billing_cycle varchar(20)
  scheduled_at timestamptz
  customer_gstin varchar(15)
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]
}

Table platform_payments {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  subscription_id uuid [ref: > tenant_subscriptions.id]
  provider varchar(30) [not null, note: 'RAZORPAY, STRIPE, MANUAL_UTR']
  provider_order_id varchar(100)
  provider_payment_id varchar(100)
  amount_in_paise integer [not null]
  currency varchar(10) [default: 'INR']
  status varchar(30) [default: 'PENDING', note: 'PENDING, SUCCESS, FAILED, EXPIRED']
  payment_method varchar(50)
  payment_details jsonb [default: '{}']
  idempotency_key varchar(100) [unique]
  fulfillment_status varchar(30) [note: 'FULFILLED, DUPLICATE_UNFULFILLED']
  failure_reason text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]
}

Table subscription_invoices {
  id uuid [pk]
  invoice_number varchar(50) [unique, not null]
  lab_id uuid [not null, ref: > labs.id]
  payment_id uuid [unique, not null, ref: > platform_payments.id]
  amount_inr numeric(10,2) [not null]
  tax_amount_inr numeric(10,2) [default: 0]
  subtotal_inr numeric(10,2) [not null]
  customer_gstin varchar(20)
  billing_period_start timestamptz [not null]
  billing_period_end timestamptz [not null]
  pdf_url text
  status varchar(30) [default: 'PAID']
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
}

Table webhook_events {
  id uuid [pk]
  event_id varchar(100) [unique, not null]
  provider varchar(30) [not null]
  event_type varchar(100) [not null]
  payload jsonb [not null]
  status varchar(30) [default: 'PENDING']
  error_message text
  processed_at timestamptz
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
}

// ----------------------------------------------------------------------------
// 8. NOTIFICATIONS & AUDIT LOGS
// ----------------------------------------------------------------------------

Table notification_logs {
  id uuid [pk]
  lab_id uuid [not null, ref: > labs.id]
  recipient_type smallint [default: 0, note: '0=PATIENT, 1=DOCTOR, 2=STAFF']
  recipient_name text [not null]
  destination varchar(255) [not null]
  channel smallint [default: 0, note: '0=WHATSAPP, 1=SMS, 2=EMAIL']
  notification_type smallint [default: 0, note: '0=REPORT_READY, 1=CRITICAL_ALERT, 2=PAYMENT_RECEIPT, 3=CUSTOM']
  status smallint [default: 0, note: '0=PENDING, 1=SENT, 2=DELIVERED, 3=FAILED']
  message_content text [not null]
  payload jsonb
  provider varchar(50) [default: 'mock']
  provider_message_id varchar(255)
  sent_at timestamptz
  delivered_at timestamptz
  failure_reason text
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]
  version integer [default: 1]

  Indexes {
    (lab_id, created_at)
    (lab_id, status)
    (lab_id, destination)
  }
}
```

---

## 3. Database Invariants & Architecture Rules

1. **UUID Primary Keys**: Every entity extends `BaseDomainEntity` utilizing UUIDs generated via `gen_random_uuid()` or TypeORM `@PrimaryGeneratedColumn('uuid')`.
2. **Tenant Isolation via `lab_id`**: Every tenant-scoped entity carries a foreign key `lab_id UUID REFERENCES labs(id)`. This is enforced at the database level using PostgreSQL Row Level Security (RLS) policies.
3. **Integer Enum Mappers**: Domain enums (`RoleEnum`, `ReportStatusEnum`, `SampleStatusEnum`, `SexEnum`, `PaymentStatusEnum`, `CommissionTypeEnum`, `ExpenseCategoryEnum`, `NotificationChannelEnum`) are stored as compact `smallint` values in PostgreSQL via TypeORM `createEnumTransformer` value transformers.
4. **Optimistic Concurrency Control**: All stateful aggregate roots include an auto-incrementing `version integer DEFAULT 1` column managed by TypeORM `@VersionColumn()`.
5. **Auditable Soft Deletes**: Clinical entities (`patients`, `reports`, `test_panels`) feature `@DeleteDateColumn({ name: 'deleted_at' })` to guarantee medical audit history retention and avoid accidental data loss.
