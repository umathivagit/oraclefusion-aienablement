# Supplier Onboarding Approval Workflow
## Solution Design Document — Oracle Fusion Cloud Procurement (Supplier Management)

| Attribute | Detail |
|---|---|
| Module | Oracle Fusion Cloud Procurement → Supplier Management / Supplier Portal |
| Sub-components | Supplier Model, Supplier Registration, Approval Management (AMX/BPM), Supplier Portal, Supplier Qualification (optional) |
| Document type | Functional & Technical Solution Design |
| Prepared by | Oracle Fusion Functional/Technical Consultant |
| Version | 1.0 (baseline) |
| Status | Draft for review |

> Note on release accuracy: Oracle Fusion is updated on a quarterly cadence. Exact task names, navigation paths, and REST endpoint versions should be validated against your tenant's current release before build. The design concepts below (Supplier Model, business-relationship lifecycle, AMX approval rules, BPM human task) are stable across recent releases.

---

## 1. Executive Summary

### 1.1 Business context
The organization needs a controlled, auditable process to onboard new suppliers before any spend (purchase orders, invoices, payments) can be transacted against them. Today, supplier creation is either uncontrolled or handled offline, creating risks around duplicate vendors, unverified bank details (payment-fraud exposure), missing tax/compliance documentation, and no audit trail of who approved what.

### 1.2 Objective
Implement a standardized supplier onboarding approval workflow in Oracle Fusion that:
- Captures supplier master data, tax identifiers, banking, classifications, products & services, and supporting documents at the point of registration.
- Routes each registration through a configurable, tiered approval chain based on risk and spend authority.
- Differentiates **Prospective** suppliers (sourcing-only, no spend) from **Spend Authorized** suppliers (transaction-enabled).
- Provides full audit history, notifications, and reporting.

### 1.3 Solution approach
The solution leverages **seeded Oracle Fusion functionality** with **configuration only** (no custom code in the core flow):
- **Supplier Registration** (external self-service and internal channels) to capture the request.
- **Approval Management Extensions (AMX)** rules executed by the seeded **Supplier Registration Approval** BPM human task to route approvals.
- **Approval Groups** and **list builders** (supervisory, job level, approval group) for tiered routing by spend, category, and risk.
- **Supplier Spend Authorization Approval** for promoting prospective suppliers to spend-authorized.
- **BPM/BI Publisher notifications** for approvers and requesters.

### 1.4 Key benefits
| Benefit | Description |
|---|---|
| Risk reduction | Mandatory tax, banking, and compliance capture before activation |
| Segregation of duties | Tiered approvals (Procurement, Finance, Tax, Legal) by spend and risk |
| Fraud control | Bank detail and tax ID verification gates prior to spend authorization |
| Auditability | Complete approval history retained in the supplier registration record |
| Scalability | Self-service registration reduces manual data entry effort |

### 1.5 Key assumptions
- Procurement offering and Supplier Management functional area are licensed and enabled.
- Approver hierarchy (supervisory or position) and HR person records exist for internal approvers.
- Spend thresholds, approving roles, and required documents are confirmed with the business during the design workshop.

### 1.6 Out of scope (this baseline)
- Third-party tax/sanctions screening integration (noted as an optional enhancement).
- Supplier Qualification Management questionnaires beyond the basic registration questionnaire (optional add-on).
- Custom OIC integrations (referenced in Technical Design as extension points).

---

## 2. Business Process Model (BPM)

### 2.1 Process actors
| Actor | Role |
|---|---|
| Prospective supplier | External party completing self-service registration |
| Internal requester / Procurement Agent | Initiates registration on behalf of a supplier |
| Procurement Approver | First-line approval, validates commercial fit |
| Finance / AP Approver | Validates banking and payment setup |
| Tax/Compliance Approver | Validates tax IDs, classifications, sanctions |
| Supplier Administrator | Owns configuration, monitors stuck requests |

### 2.2 To-be process narrative (swimlane description)

**Lane: Supplier / Requester**
1. Registration is initiated through one of two channels:
   - *External*: supplier accesses the public **Supplier Registration** URL and self-registers.
   - *Internal*: a procurement agent uses **Suppliers → Register Supplier** to register on the supplier's behalf.
