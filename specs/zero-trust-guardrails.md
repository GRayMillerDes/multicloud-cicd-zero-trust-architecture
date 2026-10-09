# Enterprise Zero-Trust & Zero-ClickOps Guardrails

## 1. Zero-Secret State Mandate
- **Rule 1.1**: Prohibit passing raw plaintext secrets through Terraform `values = [yamlencode(...)]` or `variables.tf`.
- **Rule 1.2**: Mandate all runtime secrets be injected via External Secrets Operator (ESO) backed by cloud KMS/Vault providers.
- **Rule 1.3**: Automated pre-commit hooks and CI linters (`tfsec`, `trufflehog`) must fail any commit containing potential credentials.

## 2. Zero-ClickOps Mandate
- **Rule 2.1**: Revoke public cloud console write/mutate permissions for human accounts across production VPCs and Kubernetes clusters.
- **Rule 2.2**: Infrastructure state mutation must originate from approved Pull Requests executed by Terraform Cloud or Argo CD runners.
- **Rule 2.3**: Automated drift detection jobs run every 4 hours. Unapproved drift triggers automated reconciliation or alerts SRE teams.

## 3. Workload Hardening Standards (CIS Benchmark)
- **Rule 3.1**: Disallow root user execution (`runAsNonRoot: true`, `runAsUser: 1000`).
- **Rule 3.2**: Enforce read-only root filesystems (`readOnlyRootFilesystem: true`) with temporary mounts restricted to `emptyDir`.
- **Rule 3.3**: Drop all Linux kernel capabilities (`capabilities.drop: ["ALL"]`) and set `allowPrivilegeEscalation: false`.
- **Rule 3.4**: Default Seccomp profile configured to `RuntimeDefault`.
