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

## 1. What Is This Architecture? (30-Second Primer)

If you are new to multi-cloud platform engineering, here is the problem and solution in simple terms:

* **The Problem**: Global enterprises run workloads across multiple clouds (AWS in North America, Tencent Cloud in APAC, and on-premises Bare-Metal IDCs). Traditional setups either:
  1. Store shared passwords in code/Terraform state, leading to **severe security leaks**.
  2. Grant engineers direct console/SSH/`kubectl exec` permissions, violating **financial compliance (PCI-DSS/ISO27001)**.
  3. Route build traffic across cloud boundaries, causing **high cross-cloud egress bills and network latency**.
* **The Solution**: A **Hub-and-Spoke Zero-Trust Architecture**:
  * **Central Hub (Control Plane)**: Manages global access, licenses, and RBAC from one place.
  * **Regional Spokes (Data Planes)**: Execute builds locally inside private VPCs in AWS, TKE, and IDC without cross-cloud network hops.
  * **In-Memory Secrets**: External Secrets Operator (ESO) pulls cloud KMS keys directly into cluster RAM. Passwords never touch Git or Terraform state files.

```text
[ Developer PR ] ──> [ GitHub Enterprise ] ──> [ Terraform Cloud ]
                                                        │
                      ┌─────────────────────────────────┴─────────────────────────────────┐
                      ▼                                                                   ▼
       [ Central Control Plane ]                                         [ Regional Data Planes ]
       • Single Sign-On & Global RBAC                                    • AWS EKS / TKE / IDC Workers
       • Outbound mTLS JNLP Hub                                          • Local VPC Build Pods (Zero Egress)
                                                                         • Ephemeral In-Memory Secrets (ESO)
```

---

## 2. Quickstart: Consuming & Applying This Blueprint

This repository is organized as an actionable engineering toolkit. You can evaluate and apply it progressively:

### Step 1: Test & Dry-Run the Security Policy (30 Seconds)
Validate the production zero-interactive-exec RBAC matrix against any Kubernetes cluster without making changes:
```bash
# Validates client-side syntax and least-privilege RBAC definitions
kubectl apply --dry-run=client -f ./specs/least-privilege-rbac-matrix.yaml
```

### Step 2: Spin Up the Local Reproduction Sandbox (3 Minutes)
Run the companion 3-node Kind cluster locally to test the architecture end-to-end with zero cloud cost:
```bash
git clone https://github.com/GRayMillerDes/hybrid-gitops-sre-lab.git
cd hybrid-gitops-sre-lab && ./scripts/setup-local-env.sh
```

### Step 3: Deep Dive into Specifications
- **[Deep-Dive Architecture Guide](./architecture/architecture-deep-dive.md)**: Full breakdown of control plane separation, mTLS tunnels, and in-memory secret lifecycle.
- **[Zero-Trust Guardrails](./specs/zero-trust-guardrails.md)**: CIS Kubernetes benchmark hardening rules and container isolation policies.
- **[Terraform Cloud VCS Spec](./specs/terraform-cloud-vcs-spec.md)**: Enterprise PR iteration, speculative plan checks, remote runners & zero-clickops.

---

## 3. High-Level Architectural Topology

The diagram below maps the separation between the central management plane and multi-cloud regional data planes:

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

## 4. Deep-Dive: Core Architectural Mechanisms

Detailed technical implementations are documented in **[architecture-deep-dive.md](./architecture/architecture-deep-dive.md)**. Below is an engineering overview of the 4 foundational pillars:

### Pillar I: Decoupled In-Memory Secret Delivery
* **Problem**: Passing secrets in Terraform Helm values embeds plaintext credentials into `.tfstate`, exposing passwords to anyone with state bucket access.
* **Solution**: External Secrets Operator (ESO) bridges cloud KMS systems directly into Kubernetes etcd in-memory. Sensitive data is injected into pod runtime memory without touching Git or Terraform state.

### Pillar II: Outbound-Only mTLS JNLP Networking
* **Problem**: Direct cross-cloud management usually requires opening risky public ingress ports or maintaining expensive full-mesh site-to-site VPNs.
* **Solution**: Managed Controllers in remote clouds initiate outbound-only TLS/mTLS tunnels back to the Operations Center. Private VPCs remain fully shielded from incoming public traffic.

### Pillar III: Traffic Localization & Zero Egress
* **Problem**: Running builds from a centralized cluster across public clouds introduces network latency and massive egress transfer bills.
* **Solution**: Build agents are spawned ephemerally inside the local Kubernetes cluster where source code, dependencies, and caches live. Cross-cloud network traffic is strictly limited to lightweight control plane signals.

### Pillar IV: Non-Interactive Troubleshooting (Zero-kubectl-exec)
* **Problem**: Regulatory audits (SOC 2, PCI-DSS) strictly prohibit SSH or `kubectl exec` shell access in production, leaving engineers without diagnostic tools during pod crashes.
* **Solution**: Automated CI/CD diagnostic workers query `.status.containerStatuses` to extract container termination exit codes (e.g. 137 for OOMKilled) and fetch previous stderr logs (`--previous`) automatically into pipeline outputs.

👉 *[Read Complete Architectural Specification →](./architecture/architecture-deep-dive.md)*

---

## 5. Release Governance: VCS Pipeline & Manual Helm Promotion

How infrastructure changes and controller upgrades move safely from code to production:

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

## 6. Security & Compliance Specifications Matrix

Production guardrails from **[least-privilege-rbac-matrix.yaml](./specs/least-privilege-rbac-matrix.yaml)** and **[zero-trust-guardrails.md](./specs/zero-trust-guardrails.md)**:

| Compliance Domain | Enforced Specification | Implementation Artifact | Threat Prevented |
| :--- | :--- | :--- | :--- |
| **Interactive Access** | Zero `kubectl exec` / `attach` | [`least-privilege-rbac-matrix.yaml`](./specs/least-privilege-rbac-matrix.yaml) | Eliminates shell escalation & unauthorized data tampering (PCI-DSS) |
| **Container Sandbox** | `readOnlyRootFilesystem: true`, `drop: ALL` | [`zero-trust-guardrails.md`](./specs/zero-trust-guardrails.md) | Blocks malicious binary downloads & Linux kernel privilege escalation |
| **Workload Identity** | Non-root UID `1000`, `RuntimeDefault` seccomp | [`zero-trust-guardrails.md`](./specs/zero-trust-guardrails.md) | Prevents container breakout to host node |
| **State Sanitization** | External Secrets Operator (In-Memory K8s Secrets) | [`architecture-deep-dive.md`](./architecture/architecture-deep-dive.md) | Prevents credential leaks in `terraform.tfstate` |
| **Version Drift** | No `:latest` tags / Explicit semver in Git | [`terraform-cloud-vcs-spec.md`](./specs/terraform-cloud-vcs-spec.md) | Guarantees reproducible builds & instant deterministic rollbacks |

---

## 7. Architectural Decision Records (ADR) Summary

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
