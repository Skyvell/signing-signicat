# 1. Overview

Build a serverless AWS solution that (a) creates one contract PDF per vehicle from the nightly NSC interface file, supporting single-page and **bundled** multi-page PDFs with duplicate protection; (b) initiates and tracks e-signing via **Signicat** to the authorized dealer signee; (c) enforces a **visible e-signature stamp on all pages**; and (d) delivers the **signed PDF + signing log/ID + one DIP per vehicle to OnBase** (for bundles, the same file is referenced per vehicle). Additionally, generate and email an **SF “momsfaktura” PDF** from invoice XML to the dealer and SF mailboxes.

---

# 2. System Architecture

Serverless, event-driven. **Ingestion stays thin**; long-lived orchestration runs in a single **AWS Step Functions (Standard)** execution per bundle:

**Initialize Bundle → Process Vehicles (Map) → Assemble Bundle PDF → Create Signing Session (Task Token) → Deliver to OnBase (Map).

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

### Steps 1–6 — Per-bundle state machine (inside SFN)

> **Note:** We can merge smaller states into larger Lambdas depending on throughput and cost goals. Payloads remain minimal — Step Functions just passes `bundle_id` (and in Maps, `bundle_id + contract_id`).


**1) Initialize Bundle (Lambda)**  
* Query DynamoDB for the header and all vehicles for the bundle (`PK = bundle_id`).  
* Validate uniqueness and that each vehicle has a `sequence_no`.  
* Count vehicles and update the header with `vehicle_count`.  
* Set `status = READY`.  

**2) Process Vehicles (Map)**  
* Input per item: `{ bundle_id, contract_id }`.  
* Iterator Lambda responsibilities:  
  * Read the vehicle item (`PK=bundle_id, SK=VEHICLE#contract_id`).  
  * Enrich with external lookups if required.  
  * Render per-vehicle contract PDF pages into S3 storage.  
  * Update vehicle `status = RENDERED` in DynamoDB.  

**3) Assemble Bundle PDF (Lambda)**  
* Query all vehicle items for the bundle.  
* Sort them by `sequence_no` from DynamoDB.  
* Read their rendered pages from S3 and concatenate into a single unsigned PDF.  
* Store the unsigned bundle in S3 and update the header with `unsigned_bundle_uri`.  

**4) Create Signing Session (Task Token) (Lambda)**  
* Read `unsigned_bundle_uri` from the header.  
* Start a signing session with Signicat (provide pre-signed S3 URL).  
* Store `sign_request_id` and `wait_task_token` on the header.
* Pause execution until webhook callback arrives.  

**Webhook Callback (API GW + Lambda)**  
* Receive signing status for the bundle.  
* Look up the header with `bundle_id`.  
* Use the stored task token to resume the waiting Step Functions execution.  

**5) Finalize Signing (Lambda)**  
* Download the signed PDF from Signicat.    
* Write the finalized signed bundle to S3.  
* Update the header with `signed_bundle_uri` and `status = SIGNED`.  
* Store `signing_log_uri` if provided by Signicat.  

**6) Deliver to OnBase (Map)**  
* Input per item: `{ bundle_id, contract_id }`.  
* Iterator Lambda responsibilities:  
  * Read the vehicle item and the header’s `signed_bundle_uri` + `signing_log_uri`.  
  * Build a DIP package for the vehicle.  
  * Deliver the signed bundle, signing log/ID, and DIP to OnBase.  
  * Update vehicle `status = DELIVERED`, plus `dip_id`, `onbase_receipt`, and `delivered_at`.  
