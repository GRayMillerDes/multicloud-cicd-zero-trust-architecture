# Multi-Cloud Enterprise CI/CD Zero-Trust Architecture Deep Dive

This technical reference provides the engineering specifications for building a unified, multi-cloud platform engineering fabric spanning **AWS (EKS)**, **Tencent Cloud (TKE)**, and **On-Premises Bare-Metal Kubernetes**, under strict financial and gaming compliance guardrails.

---

## 1. High-Level Architectural Topology

![Multi-Cloud Architecture Topology](./multicloud-cicd-topology.svg)

### Architectural Invariants
1. **Separation of Control & Execution**: The central control plane (CloudBees Operations Center) handles RBAC, licensing, and global configuration bundles, while dynamic build workloads are executed locally within tenant-isolated data planes.
2. **Zero-ClickOps Infrastructure**: Infrastructure mutation via cloud consoles is prohibited. All state transitions flow through Git Pull Requests and Terraform Cloud runners.
3. **Zero Plaintext Credentials in State**: Static API keys, database credentials, and cluster tokens are forbidden inside Terraform state files (`.tfstate`). Credentials are dynamically provisioned in-memory by External Secrets Operator.

---

## 2. Deep Dive: The 3 Core Architectural Pillars

### Pillar I: Decoupled Secret Delivery via External Secrets Operator (ESO)
In classic Terraform Helm deployments, secrets are frequently passed using values injections:
```hcl
# ANTI-PATTERN: Commits plaintext credentials directly to terraform.tfstate
values = [
  yamlencode({
    database = {
      password = data.aws_ssm_parameter.db_secret.value
    }
  })
]
```
Even if encrypted in S3, anyone with state access (or CI build logs) can read these credentials. 

**The Production Solution:**
- Terraform provisions the Kubernetes clusters and installs the ESO operator along with cloud IAM identity bindings (AWS EKS Pod Identity / IRSA, Tencent Cloud CAM role).
- ESO defines `ClusterSecretStore` and `ExternalSecret` custom resources:
  - AWS clusters pull from **AWS SSM Parameter Store / KMS**.
  - Tencent Cloud clusters pull from **Tencent Cloud KMS**.
  - Bare-metal IDC clusters authenticate to **HashiCorp Vault** using short-lived AppRole tokens.
- ESO synthesizes standard Kubernetes `Secret` resources entirely in-memory inside the cluster.

---

### Pillar II: Non-Interactive Troubleshooting Under Least-Privilege Guardrails
In regulated production environments, granting engineers `kubectl exec` or interactive shell capabilities violates PCI-DSS, SOC2, and ISO27001 compliance standards. When a controller crashes with dynamic sidecars:
- Helm simply times out: `timed out waiting for the condition`.
- Engineers cannot run `kubectl exec -it <pod> -- sh`.

**The Production Solution:**
- Automated post-apply diagnostic hooks are integrated into Terraform Cloud and CI runners.
- On deployment failure, the diagnostic worker calls Kubernetes API endpoints to read `.status.containerStatuses`.
- It identifies the exact `lastState.terminated.exitCode` (e.g., 137 for OOMKilled, 1 for JVM configuration panic) and queries previous execution logs (`kubectl logs --previous -c <sidecar>`).
- Diagnostics are dumped directly into the pipeline run output, achieving rapid root-cause analysis (RCA) without human access escalation.

---

### Pillar III: Immutable Infrastructure with HashiCorp Packer & Terraform Cloud
To eradicate configuration drift on worker host instances:
- Golden VM machine images (AMIs for AWS, CVM images for Tencent Cloud) are built continuously using **HashiCorp Packer**.
- CIS OS benchmarks, vulnerability scanner agents, and container runtimes are baked in immutably.
- Image IDs are declared in Terraform Cloud workspaces. Node rotations are executed via RollingUpdate without in-place SSH patching.
