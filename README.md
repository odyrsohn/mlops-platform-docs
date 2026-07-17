# mlops-platform-docs

Shared engineering documentation for the three-repo MLOps/LLMOps platform —
`multi-tenant-ingestion`, `data-versioning`, `autorater`. Each of those repos
owns its own `docs/` (architecture, cloud portability, FinOps, specs); this
repo holds everything that's about the platform as a whole rather than any
one system.

The three systems are one pipeline — **ingest → redact → store → mine → judge → alert** —
with dataset versioning and GDPR right-to-erasure as the governance layer over it.

## What's here

### [Architecture review](https://htmlpreview.github.io/?https://github.com/odyrsohn/mlops-platform-docs/blob/main/docs/architecture-review.html) — the platform as a reviewer would interrogate it

Seven sheets. The system context and the actors who touch it. The AWS↔Azure deployment
topology, where every resource lane has a symmetric twin on the other cloud and the one
remaining asymmetry is called out rather than buried. The path PII takes in, the point
it gets scrubbed, and how tenant data decays or is erased across storage tiers. Which
identity holds which privilege across the trust boundary — the erasure orchestrator
alone can permanently delete tenant data. Two runtime sequences you can step through: a
mining sweep, and an erasure saga surviving a mid-flight crash. A register of thirteen
known items — nine cloud-parity gaps, four open design questions — plotted by impact
against effort to close. And seven governance capabilities mapped to the systems that
deliver them.

Rendered above through htmlpreview; [the source file](docs/architecture-review.html) opens standalone in a browser.

### [`diagrams-system-overview.md`](diagrams-system-overview.md) — how the data actually moves

End-to-end flow across all three systems, then a detail pass on each: the ingestion and
redaction pipeline, the autorater mining/judging/alerting path, and the versioning and
GDPR wipeout engine. Closes with the shared CI/CD pipeline and the tenant isolation model.

### [`DEPLOYMENT.md`](DEPLOYMENT.md) — bringing the platform up

The order the three repos deploy in and why each stage produces what the next one
consumes. Two paths: the full pipeline on a laptop with no cloud credentials, and the
per-repo Terraform sequence (secrets → apply → images) against AWS or Azure. Ends with
an end-to-end verification — plant PII and watch it get masked, trip a safety alert,
erase a tenant and confirm the prefix is empty.

### [`ROADMAP.md`](ROADMAP.md) — what was found, what was closed, what was deferred

The log of a review-and-harden pass against the spec: each gap found, and either the
code that closed it or the milestone that will. Carries the deliberate simplifications
with a named production path for each, the reasoning behind the Claude Sonnet 5 judge
default and the two-tier escalation design meant to supersede it, and the forward
roadmap mapped to a 30/90/180-day engagement.

### [`design-briefs/`](design-briefs/) — what each repo was told to be

The original architectural briefs, published as-is: specified boundaries, required
points of complexity, governance mandates. Reading a brief against its repository is
the point of keeping them.

## Why this repo exists

The three system repos are independently deployable and each is
self-documenting for its own scope. Nothing here should duplicate
repo-local docs — if a fact is true of one system only, it belongs in that
system's own `docs/`, not here.
