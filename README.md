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
* **Steps 1–7 (inside Step Functions):** The **per-bundle** state machine processes vehicles, assembles the bundle PDF, creates the signing session (task token + webhook), stamps/seals, delivers to OnBase, and closes out.

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

* `Query(bundle_id)` to load header + vehicles
* Validate presence, uniqueness, and ordering
* Emit:

```json
{
  "bundle_id": "2025-09-15-DEALER123",
  "bundle_prefix": "2025-09-15-DEALER123/",
  "vehicles": [
    { "contract_id": "35972395", "sequence_no": 12 },
    { "contract_id": "35972396", "sequence_no": 13 }
  ],
  "vehicle_count": 2
}
```


**2) Process Vehicles (Map)**
Per vehicle:

* **Enrich** with third-party lookups; upsert to DynamoDB; set `status`
* **Render** pages in memory; write short-lived artifacts to S3:
  `s3://contract-artifacts/{bundle_prefix}{contract_id}/pages/page_0001.pdf`
* Return **tiny pointers** only: `{ contract_id, vehicle_page_prefix }`

**3) Assemble Bundle PDF (Lambda)**

* Read `bundle_order.json` + `vehicle_page_prefix` values
* **Stream-concatenate** pages into a single multi-page PDF **in memory**
* Write one **unsigned** bundle to:
  `s3://contract-artifacts/{bundle_prefix}bundle.pdf` (SSE-KMS; lifecycle 1–2 days)
* Return pointer:

  ```json
  {
      "bucket": "<bucket_name>",
      "key": "<key>",
      "version_id": "<version_id>",
      "sha256": "<hash>",
      "size_bytes": "123456"
  }
  ```
  
**4) Create Signing Session (Task Token) (Lambda)**

* Stream the unsigned `bundle.pdf` to Signicat (or provide a pre-signed GET URL)
* Store `sign_request_id` on the header
* Store `wait_task_token` on the header
* Use **bundle-aware callback URL** (e.g., `/sign-webhook/{bundle_id}`) *or* set Signicat `external_reference=bundle_id`
* **Wait** for the webhook (task token)

**Webhook Callback (API GW + Lambda)**

* Extract `bundle_id` from the URL path (or from `external_reference` in the payload)
* `GetItem` header → read `wait_task_token` → call `SendTaskSuccess/Failure` to resume SFN

**5) Stamp & (Optional) Seal (Lambda)**

* Download signer’s PDF; **stamp all pages** visibly
* If required, apply an organizational **seal** (second PAdES) to protect post-processing
* Write final to **permanent** bucket: `s3://contract-signed/bundles/{bundle_id}/bundle_stamped.pdf`
* Update header `signed_uri`
* **Cleanup** unsigned artifacts (including the temp `bundle.pdf`)

**6) Deliver to OnBase (Map)**
Per vehicle:

* Build **DIP**, deliver the **same signed bundle** + signing log/ID to OnBase
* Update per-vehicle `status`; capture delivery receipts; optional notifications

**7) Closeout & Notifications (Lambda)**

* If all delivered → header `status=DELIVERED`, else `PARTIAL_FAILED`
* Emit summary notification (email/SNS/Teams)

> **Handoff rule:** States pass **only small JSON pointers** (S3 `bucket/key/version_id/sha256`). Never pass PDF bytes through SFN (256 KB limit). If strict on unsigned persistence, **merge Steps 3 & 4**: assemble in memory and stream directly to Signicat in the same task.

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

**Access patterns (no GSIs)**

* **Ingest:** conditional `PutItem` on `attribute_not_exists(sk)` for vehicles; `UpdateItem` header with `attribute_not_exists(started_at)`
* **Workflow:** `Query(bundle_id)` to read header + vehicles; update header `status`, `sign_request_id`, `wait_task_token`, `signed_uri`
* **Webhook:** `bundle_id` from path or external reference → `GetItem` header → use `wait_task_token` to `SendTaskSuccess/Failure`
* **Ops:** CloudWatch + Step Functions metrics/logs/dashboards for cross-bundle visibility (no indexes needed)

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

# 4. Non-Functional Requirements & SLAs

* **Security:** SSE-KMS on all buckets; IAM least privilege; secrets in AWS Secrets Manager; optional private egress (VPC endpoints). Default: **do not retain unsigned PDFs** beyond short-lived artifacts; retain **final signed PDF + evidence** long-term with lifecycle to archive.
* **Reliability:** conditional writes for idempotency; per-task `Retry` with backoff/jitter; explicit `TimeoutSeconds`; DLQs/alerts.
* **Performance:** Map `MaxConcurrency` sized to vendor RPS (e.g., 10–50); nightly volume ≈ 1,000 records.
* **Observability:** per-stage metrics; dashboards; alarms for SFN execution failures and long signing waits; structured logs with `bundle_id` / `contract_id`.
* **Compliance/Audit:** store signing IDs/logs; make final documents optionally LTV (persist OCSP/CRL data).

---

# 5. Delivery Timeline & Estimates

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

# 6. Deliverables

* **IaC** for S3, Lambda, Step Functions, API Gateway, DynamoDB, IAM, alarms
* **DynamoDB schema** (single table, PK/SK only; timestamps; no GSIs)
* **Lambdas:** Ingest, Initialize Bundle, Enrich, Render, Assemble Bundle PDF, Create Signing Session (task-token), Webhook, Stamp & (Optional) Seal, Deliver to OnBase, Closeout, Invoice
* **Templates:** contract & invoice (assets included)
* **Operational docs:** runbooks (reprocess bundle, resend signing, partial retries)
* **Monitoring:** dashboards, alerts, metrics
* **Test assets:** sample interface files & invoice XML; E2E test plan

---

# 7. Environments & CI/CD

* **Environments:** `dev`, `stage`, `prod` (isolated KMS keys and buckets)
* **Pipelines:** lint/type-check → unit tests → synth/deploy IaC → integration tests (stage) → manual approval → prod
* **Secrets:** per-env in Secrets Manager; rotation policy documented

---

# 8. Test Strategy

* **Unit/contract tests** for Lambdas (parsing, enrichment, rendering)
* **Integration tests:** Step Functions happy path with mocked Signicat & OnBase; webhook path
* **End-to-end:** nightly file → signed & delivered bundle → per-vehicle DIP
* **Performance:** 1k-row batch; RPS caps; PDF size checks
* **Security:** IAM policy validation; S3/KMS enforcement; secret handling

---

# 9. Risks & Mitigations

* **Vendor RPS/latency:** bound Map concurrency; retries with jitter; circuit breakers
* **Large PDFs:** optimize assets; split to multi-document session if needed
* **Webhook failures:** retries + DLQ; alerting; manual resume runbook
* **OnBase variability:** early E2E spike; clear error semantics; DLQ + replay

---

# 10. Outstanding Decisions

* Final policy on **artifacts** (temporary page artifacts vs. strict in-memory re-render)
* Required **per-page stamp** content and whether to **seal** post-stamp
* Source of truth and cadence for **dealer ↔ signee** mappings
