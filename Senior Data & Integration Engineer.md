Senior Data & Integration Engineer | EWX Platform Rebuild — Discovery Phase (Phase 1) |

- own the data and integration sections
- analyze client-provided documentation
- research five external integrations
- and design detailed data models and ETL architectures
- deliver technically coherent, production-ready specifications
- - Producing the **data-focused half of the TRD**, while the Lead Engineer handles platform architecture.

Multi-tenant energy platform **B2B dashboard platform** | Client: EWX Power

- for energy companies managing Solar, Generation, Storage, Load
- project emphasizes time-series telemetry ingestion
- cascading statistics computation
- handling inconsistent third-party API data

## 1. Executive Summary (5–10 sentences)

This document defines the role, scope, and expectations for a **Senior Data & Integration Engineer** engaged in the **EWX Platform Rebuild — Discovery Phase (Phase 1)**. The role focuses on owning the **data and integration sections** of the Technical Requirements Document (TRD) for a complex, multi-tenant energy platform. The engineer is expected to independently analyze extensive client-provided documentation, research five external integrations, and design detailed data models and ETL architectures. The project emphasizes time-series telemetry ingestion, cascading statistics computation, and handling inconsistent third-party API data. The engagement is short-term (3 weeks), high-intensity (75% dedication), and documentation-focused rather than implementation-focused. Success depends on the engineer’s ability to work autonomously in parallel with the Lead Engineer and deliver technically coherent, production-ready specifications.

---

## 2. Detailed Section-by-Section Summary

### **Document Header / Engagement Overview**

- **Client:** EWX Power
- **Engagement:** Platform Rebuild — Discovery Phase (Phase 1)
- **Source:** Heaptrace
- **Duration:** 3 weeks
- **Dedication:** 75%
- **Start:** Pending MSA execution
- **Conditional Role:** Required only if client selects a 3-week proposal (P3 or P4).

---

### **Role Summary**

- The role owns the **data and integration portions** of the TRD for an enterprise energy platform rebuild.
- Responsibilities include:
  - Deep review of ~4,300 lines of client documentation (field specs, ETL processes, data models).
  - Researching **five external integrations**.
  - Producing the **data-focused half of the TRD**, while the Lead Engineer handles platform architecture.

- The role operates as a **parallel workstream**, requiring independence and minimal guidance.
- Both TRD halves are merged in **Week 3**, requiring the data sections to be technically complete and standalone.

---

### **Project Context**

- Multi-tenant **B2B dashboard platform** for energy companies managing:
  - Solar
  - Generation
  - Storage
  - Load

- **Complex ETL pipeline**:
  - Sol-Ark inverter telemetry ingested every **5–15 minutes**.
  - Data processed through **three cascading statistics levels**:
    - Component → Location → Company

- **Time-series processing requirements**:
  - Linear interpolation (imputation)
  - Standard deviation calculations

- **External integrations (5 total)** with inconsistent documentation quality.
- **Sol-Ark API challenge**:
  - Inconsistent field population across inverter models (12k, 15k, 30k, 60k), requiring model-specific mappings.

- Client-provided documentation includes:
  - 1,500+ lines of ETL documentation
  - 2,800+ lines of field-level specifications

---

### **Responsibilities**

#### **Week 1: Research & Documentation Review**

- Deep-read ETL requirements and field-level specifications.
- Research APIs for:
  - Sol-Ark
  - EWX IoT
  - Weather
  - Solar Forecast
  - Google Maps

- Review data models, sample data, and multi-inverter scenarios.
- Document gaps, ambiguities, and clarification questions.
- Validate Visio data flow diagrams against written requirements.

---

#### **Weeks 2–3: TRD Data Sections**

- **Data Model Design**
  - Multi-tenant schema design (MySQL or recommended alternative)
  - Entity relationships
  - Indexing strategy