2. The registrant completes the guided train: Company Details → Contacts → Addresses → Business Classifications → Bank Accounts → Products & Services → Questionnaire → Attachments → Review & Submit.
3. On submit, a **Supplier Registration Request** is created with status *Pending Approval*.

**Lane: System (Oracle Fusion)**
4. Mandatory-field and format validations run at submission (e.g., tax registration number format, required attachments).
5. The **Supplier Registration Approval** workflow is triggered; AMX evaluates approval rules and builds the approver list.

**Lane: Approvers**
6. Gate 1 — **Completeness/Procurement review**: validates business need and data completeness. Can *Approve*, *Reject*, or *Request More Information* (returns to requester).
7. Gate 2 — **Finance/Tax review** (parallel or serial per configuration): validates banking and tax/compliance.
8. Gate 3 — **Spend authority approval** (tier-based): routed by intended spend/relationship type.

**Lane: System (outcome)**
9. On full approval, the supplier record is created as **Prospective** or **Spend Authorized** per the requested business relationship.
10. Supplier contact(s) flagged for portal access are provisioned a **Supplier Portal** user account with appropriate roles.
11. Requester and supplier are notified of the outcome (approved / rejected with reason).

### 2.3 Process stages and decision gates
| Stage | Gate | Possible outcomes |
|---|---|---|
| Registration submitted | Completeness check (system + Gate 1) | Pass → continue; Incomplete → return to supplier |
| Risk & compliance | Tax/Finance review (Gate 2) | Pass → continue; Fail → reject |
| Approval routing | Spend-tier approval (Gate 3) | Approved → create vendor; Declined → notify |
| Activation | Create supplier + provision portal user | Prospective or Spend Authorized |

### 2.4 RACI
| Activity | Supplier | Requester | Procurement | Finance/Tax | Supplier Admin |
|---|---|---|---|---|---|
| Submit registration | R | R | C | I | I |
| Data completeness review | I | C | **A/R** | I | C |
| Banking/tax validation | I | I | C | **A/R** | I |
| Final spend approval | I | I | **A/R** | C | I |
| Configuration & monitoring | – | – | C | C | **A/R** |

---

## 3. Functional Design

### 3.1 Solution overview in Oracle Fusion
Supplier onboarding in Fusion is delivered through the **Supplier Model** and the **Supplier Registration** flow. A registration captures all supplier attributes and, on approval, instantiates a Supplier with the requested **Business Relationship**:
- **Prospective** — supplier can be used for sourcing/negotiations only; cannot transact spend.
- **Spend Authorized** — supplier is fully enabled for POs, invoices, and payments.

Approvals are handled by the seeded **Supplier Registration Approval** BPM human task, configured through **Approval Management Extensions (AMX)** rules. No custom workflow is required.

### 3.2 Registration channels
| Channel | How initiated | Typical use |
|---|---|---|
| External self-service | Public Supplier Registration URL | Net-new suppliers registering themselves |
| Internal | Suppliers work area → **Register Supplier** | Agent-driven onboarding |
| Supplier Portal (existing) | Existing supplier updates profile | Profile change requests (separate change-approval flow) |

### 3.3 Data captured at registration
| Section | Key attributes | Notes |
|---|---|---|
| Company details | Supplier name, tax org type, tax registration number, taxpayer ID, D-U-N-S | Drives tax validation and duplicate check |
| Contacts | Name, email, phone, portal-access flag | Portal flag triggers user provisioning |
| Addresses | Address purpose (ordering, RFQ, remit-to) | |
| Business classifications | Diversity/small-business certifications, expiry | Used for spend reporting |
| Bank accounts | IBAN/account number, bank, branch, currency | High-risk: routed to Finance gate |
| Products & services | Category assignments | Drives category-based routing & sourcing |
| Questionnaire | Configurable compliance questions | Optional, via registration config |
| Attachments | W-9/tax cert, insurance, banking letter | Mandatory set enforced by policy |

### 3.4 Configuration (Functional Setup Manager)
Performed under **Setup and Maintenance → Offering: Procurement → Functional Area: Suppliers**.

