# ADR-009: Terraform for VPS infrastructure provisioning

**Status:** Accepted  
**Date:** 2026-05-09  
**Author:** Mark

## Context

The Job Tracker application runs on an Akamai (Linode) VPS in production. A decision is required on how the server is provisioned — whether to configure it manually via the Akamai console or to declare the infrastructure as code so it can be reproduced, version-controlled, and torn down and rebuilt reliably.

## Decision

The VPS infrastructure is provisioned using Terraform with the official Linode provider. Three resources are declared:

| Resource | Type | Purpose |
|---|---|---|
| `linode_sshkey.deployer` | SSH key | Registered with Akamai; used for root access to the VM |
| `linode_firewall.job_tracker` | Firewall | Default DROP inbound; allows SSH (22), HTTP (80), HTTPS (443) only |
| `linode_instance.job_tracker` | VM | Debian 12, g6-standard-2 (4 GB shared CPU), tagged for the project |

Configuration lives in `terraform/` within `job-tracker-app-infrastructure`:
- `main.tf` — provider, firewall, SSH key, and VM resources
- `variables.tf` — all configurable inputs (token, region, root password, SSH key)
- `outputs.tf` — public IP and SSH command for immediate use after `apply`
- `terraform.tfvars.example` — credential template (actual `terraform.tfvars` is gitignored)

## Rationale

- **Reproducibility:** The entire server can be destroyed and re-provisioned with `terraform apply` in under two minutes. No tribal knowledge about console clicks.
- **Version control:** Infrastructure changes go through the same git workflow as code — visible, reviewable, and revertable.
- **Portfolio signal:** IaC is a standard expectation at senior engineer and architect level. Having it in the repo demonstrates the practice, not just awareness of it.
- **Firewall as code:** The firewall rules are declared alongside the VM, so there is no risk of provisioning a server and forgetting to lock it down.

## Alternatives considered

- **Manual provisioning via Akamai console:** Fast for one-off setup but not reproducible and leaves no audit trail. Rejected.
- **Ansible:** Better suited to configuration management (what runs on the server) than provisioning (what the server is). Terraform handles provisioning; Docker Compose handles the application stack. Ansible would be redundant here.
- **Pulumi:** Code-native IaC with stronger typing. Valid choice but Terraform's Linode provider is more mature and Terraform is more widely recognised in the portfolio context.

## Consequences

- `terraform.tfvars` and `terraform.tfstate` are gitignored — credentials and state are never committed.
- `terraform.tfvars.example` documents all required inputs without exposing values.
- Application deployment (Docker Compose, `.env`, service configuration) is handled separately after provisioning — Terraform owns the VM, Docker Compose owns the application stack.
- TLS termination and reverse proxying are handled by Nginx Proxy Manager running in a separate Docker Compose stack on the same VM. See ADR-012 for the TLS termination decision.
- Future work: remote state backend (Terraform Cloud or S3) for team use; DNS record management via Linode DNS provider.
