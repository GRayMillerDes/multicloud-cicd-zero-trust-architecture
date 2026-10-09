# Enterprise Multi-Cloud CI/CD Zero-Trust Architecture Blueprint

[![Architecture](https://img.shields.io/badge/Architecture-Multi--Cloud-blue?logo=diagramsdotnet&logoColor=white)](https://github.com/GRayMillerDes/multicloud-cicd-zero-trust-architecture)
[![Zero-Trust](https://img.shields.io/badge/Security-Zero--Trust%20%7C%20ESO-green?logo=security)](https://external-secrets.io/)
[![Multi-Cloud](https://img.shields.io/badge/Clouds-AWS%20%7C%20Tencent%20%7C%20Bare--Metal-orange?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![CI/CD](https://img.shields.io/badge/Platform-CloudBees%20%7C%20Jenkins-red?logo=jenkins&logoColor=white)](https://www.cloudbees.com/)
[![SRE](https://img.shields.io/badge/SRE-Non--Interactive%20Triage-purple?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

> **Hands-on Runnable Sandbox**: To deploy and test the local multi-node Kind & GitOps environment that mirrors this architecture, visit [hybrid-gitops-sre-lab](https://github.com/GRayMillerDes/hybrid-gitops-sre-lab).

This repository provides production-grade architectural blueprints, security guardrails, and VCS delivery specifications for an enterprise multi-cloud CI/CD platform engineered under strict financial least-privilege standards.

---

## Quickstart: Consuming & Applying This Blueprint

This repository is designed as an Enterprise Architecture Toolkit. To validate, inspect, and apply these architectural assets:

### 1. Test & Dry-Run the Security RBAC Policy
Validate the production zero-interactive-exec RBAC matrix against your existing Kubernetes cluster:
```bash
# Perform dry-run client side validation of the least-privilege RBAC definitions
kubectl apply --dry-run=client -f ./specs/least-privilege-rbac-matrix.yaml
```

### 2. Inspect Specifications & Hardening Rules
- **[Deep-Dive Architecture Guide](./architecture/architecture-deep-dive.md)**: Executive problem statement, control plane vs data plane segregation, mTLS JNLP tunnels, in-memory secret lifecycle.
- **[Zero-Trust Guardrails](./specs/zero-trust-guardrails.md)**: CIS Kubernetes benchmark hardening rules and automated drift detection specifications.
- **[Terraform Cloud VCS Spec](./specs/terraform-cloud-vcs-spec.md)**: GitHub Enterprise PR iteration, speculative plan checks, remote runners & zero-clickops.

### 3. Deploy the Companion Runnable Sandbox
To spin up a live 3-node Kind cluster reproducing this architecture (Argo CD, ESO, Prometheus, Grafana) locally on your workstation:
```bash
git clone https://github.com/GRayMillerDes/hybrid-gitops-sre-lab.git
cd hybrid-gitops-sre-lab && ./scripts/setup-local-env.sh
```

---

## Architecture Deep-Dive Highlights

Detailed technical breakdowns are documented in **[architecture-deep-dive.md](./architecture/architecture-deep-dive.md)**. Below are the key system mechanisms:

1. **Control Plane vs. Data Plane Segregation**:
   The central management plane (CloudBees Operations Center) handles RBAC, licensing, and global configuration bundles. Workload execution is delegated to isolated regional data planes (AWS EKS, Tencent Cloud TKE, Bare-Metal IDC).
2. **Outbound-Only mTLS JNLP Networking**:
   Controllers establish outbound-only secure tunnels back to the control plane, eliminating inbound firewall rules into private VPCs.
3. **Zero-Secret In-Memory Lifecycle**:
   Credentials never touch Git or `terraform.tfstate`. External Secrets Operator (ESO) reconciles secrets directly from AWS SSM / Tencent Cloud KMS / Vault into ephemeral cluster memory.
4. **Traffic Localization & Zero Egress**:
   Dynamic build agents are spawned strictly within local VPCs where source code and caches reside, preventing cross-cloud latency and egress fees.

👉 *[Read Complete Architectural Specification →](./architecture/architecture-deep-dive.md)*

---

## Architectural Topology

![Multi-Cloud Architecture Topology](./architecture/multicloud-cicd-topology.svg)

```mermaid
graph TB
    classDef gitops fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef secrets fill:#022c22,stroke:#10b981,stroke-width:2px,color:#d1fae5;
    classDef core fill:#31104b,stroke:#a855f7,stroke-width:2px,color:#faf5ff;
    classDef aws fill:#451a03,stroke:#f59e0b,stroke-width:2px,color:#fef3c7;
    classDef tke fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#ecfdf5;
    classDef idc fill:#1e293b,stroke:#64748b,stroke-width:2px,color:#f1f5f9;
    classDef obs fill:#4c0519,stroke:#f43f5e,stroke-width:2px,color:#ffe4e6;
    classDef pod fill:#1e1b4b,stroke:#6366f1,stroke-width:2px,color:#e0e7ff;

    subgraph LAYER_GITOPS["Git & Governance Layer (Zero-ClickOps)"]
        GH["GitHub Enterprise / Monorepo"]:::gitops
        PR["PR Validation & Drift Check"]:::gitops
        TFC["Terraform Cloud Workflows<br/>(Execution Engine)"]:::gitops
    end

    subgraph LAYER_SECRETS["Identity & Secret Fabric (Zero-Trust)"]
        SSM[("AWS SSM Parameter Store / KMS")]:::secrets
        TCC_KMS[("Tencent Cloud KMS / Secrets")]:::secrets
        VAULT[("Enterprise Vault / Central PKI")]:::secrets
    end

    subgraph LAYER_CONTROL["Enterprise CI/CD Control Plane"]
        OC["CloudBees Operations Center (OC)<br/>• Central RBAC & Licensing<br/>• Global CasC Bundles & Governance"]:::core
    end

    subgraph LAYER_DATA["Multi-Cloud Kubernetes Data Planes"]
        subgraph CLOUD_AWS["AWS Estate (EKS Cluster)"]
            ESO_AWS["External Secrets Operator"]:::secrets
            MC_AWS["Managed Controller (AWS Apps)"]:::aws
            POD_AWS["Ephemeral Build Pods (Agents)"]:::pod
        end

        subgraph CLOUD_TKE["Tencent Cloud Estate (TKE Cluster)"]
            ESO_TCC["External Secrets Operator"]:::secrets
            MC_TCC["Managed Controller (Core Platform)"]:::tke
            POD_TCC["Ephemeral Build Pods (Agents)"]:::pod
        end

        subgraph CLUSTER_IDC["On-Premises IDC (Bare-Metal K8s)"]
            ESO_IDC["External Secrets Operator"]:::secrets
            MC_IDC["Managed Controller (Legacy/Hybrid)"]:::idc
            POD_IDC["Ephemeral Build Pods (Agents)"]:::pod
        end
    end

    subgraph LAYER_OBS["SRE & Non-Interactive Observability"]
        LOGS["TFC Diagnostic Sidecar / Hooks<br/>(Exit Code & Stderr Aggregator)"]:::obs
        SPLUNK["Enterprise Splunk & Prometheus"]:::obs
    end

    GH ==>|1. Pull Request Trigger| PR
    PR ==>|2. Automated Policy Approval| TFC
    TFC -->|3. Declarative Helm Deploy| OC
    TFC -->|3. Declarative Helm Deploy| MC_AWS
    TFC -->|3. Declarative Helm Deploy| MC_TCC
    TFC -->|3. Declarative Helm Deploy| MC_IDC

    SSM -.->|4. KMS Sync| ESO_AWS
    TCC_KMS -.->|4. KMS Sync| ESO_TCC
    VAULT -.->|4. Dynamic TLS / PKI| ESO_IDC

    ESO_AWS ==>|5. Ephemeral In-Memory Injection| MC_AWS
    ESO_TCC ==>|5. Ephemeral In-Memory Injection| MC_TCC
    ESO_IDC ==>|5. Ephemeral In-Memory Injection| MC_IDC

    OC ===|6. Secure mTLS / JNLP Protocol| MC_AWS
    OC ===|6. Secure mTLS / JNLP Protocol| MC_TCC
    OC ===|6. Secure mTLS / JNLP Protocol| MC_IDC

    MC_AWS -->|7. Dynamic Pod Spawn| POD_AWS
    MC_TCC -->|7. Dynamic Pod Spawn| POD_TCC
    MC_IDC -->|7. Dynamic Pod Spawn| POD_IDC

    POD_AWS -.->|8. Stderr Extraction| LOGS
    POD_TCC -.->|8. Stderr Extraction| LOGS
    POD_IDC -.->|8. Stderr Extraction| LOGS
    LOGS ==>|9. SLO & Alert Stream| SPLUNK
```

---

## VCS Pipeline & Release Governance Specification

Detailed delivery flows are codified in **[terraform-cloud-vcs-spec.md](./specs/terraform-cloud-vcs-spec.md)**. All infrastructure and controller version upgrades follow dual-control GitOps iteration:

```mermaid
sequenceDiagram
    autonumber
    actor Admin as SRE / Release Manager
    participant CI as Image Bake Pipeline (.github/workflows)
    participant REG as Enterprise OCI Registry (ghcr.io)
    participant HELM as GitOps Helm Values (values.yaml)
    participant GHE as GitHub Enterprise PR Gate
    participant TFC as Terraform Cloud & Argo CD

    Admin->>CI: Trigger Build with Upstream Base + Patch (e.g. 2.440.3.1-p1)
    CI->>CI: CIS Hardening + Trivy CVE Security Scan
    CI->>REG: Push Immutable Golden Image (OC / MC)
    Note over Admin,HELM: Manual Promotion Gate (Separation of Duties)
    Admin->>HELM: Manually bump image.tag to 2.440.3.1-p1
    Admin->>GHE: Submit PR for Peer Review
    GHE->>TFC: Speculative Plan Check
    Admin->>GHE: Merge PR into main
    TFC->>TFC: Declarative Sync: Argo CD rolls out new Immutable Controller Pods
```

👉 *[Read Complete VCS & Promotion Specification →](./specs/terraform-cloud-vcs-spec.md)*

---

## Security & Compliance Specifications Matrix

Production policies from **[least-privilege-rbac-matrix.yaml](./specs/least-privilege-rbac-matrix.yaml)** and **[zero-trust-guardrails.md](./specs/zero-trust-guardrails.md)** are summarized below:

| Compliance Domain | Enforced Specification | Implementation Artifact | Threat Prevented |
| :--- | :--- | :--- | :--- |
| **Interactive Access** | Zero `kubectl exec` / `attach` | [`least-privilege-rbac-matrix.yaml`](./specs/least-privilege-rbac-matrix.yaml) | Eliminates shell escalation & unauthorized data tampering (PCI-DSS) |
| **Container Sandbox** | `readOnlyRootFilesystem: true`, `drop: ALL` | [`zero-trust-guardrails.md`](./specs/zero-trust-guardrails.md) | Blocks malicious binary downloads & Linux kernel privilege escalation |
| **Workload Identity** | Non-root UID `1000`, `RuntimeDefault` seccomp | [`zero-trust-guardrails.md`](./specs/zero-trust-guardrails.md) | Prevents container breakout to host node |
| **State Sanitization** | External Secrets Operator (In-Memory K8s Secrets) | [`architecture-deep-dive.md`](./architecture/architecture-deep-dive.md) | Prevents credential leaks in `terraform.tfstate` |
| **Version Drift** | No `:latest` tags / Explicit semver in Git | [`terraform-cloud-vcs-spec.md`](./specs/terraform-cloud-vcs-spec.md) | Guarantees reproducible builds & instant deterministic rollbacks |

---

## Architectural Decision Records (ADR) Summary

| Decision ID | Context & Problem | Decision Made | Trade-offs & Consequences |
| :--- | :--- | :--- | :--- |
| **ADR-001** | Terraform state files store credentials in plaintext when using `helm_release` values. | Adopt External Secrets Operator (ESO) with ephemeral in-memory K8s secrets. | Slightly higher initial CRD reconciliation complexity; completely eliminates credential leakage in CI/CD state. |
| **ADR-002** | Financial compliance forbids `kubectl exec` / SSH into production worker nodes. | Implement non-interactive diagnostic sidecars and stderr extractors in CI runners. | Eliminates manual ad-hoc troubleshooting; enforces deterministic RCA and structured logging. |
| **ADR-003** | Multi-cloud latency and network isolation across AWS, Tencent Cloud, and IDC. | Centralize Control Plane (CloudBees OC) and distribute Data Plane controllers per cloud. | Requires outbound-only mTLS JNLP connectivity; local build pods avoid cross-cloud egress costs. |

---

## Document & Asset Index

| Focus Area | Reference Document | Engineering Scope |
| :--- | :--- | :--- |
| **System Architecture** | **[Deep-Dive Architecture Guide](./architecture/architecture-deep-dive.md)** | Control Plane vs Data Plane segregation, mTLS JNLP tunnels, in-memory secret lifecycle |
| **IaC Delivery & VCS** | **[Terraform Cloud VCS Spec](./specs/terraform-cloud-vcs-spec.md)** | GitHub Enterprise PR iteration, speculative plan checks, remote runners & zero-clickops |
| **Security & Compliance** | **[Least-Privilege RBAC Matrix](./specs/least-privilege-rbac-matrix.yaml)** | Production-grade RBAC enforcing zero-interactive-exec policies across multi-tenant clusters |
| **Hardening Benchmarks** | **[Zero-Trust Guardrails](./specs/zero-trust-guardrails.md)** | CIS Kubernetes benchmark hardening rules and automated drift detection specifications |
| **Runnable Demo** | **[Local SRE Sandbox Repo](https://github.com/GRayMillerDes/hybrid-gitops-sre-lab)** | Local 3-node Kind cluster with ESO, Argo CD, and SRE Golden Signals telemetry |

---

## Repository Layout

```text
multicloud-cicd-zero-trust-architecture/
├── README.md                              # Enterprise Architecture Overview & Navigation
├── architecture/
│   ├── architecture-deep-dive.md          # Technical Deep-Dive: Segregation & In-Memory Secrets
│   ├── multicloud-cicd-topology.svg       # Vector Topology Architecture Diagram
│   ├── multicloud-cicd-topology.png       # High-Resolution Architectural Topology
│   └── multicloud-cicd-topology.mermaid   # Mermaid Source Graph
└── specs/
    ├── least-privilege-rbac-matrix.yaml   # Production Kubernetes RBAC (Zero-Exec Policy)
    ├── terraform-cloud-vcs-spec.md        # Terraform Cloud & GitHub Enterprise VCS Spec
    └── zero-trust-guardrails.md           # CIS Benchmarks & Drift Detection Specifications
```

---

## License

Distributed under the Apache-2.0 License. See [LICENSE](./LICENSE) for details.