- **ETL Pipeline Architecture**
  - Event-driven ingestion (Sol-Ark, EWX IoT)
  - Cascading statistics computation
  - FIFO guarantees per location
  - Error handling and retry logic

- **Integration Specifications**
  - API contracts
  - Authentication
  - Rate limiting
  - Data mapping
  - Failure modes (all 5 integrations)

- **Time-Series Processing**
  - Interpolation approach
  - Standard deviation calculations
  - Data retention and partitioning

- **Notification System**
  - Alert rules
  - Delivery channels
  - Tenant-scoped architecture

---

#### **Week 3: Cross-Review & Merge**

- Cross-review Lead Engineer’s platform sections for data consistency.
- Participate in weekly syncs on cross-cutting concerns (e.g., caching).
- Revise TRD sections based on feedback prior to final assembly.

---

### **Required Experience**

- 4+ years in data or backend engineering with heavy data focus.
- Strong ETL/data pipeline design experience.
- Relational database design (MySQL, PostgreSQL, or equivalent).
- Third-party API integration experience.
- Time-series data understanding (storage, querying, aggregation, interpolation).
- Event-driven architecture experience.
- Ability to produce independent, detailed technical documentation.

---

### **Nice to Have**

- IoT/telemetry ingestion experience.
- Energy or industrial monitoring domain knowledge.
- Sol-Ark API experience.
- AWS data services (SQS, SNS, Lambda, EventBridge, Kinesis).
- Multi-tenant data architecture experience.
- Statistical processing across multiple granularities.

---

### **Engagement Details**

- Duration: ~80 working hours over 3 weeks.
- Reports to:
  - Lead Engineer (technical)
  - Director of Product Engineering (oversight)

- Coordination:
  - Lead Engineer (daily)
  - TPM (Gabi) for scheduling

- Client interaction: None directly.
- Deliverables: Written TRD data sections (no code).
- Critical dependency: Must be fully self-sufficient on the data track.

---

## 3. Key Points & Critical Insights

- The role is **documentation-first**, not implementation-focused.
- Independence is mandatory due to parallel execution with the Lead Engineer.
- Time-series ETL complexity and inconsistent third-party APIs are core challenges.
- Multi-tenant data modeling is a foundational concern.
- Failure to operate autonomously jeopardizes the 3-week timeline.

---

## 4. Important Data & Statistics

- Duration: **3 weeks**
- Dedication: **75%**
- Total effort: **~80 hours**
- Telemetry ingestion frequency: **Every 5–15 minutes**
- Cascading statistics levels: **3**
- External integrations: **5**
- ETL documentation: **1,500+ lines**
- Field specifications: **2,800+ lines**
- Total documentation reviewed: **~4,300 lines**

---

## 5. Keywords & Key Terms

- **TRD (Technical Requirements Document)** – Written technical specification output.
- **ETL Pipeline** – Data ingestion and transformation flow.
- **Time-Series Data** – Telemetry data indexed by time.
- **Multi-Tenant Architecture** – Shared platform with tenant isolation.
- **Cascading Statistics** – Aggregation across component, location, and company levels.
- **Event-Driven Architecture** – Asynchronous ingestion via events.
- **FIFO Guarantees** – Order preservation per location.

---

## 6. Action Items / Recommendations

- Deeply review and reconcile all client-provided documentation before design.
- Identify and document integration ambiguities early (Week 1).
- Produce data sections of the TRD that are independently complete.
- Align with Lead Engineer during Week 3 to resolve cross-cutting concerns.

---

## 7. Notable Quotes

- _“This is a parallel workstream — you need to operate independently without hand-holding from the Lead Engineer.”_
- _“If this role requires significant guidance from the Lead Engineer, the 3-week timeline breaks.”_

---

If you want, I can next:

- Convert this into an **interview briefing**
- Extract **risk areas**
- Map responsibilities to **expected deliverables**
- Create a **TRD outline** based strictly on this JD
