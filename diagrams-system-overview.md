# System Architecture — High-Level Diagrams

Three independent repositories form a complete MLOps/LLMOps pipeline. They are designed to work together but can be deployed separately.

---

## 1. End-to-End Data Flow (All Systems)

```
                    ┌─────────────────────────────────────────────────────────────────┐
                    │                   PRODUCTION LLM TRAFFIC                        │
                    │         (automotive telemetry, chat, inference logs)            │
                    └──────────────────────────┬──────────────────────────────────────┘
                                               │ POST /v1/ingest
                                               ▼
┌──────────────── multi-tenant-ingestion ──────────────────────────────────────────────┐
│                                                                                       │
│   ┌──────────────────────────────────┐        ┌───────────────────────────────────┐  │
│   │  Go Ingestion API (:8080)        │        │  Python Redaction Worker          │  │
│   │  • per-tenant token-bucket       │  SQS   │  • zero-copy streaming redaction  │  │
│   │    rate limiter (50 rps/burst)   │───────▶│  • email, SSN, credit card, phone │  │
│   │  • tenant_id validation          │        │    IPv4, VIN (automotive) masking  │  │
│   │  • body size cap (1 MiB)         │  DLQ   │  • bytearray/memoryview, O(chunk) │  │
│   │  • SQS publish with tenant attr  │  (x5)  │    memory regardless of blob size │  │
│   └──────────────────────────────────┘        └────────────────┬──────────────────┘  │
│                                                                 │ KMS-encrypted PUT   │
│                                                                 ▼                     │
│                                                   S3: tenants/<id>/YYYY/MM/DD/*.json  │
│                                                   • per-tenant IAM prefix isolation   │
│                                                   • lifecycle: IA@30d, Glacier@180d   │
└───────────────────────────────────────────────────────────────┬───────────────────────┘
                                                                │ poll (S3Source)
                                                                ▼
┌──────────────────────── autorater ───────────────────────────────────────────────────┐
│                                                                                       │
│   ┌────────────────────────────────────────────────────────────────────────────────┐  │
│   │  Python Mining Worker (async, EventBridge cron every 15 min)                  │  │
│   │                                                                                │  │
│   │  classify_failure()                                                            │  │
│   │    • retrieval failures (marker strings + empty retrieved_docs)               │  │
│   │    • non-terminating loops (chunk-repeat heuristic)                           │  │
│   │    • truncated outputs (finish_reason=max_tokens + no sentence end)           │  │
│   │    • explicit error_type field                                                 │  │
│   │           │                                                                    │  │
│   │           ▼                                                                    │  │
│   │  SlidingWindowDetector (EWMA baseline, z-score ≥ 3σ → anomalous)             │  │
│   │           │                                                                    │  │
│   │           ▼  (failures only)                                                   │  │
│   │  SemanticDeduplicator (shingle Jaccard ≥ 0.8 → suppressed / cost gate)        │  │
│   │           │                                                                    │  │
│   │           ▼  (novel failures only)                                             │  │
│   │  MockClaudeJudge / real anthropic.messages.create()                           │  │
│   │    RUBRIC_PROMPT → score 0–100 + verdict + rationale                          │  │
│   │           │                                                                    │  │
│   │           ▼  (score ≥ 70 OR window.anomalous)                                 │  │
│   │  AlertClient → POST /v1/alerts (Go alerting engine)                           │  │
│   └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│   ┌──────────────────────────────────────────────────────────────────────────────┐    │
│   │  Go Alerting Engine (:8070)                                                  │    │
│   │  • fingerprint TTL-dedupe (one page per unique regression window)            │    │
│   │  • high severity → Slack                                                     │    │
│   │  • critical severity → Slack + PagerDuty (Events API v2)                    │    │
│   └──────────────────────────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────── data-versioning ─────────────────────────────────────────────────┐
│                                                                                       │
│  GDPR Wipeout Path                          Eval-Set Lineage Path                    │
│  ─────────────────                          ──────────────────────                   │
│  DSAR tooling                               Dataset CI job                            │
│       │                                          │                                    │
│       │ POST /v1/forget                          │ datavault commit <dataset> <dir>   │
│       │ (bearer token, certified=true)           │                                    │
│       ▼                                          ▼                                    │
│  Go Orchestrator (saga)                    .datavault/<ds>/<version>.json             │
│  • Start (idempotent by request_id)        • content-addressed version (sha256)       │
│  • Execute: hot → cold → archive →         • parent chain (lineage)                  │
│    metadata tiers                          • golden flag + signed_off_by              │
│  • atomic JSON checkpoint after each       • tamper-evident validation                │
│    step (FileStore / EFS)                        │                                    │
│  • Resume on crash from checkpoint               │ mirror                             │
│  • GET /v1/forget/{id} status poll               ▼                                    │
│       │                                    S3 metadata bucket                         │
│       ▼                                    (no tenant PII; never purged)              │
│  S3 tier buckets purged per tenant:                                                   │
│  tenants/<id>/* wiped from hot, cold,                                                 │
│  archive, and metadata in sequence                                                    │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Multi-Tenant Ingestion Pipeline (Detail)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  multi-tenant-ingestion                                                          │
│                                                                                  │
│  ┌──────────────┐    ┌──────────────────────────────────────────────────────┐   │
│  │ HTTP Client  │    │  Go: ingestion/server/server.go                      │   │
│  │ (any tenant) │    │                                                      │   │
│  │              │───▶│  POST /v1/ingest                                     │   │
│  │  tenant_id   │    │   1. MaxBytesReader (1 MiB guard)                   │   │
│  │  prompt      │    │   2. JSON decode → Record{TenantID,Prompt,Response}  │   │
│  │  response    │    │   3. Validate tenant_id + payload presence           │   │
│  │  model       │    │   4. ratelimit.Allow(tenant_id) ←──────────────┐    │   │
│  │  metadata    │    │   5. Set ReceivedAt = time.Now().UTC()          │    │   │
│  └──────────────┘    │   6. pub.Publish(ctx, tenant_id, body)          │    │   │
│                      │   7. 202 Accepted {"status":"queued"}           │    │   │
│                      └──────────────────────┬───────────────────────── │───┘   │
│                                             │                          │        │
│                    ┌────────────────────────┘      ┌──────────────────┘        │
│                    │                               │                            │
│                    ▼                               ▼                            │
│         ┌──────────────────┐          ┌──────────────────────────────────────┐ │
│         │  SQS Queue       │          │  ratelimit.Limiter                   │ │
│         │  • SSE enabled   │          │  • token-bucket per tenant           │ │
│         │  • 20s long poll │          │  • lazy refill (no timer goroutines) │ │
│         │  • 120s visibility│         │  • RWMutex + per-bucket Mutex        │ │
│         │  • DLQ after x5  │          │  • janitor evicts idle tenants       │ │
│         └────────┬─────────┘          │  • no Redis dependency               │ │
│                  │                    └──────────────────────────────────────┘ │
│                  │ consume (long poll)                                          │
│                  ▼                                                              │
│         ┌──────────────────────────────────────────────────────────────────┐   │
│         │  Python: worker/worker.py                                         │   │
│         │                                                                   │   │
│         │  for msg in sqs.receive_message(MaxNumberOfMessages=10):          │   │
│         │    record = json.loads(msg.Body)                                  │   │
│         │    clean  = process_record(record, redactor)                      │   │
│         │    committer.commit(tenant_id, clean)                             │   │
│         │    sqs.delete_message(ReceiptHandle)                              │   │
│         │    # KeyError/ValueError → leave for DLQ (poison msg)            │   │
│         └────────────┬─────────────────────────────────────────────────────┘   │
│                      │                                                          │
│                      │   StreamingRedactor.redact_inplace(bytearray)           │
│                      │   • memoryview slices — no payload copy                 │
│                      │   • email / SSN / credit-card / phone / IPv4 / VIN      │
│                      │   • Luhn check on credit-card false-positive cut         │
│                      │   • chunk-overlap streaming (MAX_PII_SPAN=64 carry)      │
│                      │                                                          │
│                      ▼                                                          │
│         ┌──────────────────────────────────────────────────────────────────┐   │
│         │  S3 Data Lake                                                     │   │
│         │  Key: tenants/<tenant_id>/YYYY/MM/DD/<uuid>.json                 │   │
│         │  • KMS CMK (key rotation enabled)                                │   │
│         │  • TLS-only bucket policy                                        │   │
│         │  • per-tenant IAM prefix policies (infra/iam.tf)                 │   │
│         │  • lifecycle: STANDARD → IA@30d → GLACIER@180d                   │   │
│         └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Autorater & Alerting Pipeline (Detail)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  autorater                                                                           │
│                                                                                      │
│  EventBridge cron (rate(15 minutes))                                                 │
│       │  launches Fargate task                                                       │
│       ▼                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │  MiningWorker.run()                                                          │    │
│  │                                                                              │    │
│  │  source.poll()  ←── S3Source (list_objects_v2, cursor via _seen set)        │    │
│  │       │                                                                      │    │
│  │       ▼  per record                                                          │    │
│  │  classify_failure(record)                                                    │    │
│  │       │                         ┌─────────────────────────────────────────┐ │    │
│  │       ├──── None ──────────────▶│  SlidingWindowDetector.observe(False)   │ │    │
│  │       │                         │  (healthy record feeds baseline EWMA)   │ │    │
│  │       ▼                         └─────────────────────────────────────────┘ │    │
│  │  failure_type = "retrieval_failure" | "non_terminating_loop" | ...           │    │
│  │       │                                                                      │    │
│  │       ▼                                                                      │    │
│  │  SlidingWindowDetector.observe(True)                                         │    │
│  │       │ → WindowVerdict{anomalous, failure_rate, baseline_rate, z_score}     │    │
│  │       │                                                                      │    │
│  │       ▼                                                                      │    │
│  │  SemanticDeduplicator.is_duplicate(gate_text)                                │    │
│  │       │                                                                      │    │
│  │       ├── True ──▶ suppressed (dedup.suppressed++) — no judge call           │    │
│  │       │                                                                      │    │
│  │       ▼ False (novel failure)                                                │    │
│  │  MockClaudeJudge.score(case)                                                 │    │
│  │       │  RUBRIC_PROMPT → _invoke() → Verdict{score, verdict, rationale}      │    │
│  │       │  [prod: replace _invoke with anthropic.messages.create()]            │    │
│  │       │                                                                      │    │
│  │       ▼                                                                      │    │
│  │  if score ≥ 70 OR window.anomalous:                                          │    │
│  │       │                                                                      │    │
│  │       ▼                                                                      │    │
│  │  AlertClient.send({fingerprint, case_id, tenant_id, failure_type,            │    │
│  │                    severity, score, summary, window_failure_rate})            │    │
│  │       │  POST → http://alerting.local:8070/v1/alerts                        │    │
│  └───────│──────────────────────────────────────────────────────────────────────┘    │
│          │                                                                            │
│          ▼                                                                            │
│  ┌───────────────────────────────────────────────────────────────────────────────┐   │
│  │  Go Alerting Engine (always-on ECS Fargate × 2 replicas)                      │   │
│  │                                                                                │   │
│  │  POST /v1/alerts                                                               │   │
│  │    1. Decode & validate (fingerprint + severity required)                      │   │
│  │    2. dedupe.Cache.Admit(fingerprint) — TTL window, admits first occurrence    │   │
│  │       └── duplicate → 200 suppressed                                           │   │
│  │    3. Route by severity:                                                       │   │
│  │       high     → [Slack]                                                       │   │
│  │       critical → [Slack, PagerDuty]                                            │   │
│  │    4. dispatch.Slack.Dispatch()  → Block Kit webhook payload                   │   │
│  │       dispatch.PagerDuty.Dispatch() → Events API v2 trigger                    │   │
│  │       (no URL configured → mock mode: logs payload, returns nil)               │   │
│  │    5. 202 Accepted {"status":"dispatched","channels":[...]}                    │   │
│  └───────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Versioning & GDPR Wipeout Engine (Detail)

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  data-versioning                                                                     │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  datavault CLI (Python)                                                       │   │
│  │                                                                               │   │
│  │  datavault commit <dataset> <dir> [--golden] [--signed-off-by <user>]        │   │
│  │       │                                                                       │   │
│  │       ├── build_manifest(dir):                                                │   │
│  │       │     • sha256 every file (chunked, 1 MiB)                             │   │
│  │       │     • version = sha256(sorted file hashes) — content-addressed       │   │
│  │       │     • parent = previous HEAD version (linked list)                   │   │
│  │       │     • golden flag requires signed_off_by                             │   │
│  │       │                                                                       │   │
│  │       ├── validate: version unchanged → no-op                                │   │
│  │       │                                                                       │   │
│  │       └── write .datavault/<dataset>/<version>.json                          │   │
│  │                                                                               │   │
│  │  datavault verify <dataset>                                                   │   │
│  │       │                                                                       │   │
│  │       └── validate_chain():                                                   │   │
│  │             • re-hash all files → detect tampered manifests                  │   │
│  │             • check all parent refs resolve                                   │   │
│  │             • detect cycles (walk to root, track visited set)                 │   │
│  │             • verify single root (no forks in the lineage DAG)               │   │
│  │             • golden versions must have signed_off_by                         │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │  GDPR Wipeout Engine — Go Orchestrator                                        │   │
│  │                                                                               │   │
│  │  POST /v1/forget (Bearer token, constant-time compare)                       │   │
│  │  {request_id, tenant_id, certified:true, requested_by}                       │   │
│  │       │                                                                       │   │
│  │       ▼                                                                       │   │
│  │  saga.Start(request_id, tenant_id)   ← idempotent by request_id             │   │
│  │       │  persist new Saga to FileStore (atomic rename on POSIX)              │   │
│  │       ▼                                                                       │   │
│  │  202 Accepted {"saga_id":..., "status":"running"}                            │   │
│  │       │  (response is immediate; execution is async in goroutine)            │   │
│  │       │                                                                       │   │
│  │  saga.Execute() — async goroutine:                                           │   │
│  │       │                                                                       │   │
│  │       ├── Step: hot tier   → DeletePrefix("tenants/<id>/")                   │   │
│  │       │         checkpoint StepDone (atomic JSON write)                      │   │
│  │       ├── Step: cold tier  → DeletePrefix("tenants/<id>/")                   │   │
│  │       │         checkpoint StepDone                                           │   │
│  │       ├── Step: archive tier → DeletePrefix("tenants/<id>/")                 │   │
│  │       │         checkpoint StepDone                                           │   │
│  │       └── Step: metadata tier → DeletePrefix("tenants/<id>/")                │   │
│  │                 checkpoint Completed                                          │   │
│  │                                                                               │   │
│  │  On crash/timeout at any step:                                                │   │
│  │       • saga checkpointed as StatusFailed with last error                    │   │
│  │       • Resume() on boot reloads all incomplete sagas                        │   │
│  │       • already-Done steps are skipped (idempotency contract)                │   │
│  │       • forward-only: no rollback (purge is compensation-free)               │   │
│  │                                                                               │   │
│  │  GET /v1/forget/{id} → returns current Saga JSON                             │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  Storage Tiers (Terraform)                                                           │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────┐  ┌──────────────────────────┐ │
│  │  hot S3    │  │  cold S3   │  │  archive S3     │  │  metadata S3             │ │
│  │  STANDARD  │  │  → IA@30d  │  │  → GLACIER@1d   │  │  datavault manifests     │ │
│  │  expire@90d│  │  versioned │  │  versioned      │  │  no tenant PII           │ │
│  │  NCV@14d   │  │  NCV@30d   │  │  NCV@30d        │  │  never purged            │ │
│  └────────────┘  └────────────┘  └─────────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. CI/CD Pipeline (All Repos)

```
Pull Request                          Merge to main
─────────────                         ──────────────

  ┌──────────────────────────┐          ┌───────────────────────────────────────┐
  │  Pre-checks (parallel)   │          │  terraform apply                      │
  │  • gofmt + go vet        │  all     │  (OIDC → AWS IAM role, no static keys)│
  │  • go test -race -cover  │  pass    │                                        │
  │  • ruff check + format   │────────▶ │  docker build + push to ECR           │
  │  • python unittest       │          │  (SHA-tagged image, IMMUTABLE repo)    │
  │  • terraform fmt -check  │          │                                        │
  │  • terraform validate    │          │  aws ecs update-service                │
  │  • tflint                │          │  --force-new-deployment                │
  │  • Trivy IaC scan        │          │                                        │
  │    (HIGH/CRITICAL exit 1)│          │  Note: miner needs no ECS rollout —    │
  └──────────────────────────┘          │  EventBridge picks up the new image     │
                │                       │  on the next scheduled sweep           │
                ▼                       └───────────────────────────────────────┘
  ┌──────────────────────────┐
  │  terraform plan          │
  │  posted as PR comment    │
  └──────────────────────────┘