* After the Map completes, set bundle header `status = DELIVERED` or `PARTIAL_FAILED`.  

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

  "status": "NEW",                       // NEW | READY | SIGNED | DELIVERED | PARTIAL_FAILED | FAILED
  "vehicle_count": 42,                   // convenience count

  "unsigned_bundle_uri": null,           // set after assemble
  "signed_bundle_uri": null,             // set after signing + stamping
  "signing_log_uri": null,               // set after signing

  "sign_request_id": null,               // set when signing session is created
  "wait_task_token": null,               // set before SFN wait; read by webhook

  "started_at": "2025-09-15T21:10:00Z",  // start-once lock
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

  "contract_id": "35972395",
  "sequence_no": 12,                 // deterministic bundle order
  "status": "READY",                 // READY | RENDERED | DELIVERED | FAILED

  "dip_id": null,                    // set on delivery
  "onbase_receipt": null,
  "delivered_at": null,

  "created_at": "2025-09-15T21:03:40Z",
  "updated_at": "2025-09-15T21:04:12Z"
}
```

---

# 3. Implementation Plan (Agile / Chronological)

## Step 1 — Foundations & Infrastructure
- Set up baseline IaC for core AWS services (S3, DynamoDB, Step Functions, Lambda, API Gateway).  
- Establish CI/CD pipeline. 
- Create isolated dev/staging environments with their own resources and keys.  

**Goal:** Have a deployable skeleton environment with minimal resources.  

## Step 2 — Ingest & Orchestration
- Implement S3-triggered Ingest Lambda:  
  - Parse interface files.  
  - Validate rows.  
  - Write header + vehicles into DynamoDB with conditional puts (idempotent).  
- Add start-once lock on header (`started_at`).  
- Launch one Step Functions execution per bundle with `{ bundle_id }`.  

**Goal:** Reliable ingestion flow that seeds DynamoDB and starts orchestration.  

## Step 3 — Bundle Initialization
- Implement Initialize Bundle Lambda:  
  - Query DynamoDB for header + vehicles.  
  - Validate uniqueness and that every vehicle has a `sequence_no`.  
  - Count vehicles and update header with `vehicle_count`.  
  - Set header status to `READY`.  

**Goal:** Deterministic bundle setup with ordering and vehicle count tracked in DynamoDB.  

---

## Step 4 — Vehicle Processing & Rendering
- Implement vehicle processing in a Step Functions Map state:  
  - Each iterator Lambda receives `{ bundle_id, contract_id }`.  
  - Read vehicle item from DynamoDB.  
  - Enrich with external lookups if required.  
  - Render contract PDF pages into S3 storage.  
  - Update vehicle status to `RENDERED`.  

**Goal:** Individual vehicle contracts are enriched and rendered correctly.

## Step 5 — Assemble Bundle
- Implement Assembly Lambda:  
  - Query all vehicle items for the bundle.  
  - Sort vehicles by `sequence_no`.  
  - Concatenate their rendered pages into one unsigned PDF.  
  - Store the unsigned bundle in S3 and update header with `unsigned_bundle_uri`.  

**Goal:** Bundled contracts are available as a single unsigned PDF, tracked in DynamoDB.  

## Step 6 — Signing Integration
- Implement Create Signing Session Lambda:  
  - Read `unsigned_bundle_uri` from the header.  
  - Initiate signing session with Signicat (pre-signed S3 URL).  
  - Store `sign_request_id` and `wait_task_token` on the header.  
- Implement API Gateway + Webhook Lambda:  
  - On callback, read `bundle_id` from payload.  
  - Look up header and use stored task token to resume Step Functions execution.  

**Goal:** Full signing loop established, with workflow paused and resumed via webhook.

## Step 7 — Finalization
- Implement Finalize Signing Lambda:  
  - Download signed bundle from Signicat.  
  - Write finalized signed bundle to permanent S3 storage.  
  - Update header with `signed_bundle_uri`, `signing_log_uri`, and status = `SIGNED`.  

**Goal:** Produce finalized, signed contracts ready for delivery and archive.  

## Step 8 — Delivery to OnBase
- Implement delivery Map state:  
  - Each iterator Lambda receives `{ bundle_id, contract_id }`.  
  - Read vehicle item and the header’s `signed_bundle_uri` + `signing_log_uri`.  
  - Build a DIP package for the vehicle.  
  - Deliver signed bundle + signing log/ID + DIP to OnBase.  
  - Update vehicle with `status = DELIVERED`, `dip_id`, `onbase_receipt`, and `delivered_at`.  
- After Map finishes, update header status to `DELIVERED` or `PARTIAL_FAILED`.  

**Goal:** Signed contracts are reliably delivered to OnBase with per-vehicle tracking.  

## Step 9 — Parallel Invoice Flow
- Implement invoice processing Lambda:  
  - Parse invoice XML.  
  - Generate invoice PDF.  
  - Distribute via email to dealer + SF mailboxes.  

**Goal:** Independent invoice distribution path runs in parallel to contract workflow.  

---

# 4. Delivery Timeline & Estimates

| Work Package / Step                         | Estimate  |
| ------------------------------------------  | --------- |
| Step 1 — Foundations & Infrastructure       | 1 week    |
| Step 2 — Ingest & Orchestration             | 2 weeks   |
| Step 3 — Bundle Initialization              | 1 week    |
| Step 4 — Vehicle Processing & Rendering     | 2 weeks   |
| Step 5 — Assemble Bundle                    | 1 week    |
| Step 6 — Signing Integration (incl. Webhook)| 2 weeks   |
| Step 7 — Stamping & Finalization            | 1 week    |
| Step 8 — Delivery to OnBase                 | 1 week    |
| Step 9 — Parallel Invoice Flow              | 1 week    |
| Integration & End-to-End Testing            | 2 weeks   |

---

# 5. Deliverables

* **IaC** for S3, Lambda, Step Functions, API Gateway, DynamoDB, IAM, alarms
* **DynamoDB schema** (single table, PK/SK only; timestamps)
* **Lambdas:** Ingest, Initialize Bundle, Enrich, Render, Assemble Bundle PDF, Create Signing Session (task-token), Webhook, Stamp, Deliver to OnBase, Invoice
* **Test assets:** sample interface files & invoice XML; E2E test plan


