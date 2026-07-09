# mlops-platform-docs

Shared engineering documentation for the three-repo MLOps/LLMOps platform —
`multi-tenant-ingestion`, `data-versioning`, `autorater`. Each of those repos
owns its own `docs/` (architecture, cloud portability, FinOps, specs); this
repo holds everything that's about the platform as a whole rather than any
one system.

| File | What it is |
|---|---|
| `docs/architecture-review.html` | Interactive, dark-mode diagram set for an Enterprise Architecture review — system context, AWS↔Azure deployment topology, data classification, security boundaries, runtime sequences, risk register, capability map. Open directly in a browser. |
| `diagrams-system-overview.md` | Hand-drawn ASCII end-to-end data flow across all three systems. |
| `ROADMAP.md` | Engineering log: gaps found in review and how each was closed, plus the forward roadmap. |
| `DEPLOYMENT.md` | Cross-repo deployment notes. |
| `design-briefs/` | The original architectural briefs each repo was built from. |

## Why this repo exists

The three system repos are independently deployable and each is
self-documenting for its own scope. Nothing here should duplicate
repo-local docs — if a fact is true of one system only, it belongs in that
system's own `docs/`, not here.
