# 1. Overview

Build a serverless AWS solution that (a) creates one contract PDF per vehicle from the nightly NSC interface file, supporting single-page and **bundled** multi-page PDFs with duplicate protection; (b) initiates and tracks e-signing via **Signicat** to the authorized dealer signee; (c) enforces a **visible e-signature stamp on all pages**; and (d) delivers the **signed PDF + signing log/ID + one DIP per vehicle to OnBase** (for bundles, the same file is referenced per vehicle). Additionally, generate and email an **SF “momsfaktura” PDF** from invoice XML to the dealer and SF mailboxes.

---

# 2. System Architecture

Serverless, event-driven. **Ingestion stays thin**; long-lived orchestration runs in a single **AWS Step Functions (Standard)** execution per bundle:

**Initialize Bundle → Process Vehicles (Map) → Assemble Bundle PDF → Create Signing Session (Task Token) → Stamp & (Optional) Seal → Deliver to OnBase (Map) → Closeout & Notifications**

Unsigned PDFS are written temporarily to S3. The invoice path runs in parallel (Figure 1).

![Alt text](architecture/pdf_signing.png)
Figure 1. Architecture overview.

## 2.1 Platform Components

* **Amazon S3 (Inbound)**

  * `inbound/interface/` — nightly NSC interface files (contracts)
  * `inbound/invoice/` — invoice XML for the momsfaktura flow
* **AWS Lambda — Ingest**

  * Trigger: S3 `Put` on interface files
  * Parse & validate rows
  * **Idempotent** writes to DynamoDB using conditional puts
  * Derive `bundle_id`; **start one Step Functions execution per bundle** with `{ bundle_id }`
* **Amazon DynamoDB (single table, minimal)**

  * **Partition key:** `bundle_id`
  * **Sort key:** `sk` (typed value, e.g., `HEADER#0`, `VEHICLE#<contract_id>`)
  * **`item_type`**: human-readable type (`"header"` or `"vehicle"`)
  * Timestamps: `created_at`, `updated_at`
* **AWS Step Functions (Standard)**
  Orchestrates all post-ingest stages (states listed below). How to distribute the code accross lambdas TBD.
* **Amazon API Gateway + Webhook Lambda**
  Receives Signicat callbacks at a **bundle-aware** endpoint, resumes the waiting Step Functions task (task token).
* **OnBase / DMS integration**
  Receives the **signed bundle** (single file), signing log/ID, and **one DIP per vehicle**.
* **Invoice Lambda**
  XML → PDF → email distribution (dealer + SF mailboxes).

## 2.2 Contract Processing Workflow (incl. Ingestion & Handoff)

**End-to-end flow:**

* **Step 0 (outside Step Functions):** S3-triggered **Ingest Lambda** parses the nightly file, writes records to DynamoDB **idempotently**, and **starts one Step Functions execution per bundle**.
* **Steps 1–6 (inside Step Functions):** The **per-bundle** state machine processes vehicles, assembles the bundle PDF, creates the signing session (task token + webhook), stamps, delivers to OnBase, and closes out.

### Step 0 — Ingest & Orchestration Handoff (outside SFN)

1. **Parse & validate** each row.
2. **Idempotent upsert** to DynamoDB using a conditional put on the real key:

   * Table `contracts`, partitioned by `bundle_id`
   * Vehicle item key: `PK=bundle_id`, `SK="VEHICLE#<contract_id>"`
   * Condition: `attribute_not_exists(sk)` to drop duplicates caused by retries
   * Ensure header: `PK=bundle_id`, `SK="HEADER#0"`, `status="NEW"`
3. **Start-once lock:** set `started_at` on the header with `attribute_not_exists(started_at)` to avoid double starts.
4. **Start SFN per bundle:** `StartExecution` with payload `{ "bundle_id": "<id>" }`. If the file contains multiple bundles, repeat for each.

**Output to SFN:**

```json
{ "bundle_id": "<id>" }
```

### Steps 1–7 — Per-bundle state machine (inside SFN)

**1) Initialize Bundle (Lambda)**  
* Load header + vehicles for the bundle  
* Validate uniqueness and ordering  
* Persist a deterministic bundle manifest (`bundle_order.json`)  
* Emit references for artifacts, bundle prefix, and the vehicle list  

---

**2) Process Vehicles (Map)**  
_Per vehicle:_  
* Enrich data with lookups  
* Update vehicle status in DynamoDB  
* Render PDF pages and store short-lived artifacts  
* Emit small references to the generated pages  

---

**3) Assemble Bundle PDF (Lambda)**  
* Read the bundle manifest + per-vehicle page references  
* Concatenate all pages in memory into a single PDF  
* Write one unsigned bundle to storage  
* Emit a pointer to the unsigned bundle  

---

**4) Create Signing Session (Task Token) (Lambda)**  
* Provide the unsigned bundle to Signicat (stream or pre-signed URL)  
* Store signing request info and task token on the bundle header  
* Wait for the webhook callback to resume execution  

**Webhook Callback (API GW + Lambda)**  
* Receive signing status for the bundle  
* Resolve the waiting task token to continue the workflow  

---

**5) Stamp (Lambda)**  
* Download signed PDF from Signicat  
* Apply visible stamps on all pages  
* Write the final stamped PDF to permanent storage  
* Update bundle header

---

