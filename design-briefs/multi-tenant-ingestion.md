Act as a Principal Cloud & MLOps Infrastructure Architect. I need you to generate a lean, production-ready repository for an LLM Ingestion and PII Redaction Pipeline. 

### Core Architecture & Tech Stack
*   **Ingestion (Go):** A high-throughput API that accepts JSON payloads of production LLM traffic (prompts/responses) annotated with a `tenant_id`. 
*   **Worker (Python):** Consumes payloads from a message queue, evaluates them for PII, runs an automated redaction/masking filter, and commits the clean records to long-term cloud storage.
*   **Infrastructure (Terraform):** Provision a message queue (AWS SQS or Azure Service Bus) and a multi-tenant cloud storage layer (AWS S3 or Azure Blob Storage) implementing strict path or bucket-level tenant isolation.
*   **CI/CD (GitHub Actions):** Build a unified pipeline containing:
    1. Pre-checks: Go/Python linting and testing, `terraform validate` and `tflint`.
    2. IaC Phase: Automated `terraform plan` on Pull Requests and `terraform apply` on main branch merge.
    3. App Deploy: Containerizes both services and deploys them to container runtimes.

### Required Point of Complexity
*   **Go Ingestion Service:** Implement a highly concurrent, memory-efficient token-bucket rate limiter evaluated per `tenant_id` using native channels/mutexes without external cache dependencies.
*   **Python Worker:** Implement a zero-copy stream processing mechanism for real-time regex/NER masking so that massive automotive telemetry text payloads are processed without exceeding tight memory profiles.

### FinOps & Governance Mandate
*   **IaC Default Tags:** You must enforce a strict global tagging strategy within the provider block using `default_tags`. Every resource must inherit: `app:name`, `app:projectName`, `app:component`, `app:teamName`, `app:env`.
*   **FinOps Policy Document (`docs/finops-policy.md`):** Write a concise, mandatory policy explaining how these tags map directly to cost-allocation tags for dashboard filtering, granular budget tracking, and real-time cost exploration.

### Deliverables Needed
1. Complete repository directory structure map.
2. Complete, executable application files (`main.go`, `worker.py`, dependency lockfiles) with mocked ML models but complete cloud SDK hooks.
3. Clean, modular Terraform files leveraging default tags and multi-tenant resource structures.
4. Production-grade GitHub Actions workflows (`.github/workflows/ci-cd.yml`).
5. A crisp, lean `README.md` showing local setup (Docker Compose), deployment sequences, and a highlighted section on the applied FinOps strategy.
