Act as a Principal Cloud & MLOps Infrastructure Architect. I need you to generate a lean, production-ready repository for an Automated Evaluation-Mining Pipeline and an LLM-as-Judge Autorater.

### Core Architecture & Tech Stack
*   **Mining Worker (Python):** An asynchronous worker that continuously monitors or polls ingested production data inside cloud storage, detects potential runtime failures (e.g., retrieval failures, non-terminating loops), extracts those cases, and evaluates them using an LLM-as-Judge scoring method (mocking a Claude API invocation).
*   **Alerting Engine (Go):** A dedicated microservice exposing a webhook that the Python worker calls whenever a severe prompt/model regression or failure is caught. It processes, deduplicates, and dispatches regression alerts (mocked Slack/PagerDuty payload).
*   **Infrastructure (Terraform):** Provision the compute cluster workloads, serverless trigger mechanisms or cron engines, and an integrated cloud observability/tracing hub (AWS CloudWatch/X-Ray or Azure Application Insights).
*   **CI/CD (GitHub Actions):** Build a unified pipeline containing:
    1. Pre-checks: Code linting, unit tests, `terraform validate`, and infrastructure security scans.
    2. IaC Phase: Automated `terraform plan` on Pull Requests and `terraform apply` on main branch merge.
    3. App Deploy: Validates, builds, and pushes the services to their target runtimes.

### Required Point of Complexity
*   **Python Miner Service:** Implement a sliding-window anomaly detection and semantic deduplication gate *before* hitting the LLM-as-Judge scoring block. This ensures identical production failures do not repeatedly invoke the LLM API, directly embedding programmatic cost controls into the ML system design.

### FinOps & Governance Mandate
*   **IaC Default Tags:** You must enforce a strict global tagging strategy within the provider block using `default_tags`. Every resource must inherit: `app:name`, `app:projectName`, `app:component`, `app:teamName`, `app:env`.
*   **FinOps Policy Document (`docs/finops-policy.md`):** Write a concise policy explaining how these tags allow teams to break down observability and evaluation compute costs on an executive billing dashboard.

### Deliverables Needed
1. Complete repository directory structure map.
2. Fully written application code for the Python mining worker and Go webhook alerting service.
3. Clean, production-grade Terraform files ensuring default tags are correctly declared and propagated down to individual cloud metrics engines.
4. GitHub Actions workflow files for testing, planning, and deployment execution.
5. A lean, direct `README.md` detailing local evaluation runs, deployment guides, and a specific highlight on the FinOps strategy applied to the pipeline's external API consumption.
