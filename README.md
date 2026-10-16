# LabOS — Diagnostic Laboratory Operating System

> An enterprise-grade, multi-tenant operating system designed for India's independent diagnostic laboratories, delivering clinical precision, real-time phlebotomy workflows, automated B2B GST Rule 46 billing, and NABL/ISO 15189 compliance.

---

## 🏛️ Ecosystem Architecture & Submodules

LabOS is organized as an umbrella platform coordinating 4 dedicated applications:

| Submodule / Service | Repository | Tech Stack | Role & Purpose |
| :--- | :--- | :--- | :--- |
| **`labOS-service`** | [github.com/shivamdeswal5/labOS-service](https://github.com/shivamdeswal5/labOS-service) | NestJS, TypeORM, PostgreSQL | Core Modular Monolith API across 8 DDD bounded contexts. |
| **`labOS-app`** | [github.com/shivamdeswal5/labOS-app](https://github.com/shivamdeswal5/labOS-app) | Next.js (App Router), Tailwind, shadcn/ui | Laboratory Workstation (accessions, result entry, patient portal). |
| **`labOS-admin`** | [github.com/shivamdeswal5/pathiq-admin](https://github.com/shivamdeswal5/pathiq-admin) | Next.js, TanStack Query | Super Admin Operations Cockpit (onboarding, billing, UTR verification). |
| **`labOS-web`** | [github.com/shivamdeswal5/pathiq](https://github.com/shivamdeswal5/pathiq) | Next.js, Vanilla CSS | Marketing, pricing, and public conversion website. |

---

## 📐 Master Architecture & System Documentation

- **[System Architecture](docs/architecture/system-design.md)**: Hexagonal architecture, CQRS vertical slices, CloudEvents real-time gateway.
- **[Data Model & Schema](docs/architecture/data-model.md)**: Comprehensive relational schema, integer enum transformers, Row Level Security (RLS).
- **[Entity Relationship Diagram (ERD)](docs/architecture/erd.md)**: Visual Mermaid diagram and code-verified DBML ([dbdiagram.io Live](https://dbdiagram.io/d/PathIQ-6ad1b1b3064322c8696969c9) · [Eraser.io Live](https://app.eraser.io/workspace/hJc9cXMMs4DR3jCNpgR6?origin=share)).
- **[Billing & Payments Architecture](docs/architecture/billing-and-payments.md)**: JIT gateway settlement, duplicate hold policy, proration engine.
- **[Product Requirements Document (PRD)](docs/product/prd.md)**: Target market, clinical standards, and product roadmap.
- **[Coding Standards & Conventions](docs/conventions/coding-standards.md)**: Frontend and backend engineering conventions.

---

## 🚀 Quickstart & Local Orchestration

### Prerequisites
- Node.js 20+ & npm 10+
- Docker & Docker Compose
- Supabase account / local PostgreSQL 15+

### Clone with Submodules
```bash
git clone --recurse-submodules https://github.com/shivamdeswal5/labOS-platform.git
cd labOS-platform
```

### Start Database & Backend Services
```bash
docker compose up -d
```
