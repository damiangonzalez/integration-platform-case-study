# Deep Technical Research: Systems & Integration Patterns

**Purpose:** Detailed technical research on the specific systems, APIs, data models, and integration patterns relevant to the Albert Invent case study. This document informs architecture decisions for the 4 workstreams.

**Prerequisite:** Read `s0-context-primer.md` first for foundational domain knowledge and acronym definitions.

**Note on background context files:** The speculative analyses in `../technical-analysis.md` and `../platform-architecture-and-state.md` were written before the case study prompt was received. They contain reasonable hypotheses about Albert's integration maturity but are NOT confirmed ground truth. This document relies on verified public research.

---

## Table of Contents

1. [LabVantage LIMS](#1-labvantage-lims)
   - [1.1 REST API](#11-rest-api)
   - [1.2 SOAP Web Services](#12-soap-web-services)
   - [1.3 Data Model](#13-data-model)
   - [1.4 Result Delivery Patterns](#14-result-delivery-patterns)
   - [1.5 Authentication & Security](#15-authentication--security)
   - [1.6 Known Limitations & Integration Gotchas](#16-known-limitations--integration-gotchas)
   - [1.7 Connector Ecosystem](#17-connector-ecosystem)
2. [Biovia ELN](#2-biovia-eln)
   - [2.1 Product Portfolio](#21-product-portfolio)
   - [2.2 File & Attachment Storage Architecture](#22-file--attachment-storage-architecture)
   - [2.3 Export Capabilities](#23-export-capabilities)
   - [2.4 Migration Approaches](#24-migration-approaches)
   - [2.5 Volume Expectations](#25-volume-expectations)
   - [2.6 Access Control Model](#26-access-control-model)
3. [Biovia CISPro](#3-biovia-cispro)
   - [3.1 Product Overview](#31-product-overview)
   - [3.2 Data Model](#32-data-model)
   - [3.3 Available Chemicals Directory (ACD) Integration](#33-available-chemicals-directory-acd-integration)
   - [3.4 Export Capabilities](#34-export-capabilities)
   - [3.5 CISPro → Albert Inventory Mapping](#35-cispro--albert-inventory-mapping)
   - [3.6 Regulatory & Compliance Data](#36-regulatory--compliance-data)
4. [Azure Enterprise Data Platform](#4-azure-enterprise-data-platform)
   - [4.1 SaaS-to-Azure Integration Patterns](#41-saas-to-azure-integration-patterns)
   - [4.2 ADF Pipeline Patterns](#42-adf-pipeline-patterns)
   - [4.3 Cross-Tenant Authentication](#43-cross-tenant-authentication)
   - [4.4 Data Formats & Landing Patterns](#44-data-formats--landing-patterns)
   - [4.5 Albert's Data Warehouse (Inferred Capabilities)](#45-alberts-data-warehouse-inferred-capabilities)
5. [Materials Science Lab Data Workflows](#5-materials-science-lab-data-workflows)
   - [5.1 Test Request Submission (Albert → LabVantage)](#51-test-request-submission-albert--labvantage)
   - [5.2 Test Result Return (LabVantage → Albert)](#52-test-result-return-labvantage--albert)
   - [5.3 Typical Turnaround Times](#53-typical-turnaround-times)
   - [5.4 Volume of Test Requests](#54-volume-of-test-requests)
   - [5.5 Integration Design Implications](#55-integration-design-implications)
6. [Inventory Data in Materials Science](#6-inventory-data-in-materials-science)
   - [6.1 Data Model](#61-data-model)
   - [6.2 Typical Volumes](#62-typical-volumes)
   - [6.3 Lot Tracking & Qualification](#63-lot-tracking--qualification)
   - [6.4 Regulatory Data per Substance](#64-regulatory-data-per-substance)
   - [6.5 Virtual vs Physical Inventory](#65-virtual-vs-physical-inventory)
7. [Key Findings for Architecture Decisions](#key-findings-for-architecture-decisions)
   - [For the LIMS Integration (Workstream 2)](#for-the-lims-integration-workstream-2)
   - [For the ELN Migration (Workstream 1)](#for-the-eln-migration-workstream-1)
   - [For the Inventory Migration (Workstream 4)](#for-the-inventory-migration-workstream-4)
   - [For the Azure Integration (Workstream 3)](#for-the-azure-integration-workstream-3)

---

## 1. LabVantage LIMS

### 1.1 REST API

#### URL Structure

LabVantage exposes REST services at:

```
https://<hostname>:<port>/<webapp>/rest/...
```

Core endpoint categories:

| Endpoint Pattern | Purpose |
|---|---|
| `/rest` | API operational status check (no auth required) |
| `/rest/api` | Built-in API documentation (reflects enabled RESTPolicy resources) |
| `/rest/connections` | Session/connection management |
| `/rest/actions` | System Action execution (business logic) |
| `/rest/sdc/{resource}` | SDC (Structured Data Container) resource CRUD -- the primary integration surface |

#### SDC Resource Endpoints

This is the core pattern for working with LIMS entities:

```
/rest/sdc/{resource}                                              # GET list, POST create
/rest/sdc/{resource}/{resourceid}                                 # GET, PUT, DELETE single item
/rest/sdc/{resource}/{resourceid}/{subresource}                   # Sub-resources
/rest/sdc/{resource}/{resourceid}/{subresource}/{subresourceid}   # Specific sub-resource
```

**Concrete examples for the Albert ↔ LabVantage integration:**

```
GET    /rest/sdc/samples?queryid={queryid}              # List samples via named query
GET    /rest/sdc/samples/{sampleid}                      # Get single sample
POST   /rest/sdc/samples                                 # Create sample (JSON body)
PUT    /rest/sdc/samples/{sampleid}                      # Update sample

GET    /rest/sdc/samples/{sampleid}/tests                # List tests for a sample
POST   /rest/sdc/samples/{sampleid}/tests                # Add tests to a sample
PUT    /rest/sdc/samples/{sampleid}/tests/{testid}       # Update a test
DELETE /rest/sdc/samples/{sampleid}/tests/{testid}       # Remove a test
```

#### Request/Response Format

**JSON exclusively.** No XML on the REST API. POST bodies are JSON key-value pairs. All responses return JSON with HTTP status codes.

For **POST (create)**: Body contains column values. Response returns created resource data plus available Action call definitions.

For **PUT (update at root)**: Body contains a collection of keys and data values with resource paths -- supports **bulk updates** in a single request.

#### Querying and Filtering

LabVantage uses server-side **Query objects** (not standard URL query parameters):

- `?queryid={queryid}` -- reference a predefined LabVantage Query
- `?param1=val&param2=val` -- pass parameters to the query
- `?keyid1=val1;val2;val3` -- key-list based retrieval (semicolon-separated)
- `?fields=col1,col2` -- custom column selection
- `?queries=Y` -- list available queries and their documentation

**No standard offset/limit pagination** is documented. Filtering is done through server-side Query objects configured by the LabVantage admin.

#### Actions API (Business Logic Execution)

```
GET    /rest/actions                    # List all available actions
GET    /rest/actions/{actionid}         # Get action properties/schema
POST   /rest/actions                    # Execute action block (JSON)
POST   /rest/actions/{actionid}         # Execute specific action with properties
```

**This is important:** Creating samples with full business logic (auto-ID generation, test template application, workflow triggering) may require calling Actions rather than direct SDC POST. The Actions endpoint invokes the same business logic the UI uses.

#### RESTPolicy -- Critical Security Layer

The REST API operates under a **deny-by-default** security model controlled by RESTPolicy configuration:

- Every SDC resource, sub-resource, HTTP method, and returnable column must be **explicitly enabled**
- Each HTTP method can be mapped to a specific LabVantage System Action
- **Restrictive Where** clauses can limit which records an API consumer sees (e.g., only samples of type 'QC')
- Column-level enforcement with three modes: Ignore (silently filter), Error (reject), Permissive (allow all)
- Data masking based on user roles

**Integration implication:** The LabVantage admin must configure RESTPolicy nodes specifically for the Albert integration, defining exactly which resources, methods, columns, and queries are exposed.

---

### 1.2 SOAP Web Services

LabVantage provides two SOAP implementations:

#### SapphireWS (Apache Axis)

**WSDL:** `http://<host>:<port>/<webapp>/services/SapphireWS?wsdl`

Requires stub generation (produces 19 Java source files). Uses Transport Beans for complex data types.

| Category | Method | Description |
|---|---|---|
| Connection | `getConnectionId(databaseid, userid, password)` | Obtain session |
| Connection | `checkConnection(connectionid)` | Validate session |
| Connection | `clearConnection(connectionid)` | Close session |
| Data | `getSqlDataSet(connectionid, sql)` | **Execute raw SQL, return XML DataSet** |
| Data | `getSDIData(connectionId, SDIRequestTransportBean)` | Query SDI entities |
| Actions | `processAction(connectionid, actionid, versionid, propertyListXML)` | Execute single action |
| Actions | `processActionBlock(connectionid, ActionBlockTransportBean)` | Execute action sequence |
| Actions | `processMessage(connectionid, BaseSECMessage, processingMode)` | Enterprise Connector messages (async/sync/manual) |
| Attachments | `getSDIAttachment()`, `addSDIAttachment()`, `editSDIAttachment()`, `deleteSDIAttachment()` | Document management |
| Utility | `getPublicKey()` | RSA encryption key |

#### SapphireBasicWS (JAX-WS)

**WSDL:** `http://<host>:<port>/<webapp>/services/SapphireBasicWS?wsdl`

No stub generation required. All methods accept only **primitive types** (WS-I compliant). Same core operations but without Transport Beans or attachment handling.

#### Axis vs JAX-WS Comparison

| Aspect | SapphireWS (Axis) | SapphireBasicWS (JAX-WS) |
|---|---|---|
| Stub generation | Required | Not required |
| Data types | Transport Beans + XML strings | Primitive strings only |
| Attachment handling | Yes | Not documented |
| Client complexity | Higher | Lower |
| .NET interop | Good (Transport Beans) | Good (primitives) |

#### SOAP-Only Capabilities

Operations available via SOAP but **not REST**:
- `getSqlDataSet()` -- raw SQL execution against LabVantage database
- `processMessage()` -- Enterprise Connector message handling with sync/async modes
- Attachment CRUD operations (Axis only)
- `getPublicKey()` -- RSA key for password encryption
- `getSequence()` -- sequence number management

---

### 1.3 Data Model

#### Core Architecture: SDCs and SDIs

LabVantage uses a **metadata-driven architecture**:

- **SDC (Sapphire Data Collection):** A class/entity definition spanning one or more database tables (e.g., "Sample", "Batch", "User")
- **SDI (Sapphire Data Item):** An individual record/instance belonging to an SDC (e.g., sample "SAM-001")

Core metadata tables:

| Table | Purpose |
|---|---|
| `SDC` | Master entity definitions |
| `SysTable` | Maps SDCs to implementation tables |
| `SysColumn` | Data dictionary layer |
| `SDCLink` | Inter-SDC relationships (types, foreign keys) |

SDC types: **Core/System** (read-only, out-of-box), **User** (fully customizable), **Definition** (read-only but extendable).

#### The Critical Hierarchy: Sample → Test → DataSet → DataItem

This is the entity chain that matters for the Albert ↔ LabVantage integration:

```
Sample (SDI)
  └── Test (SDIWorkitem) ── instance of a Test Method (Workitem template)
        └── DataSet (SDIData) ── instance of a Parameter List (defines what to collect)
              └── DataItem (SDIDataItem) ── individual measured value/parameter
```

**Terminology mapping:**
- Test Method = Workitem (template/definition of what to test)
- Test = SDIWorkitem (instance applied to a specific Sample)
- Parameter List = template defining which data items to collect
- DataSet = SDIData (instance of a Parameter List on a Test)
- DataItem = SDIDataItem (the actual individual result value)

#### Sample Entity

Key fields:
- `sampleid` -- unique identifier (auto-generated or user-specified)
- `sampletypeid` -- classification (QC, Raw Material, Finished Goods)
- `productid` -- associated product
- `batchid` -- associated batch
- `status` -- lifecycle state
- Department assignments (assigned, work area, testing department)
- External Sample ID + External ID Type (for cross-system correlation)

**Sample lifecycle:**
```
Initial → Received → InCirculation → InProgress → Completed → Reviewed → Disposed
```
Also: `Cancelled`, `NotReceived`

#### Test Status Transitions

```
Initial → InProgress → DataEntered → Released → Completed
```
Also: `Cancelled`

- **InProgress:** Partial mandatory data entered
- **DataEntered:** All mandatory data entered
- **Released:** All mandatory DataItems released
- **Completed:** All DataSets approved

**Status rolls up:** DataItem → DataSet → Test → Sample → Batch. For a Sample to be "Completed," all its Tests and DataSets must be "Completed."

#### Result (DataItem) Structure

Each DataItem carries:
- The measured value/result
- Method association (via Parameter List)
- Instrument ID
- Specification limits (from Product)
- Quality flags: **OOS** (Out of Specification), **OOT** (Out of Trend)
- External Reference field (for audit trail back to source system -- **critical for Albert correlation**)
- Release status, approval status
- `blockFlag` -- prevents duplicate processing (used by Empower connector pattern)
- Analyst assignment and certification

**Three operational levels:**
1. **Release** -- applied at DataItem level (signals readiness for approval)
2. **Approval** -- applied at DataSet level (indicates review readiness)
3. **Review** -- applied at Sample level (quality examination)

#### Batch Entity

- Each Batch is associated with one Product
- Product definition determines required Samples and Tests via a **Sampling Plan**
- During Batch creation, Samples auto-generate from the Product's plan

**Batch lifecycle:**
```
Initial → Received → Active → PendingRelease → Released
```
Also: `Rejected`, `OnHold`, `PreliminaryRelease`

Auto-transitions: `Received → Active` (when any sample moves to Received/InProgress), `Active → PendingRelease` (when all samples finish), `PendingRelease → Released` (auto-release if enabled and all specs pass).

---

### 1.4 Result Delivery Patterns

#### No Native Webhooks

**LabVantage does NOT natively support webhooks or push-based callbacks to external systems.** There is no webhook registration endpoint or callback URL configuration.

#### Options for Getting Results Back to Albert

**Option 1: Albert Polls LabVantage REST API (Simplest)**
```
GET /rest/sdc/samples?queryid={query_for_completed_samples}&param1={since_timestamp}
GET /rest/sdc/samples/{sampleid}/tests
```
- Requires a LabVantage Query configured to filter by status and/or date range
- Albert manages polling interval and watermark tracking
- Latency = polling interval (e.g., 5-15 minutes)

**Option 2: LabVantage Event Service (Server-Side Automation)**
- LabVantage has a Scheduler Module and Event Service that can automate server-side actions
- A poller runs on a configurable interval, checks for status changes, and invokes actions
- Could be configured to call Albert's API when results are approved
- Requires **LabVantage-side configuration/customization**

**Option 3: Custom LabVantage Action on Approval**
- Build a custom System Action that fires on test/sample completion
- The action sends an outbound REST call to Albert's webhook/API
- Most real-time option but requires LabVantage development

**Option 4: Sapphire Enterprise Connector (SEC)**
- LabVantage's primary bidirectional enterprise integration mechanism
- Certified with SAP (IDoc messaging via SAP NetWeaver PI)
- Uses `processMessage()` SOAP operation with async/sync/manual modes
- Could be adapted for Albert, but SEC is primarily designed for SAP integration

#### Reference Pattern: Waters Empower Connector

The best-documented bidirectional integration in LabVantage:

**Download (LabVantage → Empower):** Pending samples mapped to Empower SampleSetMethods. Sample attributes flow out. Blocking mechanism (`blockFlag = Y`) prevents duplicates.

**Upload (Empower → LabVantage):** Triggered by signoff events in Empower. Results persisted to SDIDataItems. External Reference metadata maintained for full audit trail:
```
"project=<project>; database=<database>; samplesetname=<sampleset>;
 resultsetid=<resultsetid>; resultid=<resultid>; channelid=<channelid>; Added=<Y>"
```
Reupload: previously auto-created items deleted before re-upload. Released result handling: configurable (Error/Ignore/Override).

---

### 1.5 Authentication & Security

#### Token-Based Authentication (Recommended for Integration)

**Setup process:**
1. Register an **External Application** (e.g., "AlbertInvent")
2. Create an **External App User** linked to that application
3. Generate an **Auth Token** tied to that user

**External Application configuration:**
- Status: Active/Disabled (kill switch for the integration)
- REST WebServices: Enable + assign specific RESTPolicy Node
- SOAP WebServices: Enable/disable separately
- Default Token Expiry Days

**Token characteristics:**
- API-only access (web UI blocked)
- Sessions **do not time out** like normal user sessions
- Configurable expiration date
- Restrict to specific protocol (SOAP, REST, or Controller only)
- **Does NOT consume named or concurrent user licenses**
- External apps never store username/password -- only the token

**Token delivery methods:**
1. Query parameter: `?token=xxx`
2. Cookie: `token=xxx`
3. HTTP Authorization header: `"Token xxxx"`

**Process-As impersonation options:**
- **Never:** Actions execute as the External App User
- **Specific User:** Token always processes as a predetermined user
- **Department/Role-Based:** Restrict allowable users to specific departments/roles

#### ConnectionId-Based (Alternative)

```
POST /rest/connections
Body: { "databaseid": "labdb", "username": "svc_albert", "password": "..." }
Response: ConnectionId (e.g., "lab060040jbs512|(system)1669603006983")
```

Transmitted via Cookie, Authorization header, or query parameter. Requires periodic heartbeats.

#### Service Account Pattern

```
External Application: "AlbertInvent" (REST enabled, dedicated RESTPolicy node)
   └── External App User: "svc_albert" (role: IntegrationService, dept: QualityLab)
         └── Auth Token: "abc123..." (active, expires 2026-12-31, REST only)
```

---

### 1.6 Known Limitations & Integration Gotchas

| Issue | Impact |
|---|---|
| **No pagination** | Large result sets from queries may be difficult to handle incrementally. No standard offset/limit. |
| **No bulk POST** | Cannot create multiple samples atomically in one REST request. |
| **Non-atomic sample + test creation** | Creating a sample and adding tests requires separate HTTP requests. Use `POST /rest/actions` (action block) for sequential chaining. |
| **Deny-by-default RESTPolicy** | Every endpoint, method, column, and query must be explicitly enabled. Misconfiguration = silent 403s. |
| **No WebSocket / real-time feeds** | All external consumption is request/response. No streaming. |
| **Status rollup cascading** | Updating a DataItem triggers DataSet → Test → Sample → Batch recalculation. Side effects must be understood. |
| **Blocking flags needed** | The Empower pattern uses `blockFlag` to prevent duplicate processing. Albert integration should implement similar. |
| **Test addition vs. application** | Creating a test (AddTest) is distinct from applying it (which creates DataItem records). Both steps needed. |
| **Session capacity** | 1 BCU (4 cores, 16GB RAM) supports ~25-80 concurrent HTTP sessions. High-frequency polling could exhaust capacity. Client-side throttling required. |
| **No CDC mechanism** | No database-level Change Data Capture. Polling with watermarks is the only option for detecting changes. |

---

### 1.7 Connector Ecosystem

| Connector | Type | Data Flow |
|---|---|---|
| **SAP (SEC)** | Enterprise Connector | Bidirectional. IDoc messaging via SAP NetWeaver PI. Certified. |
| **Waters Empower** | Chromatography Data System | Bidirectional. Download samples, upload results. Signoff-triggered. |
| **Tulip** | Manufacturing Execution | REST API connector. Shop floor to LIMS. |
| **LIMS CI (Talend-based)** | Complex Instruments | 200+ drag-and-drop ETL components. File or TCP/IP. |
| **SDMS** | Scientific Data Management | Embedded. Attachment handlers via Talend or Java jobs. |

No formal "App Store" or connector marketplace. Custom integrations built using REST/SOAP APIs.

Cloud vs On-Premise: Same API surface across deployment models. Cloud/SaaS may have stricter security policies. `getSqlDataSet()` may be restricted in SaaS.

---

## 2. Biovia ELN

### 2.1 Product Portfolio

Biovia offers three ELN products (see `s0-context-primer.md` for full descriptions):

| Product | Architecture | Key Characteristic |
|---------|-------------|-------------------|
| **BIOVIA Notebook** | Web-based (ScienceCloud or on-premises) | Simple, rapid adoption |
| **BIOVIA Workbook** | On-premise/hosted | Heavily configurable, multi-discipline |
| **BIOVIA Scientific Notebook** | Cloud-native (3DEXPERIENCE platform) | Can index records from Workbook, Notebook, and third-party ELNs |

### 2.2 File & Attachment Storage Architecture

**Organizational hierarchy:**
```
Projects → Notebooks → Experiments → Sections → Attachments
```

**Supported file formats stored as attachments:**
- Chemical structures: MOL, SDF, RXN, RD, RDF
- Biological sequences: FASTA, HELM, XHELM
- Data files: CSV, TXT, Excel-compatible
- Visualization: PDB, CIF, XYZ
- General: images, PDFs, Office documents, instrument data files

**Storage implementation:** Not publicly documented. Enterprise ELNs of this class typically use a **relational database (Oracle or SQL Server) for metadata with BLOB storage or a managed file store for binary attachments**. Given Dassault Systèmes' 3DEXPERIENCE platform runs on Oracle, Oracle BLOB or Oracle-managed file store is the likely implementation for on-premises. Cloud deployments likely use object storage.

**BIOVIA Desktop Connector:** A utility that bridges local file systems with BIOVIA applications, enabling upload, edit, and management of files including Office documents directly within the ELN.

### 2.3 Export Capabilities

**Critical finding: No publicly documented REST API for bulk data extraction exists.**

**Available extraction approaches:**

| Approach | Description | Suitability for Bulk Migration |
|----------|-------------|-------------------------------|
| **Pipeline Pilot protocols** | BIOVIA's ETL/integration platform. Database integration via ODBC/JDBC, SOAP web services, command-line, Java/Perl components. Can build custom extraction workflows. | **Best option** -- programmatic, repeatable, handles chemical structure data |
| **Direct database access (SQL)** | Query the underlying Oracle/SQL Server database directly. Requires knowledge of schema. | **Good fallback** -- requires schema documentation from Dassault or consultant |
| **Scientific Notebook indexing** | Use Scientific Notebook's built-in indexer to consolidate from legacy ELNs | **Intermediate step** -- consolidation within BIOVIA ecosystem, not migration out |
| **PDF/HTML export** | Individual experiment export as signed PDF | **Not suitable for bulk** -- manual, one-at-a-time |

**Pipeline Pilot capabilities relevant to extraction:**
- Database integration via standard SQL with ODBC/JDBC (Oracle, SQL Server)
- Web services integration via SOAP
- Command-line, SSH, FTP protocols
- Text mining including PDF text extraction
- Pre-built cheminformatics components for chemical structure handling
- Can create repeatable "protocols" (workflows)

### 2.4 Migration Approaches

**How other companies have migrated off Biovia ELN:**

| Approach | Description | When to Use |
|----------|-------------|-------------|
| **Start fresh, pull files as needed** | Minimal migration. Only bring specific files forward when actively needed. | Low urgency, small archive, clean break preferred |
| **Prioritize active projects** | Migrate only files from currently active projects first. Historical data stays read-only in old system or gets migrated later. | Time pressure, Pareto principle (20% of projects = 80% of value) |
| **Bulk migration of prioritized data** | Extract and load file archives in batches, organized by project/team | **Most relevant to the case study** -- Biovia is going away, all files must move |
| **Scientific Notebook as consolidation layer** | Index legacy data in Scientific Notebook first, then migrate from there | When customer has Pipeline Pilot and Scientific Notebook licenses |

**Key migration challenges (specific to Biovia ELN → Albert Notebook):**
- **Organizational mapping:** How do Biovia Projects/Notebooks map to Albert Projects? This requires customer collaboration to define the mapping.
- **Access control mapping:** Biovia project-level groups → Albert Teams. Roles (Author, Reviewer, Admin, Read-Only) → Albert permissions.
- **Metadata preservation:** Timestamps, authors, electronic signatures must be preserved or archived.
- **File volume:** Could range from 50GB to 10TB+ depending on whether analytical instrument data is stored inline.
- **Chemical structure files:** MOL/SDF files require special handling to preserve stereochemistry and enhanced structural data.

**Consulting firms specializing in Biovia migrations:**

| Firm | Specialty |
|------|-----------|
| **Astrix Technology Group** | Pipeline Pilot integration, BIOVIA consulting |
| **CSols Inc.** | BIOVIA ELN implementation, change management (1,500+ user deployments) |
| **Kalleid** | Chemistry data migration, structure format conversion |
| **Zifo RnD Solutions** | ELN software services, system management |
| **TECHNIA** | BIOVIA reseller/implementer |

### 2.5 Volume Expectations

No precise public benchmarks exist. Estimates based on case studies and industry knowledge:

| Metric | Small-Medium Company | Large Enterprise |
|--------|---------------------|-----------------|
| **Active ELN users** | 50-200 | 500-2,000+ |
| **Years of data** | 5-10 | 10-20+ |
| **Total experiments** | 12,500-100,000 | 100,000-400,000+ |
| **Attachments per experiment** | 2-10 | 2-10 |
| **Total file archive** | 50-500 GB | 1-10+ TB |
| **Individual file sizes** | 10 KB - 500 MB | 10 KB - 500 MB |
| **File types** | PDFs, images, spectra, chromatograms, Excel, chemical structure files | Same + raw instrument data, videos |

### 2.6 Access Control Model

**Permission structure:**
- Managed at the **project level and group level**
- Groups/teams defined; users assigned to groups
- Hierarchy follows: Project > Notebook > Experiment

**Typical roles:**

| Role | Permissions |
|------|-------------|
| **Author/Owner** | Create, edit, own experiments |
| **Co-Author** | Edit experiments alongside primary author |
| **Reviewer/Witness** | Review, countersign, attest to content (critical for IP/patent) |
| **Administrator** | System-wide configuration, user management, templates |
| **Read-Only** | View only |

**Mapping to Albert:**
- Biovia Project Groups → Albert Teams
- Biovia Author/Reviewer/Admin roles → Albert user permissions
- For a files-only migration, access control mapping is simplified -- files land in the correct Albert project with appropriate read access for the team

---

## 3. Biovia CISPro

### 3.1 Product Overview

CISPro (Chemical Inventory System Pro) is a web-based, multi-site chemical and material inventory management system. It tracks chemicals through their complete lifecycle: receipt → verification → approval → storage → dispensing → expiry → disposal.

Deployed on ScienceCloud (cloud) or on-premises. Part of BIOVIA ONE Lab / Unified Lab Management suite.

**Integration ecosystem:** Connects to BIOVIA Notebook ELN, third-party LIMS/LES, BIOVIA Available Chemicals Directory (ACD), fire code reporting, SDS management, regulatory compliance tools, and lab instruments.

### 3.2 Data Model

#### Core Entities

| Entity | Description | Key Fields |
|--------|-------------|-----------|
| **Substance** | Chemical compound identity | CAS number, chemical name (IUPAC + common), molecular formula, molecular weight, chemical structure (searchable via ACD), hazard classification (GHS), storage class, physical state, regulatory list memberships |
| **Container** | Physical unit (bottle, drum, jar) | Barcode (unique at receipt), location, quantity, unit, status (active/expired/disposed), receipt date, expiry date, lot association |
| **Location** | Physical storage hierarchy | Building > Room > Cabinet > Shelf. Multi-site support. Temperature, humidity, compatibility restrictions. |
| **Lot/Batch** | Specific production batch from a vendor | Lot number, purity, grade, certificate of analysis, supplier |
| **Vendor/Supplier** | Chemical supplier | Name, contact info, catalog integration via ACD |
| **User/Owner** | Responsible party | Scientist or group assigned to a container |

**Entity relationships:**
```
Substance (1) ──→ (many) Lots ──→ (many) Containers ──→ (1) Location
                                                       ──→ (1) User/Owner
```

**Underlying database:** Not explicitly documented, but based on Dassault Systèmes / Accelrys lineage and Pipeline Pilot's ODBC/JDBC Oracle connectivity, **Oracle Database** is the primary backend. SQL Server may be supported as an alternative.

**Custom attributes:** CISPro is described as "highly customizable" with support for user-defined attributes on substance and container records.

### 3.3 Available Chemicals Directory (ACD) Integration

ACD is a critical reference database integrated with CISPro:

- **Largest structure-searchable collection of commercially available chemicals**
- Contains data for **24+ million products** and **64+ million packages** from **970+ catalogs**
- Fields per chemical: structure, name, CAS number, grade, package size, pricing, supplier
- Fully integrated as a cloud solution within CISPro

**Migration implication:** ACD is a reference database, not customer-owned data. Customer-specific substance records, containers, lots, and custom pricing are what need to be migrated. ACD data can potentially be re-referenced in Albert (which has its own 400,000+ substance database).

### 3.4 Export Capabilities

**Critical finding: No publicly documented REST API or SOAP web service for CISPro.**

**Practical extraction approaches:**

| Approach | Description | Suitability |
|----------|-------------|------------|
| **Direct database access (SQL)** | Query the underlying Oracle/SQL Server. Requires schema documentation from Dassault or a consulting partner. | **Most reliable for bulk** |
| **Pipeline Pilot protocols** | Build ETL workflows connecting to CISPro's database via ODBC/JDBC. Pre-built cheminformatics components handle chemical structure data properly. | **Best for chemical data integrity** |
| **Web UI report export** | Export search results and reports to CSV/Excel via the CISPro web interface. | **Not suitable for bulk migration** (manual) |
| **SDS document extraction** | SDS documents are linked files (PDFs). Extract file references from DB and copy files from file store. | **Separate sub-task** |

**Chemical structure handling:** CISPro uses a chemical cartridge (likely BIOVIA Direct for Oracle) for structure storage and searching. During migration, structures must be exported in a standard format (MOL, SDF, SMILES, InChI) to avoid loss of stereochemistry.

### 3.5 CISPro → Albert Inventory Mapping

| CISPro Entity | Albert Inventory Entity | Notes |
|---------------|------------------------|-------|
| **Substance** | **Substance** | 1:1 mapping. CAS, name, formula, MW, structure. |
| **Lot/Batch** | **Lot** | 1:1 mapping. Lot number, vendor, purity, grade, CoA. |
| **Container** | **Inventory Item / Lot** | May need restructuring depending on Albert's container vs. lot distinction. |
| **Location** | **Location** | Confirm Albert supports equivalent hierarchy depth (Building > Room > Cabinet > Shelf). |
| **Custom Properties** | **Attributes** | Map field names and data types. Validate completeness. |
| **Vendor/Pricing** | **Pricing** | Distinguish catalog pricing (ACD) from actual purchase pricing. |
| **Hazard Classification (GHS)** | **Hazards** | Pictograms, signal words, H-statements, P-statements. |
| **Regulatory Lists** | **Regulatory/Compliance** | DEA schedules, TSCA, REACH, state-specific lists. |
| **SDS Documents** | **Attached Files** | PDF files linked to substances. |

**Typical data transformation needed:**
- CAS number validation and deduplication (CISPro may have duplicate substance records)
- Chemical structure format conversion (BIOVIA Direct cartridge → standard MOL/SDF)
- Location hierarchy normalization
- Unit of measure standardization
- Vendor name deduplication
- Custom property/attribute name mapping
- Status field mapping (active, expired, disposed, quarantined)
- Date format normalization

**Common data quality issues in CISPro:**
- Duplicate substance records (same chemical, slightly different names)
- Orphaned containers (linked to non-existent substances or changed locations)
- Expired containers still marked as active
- Inconsistent custom field usage across sites/departments
- Missing or incorrect CAS numbers
- Outdated or missing SDS documents
- Location data that doesn't reflect current physical layout

### 3.6 Regulatory & Compliance Data

**GHS classification tracked per substance:**
- Hazard pictograms (flame, skull-and-crossbones, exclamation mark, etc.)
- Signal words (Danger, Warning)
- Hazard statements (H-codes, e.g., H225: Highly flammable)
- Precautionary statements (P-codes, e.g., P210: Keep away from heat)

**Controlled substances:** CISPro includes the **Scitegrity CS2 module** for DEA-scheduled substance identification and tracking. Chain-of-custody records, usage tracking, DEA reporting compliance.

**Regulatory lists tracked:**
- TSCA (US Toxic Substances Control Act) inventory
- REACH (EU Registration, Evaluation, Authorization, Restriction of Chemicals)
- California Proposition 65
- DEA controlled substance schedules
- Fire code thresholds (NFPA, IFC)
- Export control lists (ITAR, EAR) depending on configuration

**What MUST be preserved during migration:**
- All GHS classification data
- DEA schedule classifications and usage history
- Regulatory list memberships per substance
- SDS documents (current and historical versions)
- Fire code threshold data per location
- Chain-of-custody records for controlled substances
- Audit trail history (may need separate archiving if Albert's model differs)

---

## 4. Azure Enterprise Data Platform

### 4.1 SaaS-to-Azure Integration Patterns

#### Pattern 1: Customer-Pull via ADF (Most Common in Enterprise)

The customer's Azure Data Factory reaches out to Albert's API and pulls data:

```
Albert Invent REST API → ADF Copy Activity → ADLS Gen2 (Bronze) → ADF Data Flow → Silver → Gold
```

**Why this is preferred:**
- Customer retains full control over ingestion timing, frequency, and transformations
- Customer's security team doesn't need to grant inbound write access to an external vendor
- Albert only needs to expose a well-documented REST API
- ADF has native REST connectors handling pagination, OAuth token refresh, and incremental extraction

#### Pattern 2: Vendor-Push (Direct Write to Customer Storage)

Albert writes data directly to the customer's ADLS Gen2:

```
Albert Invent → Writes Parquet/JSON files → Customer ADLS Gen2 (Landing Zone)
                                                → Event Grid triggers ADF pipeline
                                                → Bronze → Silver → Gold
```

Used when: very large data volumes (API polling inefficient), near-real-time delivery needed, or Albert has a dedicated "data warehouse sync" feature.

#### Pattern 3: Intermediate Landing Zone (Hybrid)

Albert exports to a shared intermediate location (vendor's blob storage, SFTP, or shared Azure container). Customer's ADF copies from there. Avoids giving Albert direct write access to the customer's lake.

#### Pattern 4: Event-Driven Streaming

Albert publishes events to Azure Event Hubs or Event Grid. Customer's Azure services consume events and land them in ADLS Gen2. Best for real-time needs.

**For the case study, Pattern 1 (Customer-Pull via ADF) or Pattern 2 (Vendor-Push) are most likely**, depending on what Albert's Data Warehouse component provides.

### 4.2 ADF Pipeline Patterns

#### REST API Connector

ADF's built-in REST connector supports:
- HTTP Methods: GET and POST for reading; POST, PUT, PATCH for writing
- Authentication: Anonymous, Basic, Service Principal, **OAuth2 Client Credential**, System/User-Assigned Managed Identity
- Pagination: Configurable rules supporting next-link, offset-based, cursor-based patterns
- Response handling: JSON, XML, binary

#### Typical OAuth2 Pipeline (Two-Step)

1. **Web Activity** (Get Token): POST to Albert's token endpoint → receives `access_token`
2. **Copy Activity** (Fetch Data): Uses token as Bearer header → copies REST source to ADLS Gen2 sink

#### Ingestion Patterns

**Full Load:**
```
ForEach(entity in [experiments, samples, results, inventory])
  → Copy Activity (REST Source → ADLS Gen2 Parquet Sink)
```

**Incremental/Delta Load:**
```
Lookup (get last watermark from control table)
  → Copy Activity (REST with ?modifiedSince={watermark} → ADLS Gen2)
    → Stored Procedure (update watermark)
```

**Event-Triggered:**
```
Albert webhook → Event Grid → Triggers ADF pipeline run → Copy + Transform
```

### 4.3 Cross-Tenant Authentication

#### Multi-Tenant Service Principal (Recommended for Ongoing Integration)

1. Albert creates a **Multi-Tenant App Registration** in their Entra ID tenant
2. Customer admin **consents** to the app (creates a Service Principal in customer's tenant)
3. Customer assigns **RBAC roles** (e.g., `Storage Blob Data Contributor`) scoped to specific container
4. Albert authenticates with client ID + secret against the customer's tenant endpoint

```
POST https://login.microsoftonline.com/{customer-tenant-id}/oauth2/v2.0/token
grant_type=client_credentials
&client_id={albert-app-client-id}
&client_secret={albert-app-client-secret}
&scope=https://storage.azure.com/.default
```

**Key constraint: Managed Identities do NOT work cross-tenant.** Must use Service Principals with credentials.

#### SAS Tokens (Alternative)

Shared Access Signatures provide time-limited, scoped access without Entra ID:
- Scoped to specific container, permissions (read/write/list), time window, IP range
- **Cannot be individually revoked** (only by rotating the storage account key)
- No identity-based audit trail
- Best for: time-limited, ad-hoc transfers, not ongoing integrations

#### Azure Private Link

Adds network-level security: traffic traverses Microsoft backbone, never the public internet. Customer can disable public access entirely. Cross-tenant Private Endpoints require explicit approval.

### 4.4 Data Formats & Landing Patterns

#### Recommended Formats

| Format | Use Case | Pros | Cons |
|--------|----------|------|------|
| **Parquet** | Silver/Gold layers, analytical workloads | Columnar, 5-10x compression, schema embedded, predicate pushdown | Binary, not human-readable |
| **Delta Lake** | Silver/Gold layers with ACID requirements | Parquet + transactions, schema evolution, time-travel, merge/upsert | Requires Spark/Databricks runtime |
| **JSON** | Bronze landing zone, raw API responses | Human-readable, flexible schema | Poor query performance, no compression |
| **CSV** | Legacy flat exports | Simple | No schema, no types, delimiter issues |

#### Medallion Architecture

```
Bronze (Raw)                          Silver (Validated)              Gold (Business-Ready)
─────────────────                     ─────────────────               ─────────────────
Append-only                           Cleansed, deduplicated          Pre-aggregated
Original format                       Schema enforced (Delta)         Star/snowflake schemas
Source of truth / audit trail         Business keys standardized      Optimized for BI/ML
                                      SCD (Slowly Changing Dims)      Cross-source joins

Path: /bronze/{source}/{entity}/      /silver/{source}/{entity}/      /gold/{domain}/{dataset}/
      {year}/{month}/{day}/
```

#### File Organization Convention

```
/bronze/albert-invent/experiments/year=2025/month=06/day=15/experiments_20250615_001.parquet
/bronze/albert-invent/inventory/year=2025/month=06/day=15/inventory_full_20250615.parquet
/bronze/albert-invent/test-results/year=2025/month=06/day=15/results_delta_20250615.json
/silver/albert-invent/experiments/      (Delta Lake table)
/silver/albert-invent/inventory/        (Delta Lake table)
/gold/rd-analytics/experiment-summary/  (Cross-source Delta table)
```

Hive-style partitioning (`key=value`) is natively understood by Spark, Synapse, and Databricks.

### 4.5 Albert's Data Warehouse (Inferred Capabilities)

Based on Albert's public developer documentation and platform architecture:

- **Open RESTful APIs** for programmatic access to all platform data
- **Event/Webhook system** referenced in developer docs ("continuously export new traces, scores, and metadata")
- **Bulk export to cloud storage** (S3, Google Cloud Storage, Azure Blob Storage)
- **Data Warehouse integration** explicitly supports Snowflake, BigQuery, Databricks
- **Microservice architecture** implies individual APIs per domain (inventory, ELN, LIMS)

**Most likely integration architecture for Albert → Azure:**

```
Option A (Customer-Pull):
  Albert REST API → ADF (OAuth2, incremental ?modifiedSince) → ADLS Gen2 Bronze → Silver → Gold

Option B (Albert Pushes):
  Albert Data Warehouse → Continuous Export → Customer ADLS Gen2 Landing Zone
                            → Event Grid triggers ADF → Bronze → Silver → Gold

Option C (Hybrid):
  Scheduled ADF pipeline (daily bulk sync via API)
    +
  Albert Webhooks → Azure Function → Event Hub → Databricks Streaming → Bronze
```

---

## 5. Materials Science Lab Data Workflows

### 5.1 Test Request Submission (Albert → LabVantage)

When an R&D scientist submits a test request, the following data flows from Albert to LabVantage:

**Required fields:**

| Field | Description | Example |
|-------|-------------|---------|
| Sample ID | Unique identifier | `SAMP-2025-04532` |
| Sample Name/Description | Human-readable | "Polyurethane Adhesive Batch 47-C" |
| Test Type(s) Requested | Which analyses to run | Viscosity, Tensile Strength, DSC, FTIR |
| Priority | Urgency level | Routine / Rush / Emergency |
| Due Date | When results are needed | 2025-06-20 |
| Project ID | Links to R&D project | `PROJ-2025-0089` |
| Requesting Scientist | Submitter identity | Dr. Jane Chen |
| Department/Lab | Organizational unit | Adhesives R&D, Building 3 |

**Common additional fields:**

| Field | Example |
|-------|---------|
| Specification ID | `SPEC-ADH-047 Rev 3` |
| Sample Type/Matrix | Liquid adhesive, 2-component |
| Quantity Submitted | 250 mL |
| Number of Replicates | 3 |
| Storage Conditions | Room temperature, away from moisture |
| Hazard Information | Flammable, skin sensitizer, GHS Cat 2 |
| Lot/Batch Number | `LOT-2025-0612-A` |
| ELN Reference | `EXP-2025-04532-003` (Albert experiment/worksheet ID) |
| Special Instructions | "Test at 23C and 40C; compare to reference SAMP-2025-04100" |

### 5.2 Test Result Return (LabVantage → Albert)

When results come back, each result record carries significantly more data:

| Field | Description | Example |
|-------|-------------|---------|
| Result ID | Unique identifier | `RES-2025-078441` |
| Sample ID | Links back to request | `SAMP-2025-04532` |
| Test Method | Standard used | ASTM D2196 (Viscosity) |
| Test Name | Human-readable | Brookfield Viscosity |
| **Result Value** | **The measurement** | **14,500** |
| **Unit of Measure** | **Units** | **cP (centipoise)** |
| Specification Limit (Low) | Lower acceptable | 12,000 |
| Specification Limit (High) | Upper acceptable | 18,000 |
| **Pass/Fail** | **Meets spec?** | **PASS** |
| OOS Flag | Out of Specification | false |
| OOT Flag | Out of Trend | false |
| Instrument ID | Which instrument | `INST-BF-DV2T-003` |
| Instrument Calibration Status | Last calibrated | 2025-06-01, due 2025-09-01 |
| Analyst Name | Who tested | M. Rodriguez |
| Analysis Date/Time | When tested | 2025-06-17T14:32:00Z |
| Conditions | Environment during test | 23.0C, 50% RH |
| Replicate Values | Individual measurements | [14,200, 14,600, 14,700] |
| Std Dev / %RSD | Statistical spread | 264.6 / 1.82% |
| Quality Flags | Data quality | "Within calibration", "Method validated" |
| Reviewer | Who approved | Dr. K. Patel |
| Review Date | Approval timestamp | 2025-06-17T16:45:00Z |
| Comments | Analyst notes | "Spindle #4, 20 RPM, 2-min equilibration" |
| Attachments | Supporting files | Chromatogram PDF, raw instrument data |

### 5.3 Typical Turnaround Times

| Test Category | Turnaround | Examples |
|--------------|-----------|----------|
| Simple physical tests | Same day to 1 day | Viscosity, pH, density, color |
| Standard analytical | 2-5 business days | FTIR, GC-MS, HPLC, titration |
| Mechanical/physical | 3-7 business days | Tensile strength, elongation, hardness, adhesion |
| Thermal analysis | 3-7 business days | DSC, TGA, DMA |
| External/contract lab | 7-10+ business days | Specialized tests at third-party labs |
| Rush/priority | 1-2 business days | Typically 2-3x cost premium |
| Aging/stability studies | Weeks to months | Accelerated aging at elevated temp/humidity |

### 5.4 Volume of Test Requests

| Organization Size | Test Requests/Week | Context |
|------------------|-------------------|---------|
| Small R&D team (5-10 scientists) | 10-30 | 2-5 samples/scientist/week |
| Medium R&D (20-50 scientists) | 50-200 | Multiple parallel projects |
| Large enterprise R&D (100+) | 200-1,000+ | Multiple sites, high-throughput |
| QC/QA lab (production support) | 250-2,500+ (50-500/day) | Incoming raw materials + in-process + finished goods |

### 5.5 Integration Design Implications

- **Latency tolerance:** Analytical results take hours to days -- near-real-time integration is desirable but not critical. Polling every 5-15 minutes is likely acceptable.
- **Volume per integration cycle:** Even at enterprise scale, the number of results returning per polling cycle is manageable (tens to low hundreds, not thousands).
- **Data richness:** Results carry significant metadata (instrument, method, conditions, quality flags, spec limits). The integration must preserve all of this, not just the numeric value.
- **Status tracking:** Scientists want to see where their sample is in the process (submitted, received, in-progress, completed). Status sync is as important as result delivery.
- **Error scenarios:** OOS (Out of Specification) results may trigger investigations in LabVantage. The integration needs to handle retests, amended results, and result invalidation.

---

## 6. Inventory Data in Materials Science

### 6.1 Data Model

The canonical chemical inventory data model follows this hierarchy:

```
Substance (Chemical Entity)
    ├── Lot (Batch from vendor/production)
    │     └── Container (Physical unit: bottle, drum, bag)
    │           └── Location (Where stored)
    ├── Regulatory Data (REACH, TSCA, GHS)
    ├── SDS (Safety Data Sheet -- one or more per substance)
    └── Specifications (Acceptance criteria for incoming lots)
```

#### Substance Record

| Field | Type | Example |
|-------|------|---------|
| Substance ID | String | `SUB-00012345` |
| Chemical Name | String | Methyl methacrylate |
| CAS Number | String | 80-62-6 |
| Molecular Formula | String | C₅H₈O₂ |
| Molecular Weight | Decimal | 100.12 g/mol |
| SMILES | String | CC(=C)C(=O)OC |
| InChI Key | String | VVQNEPGJFQJSBK-UHFFFAOYSA-N |
| Category | Enum | Raw Material / Intermediate / Product / Reagent |
| Physical Form | Enum | Liquid / Solid / Gas / Paste |
| Storage Class | String | "Flammable Liquids" |
| Shelf Life | Integer | 365 days |
| Preferred Suppliers | Array | [Sigma-Aldrich, BASF, Dow] |
| Default UOM | String | kg |
| Status | Enum | Active / Discontinued / Restricted / Banned |

#### Lot Record

| Field | Type | Example |
|-------|------|---------|
| Lot ID | String | `LOT-2025-0612-A` |
| Substance ID (FK) | String | `SUB-00012345` |
| Supplier Lot Number | String | MKCD5432 |
| Supplier | String | Sigma-Aldrich |
| Date Received | Date | 2025-06-12 |
| Date Manufactured | Date | 2025-05-28 |
| Expiration Date | Date | 2026-05-28 |
| Quantity Received | Decimal | 25.0 kg |
| Certificate of Analysis | File | `coa_lot_2025_0612_A.pdf` |
| Qualification Status | Enum | Pending / Quarantine / Approved / Rejected / Expired |
| Qualified By | String | Dr. K. Patel |
| Cost Per Unit | Decimal | $45.00/kg |
| Purity | Decimal | 99.5% |
| Country of Origin | String | Germany |

#### Container Record

| Field | Type | Example |
|-------|------|---------|
| Container ID | String | `CTN-2025-098765` |
| Lot ID (FK) | String | `LOT-2025-0612-A` |
| Barcode | String | 7890123456789 |
| Container Type | Enum | Bottle / Drum / Bag / Vial / IBC |
| Capacity | Decimal | 2.5 kg |
| Current Quantity | Decimal | 1.8 kg |
| Location ID (FK) | String | `LOC-BLD3-RM201-SHELF4-BIN12` |
| Status | Enum | Available / In-Use / Empty / Disposed / Quarantined |
| Opened Date | Date | 2025-06-14 |
| Assigned To | String | J. Smith |

#### Location Hierarchy

```
Site (Campus/Plant) → Building → Floor/Wing → Room/Lab → Storage Unit (Cabinet/Fridge) → Bin/Slot
```

Each location carries properties: temperature range, humidity, ventilation, fire rating, incompatibility restrictions.

### 6.2 Typical Volumes

| Category | Small-Medium Company | Large Enterprise |
|----------|---------------------|-----------------|
| **Unique substances** | 2,000-10,000 | 10,000-50,000 |
| **Active lots** | 5,000-20,000 | 20,000-100,000 |
| **Physical containers** | 10,000-50,000 | 50,000-500,000 |
| **Storage locations** | 200-1,000 | 1,000-10,000 |
| **SDS documents** | 2,000-10,000 | 10,000-50,000 |

For context: The EPA's TSCA inventory contains ~86,000 chemical substances. A large specialty chemicals company may track thousands to tens of thousands of distinct substances. Albert Invent's platform includes regulatory data for 400,000+ substances.

### 6.3 Lot Tracking & Qualification

#### Why Lot Tracking Matters

Every lot of a substance may have slightly different properties (purity, impurities, moisture content). Lot tracking enables:
- **Traceability:** If a product fails, trace backward through lots to identify the source material
- **Quality assurance:** Compare test results between lots to detect drift
- **Regulatory compliance:** DEA, FDA, and other agencies require full lot-level traceability
- **Expiration management:** Prevent use of expired materials

#### Lot Qualification Workflow

When a new lot arrives, it goes through a formal process before release:

```
1. RECEIVING & QUARANTINE
   └─ Lot arrives, quarantine status set, containers barcoded
   └─ Physical check: quantity, packaging integrity, labeling vs. PO

2. DOCUMENT REVIEW
   └─ Certificate of Analysis (CoA) reviewed
   └─ CAS number and substance confirmed
   └─ Supplier results checked against in-house specs
   └─ Shelf life verified

3. IDENTITY TESTING (minimum)
   └─ FTIR vs. reference standard, refractive index, or melting point
   └─ Confirms substance matches expectations

4. ANALYTICAL TESTING (risk-based)
   └─ Tier 1 (Trusted Supplier, Non-Critical): Identity only, rely on CoA
   └─ Tier 2 (Standard): Identity + 2-3 key specs (purity, water, color)
   └─ Tier 3 (Critical / New Supplier): Full specification testing
   └─ Submitted as a LIMS test request

5. REVIEW & APPROVAL
   └─ QC/QA reviews results vs. specification
   └─ PASS: Status → Approved, lot released for use
   └─ FAIL: Status → Rejected, OOS investigation triggered

Typical turnaround: 1-5 days (identity only) to 5-15 days (full testing)
```

### 6.4 Regulatory Data per Substance

| Regulation | What's Tracked | Example |
|-----------|---------------|---------|
| **TSCA** (US EPA) | Listed/Not listed, Active/Inactive, Section 12(b) export notification | CAS 80-62-6: Active on TSCA inventory |
| **REACH** (EU) | Registration number, tonnage band, SVHC candidate list, authorization requirements, Annex XVII restrictions | EC No. 201-297-1, registered >1000 t/year |
| **GHS** | Hazard classification, signal word, pictograms, H-statements, P-statements | Danger; H225, H335; GHS02, GHS07 |
| **Export Controls** | EAR classification (ECCN), ITAR status, sanctioned country restrictions | EAR99 (no license required) |
| **OSHA** | PEL (Permissible Exposure Limit), TLV (Threshold Limit Value) | PEL: 100 ppm TWA |
| **Transport (DOT/IATA)** | UN Number, Proper Shipping Name, Packing Group, Hazard Class | UN1247, PG II, Class 3 |
| **California Prop 65** | Listed/Not listed | Not listed |
| **DEA** | Schedule classification, usage tracking | Not scheduled |

### 6.5 Virtual vs Physical Inventory

| Type | Description | System Representation |
|------|-------------|----------------------|
| **Physical** | Actual containers on-site. Tracked with barcodes, audited periodically. | Substance + Lots + Containers with quantities > 0 |
| **Catalog-only** | Substances in the master catalog with no current stock. Used for approved substance lists, planning, procurement. | Substance record exists; 0 containers, 0 quantity on hand |
| **Historical** | Previously used substances, all containers disposed. Records retained for regulatory compliance (OSHA requires 30-year SDS retention). | Substance + archived SDS; status = Inactive/Archived |

Facilities must perform periodic **physical inventory audits** -- scanning all containers and comparing against system records. Discrepancies trigger investigations. Regulations often mandate annual audits at minimum.

---

## Key Findings for Architecture Decisions

### For the LIMS Integration (Workstream 2)

1. **LabVantage has a well-documented REST API** with JSON-only, token-based auth. The SDC/SDI pattern is standard and predictable.
2. **No webhooks** -- Albert must poll for completed results, or LabVantage must be customized to push (Event Service or custom Action).
3. **RESTPolicy is deny-by-default** -- requires LabVantage admin collaboration to enable specific endpoints for Albert.
4. **Sample→Test→DataSet→DataItem hierarchy** maps well to a request/result model. Use ExternalReference fields for cross-system correlation.
5. **Token-based External App Users** are the right auth pattern -- they don't consume licenses and don't expire like sessions.
6. **No bulk/atomic operations** -- sample creation and test assignment are separate API calls. Use Action blocks for sequencing.

### For the ELN Migration (Workstream 1)

7. **No Biovia REST API for bulk export** -- extraction requires Pipeline Pilot or direct database SQL.
8. **File volumes could range from 50GB to 10TB+** -- assess actual volume early in scoping.
9. **Organizational mapping** (Biovia Projects → Albert Projects) requires customer collaboration and is a significant scoping effort.
10. **Chemical structure files** (MOL, SDF) need special handling to preserve stereochemistry.

### For the Inventory Migration (Workstream 4)

11. **No CISPro REST API** -- extraction requires database access or Pipeline Pilot.
12. **Data quality cleanup** is a significant effort -- expect duplicate substances, orphaned containers, inconsistent custom fields.
13. **Regulatory data preservation is mandatory** -- GHS, DEA, TSCA, REACH data must be migrated completely and accurately.
14. **CISPro→Albert entity mapping is straightforward** conceptually but requires validation of Albert's exact data model via the SDK.

### For the Azure Integration (Workstream 3)

15. **Customer-Pull via ADF** is the most common enterprise pattern and should be the default recommendation.
16. **Multi-tenant Service Principal** is the right cross-tenant auth pattern (Managed Identities don't work cross-tenant).
17. **Parquet + Delta Lake** with medallion architecture is the standard target format.
18. **Albert's Data Warehouse** likely provides incremental export APIs or continuous export to cloud storage -- confirm during scoping.
