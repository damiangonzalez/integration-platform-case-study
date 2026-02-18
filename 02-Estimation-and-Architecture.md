# Estimation, Architecture Options & Productization Lens

**Purpose:** Estimation/scoping approaches, architecture options with tradeoffs, and productization lens for the Albert Invent case study. This document directly informs the presentation materials for Sections 1-3 of the interview.

**Prerequisites:** Read `s0-context-primer.md` (domain primer) and `s1-domain-research.md` (deep technical research) first.

**Acronym note:** All acronyms were defined in `s0-context-primer.md`. This document uses them freely. Quick refresher of the most common:
- **ELN** = Electronic Lab Notebook
- **LIMS** = Laboratory Information Management System
- **CISPro** = Chemical Inventory System (Biovia product)
- **ADF** = Azure Data Factory
- **ADLS** = Azure Data Lake Storage
- **SOW** = Statement of Work
- **PERT** = Program Evaluation and Review Technique
- **RACI** = Responsible, Accountable, Consulted, Informed
- **UAT** = User Acceptance Testing
- **DLQ** = Dead Letter Queue
- **iPaaS** = Integration Platform as a Service
- **pw** = person-weeks (used throughout for effort estimates)

---

## Table of Contents

- [PART A: Estimation & Scoping](#part-a-estimation--scoping)
  - [A1. Pre-Discovery Estimation: T-Shirt Sizing](#a1-pre-discovery-estimation-t-shirt-sizing)
    - [Size Definitions](#size-definitions)
    - [T-Shirt Sizing Framework: 6 Key Questions](#t-shirt-sizing-framework-6-key-questions)
    - [Applied to This Project's 4 Workstreams](#applied-to-this-projects-4-workstreams)
    - [Final T-Shirt Sizing Summary](#final-t-shirt-sizing-summary)
    - [Presenting Ranges to the Customer](#presenting-ranges-to-the-customer)
    - [Factors That Drive Sizing](#factors-that-drive-sizing)
  - [A2. Post-Discovery Estimation: PERT](#a2-post-discovery-estimation-pert)
    - [The Formula](#the-formula)
    - [Sample Decomposition: LIMS Integration (Workstream 2)](#sample-decomposition-lims-integration-workstream-2)
    - [Contingency Buffers](#contingency-buffers)
  - [A3. Handling Ambiguity](#a3-handling-ambiguity)
    - [Technique 1: Discovery as a Separate Fixed-Price Deliverable](#technique-1-discovery-as-a-separate-fixed-price-deliverable)
    - [Technique 2: Time-Boxed Technical Spikes](#technique-2-time-boxed-technical-spikes)
    - [Technique 3: Risk Register and Assumption Log](#technique-3-risk-register-and-assumption-log)
  - [A4. SOW Structure](#a4-sow-structure)
    - [Phase Structure](#phase-structure)
    - [Timeline Visualization (Feb-Aug 2026)](#timeline-visualization-feb-aug-2026)
    - [Milestone/Deliverable Matrix](#milestonedeliverable-matrix)
    - [Go/No-Go Gate Criteria](#gono-go-gate-criteria)
    - [Commercial Model: Hybrid (Recommended)](#commercial-model-hybrid-recommended)
    - [Protecting Albert from Scope Creep](#protecting-albert-from-scope-creep)
    - [Change Order Process](#change-order-process)
    - [Acceptance Criteria Process](#acceptance-criteria-process)
  - [A5. Customer Alignment & Commitment](#a5-customer-alignment--commitment)
    - [RACI Matrix](#raci-matrix)
    - [Discovery Workshop Structure (4 Weeks)](#discovery-workshop-structure-4-weeks)
    - [Managing "We Aren't Sure What Data Needs to Flow Where"](#managing-we-arent-sure-what-data-needs-to-flow-where)
    - [Customer Obligations (Must Be in the SOW)](#customer-obligations-must-be-in-the-sow)
- [PART B: Architecture Options](#part-b-architecture-options)
  - [B1. LIMS Integration Pattern (LabVantage ↔ Albert)](#b1-lims-integration-pattern-labvantage--albert)
    - [Option A: iPaaS (MuleSoft, Boomi, or Workato)](#option-a-ipaas-mulesoft-boomi-or-workato)
    - [Option B: Custom Integration Service (RECOMMENDED for first customer)](#option-b-custom-integration-service-recommended-for-first-customer)
    - [Option C: Albert-Hosted Connector with Adapter Pattern](#option-c-albert-hosted-connector-with-adapter-pattern)
    - [Recommendation](#recommendation)
    - [Unknowns to Validate](#unknowns-to-validate)
  - [B2. ELN File Migration (Biovia ELN → Albert Notebook)](#b2-eln-file-migration-biovia-eln--albert-notebook)
    - [Option A: Bulk Export + Blob Storage Staging (RECOMMENDED)](#option-a-bulk-export--blob-storage-staging-recommended)
    - [Option B: Streaming Pipeline](#option-b-streaming-pipeline)
    - [Option C: Hybrid (Bulk + Streaming)](#option-c-hybrid-bulk--streaming)
    - [Recommendation](#recommendation-1)
    - [Unknowns to Validate](#unknowns-to-validate-1)
  - [B3. Inventory Migration (CISPro → Albert Inventory)](#b3-inventory-migration-cispro--albert-inventory)
    - [Option A: ETL Pipeline with Staging Database (RECOMMENDED)](#option-a-etl-pipeline-with-staging-database-recommended)
    - [Option B: Direct Bulk Import via Albert SDK](#option-b-direct-bulk-import-via-albert-sdk)
    - [Option C: Hybrid with Shadow Load](#option-c-hybrid-with-shadow-load)
    - [Recommendation](#recommendation-2)
    - [Unknowns to Validate](#unknowns-to-validate-2)
  - [B4. Azure Data Lake Integration (Albert → Azure)](#b4-azure-data-lake-integration-albert--azure)
    - [Option A: Customer-Pull via ADF (RECOMMENDED)](#option-a-customer-pull-via-adf-recommended)
    - [Option B: Vendor-Push (Albert Pushes to ADLS)](#option-b-vendor-push-albert-pushes-to-adls)
    - [Option C: Event-Driven Streaming (Azure Event Hubs)](#option-c-event-driven-streaming-azure-event-hubs)
    - [Recommendation](#recommendation-3)
    - [Unknowns to Validate](#unknowns-to-validate-3)
  - [B5. Connector Framework / Productization](#b5-connector-framework--productization)
    - [Option A: Configuration-Driven Adapter Framework (RECOMMENDED)](#option-a-configuration-driven-adapter-framework-recommended)
    - [Option B: SDK Extension Model](#option-b-sdk-extension-model)
    - [Option C: Template-Based Code Generation](#option-c-template-based-code-generation)
    - [Recommendation](#recommendation-4)
    - [Unknowns to Validate](#unknowns-to-validate-4)
- [PART C: Productization Lens](#part-c-productization-lens)
  - [The Productization Progression](#the-productization-progression)
- [Summary](#summary)
  - [Effort Summary](#effort-summary)
  - [Calendar Timeline](#calendar-timeline)
  - [Quick Reference: All Design Choices](#quick-reference-all-design-choices)
  - [Red Flags to Watch](#red-flags-to-watch)

---

# PART A: Estimation & Scoping

*Maps to Interview Section 1: Estimate and Scoping (15 min presentation)*

---

## A1. Pre-Discovery Estimation: T-Shirt Sizing

T-shirt sizing is the right first-pass technique when requirements are ambiguous. It uses categorical sizes instead of precise hours, which honestly communicates the uncertainty level to the customer.

### Size Definitions

| Size | Person-Week Range | Characteristics |
|------|-------------------|-----------------|
| **S** | 1-3 pw | Single system, well-documented API, unidirectional, no data transformation, no deadline pressure |
| **M** | 3-8 pw | Single system, moderate data mapping, some unknowns, clear API but needs configuration |
| **L** | 8-16 pw | Multi-system or bidirectional, significant data transformation, dependency on customer resources, deadline pressure |
| **XL** | 16-32 pw | First-of-kind, bidirectional with complex error handling, multiple systems, regulatory/compliance, hard deadline |

### T-Shirt Sizing Framework: 6 Key Questions

**Methodology source:** T-shirt sizing is a standard agile estimation technique documented in Mike Cohn's "Agile Estimating and Planning" (2005) and widely used in enterprise software for pre-discovery scoping. The 6-question framework below adapts Cohn's relative sizing approach with dimensions specific to integration and migration work based on common practice at enterprise SaaS companies (Salesforce, ServiceNow, Workday).

To assign a t-shirt size, evaluate each workstream against these 6 dimensions:

| Question | Small (S) | Medium (M) | Large (L) | XL |
|----------|-----------|------------|-----------|-----|
| **1. How many systems?** | 1 system | 1-2 systems | 2-3 systems | 3+ or very complex |
| **2. Directionality?** | One-way, read-only | One-way, write | Bidirectional | Bidirectional + state sync |
| **3. Data complexity?** | Files or simple flat data | Structured data, moderate mapping | Complex relationships, heavy transformation | Unknown schema + complex rules |
| **4. API availability?** | Well-documented REST | Documented but quirky | Poorly documented | No API (must reverse-engineer) |
| **5. Ongoing vs one-time?** | One-time, no monitoring | One-time with validation | Ongoing, needs monitoring | Ongoing + SLA/compliance |
| **6. Have we done this before?** | Standard pattern | Similar to past work | First time, but familiar tech | First-of-kind |

**Sizing rule:** Count how many dimensions fall into each column. Mostly "Small" = S, Mostly "Medium" = M, Mix of Medium/Large = L, Mostly Large or any XL = XL.

---

### Applied to This Project's 4 Workstreams

**Step 1: Map each workstream to the 6-question framework**

| Question | 1. ELN Migration | 2. LIMS Integration | 3. Data Warehouse | 4. Inventory Migration |
|----------|-----------------|-------------------|------------------|----------------------|
| **Systems?** | 2 (Biovia + Albert) = M | 2 (LabVantage + Albert), bidirectional = L | 2 (Albert + Azure), customer-pull = M | 2 (CISPro + Albert) = M |
| **Directionality?** | One-way = **S** | **Bidirectional + state sync = XL** | One-way outbound = **S** | One-way = **S** |
| **Data complexity?** | Files + metadata, simple hierarchy = **S-M** | Moderate (sample/test mapping), but state machine adds complexity = L | Schema mapping, customer handles transformation = M | **Complex referential integrity (Substance → Lot → Container), data quality risk = L-XL** |
| **API?** | **No public API (DB or Pipeline Pilot) = XL** | REST exists but non-standard queries, no webhooks = M-L | Albert provides API, customer uses ADF (standard) = **S-M** | **No public API = XL** |
| **Ongoing?** | One-time with July 1 deadline = M | **Ongoing production integration (monitoring, alerting, DLQ, retry) = L** | Ongoing, customer operates pipeline = M | One-time with July 1 deadline = M |
| **Done before?** | File migrations common, but Biovia extraction isn't = M | **First LabVantage integration Albert has built = XL** | ADF integrations are standard = **S** | Similar to past, but CISPro-specific = M |

**Step 2: Count column distribution and assign size**

| Workstream | Mostly S/M | Has L traits | Has XL traits | **Assigned Size** | **Person-Week Range** |
|-----------|-----------|-------------|--------------|------------------|---------------------|
| **1. ELN Migration** | ✓ (3 Small, 2 Medium) | 0 | 1 (no API) | **M** | **6-8 pw** (high end of M) |
| **2. LIMS Integration** | 0 | 3 (bidirectional, ongoing, data) | 2 (state sync, first-of-kind) | **XL** | **16-22 pw** |
| **3. Data Warehouse** | ✓ (4 Small/Medium) | 0 | 0 | **M** | **3-5 pw** (low end of M) |
| **4. Inventory Migration** | ✓ (2 Small, 3 Medium) | 2 (data complexity, customer dependency) | 1 (no API) | **L** | **8-11 pw** |

**Key insight:** ELN and Inventory both have "no API" (XL trait), but ELN has simpler data (files) while Inventory has complex relational data with quality risk. This is why ELN stays M while Inventory moves to L.

---

### Final T-Shirt Sizing Summary

| Workstream | T-Shirt | Rationale |
|-----------|---------|-----------|
| 1. ELN Migration (Biovia → Albert Notebook) | **M** | One-time, unidirectional file migration. But: volume unknown, organizational mapping needed, AI indexing requirement, July 1 deadline |
| 2. LIMS Integration (LabVantage ↔ Albert) | **XL** | Bidirectional, ongoing, first-of-kind for Albert, complex error handling, state management, production reliability |
| 3. Data Warehouse (Albert → Azure) | **M** | Outbound only, Albert has Data Warehouse module, customer has Azure patterns. But: schema mapping and customer data model alignment add complexity |
| 4. Inventory Migration (CISPro → Albert) | **L** | Structured data with complex relationships (Substance → Lot → Container), data validation critical, CAS number matching, July 1 deadline |

### Presenting Ranges to the Customer

Present as a three-column range with explicit assumptions tied to each column:

| Workstream | Optimistic | Expected | Pessimistic | Optimistic Assumes... |
|-----------|-----------|----------|-------------|----------------------|
| 1. ELN Migration | 3 pw | 6 pw | 10 pw | Biovia provides clean bulk export, file count <50K, simple project mapping |
| 2. LIMS Integration | 10 pw | 18 pw | 28 pw | LabVantage API is accessible, sample/test lifecycle is standard, customer has clear data mapping |
| 3. Data Warehouse | 3 pw | 5 pw | 9 pw | Customer Azure team has established ADF patterns, Albert Data Warehouse schema is stable |
| 4. Inventory Migration | 4 pw | 8 pw | 14 pw | CISPro provides clean structured export, entity relationships are consistent, volume <100K records |
| **TOTAL** | **20 pw** | **37 pw** | **61 pw** | |

**Key message to customer:** "These are pre-discovery estimates. Discovery will narrow the range by 40-60%. The optimistic estimate assumes zero surprises. The pessimistic assumes significant rework."

### Factors That Drive Sizing

1. **API availability and maturity** — Does the external system have a well-documented REST API? LabVantage has one, but with non-standard query patterns (server-side Query objects, not URL params). Biovia products have **no public API** at all.
2. **Data complexity** — Files (Workstream 1) are simpler than structured relational data (Workstream 4). Bidirectional data with state machines (Workstream 2) is the most complex.
3. **Directionality** — Unidirectional is inherently simpler. Bidirectional requires conflict resolution, idempotency, and state reconciliation.
4. **Ongoing vs. one-time** — One-time migrations need correctness. Ongoing integrations need correctness PLUS reliability, monitoring, alerting, error recovery, and operational runbooks.
5. **Deadline pressure** — July 1 compresses timelines and reduces ability to iterate. Forces parallel execution and increases coordination overhead.
6. **Customer readiness** — If the customer cannot provide API credentials, test environments, data samples, or SME (Subject Matter Expert) time promptly, estimates blow up.

---

## A2. Post-Discovery Estimation: PERT

After discovery identifies specific tasks, switch from t-shirt sizing to PERT estimation at the work-package level.

### The Formula

```
Expected Duration = (O + 4M + P) / 6
Standard Deviation = (P - O) / 6
```

Where O = Optimistic, M = Most Likely, P = Pessimistic. The "4M" weighting means the most likely estimate gets 4× the weight of the extremes.

### Sample Decomposition: LIMS Integration (Workstream 2)

| Work Package | O (days) | M (days) | P (days) | PERT (days) |
|-------------|----------|----------|----------|-------------|
| Auth/connectivity setup (LabVantage API) | 2 | 3 | 8 | 3.7 |
| Data mapping: Albert Task → LV Sample/Test Request | 3 | 5 | 12 | 5.8 |
| Outbound flow: submit request from Albert to LV | 5 | 8 | 15 | 8.7 |
| Inbound flow: poll/receive results from LV to Albert | 5 | 10 | 20 | 10.8 |
| Error handling and retry logic | 3 | 5 | 12 | 5.8 |
| Status sync and state machine | 3 | 7 | 15 | 7.7 |
| Integration testing (both directions) | 5 | 8 | 15 | 8.7 |
| UAT support | 3 | 5 | 10 | 5.5 |
| Production deployment + hypercare | 2 | 3 | 5 | 3.2 |
| **TOTAL** | **31** | **54** | **112** | **59.9** |

That's ~60 person-days = ~12 person-weeks for the LIMS integration build phase alone. With design, testing, and deployment phases, the total aligns with the T-shirt XL range.

### Contingency Buffers

| Scenario | Buffer |
|----------|--------|
| **First-of-kind integration** (e.g., first LabVantage integration Albert has ever done) | **25-35%** on top of PERT expected |
| **Repeatable/known pattern** (e.g., second LabVantage integration with same architecture) | **10-15%** |
| **One-time migration with hard deadline** (Workstreams 1, 4) | **20-25%** — deadline removes ability to extend, so buffer must be in the schedule upfront |
| **Customer dependency risk** (waiting on API access, data samples, SME availability) | Additional **10-15%** for "blocked time" |

**For this project:**
- Workstream 2 (LIMS): first-of-kind → 30% contingency
- Workstream 3 (Azure): heavy customer dependency → 20% contingency + customer dependency buffer
- Workstreams 1 & 4 (migrations): hard July 1 → 25% buffer, start early

---

## A3. Handling Ambiguity

The customer said: *"We aren't yet sure what data needs to flow where — we just need everything connected."*

This is the single most dangerous statement in the brief. It signals the customer hasn't done their own internal requirements gathering. Three techniques to manage this:

### Technique 1: Discovery as a Separate Fixed-Price Deliverable

Sell the discovery phase as a standalone engagement with its own SOW.

| Attribute | Recommendation |
|-----------|---------------|
| **Duration** | 3-4 weeks (for 4 workstreams of this complexity) |
| **Cost model** | Fixed price ($40K-$60K depending on team size and depth) |
| **Team** | Principal Enterprise Engineer (lead), 1-2 integration engineers for technical spikes, customer's technical leads |
| **Deliverables** | Technical Discovery Report, Data Mapping Documents (per workstream), Integration Architecture Diagrams, Detailed PERT Estimate, Risk Register, Assumptions Log, Draft SOW for implementation |

**The discovery SOW should state:** "The output is a detailed scope document and fixed-price estimate for implementation. The customer is under no obligation to proceed, and Albert is under no obligation to hold pre-discovery estimates."

### Technique 2: Time-Boxed Technical Spikes

During discovery, run focused spikes to resolve specific unknowns:

| Spike | Duration | Purpose | Output |
|-------|----------|---------|--------|
| Biovia ELN export test | 2-3 days | Can we bulk-export files? What format? What metadata? | Export feasibility report, file count/size estimate |
| LabVantage API connectivity | 2-3 days | Can we authenticate? Create a sample? Read results? | Connectivity PoC (Proof of Concept), API capability assessment |
| CISPro data extract | 2-3 days | Can we get a structured export? How clean is the data? | Sample extract, data quality assessment |
| Albert SDK validation | 1-2 days | Does the SDK support all required operations per workstream? | SDK capability matrix |

### Technique 3: Risk Register and Assumption Log

**Risk Register (specific to this project):**

| ID | Risk | Prob | Impact | Mitigation |
|----|------|------|--------|------------|
| R1 | Biovia ELN has no bulk export API; manual extraction required | Med | High (schedule) | Spike during discovery; plan for scripted Oracle DB extraction |
| R2 | LabVantage API doesn't support required operations (e.g., creating sample requests programmatically) | Med | Critical | Verify during discovery spike; escalate to LabVantage support |
| R3 | CISPro data quality is poor (orphaned records, inconsistent CAS numbers) | High | High (rework) | Request data sample early; build data cleansing phase into estimate |
| R4 | Customer cannot provide test environment access within 2 weeks of kickoff | Med | High (schedule) | Make it a contractual prerequisite in the SOW |
| R5 | July 1 deadline not achievable for all 4 workstreams | Med | Critical | Prioritize migrations (WS 1, 4) over ongoing integrations (WS 2, 3); propose phased go-live |
| R6 | Analytical teams resist change ("perceived to be too high" — per the case study brief) | Med | High (adoption) | Start with minimal viable integration; get early wins; involve analytical leads in design |
| R7 | First-of-kind LIMS integration has no reference architecture or reusable code at Albert | High | High (effort) | Budget 30% contingency; plan for post-project productization of the connector |

**Assumption Log (specific to this project):**

| ID | Assumption | If False... |
|----|-----------|-------------|
| A1 | Biovia ELN files are <100K total, <500GB total size | Volume drives migration timeline; may need parallel processing |
| A2 | LabVantage REST API is enabled and accessible from Albert's cloud | If behind a firewall with no VPN, architecture changes significantly |
| A3 | Customer has a LabVantage admin who can configure queries and RESTPolicy | Without this, Albert cannot access the API at all (deny-by-default) |
| A4 | Albert's SDK supports all required inventory object creation | If not, must use raw API or request SDK enhancements from Albert product team |
| A5 | Customer will dedicate at least 0.5 FTE of SME time per workstream during build | Without customer input, data mapping and UAT stall |
| A6 | Biovia contracts allow data extraction/migration (no vendor lock-in clauses) | Legal review needed; could block entire migration |

---

## A4. SOW Structure

### Phase Structure

Standard for enterprise integration: **Discovery → Design → Build → Test → Deploy → Hypercare**

For 4 parallel workstreams, use a shared Discovery phase followed by workstream-specific execution:

```
Phase 0: Discovery (shared, all 4 workstreams)       3-4 weeks
Phase 1: Design (per workstream, can be parallel)     2-3 weeks each
Phase 2: Build (per workstream, can be parallel)      4-10 weeks each
Phase 3: Test (per workstream, includes integration)  2-4 weeks each
Phase 4: Deploy (per workstream, staggered)           1-2 weeks each
Phase 5: Hypercare (per workstream)                   2-4 weeks each
```

### Timeline Visualization (Feb-Aug 2026)

```
                    Feb    Mar    Apr    May    Jun    Jul    Aug
Discovery           [====]                                         (all 4 workstreams)
                          |
                          v--- Scope Lock / Go-No-Go Gate

WS1: ELN Migration       [Des][=Build==][Test][DEP]               ← July 1 cutover
WS4: Inv Migration        [Design][===Build===][Test][DEP]         ← July 1 cutover
WS3: Azure                [Des][==Build==][Test][DEP][Hypercare==]
WS2: LIMS Integration     [Design=][====Build=====][Test][DEP][HC=====]
```

**Key scheduling decisions:**
1. **Shared Discovery** — all 4 workstreams together to maximize cross-cutting learning
2. **Prioritize migrations** — WS 1 (ELN) and WS 4 (Inventory) have hard July 1 deadline; start their build first
3. **Stagger deployments** — don't attempt all 4 go-lives in the same week. Deploy in risk order: ELN (lowest risk, files) → Inventory → Data Warehouse → LIMS (highest risk, bidirectional)
4. **Shared resources with dedicated leads** — one Principal Engineer oversees all 4; each workstream has a dedicated integration engineer

### Milestone/Deliverable Matrix

**Phase 0: Discovery**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| D-1: Kickoff | Meeting minutes, RACI, communication plan | Customer signs off on RACI |
| D-2: Technical assessment | Discovery Report per workstream | Covers data mapping, API assessment, architecture options, risks |
| D-3: Scope locked | Detailed Scope Document with data flow diagrams, entity mappings, acceptance criteria | Both parties sign off. This becomes the change order baseline |
| D-4: Estimate finalized | PERT-based estimate, resource plan, schedule | Both parties agree to timeline and commercial terms |

**Phase 1: Design (per workstream)**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| DES-1: Architecture approved | Integration Architecture Document (sequence diagrams, data flows, error handling) | Both technical leads sign off |
| DES-2: Data mapping approved | Field-level Data Mapping Specification (source → target, transformations, validation rules) | Customer SME validates every mapping |
| DES-3: Environment ready | Test environments provisioned, API credentials obtained, connectivity verified | Albert team can make test API calls |

**Phase 2: Build (per workstream)**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| BLD-1: Core pipeline functional | Working end-to-end flow in dev environment | Demonstrates happy path with sample data |
| BLD-2: Error handling complete | Retry, DLQ, alerting, logging | Error scenarios documented and tested |
| BLD-3: Code review + docs | Reviewed code, operational runbook draft | Meets Albert's internal engineering standards |

**Phase 3: Test (per workstream)**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| TST-1: Integration testing | Test report against test plan | All critical test cases pass; no P1/P2 defects |
| TST-2: UAT complete | Customer sign-off on UAT results | Business users confirm requirements met |
| TST-3: Performance testing (ongoing integrations only) | Performance test report | Meets agreed throughput and latency SLAs |

**Phase 4: Deploy (per workstream)**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| DEP-1: Production deployment | Deployment runbook executed, integration live | Data flows correctly in production |
| DEP-2: Cutover (migrations only) | Migration completion report, validation report | All data migrated, validated, accessible |

**Phase 5: Hypercare (per workstream)**

| Milestone | Deliverable | Acceptance Criteria |
|-----------|------------|---------------------|
| HC-1: Hypercare complete | Summary report, issue log | No open P1/P2 issues; SLAs met for 2+ consecutive weeks |
| HC-2: Handoff to support | Finalized runbook, dashboards, escalation procedures | Support team can independently operate |

### Go/No-Go Gate Criteria

**Discovery → Design:**
- [ ] Scope document signed by both parties
- [ ] Data mapping ≥80% complete (remaining 20% identified as design-phase items)
- [ ] All external system APIs verified accessible
- [ ] Commercial terms agreed
- [ ] Customer has assigned named resources per RACI

**Design → Build:**
- [ ] Architecture document signed off by both technical leads
- [ ] Data mapping 100% complete for this workstream
- [ ] Test environments provisioned and accessible
- [ ] API credentials (non-production) obtained
- [ ] Test data available

**Build → Test:**
- [ ] All code committed and peer-reviewed
- [ ] Happy-path end-to-end flow working in test environment
- [ ] Error handling implemented for all identified failure modes
- [ ] Test plan reviewed and approved by customer
- [ ] Operational runbook drafted

**Test → Deploy:**
- [ ] All critical/high test cases pass
- [ ] No open P1 defects
- [ ] Customer UAT sign-off obtained
- [ ] Production credentials and environment access obtained
- [ ] Rollback plan documented
- [ ] For migrations: data validation report reviewed (counts, checksums, spot-checks)

### Commercial Model: Hybrid (Recommended)

| Component | Model | Rationale |
|-----------|-------|-----------|
| **Discovery** | Fixed price ($40K-$60K) | Well-scoped deliverable; low-risk entry point; demonstrates competence |
| **WS1: ELN Migration** | Fixed price | File migration is predictable post-discovery; scope is containable |
| **WS4: Inventory Migration** | Fixed price with capped change orders | Structured data migration is estimable; cap on changes protects Albert |
| **WS2: LIMS Integration** | T&M (Time & Materials) with NTE (Not-to-Exceed) cap | First-of-kind, bidirectional, high unknowns. T&M protects Albert; NTE cap gives customer budget certainty |
| **WS3: Data Warehouse** | Fixed price | Outbound, likely aligns with customer's existing patterns |
| **Hypercare** | Included (migrations) or T&M rate (integrations) | Migrations have defined hypercare. Ongoing integrations transition to SLA/support contract |

### Protecting Albert from Scope Creep

1. **Assumptions section** — every assumption that, if false, triggers a change order. Example: "Assumes LabVantage REST API is accessible from Albert's cloud without VPN. If VPN or IP whitelisting required, change order will be submitted."
2. **Explicit exclusions** — list what is NOT included. Example: "Does not include: Biovia ELN decommissioning, LabVantage configuration changes, ADF pipeline development, data cleansing of source data, end-user training."
3. **Change order threshold** — "Any request adding >8 hours to any workstream requires a formal Change Order with customer approval before work begins."
4. **Customer dependency SLAs** — "If customer does not provide [API access / test data / SME time / deliverable feedback] within X business days, timeline shifts day-for-day, and Albert may submit a change order for remobilization costs."
5. **Scope freeze date** — "No scope changes accepted after [date] for workstreams with July 1 deadline."

### Change Order Process

1. **Initiation** — either party identifies a potential scope change
2. **Documentation** — Change Request Form: description, business justification, schedule/cost/resource/risk impact, priority
3. **Assessment** — Albert assesses within 3 business days (PERT estimate, schedule impact, cost)
4. **Approval** — both parties' authorized signatories approve/reject. Approval creates a formal amendment
5. **Execution** — approved change incorporated into project plan and schedule

**Critical for July 1:** Any change affecting WS 1 or WS 4 must include timeline impact assessment. If it jeopardizes July 1, escalate to executive sponsors on both sides.

### Acceptance Criteria Process

- Each deliverable has defined acceptance criteria (see milestone tables above)
- Customer has **5 business days** to review and accept or provide specific written feedback
- **Deemed-accepted clause:** no response within 5 business days = accepted
- Albert has **5 business days** to address valid feedback, then resubmit
- UAT period: 5-10 business days per workstream
- UAT exit criteria: all critical test cases pass, all P1 defects resolved, P2 defects have agreed resolution plan
- Final acceptance: end of hypercare, all P1/P2 resolved, operational handoff complete → triggers final payment

---

## A5. Customer Alignment & Commitment

### RACI Matrix

| Activity | Albert Principal Eng | Albert Integration Devs | Customer PM | Customer IT | Customer SMEs | Customer LV Admin | Customer Azure Team |
|----------|---------------------|------------------------|-------------|-------------|---------------|-------------------|-------------------|
| Project governance | A | I | R | I | I | I | I |
| Discovery workshops | R | C | A | R | R | C | C |
| Data mapping (Albert side) | A | R | I | C | C | — | — |
| Data mapping (external systems) | C | C | I | R | A | R | R |
| Architecture design | A/R | R | I | C | I | C | C |
| API access provisioning | I | I | A | R | — | R | R |
| Test environment setup (Albert) | A | R | I | — | — | — | — |
| Test environment setup (external) | I | I | A | R | — | R | R |
| Build integration code | A | R | I | C | — | C | C |
| Integration testing | A | R | I | R | C | C | C |
| UAT execution | C | C | A | C | R | C | C |
| UAT sign-off | I | I | A | C | R | — | — |
| Production deployment | A | R | I | R | I | R | R |
| Data validation (migrations) | C | R | I | C | A | — | — |
| Hypercare support | A | R | I | R | R | C | C |

R = Responsible (does the work), A = Accountable (approves/owns), C = Consulted, I = Informed

**Key point:** Customer PM is Accountable for many activities because the customer must commit resources, provide access, and make decisions. Albert can build the integration, but cannot succeed without active customer participation.

### Discovery Workshop Structure (4 Weeks)

**Week 1: Orientation & Data Landscape**

| Session | Duration | Attendees | Outputs |
|---------|----------|-----------|---------|
| Project Kickoff | 2h | All stakeholders, exec sponsors | Aligned objectives, RACI, communication plan |
| WS1 Deep-Dive (ELN) | 2h | Albert + ELN owners, IT | File inventory estimate, org structure, access control requirements, Biovia export capabilities |
| WS4 Deep-Dive (Inventory) | 2h | Albert + inventory managers, IT | CISPro data model walkthrough, entity counts, data quality concerns, business rules |
| System Access Workshop | 1h | Albert + Customer IT | Connectivity requirements, firewall rules, VPN needs, credential provisioning plan |

**Week 2: Integration Deep-Dives**

| Session | Duration | Attendees | Outputs |
|---------|----------|-----------|---------|
| WS2 (LIMS) Part 1: Business Process | 3h | Albert + R&D scientists, analytical leads | Current-state and future-state workflow mapping, data flow requirements |
| WS2 (LIMS) Part 2: Technical | 2h | Albert + LV admin, Customer IT | API capabilities, available queries, sample/test data model, auth |
| WS3 Deep-Dive (Azure) | 2h | Albert + Azure/data platform team | Existing ingestion patterns, medallion architecture, ADF capabilities |
| Data Mapping Workshop | 3h | Albert + SMEs per workstream | Draft field-level mappings for all workstreams |

**Week 3: Technical Spikes & Validation**

| Activity | Duration | Purpose |
|----------|----------|---------|
| Biovia ELN export spike | 2-3 days | Validate export feasibility, measure volume, assess metadata quality |
| LabVantage API spike | 2-3 days | Authenticate, create test sample, read results, validate CRUD |
| CISPro data extract spike | 2-3 days | Extract sample data, validate relationships, assess data quality |
| Albert SDK validation | 1-2 days | Validate SDK supports all required operations per workstream |

**Week 4: Synthesis & Scope Lock**

| Session | Duration | Attendees | Outputs |
|---------|----------|-----------|---------|
| Findings Readout | 2h | All stakeholders | Technical discovery report, spike results, risk register |
| Data Mapping Review | 2h | Technical leads + SMEs | Finalized data mappings (or documented gaps for design phase) |
| Scope & Estimate Review | 2h | Project leads + exec sponsors | Detailed estimate, proposed timeline, commercial terms |
| Scope Lock Sign-Off | 1h | Authorized signatories | Signed scope document → basis for implementation SOW |

### Managing "We Aren't Sure What Data Needs to Flow Where"

Five tactics:

1. **Acknowledge, don't fight it.** "That's very normal at this stage. Discovery is specifically designed to answer that question."
2. **Use process mapping as a forcing function.** Instead of "what data do you need?", ask "walk me through what happens when a scientist needs to run a test." Data requirements emerge from the workflow.
3. **Provide a strawman, not a blank canvas.** Present a default data mapping based on typical lab workflows. The customer reacts to a concrete proposal faster than they generate requirements from scratch.
4. **Use LIMS as the anchor.** Define the minimum viable integration first: Outbound = Test Request (sample ID, test type, priority, due date, submitter). Inbound = Measured Results (sample ID, test ID, value, unit, status, analyst, date). Start here. Add fields iteratively.
5. **Contractually protect against open-ended requirements.** "Data mappings in Appendix A represent the agreed scope. Additional fields, transformations, or integrations not listed require a Change Order."

### Customer Obligations (Must Be in the SOW)

| Obligation | Timeline | If Not Provided... |
|-----------|----------|-------------------|
| Named project manager with decision authority | By kickoff | Project cannot start |
| API access to LabVantage (non-prod) | Within 2 weeks of kickoff | LIMS design and build blocked |
| API access to LabVantage (prod) | 2 weeks before go-live | LIMS deployment blocked |
| Biovia ELN export or database access | Within 2 weeks of kickoff | ELN migration build blocked |
| CISPro data extract (full export) | Within 2 weeks of kickoff | Inventory migration build blocked |
| Azure ADF access and pattern docs | Within 2 weeks of kickoff | Data Warehouse design blocked |
| Test environments for all external systems | By end of discovery | Build phase blocked |
| SMEs: 0.5 FTE/workstream (discovery + build), 1 FTE/workstream (UAT) | Throughout | Data mapping incomplete, UAT delayed |
| Representative test data (anonymized or synthetic) | Within 3 weeks of kickoff | Testing uses unrealistic data; production issues surface late |
| Timely feedback: 5 business day turnaround | Throughout | Schedule slips day-for-day |
| Decision authority: PM can make binding decisions | Throughout | Decision paralysis delays schedule |
| Network/firewall changes for Albert cloud access | By end of design | Build blocked |
| Named UAT testers | 2 weeks before UAT | UAT delayed |

**Contractual language:** "Albert's obligations are contingent upon Customer fulfilling Customer Obligations in Section X. If Customer fails within the specified timeline, Albert will notify in writing. If not fulfilled within 5 business days of notification: (a) timeline extends day-for-day, (b) Albert may submit a Change Order for additional costs including remobilization, (c) delivery deadlines including July 1 are adjusted accordingly."

---

# PART B: Architecture Options

*Maps to Interview Section 2: LIMS Integration Plan (20 min presentation) and Section 3: Increasing Velocity Over Time (10 min presentation)*

---

## B1. LIMS Integration Pattern (LabVantage ↔ Albert)

**Context:** Bidirectional ongoing integration. Requests submitted FROM Albert TO LabVantage. Measured results flow BACK from LabVantage to Albert.

**Critical constraint:** LabVantage has **no native webhooks or push notifications**. Albert must poll or use LabVantage's Event Service to detect new results.

### Option A: iPaaS (MuleSoft, Boomi, or Workato)

Use an iPaaS to mediate between Albert and LabVantage.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Built-in monitoring dashboard, transformation engine, logging, retry. Pre-built connectors for common systems. |
| **Technical Cons** | No pre-built LabVantage connector exists in any major iPaaS. Still must build custom adapters for both LabVantage and Albert. LabVantage's non-standard query model (server-side queries, not URL params) and deny-by-default RESTPolicy won't be easier through an iPaaS. The hard problems (state management, polling, idempotency) remain. |
| **Business Pros** | "Enterprise" brand credibility with customer. Familiar to enterprise IT teams. |
| **Business Cons** | **$36K-$180K+/year licensing cost** passed to customer or absorbed by Albert. Adds vendor dependency. Albert doesn't own the IP. Doesn't contribute to AlbertHub productization. |
| **Effort** | **4-6 pw** for custom adapters + iPaaS configuration |
| **When it makes sense** | Only if the customer already has and pays for an iPaaS and insists on using it |

### Option B: Custom Integration Service (RECOMMENDED for first customer)

Build a Python microservice that uses the Albert SDK + LabVantage REST API directly.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Full control over polling, retry, error handling. Use Albert Python SDK directly. LabVantage ExternalReference field for correlation IDs. Circuit breaker + exponential backoff + DLQ for resilience. Can be deployed as a containerized service in Albert's infrastructure. |
| **Technical Cons** | Higher initial build effort. Must implement monitoring, logging, alerting from scratch. Must handle LabVantage's non-standard query patterns. |
| **Business Pros** | **Albert owns the IP.** No licensing cost. Builds institutional knowledge. No vendor lock-in. |
| **Business Cons** | No "enterprise" iPaaS branding. Requires Albert engineering capacity for ongoing maintenance. |
| **Effort** | **6-8 pw** |

**Key design details:**

- **Polling interval: 5-10 minutes.** Justified because lab test turnaround times are hours to days. 5-minute polling is near-real-time relative to the business process. This avoids the complexity of real-time eventing for a use case that doesn't need it.
- **Outbound flow (Albert → LabVantage):** Albert user submits a test request → integration service creates a Sample + Test in LabVantage via REST POST → stores correlation ID (LabVantage SampleId ↔ Albert TaskId) → updates Albert task status to "Submitted"
- **Inbound flow (LabVantage → Albert):** Polling service queries LabVantage for test results with status = "Complete" → retrieves DataSet/DataItem values → pushes results to Albert via SDK → updates Albert task status to "Results Received"
- **State machine:** Submitted → In Progress → Complete → Approved. Each status transition is logged with timestamp and correlation ID.
- **Error handling:** Circuit breaker (trip after 3 consecutive failures, reset after 5 min). DLQ for messages that fail after max retries. Alerting on DLQ depth > 0.

### Option C: Albert-Hosted Connector with Adapter Pattern

Build a configuration-driven connector framework from the start.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | YAML configuration files define field mappings, polling intervals, endpoints. `LIMSConnector` interface with `LabVantageAdapter` implementation. Shared framework: scheduler, retry engine, DLQ, monitoring, health checks. New LabVantage customers need only a new YAML config. |
| **Technical Cons** | Highest upfront effort. Risk of premature abstraction — you don't know what to abstract until you've built the first one. May over-engineer for patterns that turn out to be customer-specific. |
| **Business Pros** | **Directly addresses Section 3 (velocity) and AlbertHub vision.** Reduces next LabVantage customer from 6-8 weeks to 2-3 weeks. Enables Tech Ops / Customer Success to deploy without engineering. |
| **Business Cons** | Delays first customer delivery. Higher upfront investment. Framework may need redesign after real-world usage reveals wrong abstractions. |
| **Effort** | **8-12 pw** |

### Recommendation

**Build Option B first. Refactor into Option C after the first customer goes live.**

Rationale:
1. **Avoid premature abstraction.** You need to build one integration to understand what's truly reusable vs. customer-specific.
2. **Deliver value faster.** Option B is 6-8 pw vs. 8-12 pw. The customer gets their integration sooner.
3. **Informed refactoring.** After go-live, you know exactly which components are generic (polling, retry, DLQ, monitoring) vs. LabVantage-specific (query patterns, status model, field mappings). The refactoring into Option C is cheaper and more accurate.
4. **Aligns with productization timeline.** The first customer integration becomes the "reference implementation" that the connector framework generalizes.

### Unknowns to Validate

- [ ] Is LabVantage REST API enabled and accessible from Albert's cloud?
- [ ] What named queries exist? Can the LV admin create a "get completed results since timestamp" query?
- [ ] Can we create Samples + Tests via REST POST, or must we use the Actions API for full business logic?
- [ ] What fields are required for sample/test creation? Does the customer use custom SDI fields?
- [ ] Network path: is LabVantage on-prem behind a firewall, or cloud-hosted?

---

## B2. ELN File Migration (Biovia ELN → Albert Notebook)

**Context:** One-time migration of files and attachments. NOT structured data. Hard July 1 deadline.

**Critical constraint:** Biovia ELN has **no public REST API** for bulk export. Options are Pipeline Pilot protocols or direct Oracle database SQL.

### Option A: Bulk Export + Blob Storage Staging (RECOMMENDED)

Extract all files to a staging area (S3 or Azure Blob), then upload to Albert.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | **Safety net:** staging area persists if Albert upload fails. SHA-256 checksums at every stage (extract, stage, upload). Manifest CSV for full audit trail. Resume-from-checkpoint for interrupted uploads. Extraction can start immediately while mapping discussions happen. |
| **Technical Cons** | Requires staging storage ($). Two-hop process (Biovia → staging → Albert) adds complexity. |
| **Business Pros** | **Deterministic proof of integrity.** Manifest CSV proves every file was extracted, staged, and uploaded correctly. Customer can validate before Biovia is decommissioned. Extraction can run in parallel with other discovery/design work. |
| **Business Cons** | Staging storage cost (minimal for S3/Blob — pennies per GB/month). |
| **Effort** | **6-8 pw** |

**Key design details:**

- **Extraction:** Pipeline Pilot (if customer has it) or direct Oracle SQL against BLOB storage. Script walks Projects → Notebooks → Experiments → Sections → Attachments hierarchy.
- **Staging:** Each file lands with metadata sidecar (original path, creation date, author, experiment context, SHA-256 hash).
- **Upload:** Albert SDK with 5-10 concurrent workers. Rate-limited to avoid overwhelming Albert's ingestion. Each upload verified against staging checksum.
- **Hierarchy mapping:** Biovia's Project/Notebook/Experiment/Section structure maps to Albert's Project/Notebook structure. Mapping defined in discovery.
- **Access control:** Biovia user/group permissions mapped to Albert Users/Teams. Mapping defined in discovery workshop.
- **AI indexing:** After upload, Albert's "Ask Albert" AI indexes the files for search/extraction. This is a post-migration step using Albert's existing capabilities.

### Option B: Streaming Pipeline

Direct Biovia-to-Albert transfer via a queue-based pipeline (no staging).

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Single-hop. Potentially faster for smaller volumes. |
| **Technical Cons** | **No staging safety net.** If Albert upload fails mid-stream, must re-extract from Biovia (which may be decommissioned by July 1). Harder to verify completeness. Not suitable for 10TB+ volumes. |
| **Business Pros** | Simpler architecture. Fewer moving parts. |
| **Business Cons** | No independent proof of extraction. If something goes wrong after Biovia is shut down, data is lost. |
| **Effort** | **5-7 pw** |

### Option C: Hybrid (Bulk + Streaming)

Bulk for historical data, streaming for recently modified files.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Captures recent changes close to cutover. |
| **Technical Cons** | Over-engineered for a one-time migration. The delta between bulk and cutover is small (days of new files at most). |
| **Business Pros** | Minimal data loss window. |
| **Business Cons** | Highest effort for marginal benefit. |
| **Effort** | **8-10 pw** |

### Recommendation

**Option A: Bulk Export + Blob Staging.**

Rationale:
1. **Safety net is essential for a hard deadline.** If anything fails during upload to Albert, the staging area persists. You don't need to re-extract from Biovia (which is being decommissioned).
2. **Checksums provide deterministic proof.** The customer can validate that every file was migrated before Biovia is shut down.
3. **Extraction can start early.** Even during discovery, you can begin extracting files to staging. This de-risks the July 1 deadline.
4. **The cost of the staging area is trivial.** S3/Blob storage is pennies per GB/month.

### Unknowns to Validate

- [ ] Does the customer have Pipeline Pilot? If not, is direct Oracle DB access available?
- [ ] Total file count and volume (estimate: 10K-400K files, 50GB-10TB)
- [ ] File type distribution (PDFs, images, Word docs, proprietary instrument formats?)
- [ ] Biovia hierarchy mapping to Albert (how many projects, notebooks, experiments?)
- [ ] Access control model: are permissions at the project, notebook, or experiment level?
- [ ] Does Biovia's contract allow data extraction before subscription ends?

---

## B3. Inventory Migration (CISPro → Albert Inventory)

**Context:** One-time migration of structured data. Entities: Substances, Lots, Containers, Locations, Pricing, Attributes. Hard July 1 deadline.

**Critical constraint:** CISPro has **no public API**. Extraction via Pipeline Pilot or direct Oracle DB access. Data quality is the dominant risk — legacy chemical inventory systems accumulate orphaned records, duplicate substances, and inconsistent custom fields over decades.

### Option A: ETL Pipeline with Staging Database (RECOMMENDED)

Extract to a PostgreSQL staging database, run quality checks, get customer review, then load to Albert.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | **Data quality checkpoint before production load.** SQL-based quality checks: duplicate substance detection, orphan detection (lots without substances, containers without locations), CAS number validation, referential integrity verification. Customer reviews data quality report before anything hits Albert. Load in referential integrity order: Locations → Substances → Lots → Containers → Pricing → Attributes. Full rollback capability. |
| **Technical Cons** | Requires provisioning and managing a staging database. Two-hop process. More moving parts. |
| **Business Pros** | **Data quality report is a customer deliverable** that demonstrates thoroughness. Customer review checkpoint builds trust. If data quality is poor (likely), issues are caught BEFORE production — not discovered by end users after go-live. |
| **Business Cons** | Higher effort. Staging DB adds infrastructure. |
| **Effort** | **8-11 pw** |

**Key design details:**

- **Extraction:** Pipeline Pilot or direct Oracle SQL from CISPro. Dump all entities with their relationships.
- **Staging DB:** PostgreSQL. Schema mirrors Albert's target model. Transformation happens in staging.
- **Data quality checks (automated):**
  - Duplicate substances (matching on CAS number, name, or both)
  - Orphaned lots (lot records with no parent substance)
  - Orphaned containers (containers with no location or no lot)
  - Invalid CAS numbers (checksum validation)
  - Missing required fields per Albert's schema
  - Referential integrity (every foreign key resolves)
- **Customer review checkpoint:** Albert generates a data quality report. Customer reviews and decides: clean the data in CISPro first, or accept and clean in Albert, or exclude problem records.
- **Load order:** Must respect referential integrity:
  1. Locations
  2. Substances
  3. Lots (reference Substances)
  4. Containers (reference Lots + Locations)
  5. Pricing (reference Substances)
  6. Attributes (reference Substances/Lots/Containers)
- **Rollback:** If load fails partway through, truncate Albert inventory objects and reload from staging. Staging is the single source of truth until migration is validated.

### Option B: Direct Bulk Import via Albert SDK

Extract from CISPro, transform in memory, load directly via Albert SDK.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Simpler architecture. Fewer moving parts. Faster for small volumes (<10K substances). |
| **Technical Cons** | **No staging safety net.** Data quality issues hit Albert production directly. In-memory transformation doesn't scale for >100K records. No customer review checkpoint before load. No easy rollback. |
| **Business Pros** | Lower effort. Faster delivery. |
| **Business Cons** | Data quality issues discovered by end users, not by the migration team. Higher risk of rework. |
| **Effort** | **4-6 pw** |

### Option C: Hybrid with Shadow Load

Option A + sandbox validation + parallel loading to a test environment first.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Lowest risk. Customer validates in sandbox before production. Issues caught before they matter. |
| **Technical Cons** | Highest effort. Requires Albert to provide a sandbox environment (may not exist). |
| **Business Pros** | Maximum customer confidence. |
| **Business Cons** | Highest cost and longest timeline. May not be feasible against July 1 deadline. |
| **Effort** | **10-13 pw** |

### Recommendation

**Option A: ETL + Staging DB.**

Rationale:
1. **Data quality is the dominant risk.** CISPro data accumulated over years/decades. There WILL be duplicates, orphans, and inconsistencies. Discovering these in production is far more expensive than catching them in staging.
2. **Customer review checkpoint is essential for trust.** The customer sees and approves the data quality report before anything enters Albert. This prevents blame disputes ("your migration broke our data").
3. **Load order enforcement prevents referential integrity failures.** You can't create a Container in Albert that references a Lot that doesn't exist yet.
4. **Rollback from staging is clean.** If the Albert load fails, the staging DB is your recovery point. Without it, you'd need to re-extract from CISPro.

### Unknowns to Validate

- [ ] CISPro data volumes: how many substances, lots, containers, locations?
- [ ] Custom attributes/fields beyond standard CISPro schema?
- [ ] Data quality: has the customer already done any cleanup? Known issues?
- [ ] CAS number coverage: what % of substances have valid CAS numbers?
- [ ] Regulatory data (GHS/SDS): is it stored in CISPro or a separate system?
- [ ] Albert's Inventory module: what are the required vs. optional fields? Any limits on bulk creation?

---

## B4. Azure Data Lake Integration (Albert → Azure)

**Context:** Ongoing outbound integration. Port data from Albert's Data Warehouse to customer's Azure Enterprise Data Platform.

**Key technical detail:** Cross-tenant Azure auth requires a multi-tenant Service Principal. Azure Managed Identities do NOT work cross-tenant.

### Option A: Customer-Pull via ADF (RECOMMENDED)

Customer's ADF pulls data from Albert's REST API or Data Warehouse exports.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | **Enterprise standard pattern.** Customer's Azure team knows ADF. Albert provides documented incremental APIs + sample ADF pipeline template as reference. Customer owns and operates their own pipeline. Fits customer's existing medallion architecture (Bronze → Silver → Gold). |
| **Technical Cons** | Albert must expose stable, documented APIs for incremental data extraction. Customer must build/maintain the ADF pipeline (but this is their core competency). |
| **Business Pros** | **Least Albert-side effort.** Customer owns the pipeline — operational burden is on their Azure team. No security concern — Albert grants READ access, not WRITE. Scales to any number of Azure customers (each pulls independently). |
| **Business Cons** | Albert cannot guarantee freshness or correctness of the customer's pipeline. Customer blames Albert if their pipeline has bugs. |
| **Effort** | **2-4 pw** (Albert side: API documentation, sample ADF template, auth setup) |

**Key design details:**

- **Auth:** Register a multi-tenant Service Principal in Albert's Azure AD (Active Directory). Customer grants it read access to their ADLS Gen2. OR: Albert provides an API key/OAuth token for the customer's ADF to call Albert's Data Warehouse API.
- **Data format:** Albert exports as JSON (Bronze landing zone). Customer's ADF transforms to Parquet for Silver/Gold layers.
- **Incremental extraction:** Albert provides a "changes since timestamp" or "watermark" API endpoint. Customer's ADF polls at their preferred frequency (hourly, daily).
- **Sample ADF template:** Albert provides a documented ADF pipeline template (JSON ARM template) that the customer adapts.

### Option B: Vendor-Push (Albert Pushes to ADLS)

Albert writes Parquet files directly to the customer's ADLS Gen2 storage.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Albert controls the pipeline end-to-end. Can guarantee format, freshness, and correctness. Pre-formatted Parquet files land directly in the customer's Bronze zone. |
| **Technical Cons** | **Requires WRITE access to customer storage** — significant security concern for enterprise customers. Albert must implement and operate the push pipeline. Must handle customer's network restrictions (Private Link, IP whitelisting). |
| **Business Pros** | Customer gets pre-formatted data with no pipeline to build. "White glove" service. |
| **Business Cons** | Higher Albert effort. Ongoing operational burden. Security concern may be a non-starter for some customers. |
| **Effort** | **4-6 pw** |

### Option C: Event-Driven Streaming (Azure Event Hubs)

Real-time streaming of Albert events to Azure Event Hubs.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Near-real-time data freshness. Event-driven architecture is modern and scalable. |
| **Technical Cons** | **Overkill for R&D data volumes and freshness requirements.** R&D data changes are infrequent (experiments take days/weeks). Event Hubs adds complexity (schema registry, partitioning, consumer groups). |
| **Business Pros** | "Real-time" marketing. Future-proof architecture. |
| **Business Cons** | Highest cost and complexity for data that doesn't need real-time freshness. Event Hubs licensing cost. |
| **Effort** | **6-8 pw** |

### Recommendation

**Option A: Customer-Pull via ADF.**

Rationale:
1. **Enterprise standard.** Every enterprise Azure customer knows ADF. This pattern is familiar and trusted.
2. **Least Albert effort.** 2-4 pw vs. 4-8 pw. Albert's deliverable is documentation + sample template + auth setup.
3. **Avoids security concern.** Albert never needs WRITE access to customer infrastructure. Read-only API access is far easier to approve.
4. **Scales.** Each new Azure customer pulls from the same Albert API. Albert doesn't need to build and operate a push pipeline per customer.

### Unknowns to Validate

- [ ] Does Albert's Data Warehouse module already have a REST API for incremental extraction?
- [ ] What data entities does the customer want in their data lake? (All Albert data? Just experiments and results? Just inventory?)
- [ ] Customer's current ADF maturity — do they have established patterns, or is this new for them?
- [ ] Network path: can customer's ADF reach Albert's API? Any IP whitelisting or Private Link requirements?

---

## B5. Connector Framework / Productization

**Context:** How to design the first LIMS integration so the NEXT one takes significantly less time. Maps directly to Interview Section 3 (Increasing Velocity Over Time).

### Option A: Configuration-Driven Adapter Framework (RECOMMENDED)

Abstract the integration into a shared framework + platform-specific adapters + YAML config files.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | `LIMSConnector` interface defines the contract: `submitRequest()`, `pollResults()`, `mapFields()`, `handleError()`. Platform-specific adapters (`LabVantageAdapter`, `StarLIMSAdapter`, etc.) implement the interface. YAML config files define: endpoints, auth, field mappings, polling intervals, retry policies. Shared framework provides: scheduler, retry engine with exponential backoff, DLQ, monitoring/health checks, correlation ID tracking, logging. |
| **Technical Cons** | Requires upfront design of the abstraction layer. Risk of wrong abstractions if built before real-world usage. |
| **Business Pros** | **Reduces next LabVantage customer from 6-8 weeks to 2-3 weeks** (new YAML config + testing only). Enables Tech Ops / Customer Success to deploy without engineering involvement. Builds toward AlbertHub vision. Creates reusable IP. |
| **Business Cons** | 4-6 pw investment post-first-customer. Won't benefit the first customer directly. |
| **Effort** | **4-6 pw** (post-first-customer, when you know what to abstract) |

**Six concrete artifacts:**

1. **Connector templates** — Boilerplate code for new LIMS adapters with the interface pre-implemented
2. **Field mapping configs** — YAML/JSON files mapping Albert entity fields to LIMS entity fields (one per customer)
3. **Deployment playbooks** — Step-by-step guides for provisioning, configuring, and deploying a connector
4. **Test harnesses** — Automated test suites that validate any connector against the `LIMSConnector` contract
5. **Monitoring runbooks** — Standardized dashboards and alerting rules per connector
6. **Developer documentation** — How to build a new adapter, how the framework works, API reference

### Option B: SDK Extension Model

Publish base classes in the Albert SDK that developers subclass for each LIMS platform.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Leverages existing Albert SDK distribution. Developers familiar with the SDK can extend it. Lighter-weight than a full framework. |
| **Technical Cons** | Still requires a developer per customer. No configuration-driven deployment. Less standardized than Option A. |
| **Business Pros** | Lower investment. Faster to ship. Enables partners/SIs (System Integrators) to build connectors. |
| **Business Cons** | Each customer still needs custom code. Doesn't enable non-developers to deploy. |
| **Effort** | **2-3 pw** |

### Option C: Template-Based Code Generation

CLI tool that generates a complete integration project from templates.

| Dimension | Assessment |
|-----------|-----------|
| **Technical Pros** | Fast project scaffolding. Consistent starting point. Includes boilerplate for auth, polling, retry, monitoring. |
| **Technical Cons** | **Generated code diverges from templates over time.** Bug fixes in the generator don't propagate to previously generated projects. Creates maintenance burden. |
| **Business Pros** | Fast onboarding for new integration engineers. |
| **Business Cons** | Doesn't reduce per-customer effort as much as Option A. Divergence creates inconsistency across customers. |
| **Effort** | **4-5 pw** |

### Recommendation

**Option A, built incrementally AFTER the first customer.**

The strategy:
1. **First customer (now):** Build Option B from Section B1 — a clean, well-structured custom integration service. ~6-8 pw.
2. **After go-live:** Refactor into the adapter framework. Extract the generic components (scheduler, retry, DLQ, monitoring) into the shared framework. Extract LabVantage-specific logic into the `LabVantageAdapter`. Extract field mappings into YAML config. ~4-6 pw.
3. **Second customer:** Deploy by writing a new YAML config + running the test harness. ~2-3 pw instead of 6-8 pw.

This avoids premature abstraction while delivering the productization the JD demands.

### Unknowns to Validate

- [ ] How many LIMS integration requests are in Albert's pipeline? (Determines urgency of productization)
- [ ] Which other LIMS platforms do Albert customers use? (LabWare, STARLIMS, SampleManager — determines adapter breadth)
- [ ] Does AlbertHub already have any framework or patterns that this should extend?
- [ ] Who will maintain connectors long-term: Albert engineering, Tech Ops, Customer Success, or partners?

---

# PART C: Productization Lens

*Connects the JD's language to concrete deliverables from this project*

The JD for Principal Enterprise Engineer uses specific phrases that signal what Albert expects from this role. Here's how each phrase maps to work products from this case study:

| JD Phrase | What It Means | This Project's Deliverable |
|-----------|--------------|---------------------------|
| **"Configuration-driven features and self-service tooling"** | Replace custom code with config files that non-engineers can modify | YAML connector configs (B5-A): field mappings, polling intervals, endpoints defined in config, not code |
| **"Playbooks and tooling for repeatable go-live migrations"** | Standardize the migration process so it's reproducible | Migration runbooks from ELN (B2) and Inventory (B3): step-by-step extraction, staging, validation, load, verification playbooks |
| **"Standardized environments, pipelines, and versioning models"** | Consistent infrastructure and deployment across customers | CI/CD for connectors, staging DB pattern from B3, containerized deployment from B1-B |
| **"Guardrails, best practices, and operational processes"** | Prevent mistakes, catch issues early, define how things are done | Go/No-Go gates (A4), data quality checkpoints (B3), scope freeze dates (A4), deemed-accepted clauses (A4) |
| **"Developer documentation and examples"** | Enable others to build integrations without tribal knowledge | ADF pipeline template (B4), Albert SDK usage guides, connector developer docs (B5-A artifact #6) |
| **"SDKs/APIs and integration patterns"** | Reusable code and architectural patterns | `LIMSConnector` interface (B5-A), `LabVantageAdapter` reference implementation, Albert SDK patterns for each module |
| **"Productization of engineering tools"** | Turn internal one-offs into repeatable, scalable products | The entire B → C progression: build one, extract the pattern, make it repeatable. Every bespoke integration should produce a reusable artifact |

### The Productization Progression

```
Customer 1 (this project)          Customer 2              Customer N
─────────────────────────          ──────────              ──────────
Custom code (B1-B)           →     Config + adapter (B5-A)  →  Self-service (AlbertHub UI)
Manual migration scripts     →     Migration playbook       →  Guided migration wizard
Custom ADF template          →     Standard template lib    →  One-click data lake setup
Ad-hoc SOW                   →     SOW template library     →  Standard pricing/packaging
```

Each customer engagement should leave behind:
1. A **working integration** for that customer
2. A **reusable artifact** that makes the next customer faster
3. **Documented learnings** that update the playbooks and templates

This is the "integration factory" vs. "building integrations" mindset.

---

# Summary

## Effort Summary

| Workstream | Recommended Option | Effort | July 1 Critical? |
|-----------|-------------------|--------|-------------------|
| WS1: ELN Migration | A: Bulk Export + Staging | 6-8 pw | **Hard deadline** |
| WS2: LIMS Integration | B: Custom Service (→ refactor to C) | 6-8 pw (build) + 4-6 pw (productize) | Desired, not hard |
| WS3: Azure Data Lake | A: Customer-Pull via ADF | 2-4 pw | Flexible |
| WS4: Inventory Migration | A: ETL + Staging DB | 8-11 pw | **Hard deadline** |
| Discovery (shared) | Separate fixed-price engagement | 3-4 pw | Must complete first |
| **Implementation Total** | | **22-31 pw** | |
| Connector Framework (post-v1) | A: Config-Driven Adapter | 4-6 pw | N/A |
| **Grand Total** | | **29-41 pw** | |

## Calendar Timeline

With 2-3 parallel engineers + Principal Engineer lead, the implementation phase is approximately **4-5 months** from scope lock.

From a mid-February start:
- **Feb-Mar:** Discovery (3-4 weeks)
- **Mar-Apr:** Design + early build (migrations first)
- **Apr-Jun:** Build + test (all workstreams in parallel)
- **Late Jun:** Deploy migrations (ELN, Inventory) — **July 1 cutover**
- **Jul-Aug:** Deploy integrations (Azure, LIMS) + hypercare

**Critical path risk:** July 1 is tight for 2 migrations starting from a mid-February engagement. Discovery must be efficient, and the customer must meet their obligations on time.

## Quick Reference: All Design Choices

| # | Design Choice | Options Evaluated | Recommended | Key Risk |
|---|--------------|-------------------|-------------|----------|
| B1 | LIMS Integration Pattern | iPaaS / Custom Service / Adapter Framework | Custom Service first, refactor to Adapter | LabVantage API maturity unknown; no webhooks forces polling |
| B2 | ELN File Migration | Bulk+Staging / Streaming / Hybrid | Bulk + Blob Staging | No public Biovia API; file volume unknown |
| B3 | Inventory Migration | ETL+Staging / Direct SDK / Shadow Load | ETL + Staging DB | CISPro data quality; referential integrity |
| B4 | Azure Integration | Customer-Pull / Vendor-Push / Event Streaming | Customer-Pull via ADF | Customer Azure maturity; API stability |
| B5 | Connector Framework | Config-Driven / SDK Extension / Code Gen | Config-Driven Adapter (post-v1) | Premature abstraction; unknown LIMS diversity |

## Red Flags to Watch

| Flag | Severity | Why |
|------|----------|-----|
| "We aren't sure what data needs to flow where" | Critical | Customer hasn't done internal requirements analysis; discovery will take longer |
| July 1 deadline for 2 migrations (~4.5 months away) | High | After 3-4 week discovery, only ~12 weeks for design+build+test+deploy |
| "Moving analytical teams was perceived to be too high a change" | High | Signals organizational resistance; LIMS integration success depends on adoption |
| First-of-kind LIMS integration | High | No reference architecture, no reusable code, no lessons learned at Albert |
| Biovia vendor lock-in | Medium | No public APIs; extraction may require vendor assistance or direct DB access |
| 4 workstreams, 1 Principal Engineer | Medium | Resource contention; need dedicated leads per workstream |