**6) Deliver to OnBase (Map)**  
_Per vehicle:_  
* Build a delivery package (DIP)  
* Deliver the same signed bundle + signing log/ID  
* Update per-vehicle status and record delivery receipts
* Set bundle status: `DELIVERED` or `PARTIAL_FAILED`


---

** To be decided how to structure the lambdas inside the step function. Probably it makes sense to merge some of them into a bigger lambda.

## 2.3 Data Model (DynamoDB) — Minimal

**Table:** `contracts` (single table)

**Primary keys**

* **PK:** `bundle_id`
* **SK:** `sk` (typed value: `HEADER#0`, `VEHICLE#<contract_id>`)

**Header item (minimal)**

```json
{
  "bundle_id": "2025-09-15-DEALER123",
  "sk": "HEADER#0",
  "item_type": "header",

  "status": "NEW",                       # NEW | READY | SENT | SIGNED | DELIVERED | PARTIAL_FAILED | FAILED
  "sign_request_id": null,               # set when signing session is created
  "wait_task_token": null,               # set before SFN wait; read by webhook
  "signed_uri": null,                    # set after Stamp/(Optional) Seal

  "started_at": "2025-09-15T21:10:00Z",  # start-once lock
  "created_at": "2025-09-15T21:00:00Z",
  "updated_at": "2025-09-15T21:10:00Z"
}
```

**Vehicle item (minimal)**

```json
{
  "bundle_id": "2025-09-15-DEALER123",
  "sk": "VEHICLE#35972395",
  "item_type": "vehicle",

  "contract_id": "35972395",        # optional (kept for readability)
  "sequence_no": 12,                # deterministic bundling order
  "status": "READY",

  "created_at": "2025-09-15T21:03:40Z",
  "updated_at": "2025-09-15T21:04:12Z"
}
```

---

# 3. Implementation Plan

## 3.1 Work Packages

* **WP1 — Ingest & DynamoDB**

  * S3 trigger, parsing/validation, conditional idempotent writes
  * DynamoDB single-table schema (PK=`bundle_id`, SK=`sk`), timestamps, start-once lock
  * Start Step Functions per bundle
* **WP2 — Contract Processing Workflow**

  * Step Functions definition; Lambdas: Initialize Bundle, Process Vehicles (Map → Enrich/Render), Assemble Bundle PDF, Create Signing Session (task token), Webhook, Stamp & (Optional) Seal, Deliver to OnBase (Map), Closeout
* **WP3 — Templates & Rendering**

  * Contract template(s), rendering engine, artifact layout (or re-render path), PDF optimization
* **WP4 — Signicat Integration**

  * Upload/stream, session creation, **bundle-aware callback URL** or `external_reference=bundle_id`, store/read `wait_task_token`, idempotent retries
* **WP5 — OnBase / DIP Integration**

  * DIP creation API, delivery semantics, receipts, retries, idempotency
* **WP6 — Invoice Flow**

  * XML parsing, invoice template, emailing (dealer + SF mailboxes)
* **WP7 — Observability & Operations**

  * Metrics, dashboards, alarms (SFN failures, long waits), DLQs, runbooks (reprocess bundle, resend signing, partial retry)
* **WP8 — Security & Compliance**

  * KMS policies, least-privilege IAM, Secrets Manager, optional VPC endpoints, LTV (OCSP/CRL) if needed

## 3.2 Acceptance (per WP)

* Idempotent ingest (duplicates dropped), SFN starts per bundle, end-to-end happy path to **signed + delivered**; alerts on failure; invoice emails sent.

---

# 4. Delivery Timeline & Estimates

| Work Package                               | Estimate  |
| ------------------------------------------ | --------- |
| CI/CD & baseline IaC                       | 1 week    |
| WP1 Ingest & DynamoDB                      | 3 weeks   |
| WP2 Workflow (SFN + Lambdas)               | 5 weeks   |
| WP3 Templates & Rendering                  | 1–2 weeks |
| WP4 Signicat Integration                   | 1–2 weeks |
| WP5 OnBase / DIP Integration               | 2 weeks   |
| WP6 Invoice Flow                           | 1 week    |
| WP7–WP8 Observability, Security, hardening | 1–2 weeks |
| Testing (unit/integration/E2E/perf)        | 1–2 weeks |

*Estimates overlap; external integrations drive variance.*

---

# 5. Deliverables

* **IaC** for S3, Lambda, Step Functions, API Gateway, DynamoDB, IAM, alarms
* **DynamoDB schema** (single table, PK/SK only; timestamps; no GSIs)
* **Lambdas:** Ingest, Initialize Bundle, Enrich, Render, Assemble Bundle PDF, Create Signing Session (task-token), Webhook, Stamp & (Optional) Seal, Deliver to OnBase, Closeout, Invoice
* **Templates:** contract & invoice (assets included)
* **Operational docs:** runbooks (reprocess bundle, resend signing, partial retries)
* **Monitoring:** dashboards, alerts, metrics
* **Test assets:** sample interface files & invoice XML; E2E test plan

---


# 8. Test Strategy

* **Unit/contract tests** for Lambdas (parsing, enrichment, rendering)
* **Integration tests:** Step Functions happy path with mocked Signicat & OnBase; webhook path
* **End-to-end:** nightly file → signed & delivered bundle → per-vehicle DIP
* **Performance:** 1k-row batch; RPS caps; PDF size checks
* **Security:** IAM policy validation; S3/KMS enforcement; secret handling


