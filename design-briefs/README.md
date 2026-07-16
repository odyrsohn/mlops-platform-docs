# Design Briefs

These are the original architectural briefs used to direct AI-augmented implementation of the three repositories — the AyaMind delivery method: a senior architect specifies the system boundaries, the required points of complexity (zero-copy streaming redaction, Saga-pattern erasure, semantic-dedup cost gating), and the governance mandates; AI tooling executes the implementation under review.

The briefs are published as-is, deliberately. Comparing a brief against its repository shows exactly what this method produces per unit of specification — and the subsequent review-and-harden commits (see [`../ROADMAP.md`](../ROADMAP.md)) show how the output is audited and closed to production posture.

| Brief | Repository | What it covers |
|---|---|---|
| [`multi-tenant-ingestion.md`](multi-tenant-ingestion.md) | [odyrsohn/multi-tenant-ingestion](https://github.com/odyrsohn/multi-tenant-ingestion) | Ingestion API, streaming PII redaction, per-tenant isolation |
| [`autorater.md`](autorater.md) | [odyrsohn/autorater](https://github.com/odyrsohn/autorater) | Eval mining, LLM-as-Judge, safety classification, regression alerting |
| [`data-versioning.md`](data-versioning.md) | [odyrsohn/data-versioning](https://github.com/odyrsohn/data-versioning) | Dataset lineage / golden-set provenance, GDPR right-to-erasure engine |
