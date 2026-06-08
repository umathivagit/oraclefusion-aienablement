# Technical Design Document
## Integration: Salesforce → Oracle Fusion Cloud — AP Invoice Import

| | |
|---|---|
| **Document Title** | Salesforce to Oracle Fusion AP Invoice Import – Technical Design |
| **Integration ID** | INT-AP-001 |
| **Source System** | Salesforce (CRM / Custom Invoice Object) |
| **Target System** | Oracle Fusion Cloud — Payables (AP) |
| **Middleware** | Oracle Integration Cloud (OIC) |
| **Pattern** | Asynchronous, scheduled / event-driven, bulk (FBDI) |
| **Author** | Oracle Fusion Technical Consultant |
| **Version** | 1.0 |
| **Status** | Draft for Review |

### Version History
| Version | Date | Author | Description |
|---|---|---|---|
| 0.1 | — | Tech Consultant | Initial draft |
| 1.0 | — | Tech Consultant | Baselined for review |

---

## 1. Introduction

### 1.1 Purpose
This document describes the technical design for an integration that transfers supplier invoice data created in **Salesforce** into **Oracle Fusion Cloud Payables** for processing as Accounts Payable (AP) invoices. It is the technical companion to the functional/solution design and is intended for developers, integration engineers, and reviewers.

### 1.2 Scope

**In scope**
- Extraction of approved invoice records (header + lines) from Salesforce.
- Transformation and validation of data into the Oracle Fusion AP invoice structure.
- Load into Oracle Fusion using the **Import Payables Invoices** process via **FBDI** (File-Based Data Import).
- Error capture, reconciliation, and notification.

**Out of scope**
- Supplier (vendor) master creation in Fusion (assumed pre-existing / managed by a separate interface).
- Invoice approval workflow inside Fusion.
- Payment processing and accounting.
- Salesforce-side invoice creation logic and UI.

### 1.3 Definitions
| Term | Meaning |
|---|---|
| OIC | Oracle Integration Cloud (integration middleware) |
| FBDI | File-Based Data Import — Oracle's bulk load framework |
| UCM / WCC | Universal Content Management (WebCenter Content) document repository in Fusion |
| ESS | Enterprise Scheduler Service — runs Fusion background jobs |
| AP | Accounts Payable |

---

## 2. Solution Architecture

### 2.1 Architecture Overview
The integration follows a **middleware-orchestrated, bulk file-based** pattern. Oracle Integration Cloud (OIC) is the integration hub: it retrieves data from Salesforce using the Salesforce Adapter, transforms it into the Oracle AP invoice FBDI format, and submits it to Oracle Fusion using the **Oracle ERP Cloud Adapter**, which natively handles UCM upload and ESS job orchestration.

```
 ┌──────────────┐     1. Fetch approved      ┌───────────────────────────┐
 │              │        invoices (REST/      │   Oracle Integration       │
 │  SALESFORCE  │───────  Bulk/Platform ────▶ │   Cloud (OIC)              │
 │  (Invoice    │        Event)               │                            │
 │   Object)    │◀────── 6. Status update ─── │  • Salesforce Adapter      │
 └──────────────┘                             │  • Map & Transform         │
                                              │  • Generate FBDI CSV/ZIP   │
                                              │  • ERP Cloud Adapter       │
                                              └────────────┬───────────────┘
                                                           │ 2. importBulkData
                                                           ▼
                                              ┌───────────────────────────┐
                                              │  ORACLE FUSION CLOUD (AP)  │
                                              │                            │
                                              │  3. Upload ZIP → UCM       │
                                              │  4. Load Interface File    │
                                              │     for Import (ESS)       │
                                              │       → AP_INVOICES_        │
                                              │         INTERFACE           │
                                              │       → AP_INVOICE_LINES_    │
                                              │         INTERFACE            │
                                              │  5. Import Payables         │
                                              │     Invoices (ESS)          │
                                              │       → AP_INVOICES_ALL      │
                                              │       → AP_INVOICE_LINES_ALL │
                                              └───────────────────────────┘
```

