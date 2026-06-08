# 🚀 Crossplane Platform Toolkit

> **A modular, scalable, and production-ready toolkit for managing AWS cloud infrastructure — EKS, RDS, S3, and beyond — using Crossplane and native Kubernetes APIs.**

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27%2B-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Crossplane](https://img.shields.io/badge/Crossplane-1.14%2B-00AEEF?logo=crossplane)](https://crossplane.io/)
[![AWS Provider](https://img.shields.io/badge/provider--aws-upbound-FF9900?logo=amazonaws)](https://marketplace.upbound.io/providers/upbound/provider-aws)
[![API Group](https://img.shields.io/badge/API%20Group-devops.murali.io-success)](.)

---

## 📖 About

The **Crossplane Platform Toolkit** delivers a standardized, self-service approach to provisioning and managing AWS infrastructure using Infrastructure-as-Code (IaC) principles — entirely within Kubernetes.

Rather than relying on traditional CLI tools, Terraform, or scattered CloudFormation templates, this toolkit harnesses the power of **Crossplane Compositions** to transform cloud infrastructure into first-class Kubernetes objects. Platform teams define the guardrails; application teams provision what they need — with just a YAML claim.

**Key Benefits:**

- 🏗️ **Platform Engineering First** — Abstracts AWS complexity behind clean, opinionated Kubernetes APIs
- 🔁 **GitOps Ready** — Every resource is declarative, versionable, and auditable
- 🔒 **Policy by Design** — Enforce organizational standards (Free Tier limits, tagging, naming conventions) at the composition layer
- ⚡ **Self-Service** — Developers claim infrastructure without needing AWS console access
- 🧩 **Modular** — Each XRD is independently composable and reusable across environments

---

## 🏛️ Architecture

This toolkit is built on the **Crossplane Control Plane** pattern.

```
Developer submits Claim (YAML)
        │
        ▼
┌─────────────────────────┐
│  CompositeResource (XR) │  ← Created automatically by Crossplane
│  devops.murali.io/v1    │
└────────────┬────────────┘
             │  resolved by
             ▼
┌─────────────────────────┐
│     Composition         │  ← go-templating pipeline (function-go-templating)
│  (go-template engine)   │
└────────────┬────────────┘
             │  renders
             ▼
┌────────────────────────────────────────────┐
│         Managed Resources (AWS)            │
│  RDS Instance │ EKS Cluster │ VPC │ S3 ... │
└────────────────────────────────────────────┘
```

**Custom API Group:** `devops.murali.io`

| XRD | Claim Kind | Description |
|---|---|---|
| `xrdsinstances` | `RDSInstance` | RDS PostgreSQL / MySQL (Free Tier ready) |
| `xekscluster` | `EKSCluster` | Managed EKS with node groups |
| `xnetworks` | `Network` | VPC, Subnets, NAT Gateway |
| `xsecuritygroups` | `SecurityGroup` | Bastion, EKS, DB security groups |

---

## ✅ Prerequisites

Before deploying this toolkit, ensure you have the following in place:

| Requirement | Version | Notes |
|---|---|---|
| Kubernetes Cluster | 1.27+ | EKS, Kind, or any CNCF-conformant cluster |
| [Crossplane](https://crossplane.io/) | 1.14+ | Installed via Helm |
| `provider-aws` (Upbound) | Latest | Configured with valid IAM credentials |
| `function-go-templating` | v0.4.0+ | Required for all compositions |
| `function-auto-ready` | Latest | Pipeline step for readiness tracking |
| `kubectl` | 1.27+ | Configured against your target cluster |

### IAM Permissions

The AWS provider requires an IAM role or user with permissions to manage the target services (EC2, RDS, EKS, S3, IAM). It is recommended to use **IRSA (IAM Roles for Service Accounts)** in production.

---

## 🚀 Getting Started

### 1. Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

### 2. Install Required Functions

```bash
# function-go-templating
kubectl apply -f https://raw.githubusercontent.com/crossplane-contrib/function-go-templating/main/package/crds/

# Install via Crossplane package
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.4.0
---
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-auto-ready:v0.2.1
EOF
```

### 3. Configure AWS Provider

```bash
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:latest
EOF
```

### 4. Apply XRDs and Compositions

```bash
# Apply all definitions first
kubectl apply -f definitions/

# Then apply compositions
kubectl apply -f compositions/
```

### 5. Submit a Claim

```bash
# Create a Free Tier PostgreSQL instance
kubectl apply -f examples/rds-postgres.yaml

# Create a Free Tier MySQL instance
kubectl apply -f examples/rds-mysql.yaml

# Create an EKS cluster
kubectl apply -f examples/eks-cluster.yaml
```

### 6. Monitor Provisioning

```bash
# Check claim status
kubectl get rdsinstance -n <your-namespace>

# Check composite resource
kubectl get xrdsinstance

# Describe for detailed events
kubectl describe rdsinstance <name> -n <your-namespace>

# Watch all managed resources
kubectl get managed
```

---

## 📁 Project Structure

```
.
├── definitions/                        # CompositeResourceDefinitions (XRDs)
│   ├── xrd-rds-instance.yaml           # RDS PostgreSQL / MySQL XRD
│   ├── xrd-eks-cluster.yaml            # EKS Cluster XRD
│   ├── xrd-network.yaml                # VPC / Subnet / NAT XRD
│   └── xrd-security-group.yaml         # Security Group XRD
│
├── compositions/                       # Crossplane Compositions (go-templating)
│   ├── composition-rds-instance.yaml   # RDS composition (engine branching)
│   ├── composition-eks-cluster.yaml    # EKS composition (node group loop)
│   ├── composition-network.yaml        # Network composition (subnet array)
│   └── composition-security-group.yaml # SG composition (rule loop)
│
├── examples/                           # Claim examples (ready to apply)
│   ├── rds-postgres.yaml               # Free Tier PostgreSQL claim
│   ├── rds-mysql.yaml                  # Free Tier MySQL claim
│   ├── eks-cluster.yaml                # EKS cluster claim
│   └── network.yaml                    # VPC + subnet claim
│
├── docs/                               # Design docs & architecture notes
│   ├── architecture.md
│   ├── rds-design.md
│   └── eks-design.md
│
└── README.md
```

---

## 🔧 Design Principles

| Principle | Implementation |
|---|---|
| **Reference, don't recreate** | Existing VPCs, subnets, and security groups are passed by ID — the toolkit never creates duplicate network resources |
| **Engine branching in one Composition** | A single `engine: postgresql` or `engine: mysql` parameter drives the entire composition branch |
| **Free Tier by default** | `db.t3.micro`, `gp2`, `20 GiB`, `single-az` are defaults — upgrade paths are explicitly opt-in |
| **No hardcoded values** | Region, AZ, instance class, storage, and tags are all parameterized |
| **Consistent API group** | All XRDs live under `devops.murali.io` for clear ownership and discovery |

---

## 🤝 Contributing

Contributions are welcome! Here's how to get started:

1. **Fork** this repository
2. **Create a branch** for your feature or fix:
   ```bash
   git checkout -b feature/add-s3-composition
   ```
3. **Make your changes** — add XRDs, compositions, or example claims
4. **Test locally** using `crossplane beta render` or a Kind cluster
5. **Commit and push**, then open a **Pull Request** with a clear description

Please ensure new compositions follow the existing patterns:
- Use `function-go-templating` with inline templates
- Include `function-auto-ready` as the final pipeline step
- Add a corresponding example claim under `examples/`
- Document any new XRD parameters in `docs/`

---

<div align="center">

Built with ❤️ using [Crossplane](https://crossplane.io/) · [Kubernetes](https://kubernetes.io/) · [AWS](https://aws.amazon.com/)

</div>