| Setup task | Purpose |
|---|---|
| Configure Supplier Registration | Enable external registration, default business relationship, mandatory sections, questionnaire, registration URL |
| Manage Supplier Registration Approvals | Define/maintain AMX approval rules and routing |
| Manage Supplier Products and Services Categories | Category hierarchy used for routing |
| Manage Supplier Value Sets / Lookups | Supplier types, classifications |
| Manage Tax Registrations / Tax setup | Tax ID validation |
| Specify Supplier News Content | Content on registration landing page |
| Manage Workflow Notifications (BI catalog) | Notification email templates |

### 3.5 Roles & responsibilities (security)
| Seeded/derived role | Responsibility |
|---|---|
| Supplier Administrator | Configure registration & approvals, monitor requests |
| Supplier Manager | Manage supplier master, approve registrations |
| Procurement Agent / Category Manager | Initiate internal registrations, first-line approval |
| Supplier Self Service Clerk/Administrator | Manage portal access for suppliers |
| Custom approver roles | Mapped to AMX approval groups |

### 3.6 Functional assumptions & dependencies
- Approver hierarchy data (supervisory/position) is maintained in HCM or via approval groups.
- Spend thresholds and approving roles are confirmed in a design workshop and documented in the Approval Rule Matrix (Section 5.4).
- Required-attachment policy is enforced via registration configuration and reviewer checklist.

---

## 4. Technical Design

### 4.1 Architecture overview
```
[Supplier / Requester]
        │  (Self-service URL or Register Supplier UI)
        ▼
[Supplier Registration UI]  ──► [Supplier Registration Request object]
        │ submit
        ▼
[BPM/SOA Human Task: Supplier Registration Approval]
        │ invokes
        ▼
[AMX Rules Engine]  ──► builds approver list (list builders + approval groups)
        │ approve/reject
        ▼
[Supplier Model] ──► Supplier created (Prospective / Spend Authorized)
        │
        ├──► [IAM / Identity] provision Supplier Portal user
        ├──► [UCM] store attachments
        └──► [BI Publisher] outbound notifications
```

### 4.2 Core components
| Component | Type | Description |
|---|---|---|
| Supplier Registration | Seeded application flow | Captures and stores the registration request |
| Supplier Registration Approval | Seeded BPM/SOA human task | Orchestrates the approval routing |
| AMX rules | Configuration (rule dictionary) | IF/THEN routing logic, list builders |
| Approval Groups | Configuration (BPM Worklist) | Named static/dynamic approver lists |
| Notifications | BI Publisher templates | Approver action + requester outcome emails |
| Supplier Portal user provisioning | IAM integration | Creates external user with supplier roles |

### 4.3 Approval task configuration
- **Where**: BPM Worklist → Administration → **Task Configuration**, or **Manage Task Configurations for Supplier Management** in FSM.
- **Task**: `Supplier Registration Approval` (and `Supplier Spend Authorization Approval` for promotions).
- **Stages**: configured as serial and/or parallel participant blocks.
- **List builders available**: Supervisory hierarchy, Job Level, Position hierarchy, Management Chain, Approval Group (static/dynamic), Single User, Self/Auto-approve.
- **Routing attributes (payload)** available to rules include: requested business relationship, supplier type, tax organization type, products & services categories, requester, procurement BU, country, number of bank accounts, business classifications.

### 4.4 Notifications
- Delivered via workflow; templates managed as **BI Publisher** reports in the BI catalog under the Procurement workflow notifications folder.
- Configure: approver action notification, FYI notifications, reminders, and final outcome notification (with rejection reason).
- Branding/content edited by copying seeded templates into a custom folder and re-pointing the data model layout.

### 4.5 Personalization / extensibility
| Extension point | Tool | Use |
|---|---|---|
| Registration page layout, mandatory fields | Page Composer / Application Composer in a **Sandbox** | Add help text, hide/show fields |
| Additional descriptive attributes | Supplier flexfields (DFF) | Capture custom data |
| Custom questionnaire | Registration questionnaire / Supplier Qualification | Compliance questions |
| Outbound integration (e.g., master data sync, sanctions screening) | Oracle Integration Cloud (OIC) + REST | Optional enhancement |

