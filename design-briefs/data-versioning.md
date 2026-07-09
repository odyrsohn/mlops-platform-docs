Act as a Principal Cloud & MLOps Infrastructure Architect. I need you to generate a lean, production-ready repository for a Data Versioning Pipeline and a GDPR-compliant "Right-to-Erasure" Wipeout Engine.

### Core Architecture & Tech Stack
*   **Versioning Sync (Python):** A utility that automates and handles data-versioning metadata generation (mimicking programmatic DVC or LakeFS commands) over cloud datasets to track model evaluation set lineages and golden dataset provenance.
*   **Deletion Orchestrator (Go):** A high-reliability service exposing an authenticated API endpoint. Upon receiving a certified "Forget Me" payload containing a specific `tenant_id` or user context, it systematically executes a multi-tier purge of data.
*   **Infrastructure (Terraform):** Provision object storage buckets/containers, explicitly configuring lifecycle management policies, object retention transitions (Hot to Cold/Archive tiers), and automated soft-delete/permanent-wipe configurations.
*   **CI/CD (GitHub Actions):** Build a unified pipeline containing:
    1. Pre-checks: Go and Python code format verifications, unit tests, and `terraform validate`.
    2. IaC Phase: Automatic generation of `terraform plan` for PR review and execution of `terraform apply` on main merge.
    3. App Deploy: Automated build and deployment tasks for the orchestration images.

### Required Point of Complexity
*   **Go Deletion Orchestrator:** Implement a distributed transactional state machine (a basic Saga pattern approach) that guarantees atomicity during deletion across multi-tier storage setups. It must ensure that if a deletion fails halfway through (e.g., transient cloud storage timeout), it can resume safely from a durable checkpoint without leaving orphaned residual metadata.

### FinOps & Governance Mandate
*   **IaC Default Tags:** You must enforce a strict global tagging strategy within the provider block using `default_tags`. Every resource must inherit: `app:name`, `app:projectName`, `app:component`, `app:teamName`, `app:env`.
*   **FinOps Policy Document (`docs/finops-policy.md`):** Write a concise governance document defining how tag-based lifecycle management policies drastically lower storage overhead by automatically cleaning and isolating dormant tenant footprints.

### Deliverables Needed
1. Complete repository directory structure map.
2. Modular Go source files for the transactional orchestrator and Python code for data lineage validation.
3. Terraform files mapping out object storage lifecycle rules, access controls, and explicit global `default_tags`.
4. Production-ready GitHub Actions YAML workflows.
5. A crisp, direct `README.md` highlighting local development prerequisites, execution triggers, and a short, impactful section on storage optimization for FinOps.