### 2.2 Components
| Layer | Component | Role |
|---|---|---|
| Source | Salesforce Invoice object (standard or custom `Invoice__c`) | System of record for the originating invoice |
| Source API | Salesforce REST / Bulk API 2.0 / Platform Events | Mechanism to extract records |
| Middleware | OIC Integration flow | Orchestration, mapping, transformation, error handling |
| Middleware | OIC Salesforce Adapter | Connectivity to Salesforce |
| Middleware | OIC Oracle ERP Cloud Adapter | Connectivity to Fusion; abstracts UCM + ESS |
| Target | UCM (WebCenter Content) | Staging of the FBDI ZIP |
| Target | ESS Jobs: *Load Interface File for Import*, *Import Payables Invoices* | Move data into interface tables and then into base tables |
| Target | Interface tables `AP_INVOICES_INTERFACE`, `AP_INVOICE_LINES_INTERFACE` | Landing zone validated by the import |
| Target | Base tables `AP_INVOICES_ALL`, `AP_INVOICE_LINES_ALL` | Final AP invoice records |

### 2.3 Integration Pattern Rationale
- **FBDI over REST API:** The AP Invoice REST API processes records individually and is best for low volume / real-time. For batch volumes typical of invoice loads, **FBDI is the recommended, supported, and most performant** path and leverages Oracle's standard validations.
- **OIC as the orchestrator** removes custom UCM/ESS plumbing because the ERP Cloud Adapter's `importBulkData` operation performs upload, job submission, and callback/polling.

### 2.4 Trigger / Invocation Options
| Option | When to use |
|---|---|
| **Scheduled (recommended baseline)** | OIC schedule (e.g., hourly) queries Salesforce for invoices with status = *Approved* and `Integration_Status__c = 'Pending'`. Simple, resilient, good for batch. |
| **Event-driven** | Salesforce **Platform Event** or **Outbound Message** triggers OIC near-real-time. Use if low latency is required. |

This design uses the **scheduled batch** pattern as the primary mechanism, with event-driven noted as an alternative.

---

## 3. Data Flow

### 3.1 End-to-End Sequence
1. **Extract** — On schedule, OIC queries Salesforce for invoice headers and related line items where status indicates *ready for export* and not previously sent.
2. **Stamp in-progress** — OIC marks fetched records (`Integration_Status__c = 'In Progress'`, `Batch_Id__c`) to prevent re-pickup.
3. **Transform** — OIC maps Salesforce fields to the AP Invoice FBDI columns, applies lookups (supplier number, business unit, currency, distribution combination), and derives required defaults.
4. **Generate FBDI** — OIC builds the `ApInvoicesInterface.csv` and `ApInvoiceLinesInterface.csv`, then packages them into the FBDI ZIP.
5. **Load** — Via the ERP Cloud Adapter `importBulkData`:
   - ZIP is uploaded to **UCM**.
   - **Load Interface File for Import** populates `AP_INVOICES_INTERFACE` / `AP_INVOICE_LINES_INTERFACE`.
   - **Import Payables Invoices** validates and creates rows in `AP_INVOICES_ALL` / `AP_INVOICE_LINES_ALL`.
6. **Reconcile** — OIC polls/receives callback for ESS job completion, then retrieves the **import execution report** / output to determine success vs. rejected rows.
7. **Feedback** — OIC writes status back to Salesforce per record (`Success` / `Error` + error text) and notifies the support team for failures.

### 3.2 Key Field Mapping (Header — AP_INVOICES_INTERFACE)
| Salesforce field | Fusion FBDI column | Notes / Transformation |
|---|---|---|
| `Invoice_Number__c` | `INVOICE_ID` (group) / `INVOICE_NUM` | Unique invoice number; used for duplicate check |
| `Account.AP_Supplier_Number__c` | `VENDOR_NUM` / `SUPPLIER_NUMBER` | Lookup to existing Fusion supplier; reject if not found |
| `Supplier_Site__c` | `VENDOR_SITE_CODE` | Pay site in the correct BU |
| `Business_Unit__c` | `BUSINESS_UNIT` | Map to Fusion BU name |
| `Invoice_Date__c` | `INVOICE_DATE` | Format `YYYY/MM/DD` |
| `Currency__c` | `INVOICE_CURRENCY_CODE` | ISO currency |
| `Total_Amount__c` | `INVOICE_AMOUNT` | Header gross amount |
| (constant) | `INVOICE_TYPE_LOOKUP_CODE` | e.g., `STANDARD` |
| (constant) | `SOURCE` | Custom source value, e.g., `SALESFORCE` (registered lookup) |
| `Description__c` | `DESCRIPTION` | Free text |

