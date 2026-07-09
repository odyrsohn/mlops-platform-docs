# Platform Roadmap & Engineering Log

**Scope:** three repositories — `multi-tenant-ingestion`, `autorater`, `data-versioning` — implementing the P0 skeleton of a production LLM Ops pipeline: ingest → redact → store → slice → classify → judge → mine → alert, with per-tenant isolation, GDPR right-to-erasure, and FinOps controls throughout.

**Method:** senior-architected, AI-augmented delivery. Each repo started from a written design brief (see `design-briefs/`), was implemented with AI tooling under architectural direction, then went through a structured review-and-harden pass. This document is the log of that pass and the forward roadmap.

---

## Review-and-harden log (gaps found → closed)

A deliberate internal review was run against the full requirement spec behind the design briefs. Every finding was either fixed or scheduled. That loop — build, review honestly, close — is the workflow these repos demonstrate.

| Gap found in review | Status | Where |
|---|---|---|
| No query/slicing surface for on-call debugging | ✅ **Closed** | Athena workgroup + Glue catalog over judged cases, canned named queries (`autorater/iac/analytics.tf`); CloudWatch business & ops dashboards; saved Logs Insights slicing menu — by-tenant, by-failure-mode, by-language, by-AAOS-client, by-serving-model (`*/iac/queries.tf`) over a canonical structured-log envelope |
| LLM-as-Judge mocked only | ✅ **Closed** | Real chat-completions judge via OpenRouter; provider/model is a single env var (`JUDGE_MODEL`); mock retained for keyless dev; judge failures degrade to a conservative verdict, never crash a sweep (`autorater/miner/miner/judge.py`) |
| No safety/abuse classification | ✅ **Closed** | Rule-based classifier over prompt and response — prompt-injection, self-harm, abusive language, PII leak — with severity routing into the same dedup→judge→alert pipeline; interface-shaped for a model-backed upgrade (`autorater/miner/miner/safety.py`) |
| Miner cursor was in-memory (O(n) re-scan per sweep) | ✅ **Closed** | Durable `CursorStore` (DynamoDB / file backends), `StartAfter` resume, single-runner lease, at-least-once advancement (`autorater/miner/miner/sources.py`) |
| Unstructured logs, no slice dimensions | ✅ **Closed** | Canonical single-line JSON envelope across all services (Go `slog`, Python `obslog`), stable event names as a metrics contract, no prompt/response content ever logged (`.plan/standardized-logging.md`) |
| Aggregate-only analytics surface distinct from trace-level | 📌 **Scheduled — next milestone** | See "Next milestones" below |
| Regression alerts not tied to deploy events | 📌 **Scheduled — next milestone** | See "Next milestones" below |

## Next milestones (designed, not yet implemented)

Two requirements are intentionally scheduled rather than built in the showcase — each has a concrete design ready to execute:

1. **Aggregate-only analytics surface** (~half a day). The trace-level surface exists (Athena workgroup, Glue catalog, saved queries); the second surface — aggregate-only, safe for wider access — will be: an Athena view exposing counts and dimensions only (no prompt/response columns), a dedicated workgroup, and an analyst IAM policy granting workgroup access without raw-bucket read. Hard column-level enforcement (Lake Formation on AWS / Purview on Azure) lands with the client's environment.
2. **Deploy-event hook** (~half a day). The alerting service gains a `POST /v1/deploys` endpoint (or an EventBridge route) recording `model_version`/`prompt_version` deploy events; alerts are then annotated "started after deploy X" for the affected serving model. Logging already slices by `serving_model`, so this is an annotation join, not a pipeline change.

## Judge default switched to Claude Sonnet 5 — deployment steps pending real infra

