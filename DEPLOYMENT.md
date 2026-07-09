# Deployment Guide — Platform Bring-Up Order

Three repos, one pipeline: **ingest → redact → store → mine → judge → alert**, with dataset
versioning + GDPR erasure as the governance layer. Deploy in the order below — each stage
produces the resources the next one consumes.

```
1. multi-tenant-ingestion      2. autorater                     3. data-versioning
   API + redaction worker  ──▶    miner + judge + alerting  ──▶    datavault + wipeout
   creates the data lake         consumes the lake                 governs lake + eval sets
```

Why this order:

1. **multi-tenant-ingestion** creates the tenant-isolated data lake (S3/Blob) and queue.
   Nothing else runs without the lake.
2. **autorater** points its miner at that lake, provisions the results lake +
   Athena/Synapse query surface, and stands up the alerting engine. Needs the lake bucket
   name from stage 1 and an `OPENROUTER_API_KEY` secret.
3. **data-versioning** governs what the first two produce: datavault versions mined eval
   sets; the wipeout orchestrator's saga purges tenant data across hot/cold/archive tiers.
   Deploy last so its erasure scope covers every tier that exists.

All three repos share the same conventions: Terraform in `iac/aws` (us-east-1) **and**
`iac/azure` (eastus) — pick one provider and stay consistent (`CLOUD_PROVIDER=aws|azure`
selects app backends at startup); GitHub OIDC for CI auth (no static keys); five `app:*`
cost-allocation tags via provider `default_tags`; secrets as SSM SecureStrings / Key Vault
secrets declared by Terraform, values set manually once (commands in each repo's
`iac/README.md`).

---

## Path A — local demo (no cloud credentials)

Full pipeline on a laptop, in dependency order:

```bash
# 1. Ingestion + redaction (LocalStack SQS, API on :8080)
cd multi-tenant-ingestion && docker compose up --build

# 2. Feed it synthetic traffic (healthy + failures + planted PII + safety tripwires)
cd tools/trafficgen && python3 trafficgen.py generate --mode api --count 500 --seed 42

# 3. Alerting engine (:8070, mock dispatch — payloads logged, no Slack/PagerDuty needed)
cd ../../../autorater/alerting && go run .

# 4. Miner sweep over the redacted lake output (mock judge without OPENROUTER_API_KEY;
#    export the key + JUDGE_MODEL=anthropic/claude-sonnet-5 for the real judge)
cd ../miner
LOCAL_DATA_DIR=../../multi-tenant-ingestion/data \
  ALERT_WEBHOOK_URL=http://localhost:8070/v1/alerts \
  CURSOR_FILE=./.miner-cursor.json python -m miner.worker
# run it twice: the second sweep processes 0 records (durable cursor)

# 5. Governance layer — wipeout engine (:8090) + dataset versioning
cd ../../data-versioning/orchestrator
API_TOKEN=dev-secret CHECKPOINT_DIR=/tmp/w/ckpt TIER_ROOT=/tmp/w/tiers go run .
cd ../versioning && python -m datavault.cli commit evalset ./my-dataset
```

Smoke-test skills exist per repo: `multi-tenant-ingestion:run-e2e-smoke`,
`autorater:run-e2e-smoke`, `debug-stuck-saga`.

## Path B — cloud (Terraform, per repo, in order)

Per repo the sequence is always: **secrets → `terraform apply` → images/services**.
CI does all of it on merge to `main` (plan is posted on the PR); the manual equivalent:

### 1. multi-tenant-ingestion
```bash
cd multi-tenant-ingestion/iac/aws        # or iac/azure
terraform init && terraform apply        # SQS+DLQ / Service Bus, data lake, per-tenant IAM,
                                         # dashboards, alarms (set ops_alert_email)
```
Build/push both images (ingestion API, redaction worker); services read
`SQS_QUEUE_URL` + `DATA_LAKE_BUCKET` from Terraform outputs.

### 2. autorater
```bash
# secrets first — declared by TF, values set once:
#   /projects/autorater/OPENROUTER_API_KEY, SLACK_WEBHOOK_URL, PAGERDUTY_ROUTING_KEY
cd autorater/iac/aws && terraform init && terraform apply
# ECS + EventBridge cron (miner), alerting service, DynamoDB cursor/lease,
# results lake + Glue/Athena workgroup, CloudWatch dashboard
```
Point the miner at stage 1's lake bucket. The judge defaults to
`anthropic/claude-sonnet-5` (medium effort); override via `judge_model` TF var or
`JUDGE_MODEL` env. The miner needs no rollout on redeploys — EventBridge launches the
fresh image on the next scheduled sweep; only the alerting service is rolled.

### 3. data-versioning
```bash
# secret first: /projects/data-versioning/API_TOKEN
cd data-versioning/iac/aws && terraform init && terraform apply
# tier buckets + lifecycle (Hot→IA→Glacier), scoped deletion IAM, orchestrator service
```
Wire DSAR tooling to `POST /v1/forget` (bearer auth, `certified: true`). datavault
manifests mirror to the metadata bucket from dataset CI jobs.

## Verify the deployment (end-to-end)

1. POST a payload with planted PII to `/v1/ingest` → object lands under
   `tenants/<tenant_id>/…` with PII masked (`trafficgen.py verify` leak-scans the lake).
2. Wait one miner sweep (or trigger the scheduled task) → `sweep_summary` log shows
   `judge_calls` vs `suppressed_by_dedup`; judged rows queryable in Athena
   (`autorater-<env>` workgroup) and on the CloudWatch dashboards.
3. A planted safety case (e.g. prompt injection) raises a **critical** alert →
   Slack + PagerDuty.
4. `POST /v1/forget` for a test tenant → saga completes hot→cold→archive→metadata;
   `wipeout-audit` saved query shows the trail; the tenant prefix is empty.