```

---

## 6. Tenant Isolation Model

```
┌──────────────────────────────────────────────────────────────────┐
│  Tenant Isolation — Defense in Depth                             │
│                                                                  │
│  Layer 1: HTTP                                                   │
│    tenant_id required on every ingest record                     │
│    rate limiter keyed per tenant_id (no cross-tenant spillover)  │
│                                                                  │
│  Layer 2: Storage Path                                           │
│    ALL objects: tenants/<tenant_id>/YYYY/MM/DD/<id>.json         │
│    tenant_id validated: no "/" or ".." allowed                   │
│                                                                  │
│  Layer 3: IAM Policies (Terraform — infra/iam.tf)                │
│    per-tenant policy: s3:GetObject + s3:PutObject only on        │
│    arn:aws:s3:::bucket/tenants/<tenant_id>/*                      │
│    s3:ListBucket conditioned on s3:prefix = tenants/<tenant_id>/ │
│                                                                  │
│  Layer 4: Encryption                                             │
│    CMK per bucket, key rotation enabled                          │
│    ServerSideEncryption=aws:kms on every PUT                     │
│    TLS-only bucket policy (DenyInsecureTransport)                │
│                                                                  │
│  Layer 5: GDPR Wipeout                                           │
│    Saga purges tenants/<tenant_id>/* from all 4 storage tiers    │
│    Idempotent, crash-safe, audit-trailed via checkpoint JSON      │
└──────────────────────────────────────────────────────────────────┘
```
