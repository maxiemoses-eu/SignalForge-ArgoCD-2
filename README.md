# SignalForge GitOps & ArgoCD Control Plane (`SignalForge-ArgoCD-2`)

> **Enterprise-grade GitOps delivery engine for multi-environment microservice architectures across Development, Production, and Disaster Recovery (DR) clusters.**

---

## 🌍 The Three-Repository Architecture

In the SignalForge engineering ecosystem, we strictly separate concerns across **three dedicated repositories** to maintain loose coupling, robust security boundaries, and clean auditability:

1. **Repository 1: Application Source Code (`signalforge-code`)**
   - Contains the actual application code for our microservices (`api-gateway`, `order`, `payment-service`, `product-service`, `store-ui`, `users-service`).
   - Responsible for building container images, running unit/integration tests, and publishing artifacts to our container registry.

2. **Repository 2: Infrastructure as Code / IaC (`signalforge-iac`)**
   - Contains Terraform / OpenTofu / Cloud Provisioning scripts.
   - Responsible for provisioning cloud infrastructure: VPCs, EKS/GKE Kubernetes clusters, managed PostgreSQL databases, IAM roles, and networking primitives.

3. **Repository 3: GitOps & Delivery Control Plane (`Signalforge-ArgoCD-2` — **This Repository**) **
   - Contains **no application code and no infrastructure code**.
   - Houses our **ArgoCD control plane configuration** (`argocd/`), **reusable Helm blueprints** (`charts/`), and **environment-specific configuration values** (`helm-values/`).
   - Acts as the single source of truth that binds our IaC-provisioned clusters with our compiled code artifacts via automated GitOps reconciliation.

---

## 📂 Repository Structure & Layout

```text
SignalForge-ArgoCD-2/
├── .yamllint.yaml                    # Strict YAML linter rules for formatting
├── argocd/                           # ArgoCD Control Plane Manifests
│   ├── apps-matrix.yaml              # ApplicationSet generating microservice apps dynamically
│   ├── platform-matrix.yaml          # ApplicationSet generating platform tooling dynamically
│   └── projects.yaml                 # AppProjects enforcing strict RBAC security boundaries
├── charts/                           # Reusable Helm Chart Blueprints (Code)
│   ├── network-policies/             # Zero-trust network security blueprints
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/                # Default-deny, DNS egress, intra-namespace rules
│   └── service-chart/                # Universal microservice deployment blueprint
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/                # Deployment, Service, HPA, ServiceMonitor
├── helm-values/                      # Environment Configuration Values (Config)
│   ├── dev/
│   │   ├── apps/                     # Values for microservices in Development
│   │   └── platform/                 # Values for platform tools (Prometheus, Loki, Redis, Postgres, etc.)
│   ├── dr/
│   │   ├── apps/                     # Values for microservices in Disaster Recovery
│   │   └── platform/
│   └── prod/
│       ├── apps/                     # Values for microservices in Production
│       └── platform/
└── security/                         # Cluster-wide Security & Access Control
    └── cluster-roles.yaml            # ClusterRoles & ClusterRoleBindings
```

---

## 🚀 Core Architectural Concepts

### 1. Separation of Code and Configuration
We strictly follow the Twelve-Factor App methodology by separating **code** from **config**:
- **Code** (`charts/service-chart/`) is immutable. The deployment blueprint never changes per service.
- **Config** (`helm-values/{dev,prod,dr}/apps/*.yaml`) contains all environment-specific overrides (replicas, resource quotas, database connection strings, image tags).

### 2. Automated GitOps with ArgoCD ApplicationSets
Instead of maintaining dozens of static ArgoCD Application manifests, we use **ApplicationSets with Git Generators (`goTemplate: true`)**:
- ArgoCD scans `helm-values/{dev,prod,dr}/apps/*.yaml` and `helm-values/{dev,prod,dr}/platform/*.yaml`.
- Whenever a new YAML file is added or updated in Git, ArgoCD automatically reconciles the live cluster state.
- **Self-Healing & Pruning:** If a resource is deleted in Git, ArgoCD prunes it from the cluster. If someone makes an unauthorized manual change in the cluster (`kubectl edit`), ArgoCD automatically overwrites it back to match Git.

### 3. Security Guardrails (`AppProjects`)
We divide workloads into two secure sandboxes (`argocd/projects.yaml`):
- **`signalforge-apps`**: Restricted to deploying standard microservice workloads (`Deployment`, `Service`, `Hpa`, `Ingress`) into application namespaces (`signalforge-dev`, `signalforge-prod`, `signalforge-dr`).
- **`signalforge-platform`**: Granted elevated privileges required for cluster-wide infrastructure tools (`ClusterRole`, `ClusterRoleBinding`, `CustomResourceDefinition`) across monitoring, logging, and security namespaces.

### 4. Zero-Trust Networking (`charts/network-policies`)
By default, Kubernetes allows open pod-to-pod communication. Our `network-policies` chart enforces a strict **default-deny** posture:
- All ingress and egress traffic is blocked by default (`default-deny.yaml`).
- Granular exceptions are explicitly permitted (e.g., DNS resolution, intra-namespace microservice communication, ingress-nginx traffic routing, and Prometheus metrics scraping).

---

## 🛠️ Technology Stack Rationale

| Technology | Purpose | Why We Chose It Over Alternatives |
| :--- | :--- | :--- |
| **ArgoCD** | GitOps Continuous Delivery | Pull-based reconciliation, automated self-healing, drift detection, and secure cluster-internal polling over risky push-based CI credentials. |
| **Helm** | Kubernetes Packaging | Reusable templates (`service-chart`) combined with environment values files, providing superior modularity compared to raw YAML or Kustomize overlays. |
| **Prometheus & Grafana** | Metrics & Observability | Open-source CNCF standard, native Kubernetes service discovery (`ServiceMonitor`), avoiding costly SaaS billing models like Datadog. |
| **Grafana Loki + Fluent Bit** | Log Aggregation | Indexing log metadata (labels) instead of full-text indexing, reducing memory and CPU overhead by ~80% compared to Elasticsearch. |
| **Falco** | Runtime Security | Kernel-level eBPF threat monitoring that detects active container intrusions (e.g., unexpected shell execution), going beyond static image scanners. |
| **RabbitMQ** | Message Broker | Reliable pub/sub and task queuing with lower operational complexity than Apache Kafka for our microservice messaging patterns. |
| **Redis** | Caching & Sessions | Advanced data structure support, persistence, and master-replica replication over simple key-value caches like Memcached. |
| **PostgreSQL** | Relational Database | Strict ACID compliance, robust querying, and enterprise reliability required for transactional business domains. |

---

## 📖 Getting Started for New Engineers

1. **Understand the Flow**: Changes to microservice configurations are made by editing the respective YAML files in `helm-values/{env}/apps/`.
2. **Never Touch the Charts**: Do not modify `charts/service-chart/` unless introducing a breaking architectural change across *all* microservices.
3. **Local Validation**: You can preview Helm template rendering locally using:
   ```bash
   helm template api-gateway charts/service-chart -f helm-values/dev/apps/api-gateway.yaml
   ```
4. **ArgoCD UI**: Access the ArgoCD dashboard to monitor sync health, application drift, and automated rollouts across Dev, DR, and Prod clusters.

---
*Maintained by Maxie and the SignalForge Platform Engineering Team.*