### 3.3 Key Field Mapping (Line — AP_INVOICE_LINES_INTERFACE)
| Salesforce field | Fusion FBDI column | Notes |
|---|---|---|
| `Invoice_Number__c` | `INVOICE_ID` | Foreign key linking line to header group |
| `Line_Number__c` | `LINE_NUMBER` | Sequential |
| (constant) | `LINE_TYPE_LOOKUP_CODE` | e.g., `ITEM` |
| `Line_Amount__c` | `AMOUNT` | Line amount; sum must equal header |
| `Distribution_Combination__c` | `DIST_CODE_CONCATENATED` | Charge account (or use a distribution set) |
| `Line_Description__c` | `DESCRIPTION` | Free text |

> The full column set should follow the current **AP Invoice FBDI template** from Oracle's *File-Based Data Import for Financials* guide for the target release. Only required + business-relevant columns are populated; the rest are left null.

### 3.4 Volume & Frequency (to be confirmed)
| Attribute | Assumed value |
|---|---|
| Frequency | Hourly batch |
| Avg volume | ~500 invoices / run |
| Peak | ~5,000 invoices (month-end) |
| Latency SLA | Within the next scheduled run |

---

## 4. Error Handling

Errors are handled across three tiers: **connectivity/runtime**, **import/validation**, and **business reconciliation**.

### 4.1 Connectivity & Runtime Errors (OIC)
- Wrap Salesforce and ERP adapter invocations in OIC **scope** blocks with fault handlers.
- Implement **retry with backoff** for transient faults (timeouts, HTTP 5xx, token expiry). Recommended: 3 retries, exponential delay.
- On unrecoverable fault, route to a **global fault handler** that logs the error, leaves Salesforce records in a re-processable state, and raises a notification.

### 4.2 Import / Validation Errors (Fusion)
- The **Import Payables Invoices** process performs Oracle's standard validations (supplier exists, amounts balance, GL date in open period, duplicate invoice number, valid distribution combination, etc.).
- Rejected rows remain in the interface tables with a **reject reason**; the **Import Payables Invoices Report** lists rejections.
- OIC retrieves the import report output (via the ERP Cloud Adapter / ESS output) and parses the success vs. rejected counts and reasons.
- **Header–line amount mismatch**, **missing supplier**, and **closed period** are the most common rejections and should be pre-validated in OIC where feasible to reduce round-trips.

### 4.3 Business Reconciliation & Idempotency
- Each Salesforce record carries `Integration_Status__c`, `Fusion_Invoice_Number__c`, `Error_Message__c`, `Last_Sync_DateTime__c`, and `Batch_Id__c`.
- **Idempotency / duplicate prevention:** the unique `INVOICE_NUM` + supplier combination plus the *In Progress* stamping prevents resends; Fusion's own duplicate check is the final guard.
- **Failed records** are reset to a retryable status (`Error`) with the captured reason so they are re-picked on the next run after correction (manual or automated, per policy).

### 4.4 Logging & Notification
| Mechanism | Use |
|---|---|
| OIC Activity Stream / tracking | Per-instance trace with a business identifier (invoice number) for searchability |
| OIC error notifications (email) | Sent to a support distribution list on flow faults and on import rejections |
| Reconciliation summary | Per run: total fetched / loaded / rejected, attached or emailed |
| Salesforce write-back | Per-record status visible to business users in the source UI |

### 4.5 Error Handling Flow
```
Fetch ──► Transform ──► importBulkData ──► Poll ESS ──► Parse report
  │           │              │               │              │
  ▼           ▼              ▼               ▼              ▼
adapter    mapping         adapter        timeout      rejected rows
fault      fault           fault          / fail        per invoice
  │           │              │               │              │
  └──────► Global Fault Handler ◄────────────┘              │
                  │                                          │
            notify + leave records retryable          write back Error
                                                       + reason to SFDC
```

