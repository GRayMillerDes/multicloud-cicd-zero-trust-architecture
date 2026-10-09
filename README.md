# Enterprise Multi-Cloud CI/CD Zero-Trust Architecture Blueprint

[![Architecture](https://img.shields.io/badge/Architecture-Multi--Cloud-blue?logo=diagramsdotnet&logoColor=white)](https://github.com/GRayMillerDes/multicloud-cicd-zero-trust-architecture)
[![Zero-Trust](https://img.shields.io/badge/Security-Zero--Trust%20%7C%20ESO-green?logo=security)](https://external-secrets.io/)
[![Multi-Cloud](https://img.shields.io/badge/Clouds-AWS%20%7C%20Tencent%20%7C%20Bare--Metal-orange?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![CI/CD](https://img.shields.io/badge/Platform-CloudBees%20%7C%20Jenkins-red?logo=jenkins&logoColor=white)](https://www.cloudbees.com/)
[![SRE](https://img.shields.io/badge/SRE-Non--Interactive%20Triage-purple?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

> **Hands-on Runnable Lab**: To run the local multi-node Kind & GitOps sandbox reproducing this architecture, visit [hybrid-gitops-sre-lab](https://github.com/GRayMillerDes/hybrid-gitops-sre-lab).

This repository contains the architectural blueprints, technical retrospectives, security guardrails, and topology specifications for an enterprise multi-cloud CI/CD platform engineered under strict financial least-privilege policies.

---

## 🗺️ Architectural Topology

![Multi-Cloud Architecture Topology](./architecture/multicloud-cicd-topology.svg)

```mermaid
graph TB
    %% Styling and Class Definitions
    classDef gitops fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef secrets fill:#022c22,stroke:#10b981,stroke-width:2px,color:#d1fae5;
    classDef core fill:#31104b,stroke:#a855f7,stroke-width:2px,color:#faf5ff;
    classDef aws fill:#451a03,stroke:#f59e0b,stroke-width:2px,color:#fef3c7;
    classDef tke fill:#064e3b,stroke:#10b981,stroke-width:2px,color:#ecfdf5;
    classDef idc fill:#1e293b,stroke:#64748b,stroke-width:2px,color:#f1f5f9;
    classDef obs fill:#4c0519,stroke:#f43f5e,stroke-width:2px,color:#ffe4e6;
    classDef pod fill:#1e1b4b,stroke:#6366f1,stroke-width:2px,color:#e0e7ff;

    subgraph LAYER_GITOPS["🛡️ Git & Governance Layer (Zero-ClickOps)"]
        GH["🐙 GitHub Enterprise / Monorepo"]:::gitops
        PR["🔍 PR Validation & Drift Check"]:::gitops
        TFC["🏗️ Terraform Cloud Workflows<br/>(Execution Engine)"]:::gitops
    end

    subgraph LAYER_SECRETS["🔐 Identity & Secret Fabric (Zero-Trust)"]
        SSM[("☁️ AWS SSM Parameter Store / KMS")]:::secrets
        TCC_KMS[("☁️ Tencent Cloud KMS / Secrets")]:::secrets
        VAULT[("🔒 Enterprise Vault / Central PKI")]:::secrets
    end

    subgraph LAYER_CONTROL["🏢 Enterprise CI/CD Control Plane"]
        OC["🐝 CloudBees Operations Center (OC)<br/>• Central RBAC & Licensing<br/>• Global CasC Bundles & Governance"]:::core
    end

    subgraph LAYER_DATA["🌐 Multi-Cloud Kubernetes Data Planes"]
        subgraph CLOUD_AWS["🟠 AWS Estate (EKS Cluster)"]
            ESO_AWS["🔐 External Secrets Operator"]:::secrets
            MC_AWS["🐝 Managed Controller (AWS Apps)"]:::aws
            POD_AWS["⚡ Ephemeral Build Pods (Agents)"]:::pod
        end

        subgraph CLOUD_TKE["🟢 Tencent Cloud Estate (TKE Cluster)"]
            ESO_TCC["🔐 External Secrets Operator"]:::secrets
            MC_TCC["🐝 Managed Controller (Core Platform)"]:::tke
            POD_TCC["⚡ Ephemeral Build Pods (Agents)"]:::pod
        end

        subgraph CLUSTER_IDC["⚪ On-Premises IDC (Bare-Metal K8s)"]
            ESO_IDC["🔐 External Secrets Operator"]:::secrets
            MC_IDC["🐝 Managed Controller (Legacy/Hybrid)"]:::idc
            POD_IDC["⚡ Ephemeral Build Pods (Agents)"]:::pod
        end
    end

    subgraph LAYER_OBS["📊 SRE & Non-Interactive Observability"]
        LOGS["🩺 TFC Diagnostic Sidecar / Hooks<br/>(Exit Code & Stderr Aggregator)"]:::obs
        SPLUNK["📈 Enterprise Splunk & Prometheus"]:::obs
    end

    %% Pipeline and Control Workflows
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

## 🎯 How to Use This Repository (Reading & Navigation Guide)

Depending on your role and objectives, here is the recommended path through this blueprint:

| Your Persona / Objective | Recommended Starting Point | Key Takeaway |
| :--- | :--- | :--- |
| **Recruiters / Engineering Leaders** | 💼 **[LinkedIn Pulse Retrospective](./articles/linkedin-pulse-article.md)** | High-level business context, executive summary of zero-trust wins, and SRE impact. |
| **Principal / Cloud-Native Architects** | 🗺️ **[Topology](./architecture/multicloud-cicd-topology.svg)** & 📖 **[Deep-Dive Architecture Guide](./architecture/architecture-deep-dive.md)** | Full technical breakdown of control plane separation, mTLS agent transport, and ESO secret sync. |
| **Security & Compliance Officers** | 🛡️ **[Least-Privilege RBAC Matrix](./specs/least-privilege-rbac-matrix.yaml)** & 📋 **[Zero-Trust Guardrails](./specs/zero-trust-guardrails.md)** | Auditable configurations enforcing zero-interactive-exec policies and CIS benchmark hardening. |
| **Hands-on Practitioners** | 🚀 **[hybrid-gitops-sre-lab](https://github.com/GRayMillerDes/hybrid-gitops-sre-lab)** | Jump into the runnable local sandbox to provision Kind, Argo CD, and test zero-secret workflows. |

---

## 📚 Core Repository Contents

- 📖 **[Deep-Dive Architecture Guide](./architecture/architecture-deep-dive.md)**: Technical breakdown of control plane vs data plane segregation, JNLP mTLS networking, and in-memory secret lifecycle.
- 💼 **[LinkedIn Pulse Ready Article](./articles/linkedin-pulse-article.md)**: English technical retrospective formatted specifically for LinkedIn Pulse and Featured sections.
- 🛡️ **[Least-Privilege RBAC Matrix](./specs/least-privilege-rbac-matrix.yaml)**: Declarative Kubernetes RBAC configurations enforcing zero-interactive-exec policies.
- 📋 **[Zero-Trust Guardrails](./specs/zero-trust-guardrails.md)**: Hardening benchmarks and automated drift detection specifications.

---

## 💡 Key Architectural Takeaways

1. **Eliminating the Terraform State Credential Leak**: Decoupled secret management via **External Secrets Operator (ESO)** ensures that zero credentials are committed to `terraform.tfstate`.
2. **The "No-kubectl" Dilemma**: Engineered automated diagnostic hooks within Terraform Cloud runners that inspect `.status.containerStatuses` and extract container exit codes and `--previous` stderr streams without granting shell access.
3. **Zero-ClickOps Compliance**: Automated golden machine image pipelines with **HashiCorp Packer** and exclusive declarative scheduling through **Terraform Cloud** revoke human write permissions to public cloud web consoles.

---

## 📄 License

Distributed under the Apache-2.0 License. See [LICENSE](./LICENSE) for details.
