# 🏥 MedallionHealth

> **A production-grade, multi-tenant healthcare intelligence platform** demonstrating distributed systems, lakehouse architecture, dimensional data modeling, and ML-ready pipelines — built to healthcare compliance standards.

![Architecture](https://img.shields.io/badge/Architecture-Medallion%20Lakehouse-blue)
![Cloud](https://img.shields.io/badge/Cloud-AWS-orange)
![Warehouse](https://img.shields.io/badge/Warehouse-Amazon%20Redshift-red)
![Modeling](https://img.shields.io/badge/Modeling-Kimball%20Star%20Schema-green)
![Security](https://img.shields.io/badge/Security-Row--Level%20Security-purple)
![Status](https://img.shields.io/badge/Status-In%20Development-yellow)

---

## 📐 Platform Overview

MedallionHealth is a cloud-native data platform designed for multi-hospital healthcare networks. It ingests clinical, operational, and financial data across tenants, enforces strict data isolation via row-level security, and exposes a curated analytics layer optimized for BI dashboards, executive reporting, and ML model training.

The platform is named after the **Medallion Architecture** (Bronze → Silver → Gold), with each layer progressively refining raw data into actionable intelligence.

### Core Capabilities

| Capability | Description |
|---|---|
| **Multi-Tenancy** | Hospital-scoped `tenant_id` on every table, enforced via Redshift RLS policies |
| **Dimensional Modeling** | Kimball star schema with SCD Type 2 for slowly changing dimensions |
| **Real-Time + Batch** | Kafka-based streaming for admissions + daily batch for cost/revenue |
| **PHI Protection** | Tokenized PII, role-based access control, audit logging |
| **ML-Ready** | Denormalized gold layer, pre-computed features, readmission risk flags |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
│  EHR Systems │ Billing/ERP │ Bed Sensors │ Workforce Systems        │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  FHIR / HL7 / REST / CSV
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      INGESTION LAYER                                │
│   Apache Kafka (real-time)    │    Airflow DAGs (batch)             │
│   Pydantic schema validation  │    Python connectors                │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │         BRONZE LAYER            │
          │   Raw Parquet on S3             │
          │   Apache Iceberg (time travel)  │
          │   Immutable, append-only        │
          └────────────────┬────────────────┘
                           │  dbt transformations
          ┌────────────────▼────────────────┐
          │         SILVER LAYER            │
          │   Cleaned + deduplicated        │
          │   Multi-tenant partitioning     │
          │   Data quality checks (dbt)     │
          └────────────────┬────────────────┘
                           │  dbt models
          ┌────────────────▼────────────────┐
          │          GOLD LAYER             │
          │   Amazon Redshift               │
          │   Kimball Star Schema           │
          │   RLS-enforced tenant isolation │
          │   ML feature tables             │
          └────────────────┬────────────────┘
                           │
          ┌────────────────▼────────────────┐
          │         CONSUMERS               │
          │  BI Dashboards │ ML Models      │
          │  LLM Assistants│ Audit Reports  │
          └─────────────────────────────────┘
```

---

## 📊 Data Model (Gold Layer — Amazon Redshift)

The analytics warehouse follows **Kimball dimensional modeling** with a star schema design.

### Dimension Tables

| Table | SCD Type | Description |
|---|---|---|
| `dim_hospital` | Type 1 | Hospital master data — type, region, bed capacity, accreditation |
| `dim_patient` | Type 1 | Tokenized patient demographics, coverage type, registration |
| `dim_doctor` | **Type 2** | Versioned doctor records — tracks specialty changes, transfers, promotions |
| `dim_department` | **Type 2** | Versioned department records — tracks restructuring, head-of-dept changes |
| `dim_date` | N/A | Pre-populated date spine with fiscal calendar, holidays, healthcare periods |
| `dim_region` | N/A | AWS region hierarchy with data residency zones (US / EU compliance) |

### Fact Tables

| Table | Grain | Load Pattern | Key Metrics |
|---|---|---|---|
| `fact_admissions` | One row per admission | Kafka stream + daily batch | LOS, readmission flags, ICD-10 codes |
| `fact_bed_occupancy` | Bed × hour snapshot | Hourly streaming | Occupancy rate, ICU utilization |
| `fact_doctor_utilization` | Doctor × day | Daily batch | Utilization rate, appointment completion |
| `fact_revenue` | One row per invoice | Daily batch | Net revenue, payer mix, contractual adjustments |
| `fact_cost` | Department × day | Daily batch + ERP sync | Direct/indirect cost by category |

### Key Design Decisions

- **All monetary values stored in cents** (`BIGINT`) — eliminates floating-point errors in financial calculations
- **SCD Type 2 on `dim_doctor` and `dim_department`** — fact rows point to the dimension version active *at the time of the event*, preserving historical accuracy
- **`dw_row_hash` (MD5)** on SCD2 tables — enables efficient change detection without full column comparisons
- **`DISTKEY (tenant_id)`** on all fact/dimension tables — co-locates tenant data on the same Redshift node for query performance
- **`DISTSTYLE ALL`** on small shared dimensions (`dim_date`, `dim_region`) — broadcast to all nodes to avoid shuffle joins

---

## 🔐 Security Architecture

### Row-Level Security (RLS)

Every tenant-scoped table enforces isolation via a Redshift RLS policy:

```sql
CREATE RLS POLICY tenant_isolation_policy
    USING (tenant_id = current_setting('app.current_tenant_id', true));
```

Hospital analysts can only query their own tenant's data. The policy is enforced at the database engine level — no application-layer filtering required.

### Role Hierarchy

| Role | Access | RLS |
|---|---|---|
| `platform_superuser` | Full admin (Terraform-managed, no human access) | Bypassed |
| `data_engineer_role` | Full read/write on analytics schema | Bypassed |
| `ml_engineer_role` | Read-only across all tenants | Bypassed (audit logged) |
| `hospital_analyst_role` | Read-only | **Enforced** |
| `llm_assistant_role` | Read-only, specific tables only | **Enforced** |
| `audit_role` | Read-only, audit schema only | N/A |

### PHI Handling

- Patient names and doctor names are **tokenized** before landing in Redshift (`first_name_token`, `last_name_token`)
- Full PII history is retained in the Bronze/Silver Iceberg tables with column-level encryption
- `global_patient_id` uses UUID4 format (`gpid-{uuid4}`) — no PII in the key itself

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Ingestion (streaming)** | Apache Kafka |
| **Ingestion (batch)** | Apache Airflow |
| **Schema validation** | Python + Pydantic |
| **Raw storage** | AWS S3 + Apache Iceberg |
| **Transformations** | dbt (Bronze → Silver → Gold) |
| **Analytics warehouse** | Amazon Redshift |
| **Data quality** | dbt tests + Great Expectations |
| **Infrastructure** | Terraform |
| **CI/CD** | GitHub Actions |
| **Containerization** | Docker / Docker Compose |
| **Healthcare data format** | FHIR R4 (simulated) |

---

## 📁 Repository Structure

```
medallionhealth/
├── README.md
├── docs/
│   ├── architecture.md          # This document — detailed design
│   ├── data_model.md            # Full DDL and modeling decisions
│   ├── security.md              # RLS, role hierarchy, PHI handling
│   └── adr/                     # Architecture Decision Records
│       ├── ADR-001-redshift-vs-snowflake.md
│       ├── ADR-002-iceberg-for-bronze-silver.md
│       └── ADR-003-scd2-doctor-department.md
├── ingestion/
│   ├── connectors/              # Source-specific ingestion scripts
│   │   ├── ehr_fhir_connector.py
│   │   ├── billing_erp_connector.py
│   │   └── bed_sensor_connector.py
│   ├── schemas/                 # Pydantic models for validation
│   │   ├── admission_schema.py
│   │   ├── patient_schema.py
│   │   └── revenue_schema.py
│   ├── kafka/                   # Kafka producer/consumer configs
│   └── tests/
├── warehouse/
│   ├── bronze/                  # Raw Iceberg table definitions
│   ├── silver/                  # Cleaned dbt models
│   ├── gold/                    # Analytics dbt models (Redshift)
│   │   ├── dimensions/
│   │   ├── facts/
│   │   └── marts/               # Pre-built analytical views
│   └── seeds/                   # dim_date population, reference data
├── ml_pipelines/                # Phase 2 — readmission prediction
├── security/                    # RLS policies, IAM, audit configs
├── infra/                       # Terraform modules
│   ├── redshift/
│   ├── s3/
│   ├── kafka/
│   └── iam/
├── docker-compose.yml           # Local development environment
├── .github/
│   └── workflows/
│       ├── ci.yml               # dbt test + linting on PR
│       └── deploy.yml           # Terraform plan/apply
└── Makefile                     # Common dev commands
```

---

## 🚀 Getting Started

### Prerequisites

- Docker & Docker Compose
- Python 3.11+
- AWS CLI (configured)
- dbt CLI

### Local Development

```bash
# Clone the repo
git clone https://github.com/yourusername/medallionhealth.git
cd medallionhealth

# Start local services (Kafka, local Redshift-compatible DB)
docker-compose up -d

# Install Python dependencies
pip install -r requirements.txt

# Run dbt models
cd warehouse
dbt deps
dbt seed          # Populate dim_date and reference data
dbt run           # Build all models
dbt test          # Run data quality checks
```

---

## 📈 Roadmap

- [x] Analytics warehouse dimensional model (Redshift DDL)
- [x] RLS security policies and role hierarchy
- [ ] Bronze layer — Iceberg table definitions + S3 layout
- [ ] Ingestion connectors — FHIR EHR, billing ERP (simulated)
- [ ] Kafka streaming pipeline for real-time admissions
- [ ] dbt Silver layer — cleaning, deduplication, tenant partitioning
- [ ] dbt Gold layer — star schema population from Silver
- [ ] Airflow DAGs — orchestration for batch pipelines
- [ ] GitHub Actions CI — dbt test on every PR
- [ ] ML pipeline — 30-day readmission risk model
- [ ] Terraform IaC — full AWS infrastructure as code
- [ ] Grafana dashboard — bed occupancy and doctor utilization

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

*Built as a portfolio project demonstrating production-grade data engineering patterns: multi-tenant architecture, Kimball dimensional modeling, SCD Type 2, streaming + batch ingestion, and healthcare data compliance.*