### 4.6 Integration points (extension options)
| Integration | Mechanism | Purpose |
|---|---|---|
| Supplier master sync to ERP/3rd party | Supplier REST APIs / OIC | Downstream replication |
| Tax ID / VAT validation | External service via OIC | Validate at submission |
| Sanctions / watchlist screening (D&B etc.) | OIC + DaaS | Compliance gate |
| Bank detail verification | OIC + bank validation service | Fraud control |
| Identity provisioning | OCI IAM / IDCS | Portal user accounts |

REST resources commonly used: **Suppliers**, **Supplier Addresses/Sites/Contacts**, and **Supplier Registration** resources (validate exact resource names/versions against the tenant's REST API catalog).

### 4.7 Security model
- Function security via job roles (Supplier Administrator, Supplier Manager); data security via procurement BU and data access sets.
- Approver authorization derived from AMX list builders / approval group membership.
- Supplier Portal users receive externally scoped roles only (no internal data access).

### 4.8 Reporting
- **OTBI** subject area: *Procurement – Supplier Real Time* (and Supplier Registration subject area where available) for onboarding cycle time, pending approvals, rejection reasons, approver workload.
- BPM Worklist provides per-task audit history and approval trail.

### 4.9 Environment migration
- Approval rules and configuration migrated between PODs using **FSM Configuration Packages** / **Export-Import** (CSV) for the Suppliers functional area; BPM task config and approval groups migrated via BPM export.

---

## 5. BPM Workflow Logic

### 5.1 Workflow task
- **Human task**: Supplier Registration Approval.
- **Trigger**: submission of a Supplier Registration Request (status → Pending Approval).
- **Outcome actions**: Approve, Reject, Request Information (reassign back to requester), Delegate, Escalate.

### 5.2 Stage structure (recommended)
| Stage | Mode | Participant | Purpose |
|---|---|---|---|
| Stage 1 – Completeness | Serial | Procurement Approver (approval group) | Validate data and business need |
| Stage 2 – Compliance | Parallel | Finance approver + Tax/Compliance approver | Validate banking and tax/sanctions |
| Stage 3 – Spend authority | Serial | Tiered approver by spend (supervisory/job-level) | Authorize spend relationship |

### 5.3 List-builder strategy
| Routing need | List builder |
|---|---|
| Fixed functional reviewers (Procurement, Finance, Tax) | Approval Group (static) |
| Spend-threshold escalation up the chain | Supervisory hierarchy + Job Level |
| Category-specialist routing | Approval Group (dynamic) by category |
| Low-risk auto-approval | Auto-approve / Self |

### 5.4 Approval Rule Matrix
| Rule | Condition (IF) | Approver (THEN) | Mode |
|---|---|---|---|
| R1 – Prospective only | Business relationship = Prospective AND no bank account | Procurement approval group (1 level) | Serial |
| R2 – Standard spend-authorized | Business relationship = Spend Authorized AND intended spend ≤ Tier-1 threshold | Procurement + Finance approval groups | Stage 1 serial → Stage 2 parallel |
| R3 – Mid spend | Spend Authorized AND Tier-1 < spend ≤ Tier-2 | R2 approvers + Procurement Manager (job level +1) | Serial escalation |
| R4 – High spend / strategic | Spend Authorized AND spend > Tier-2 | R3 approvers + Category Director + CFO delegate | Serial to top of chain |
| R5 – Tax/compliance gate | Any tax org type = Foreign OR sanctioned-country flag | Add Tax/Compliance approval group (mandatory) | Parallel within Stage 2 |
| R6 – Banking gate | Bank accounts count ≥ 1 | Add Finance/AP approval group | Parallel within Stage 2 |
| R7 – Auto-approve | Prospective AND internal-only AND pre-screened category | Auto-approve | – |

> Tier-1 / Tier-2 thresholds and named approval groups are confirmed during the design workshop and parameterized in the rule dictionary.

### 5.5 Routing examples
- **Low-risk prospective supplier**: R1 fires → single procurement approval → created as Prospective.
- **Spend-authorized supplier, mid spend, foreign tax ID, one bank account**: R3 + R5 + R6 fire → Stage 1 procurement → Stage 2 parallel (Finance + Tax) → Stage 3 manager escalation → created as Spend Authorized.
- **Strategic high-spend supplier**: R4 fires → full serial chain to CFO delegate.

### 5.6 Exception & control handling
| Event | Behavior |
|---|---|
| Request More Information | Task returns to requester; on resubmit, re-evaluates rules from current stage |
| Rejection | Request status → Rejected; requester notified with mandatory comment/reason; no supplier created |
| Resubmission after reject | New registration request; full re-evaluation |
| Expiration / timeout | Configurable due date; on expiry auto-escalate to next approver or supplier admin |
| Reminders | Configurable reminder cadence before due date |
| Delegation/Reassignment | Approver may delegate via BPM Worklist; audit retained |
| No approver found | Route to **Supplier Administrator** as a safety net (administrative approval group) |

### 5.7 Spend authorization promotion
For prospective→spend-authorized promotion later, the **Supplier Spend Authorization Approval** task runs an analogous rule set (typically Finance + Tax gate) before enabling transactions.

---

## 6. Test Cases

| TC ID | Scenario | Type | Steps | Expected result |
|---|---|---|---|---|
| TC-01 | External self-service registration – happy path | Positive | Access registration URL → complete all sections → submit | Request created, status Pending Approval; Stage 1 task generated |
| TC-02 | Internal registration by agent | Positive | Suppliers → Register Supplier → complete → submit | Request created; routing per rules |
| TC-03 | Mandatory attachment missing | Negative | Submit without required tax document | Validation error; submission blocked |
| TC-04 | Invalid tax registration format | Negative | Enter malformed tax ID → submit | Format validation error |
| TC-05 | Prospective-only routing (R1) | Positive | Register prospective, no bank account → submit | Single procurement approval → supplier created as Prospective |
| TC-06 | Standard spend routing (R2) | Positive | Spend authorized, spend ≤ Tier-1, one bank account | Procurement (serial) → Finance (parallel) → created Spend Authorized |
| TC-07 | Mid-spend escalation (R3) | Positive | Spend in Tier-1..Tier-2 | Adds manager (job-level +1) in chain |
| TC-08 | High-spend strategic (R4) | Positive | Spend > Tier-2 | Full serial chain to CFO delegate |
| TC-09 | Tax/compliance gate (R5) | Positive | Foreign tax org type | Tax/Compliance approver added in Stage 2 |
| TC-10 | Banking gate (R6) | Positive | One+ bank accounts | Finance/AP approver added |
| TC-11 | Parallel approval completion | Positive | Two parallel approvers both approve | Stage completes; proceeds to next stage |
| TC-12 | Parallel approval – one rejects | Negative | One parallel approver rejects | Request rejected; requester notified with reason |
| TC-13 | Request More Information loop | Positive | Approver requests info → requester updates → resubmits | Task returns and re-evaluates from current stage |
| TC-14 | Rejection with reason | Negative | Approver rejects without comment | System enforces mandatory comment; on reject, status Rejected, no supplier created |
| TC-15 | Auto-approval (R7) | Positive | Prospective, internal, pre-screened category | Auto-approved; no human task |
| TC-16 | Timeout/escalation | Positive | Leave task past due date | Auto-escalation to next approver/admin |
| TC-17 | Reminder notification | Positive | Approach due date | Reminder email sent to pending approver |
| TC-18 | No approver found fallback | Negative | Configure scenario with empty group | Routes to Supplier Administrator |
| TC-19 | Portal user provisioning | Positive | Contact flagged for portal access; approve | Supplier Portal user created with correct roles |
| TC-20 | Notifications content | Positive | Complete approval & rejection cycles | Correct approver/outcome emails with reason fields populated |
| TC-21 | Spend authorization promotion | Positive | Promote prospective supplier | Spend Authorization Approval runs; supplier becomes Spend Authorized |
| TC-22 | Duplicate supplier detection | Negative | Register with existing tax ID/name | Duplicate warning surfaced to reviewer |
| TC-23 | Security – approver visibility | Positive | Approver opens task | Sees only authorized data; no internal restricted fields for portal users |
| TC-24 | Reporting – cycle time | Positive | Run OTBI onboarding report | Pending/approved/rejected counts and cycle time accurate |

---

## 7. User Training Guide

### 7.1 For prospective suppliers (external self-service)
1. Open the **Supplier Registration** link provided by the buying organization.
2. Complete each step of the guided train; fields marked with an asterisk are required.
3. Enter accurate tax identifiers and, where requested, bank account details exactly as on official documents.
4. Upload all requested documents (e.g., tax certificate, banking letter, insurance) in the **Attachments** step.
5. Review the summary, then **Submit**. You will receive a confirmation and a tracking reference.
6. If a reviewer requests more information, you will receive an email with a link to update and resubmit.
7. On approval, you receive an invitation to access the **Supplier Portal** (if portal access was requested for your contact).

**Tips**: Save as draft if you need to gather documents. Use a monitored business email — all notifications go there.

### 7.2 For internal requesters / procurement agents
1. Navigate to **Procurement → Suppliers**.
2. Select **Register Supplier** (or **Tasks → Register Supplier**).
3. Choose the requested **Business Relationship** (Prospective vs Spend Authorized) based on whether spend is intended.
4. Complete company, contacts, addresses, categories, banking (if applicable), and attachments.
5. Submit. Track progress under **Manage Suppliers → Registration Requests**.

### 7.3 For approvers (acting on tasks)
1. Approval requests arrive via email and the **bell/notifications** icon, or in **Tools → Worklist (BPM)**.
2. Open the task to review the registration summary and attachments.
3. Choose an action:
   - **Approve** to advance the request.
   - **Reject** — a comment/reason is mandatory.
   - **Request Information** — returns the request to the requester.
   - **Delegate/Reassign** if the task belongs to a colleague.
4. For parallel stages, all required approvers must approve before the request advances.
5. Use the **history** tab to view who has acted and what comments were left.

**Tips**: Act before the due date to avoid auto-escalation. Always check banking and tax documents against the attachments before approving.

### 7.4 For supplier administrators
1. Configure registration and approvals under **Setup and Maintenance → Procurement → Suppliers**.
2. Maintain approval rules in **Manage Supplier Registration Approvals** and approval groups in **BPM Worklist → Approval Groups**.
3. Monitor stuck or overdue requests via the Worklist and OTBI onboarding reports.
4. Act as the fallback approver when no approver is resolved by the rules.
5. Manage notification templates in the BI catalog.

### 7.5 FAQ
| Question | Answer |
|---|---|
| Why can't I raise a PO for a supplier? | The supplier is likely **Prospective**; it must be promoted to **Spend Authorized** first. |
| A request is stuck — what do I do? | Check the Worklist history; if no approver was found it routes to the Supplier Administrator. |
| Can a rejected supplier reapply? | Yes — a new registration request is submitted and re-evaluated. |
| How are approvers determined? | By AMX rules using relationship type, spend, category, banking, and tax attributes. |

---

## Appendix A — Glossary
| Term | Meaning |
|---|---|
| AMX | Approval Management Extensions — the rules engine behind BPM human tasks |
| BPM | Business Process Management — Oracle's workflow/approval orchestration |
| Business Relationship | Prospective (no spend) vs Spend Authorized (transaction-enabled) |
| FSM | Functional Setup Manager (Setup and Maintenance) |
| List builder | Method AMX uses to construct an approver list |
| Approval Group | Named set of approvers used in routing rules |
| OTBI | Oracle Transactional Business Intelligence (reporting) |
| UCM | Universal Content Management (attachment storage) |

## Appendix B — Configuration checklist
- [ ] Enable external supplier registration and set the registration URL.
- [ ] Define default business relationship and mandatory registration sections.
- [ ] Configure questionnaire and mandatory attachments.
- [ ] Create approval groups (Procurement, Finance, Tax/Compliance, Admin fallback).
- [ ] Build AMX rules R1–R7 with confirmed thresholds.
- [ ] Configure stages (serial/parallel), due dates, reminders, escalation.
- [ ] Configure/brand notification templates.
- [ ] Define supplier portal roles for provisioned contacts.
- [ ] Validate OTBI onboarding report.
- [ ] Migrate configuration via FSM packages to test/prod.