The judge's default model and reasoning effort (`DEFAULT_MODEL`/`DEFAULT_REASONING_EFFORT` in `judge.py`; `judge_model`/`judge_reasoning_effort` Terraform variables in both `iac/aws` and `iac/azure`) are set to **Claude Sonnet 5 at `medium` reasoning effort**, since the target stack is Claude-first and a judge is only as useful as its ability to catch the subtle failure modes the spec calls out (hallucination vs. contamination, nuanced retrieval failures) — a fast/cheap model is the wrong place to economize once the semantic-dedup gate already controls call volume. That's the part doable inside a showcase repo. Three steps remain and require a real deployment target, so they're scheduled for the engagement rather than done here:

| Step | What it requires | When |
|---|---|---|
| Roll the new default to a live environment | A real OpenRouter key + an applied `terraform apply` against the client's AWS/Azure account (`OPENROUTER_API_KEY` in SSM/Key Vault already wired — no code change needed) | Milestone 1 (staging) |
| Confirm the miner picks it up | No service roll needed — EventBridge/Container Apps Job launches the new task definition on the next scheduled sweep | Milestone 1 |
| Measure the switch | Compare `judge-usage-by-model` (Athena/Synapse named query — avg score, call volume, token cost by judge model per day) for a few days pre/post switch against real production traffic | Milestone 2, once traffic is flowing |

### Next improvement: two-tier judge escalation

Flat Sonnet-for-everything is the correct default, but it isn't the final architecture. The stronger design — flagged here for a future milestone (M3, once the cost-gate's real suppression rate is known from production data):

- **Tier 1 — fast triage**, cheap/fast model (Haiku-class or the previous Gemini Flash default) scores every case that survives the semantic-dedup gate.
- **Escalation rule** — a case only goes to **Tier 2** when Tier 1 is inconclusive: near the `SEVERE_THRESHOLD` boundary, a safety-forced case, or a sliding-window anomaly. Everything clearly benign or clearly severe stops at Tier 1.
- **Tier 2 — confirmatory judge**, Claude Sonnet 5 (or higher effort) re-scores only the escalated fraction, catching what a fast model would misjudge on the cases that matter most.
- Implementation sketch: `BaseJudge` already separates model selection from scoring logic, so Tier 2 is a second `OpenRouterJudge` instance invoked from `MiningWorker._judge_case` when Tier 1's verdict falls in the escalation band — no change to the dedup gate or results schema, only a new `judge_tier`/`escalated` field to add to the results sink and Glue columns.
- This directly answers the "autorater methodology and contamination defenses" requirement with a concrete, cost-aware design rather than a single fixed model.

## Known simplifications (deliberate, with the production path named)

| Simplification | Production path |
|---|---|
| Safety classifier is rule-based first pass | Swap in a moderation-model screen behind the existing interface; evaluate Azure AI Content Safety vs Llama Guard vs Claude-based on the client's traffic |
| Column-level enforcement of the aggregate-only surface relies on view shape + workgroup/IAM separation | Lake Formation grants (AWS) or Purview policies (Azure) for hard column-level enforcement in the client's environment |
| Cloud target is AWS | All infra is Terraform behind interfaces; the Azure port and self-host-vs-managed benchmark are the Discovery-phase deliverable |
| Eval-suite write path & human-review PII gate are separate components, not yet one workflow | Wired end-to-end (mined case → review queue → PII gate → versioned golden set with provenance) in the production build-out phase |

## Forward roadmap (maps to a 30/90/180-day production engagement)

- **Phase 0 (weeks 1–2):** toolchain benchmark (analytics product, data-versioning layer, self-host vs managed), target architecture on the client's cloud, governance memo.
- **Phase 1 (≈ day 30):** ingestion + redaction E2E against staging data on client infrastructure; measured redaction precision/recall.
- **Phase 2 (≈ day 90):** P0 in production — safety classification, tiered storage + retention, both dashboard surfaces with RBAC, feedback channel, right-to-erasure E2E, first mined eval cases through the human-review PII gate with provenance.
- **Phase 3 (≈ day 180):** eval mining as weekly routine; alerts tuned with the AI team (not noisy, not silent) and tied to deploys; per-tenant/per-feature cost attribution reconciled monthly; new tenants onboarded from a runbook with zero pipeline rework.