---

## 5. Security

### 5.1 Authentication
| Connection | Mechanism |
|---|---|
| OIC → Salesforce | OAuth 2.0 (JWT bearer / connected app) via the Salesforce Adapter; client credentials stored in OIC connection, not in code |
| OIC → Oracle Fusion | OAuth 2.0 or dedicated **integration service account** with Basic auth over TLS via the ERP Cloud Adapter |
| Salesforce → OIC (event-driven option) | OIC inbound secured by OAuth / Basic with a dedicated integration user |

### 5.2 Authorization (Least Privilege)
- A **dedicated Fusion integration user** is provisioned with only the roles needed to run the import and access UCM:
  - Payables invoice import privileges (e.g., a custom role granting *Import Payables Invoices* and *Load Interface File for Import*).
  - UCM document upload (`FND_GRANTS` / appropriate UCM account access).
- The **Salesforce connected app** is scoped to read invoice objects and update integration status fields only.
- No interactive/named user credentials are used for the integration.

### 5.3 Data Protection
- All transport over **TLS 1.2+ (HTTPS)**.
- Credentials and certificates held in **OIC's secure connection store / certificate vault**; never embedded in mappings or files.
- FBDI files in OIC staging and UCM are transient and purged per retention policy.
- Restrict PII / sensitive financial data in logs — log **business identifiers**, not full payloads.

### 5.4 Auditability
- OIC retains instance tracking for the configured retention window.
- Fusion ESS request history and import reports provide an audit trail of every load.
- Salesforce field history tracks status changes per invoice.

---

## 6. Assumptions

1. **Suppliers/sites pre-exist** in Oracle Fusion; this integration does not create or update vendor master data. Records with unknown suppliers are rejected.
2. Salesforce holds an **invoice object (header) with related line items** and a usable unique invoice number.
3. The target Fusion release's **AP Invoice FBDI template** structure is the authoritative column specification; mappings will be confirmed against it during build.
4. **GL periods** for the relevant accounting dates are open at load time; closed-period invoices will be rejected and require business action.
5. A **dedicated integration service account** can be created in both Salesforce and Fusion with the least-privilege roles described.
6. **Distribution / charge account** logic is either provided on the Salesforce line or derived via an agreed rule (distribution set / default account).
7. **OIC is the approved middleware** and has network connectivity (and any required allow-listing) to both Salesforce and the Fusion environment.
8. **Batch (scheduled) processing** meets latency requirements; near-real-time is an alternative, not the baseline.
9. Invoice **approval occurs in Salesforce**; only approved invoices are sent. Approval within Fusion (if any) is handled by Fusion's own workflow, out of scope here.
10. **Currency and tax**: tax is either calculated by Fusion (recommended) or supplied per agreed rules; multi-currency invoices carry valid conversion details or rely on Fusion conversion setup.
11. **Volume estimates** in §3.4 are placeholders pending confirmation and may affect scheduling/throughput design.

---

## 7. Open Items / To Be Confirmed
| # | Item | Owner |
|---|---|---|
| 1 | Confirm exact Salesforce object/field names and approval status values | Salesforce team |
| 2 | Confirm Fusion BU(s), source lookup registration (`SALESFORCE`), and FBDI template version | Fusion functional |
| 3 | Confirm tax handling approach (calculate in Fusion vs. supplied) | AP / Tax |
| 4 | Confirm distribution/charge account derivation rules | AP functional |
| 5 | Confirm volumes, frequency, and latency SLA | Business |
| 6 | Confirm error-correction process (manual vs. auto-retry) | Support / Business |

---

## 8. Appendix — ESS Jobs Reference
| Job | Purpose |
|---|---|
| **Load Interface File for Import** | Reads the FBDI ZIP from UCM and populates the AP interface tables |
| **Import Payables Invoices** | Validates interface data and creates invoices in `AP_INVOICES_ALL` / `AP_INVOICE_LINES_ALL`; produces the import report with rejections |

*This document should be read alongside Oracle's current "File-Based Data Import (FBDI) for Oracle Financials Cloud" guide for the AP Invoice template specific to the target release.*
