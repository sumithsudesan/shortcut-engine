# shortcut-engine
A cloud-native URL shortener & traffic analytics platform showcasing Go microservices, REST (v1/v2), gRPC mesh routing, RabbitMQ event-driven pipelines, and consolidated PostgreSQL (SQL/JSONB) and Redis storage architectures deployed via Terraform and Helm on AWS EKS.

This repository is organized as a unified, production-grade cloud-native monorepo. It co-locates the core application microservices, the automated deployment configurations, and the modular infrastructure declarations under a single version control tree.

golang • microservices • event-driven • rabbitmq • kubernetes • helm • terraform • aws-eks • grpc • postgresql • redis

## 📁 Repository Directory Structure

The platform is organized as a unified, single-repository monorepo. This structure consolidates application logic, data contracts, infrastructure declarations, and deployment orchestrations under a single version control tree.

```text
shortcut-engine/            # <-- Single Git Repository Root
├── .github/
│   └── workflows/          # 1. CI/CD AUTOMATION (GitHub Actions Pipeline)
│
├── api/
│   └── protobuf/           # 2. CONTRACTS (Shared gRPC Definitions & Schemas)
│
├── deploy/                 # 3. PLATFORM & DEVOPS LAYER
│   ├── terraform/          #    - Infrastructure as Code (AWS VPC, EKS, RDS, RabbitMQ)
│   └── helm/               #    - Configuration as Code (Kubernetes Umbrella Charts)
│
└── services/               # 4. APPLICATION LAYER
    ├── url-core-svc/       #    - Go Microservice (Public REST API & Redirects Engine)
    ├── analytics-engine/   #    - Go Microservice (Internal gRPC Server & Redis Caching)
    └── audit-log-worker/   #    - Go Microservice (Async RabbitMQ Consumer & JSONB Writer)

---

### 🛠️ 1. Service Layer (The Application Core)

The application code is managed via a Go Multi-Module Workspace (`go.work`), ensuring strict decoupling and isolated dependency lifecycles for each engine.

* **`services/url-core-svc` (The Edge & Relational Database Engine)**
    * **Role:** Acts as the public entry point for ingress traffic, handling high-volume token lookups.
    * **Patterns & Tech:** Exposes a public REST API natively handling **v1/v2 payload versioning** via structural adapters to maintain backward compatibility for legacy clients. Executes strict **PostgreSQL ACID transactions** to ensure cryptographic short-token generation is entirely unique and collision-free.
* **`services/analytics-engine` (The High-Throughput Handshake Layer)**
    * **Role:** Absorbs real-time click metrics non-blocking from the critical web redirect loop to eliminate network latency for users.
    * **Patterns & Tech:** Exposes an ultra-fast, binary **gRPC server** that communicates synchronously with the core service. It buffers metrics by executing atomic, in-memory increment (`INCR`) operations inside **Redis**, then marshals and dispatches the payload as an async event over RabbitMQ.
* **`services/audit-log-worker` (The Asynchronous Document Archiver)**
    * **Role:** Runs completely out-of-band from the request pipeline to safely ingest, deserialize, and archive user click sessions.
    * **Patterns & Tech:** Implements a multi-threaded **RabbitMQ consumer** with delivery confirms. It captures highly volatile click metadata blocks (User-Agent headers, IP strings, regional tracking data) and flushes the raw unstructured text directly into a PostgreSQL **`JSONB` document column** backed by a high-performance **GIN index**.

---

### 🚀 2. DevOps Layer (The Automation & Pipeline Fabric)

Configuration as Code (CaC) and automated integration suites ensure that every pull request meets explicit production-grade quality baselines.

* **Continuous Integration Engine (`.github/workflows/`)**
    * **Role:** Executes strict automated quality gates on every codebase contribution.
    * **Patterns & Tech:** Orchestrates a **GitHub Actions CI** pipeline running `golangci-lint` to check for security vulnerabilities. Concurrently executes comprehensive integration testing via **Testcontainers-Go**, dynamically booting real, isolated Docker instances of PostgreSQL, Redis, and RabbitMQ directly inside the CI runtime.
* **Kubernetes Orchestration Fabric (`deploy/helm/`)**
    * **Role:** Standardizes scheduling, environment boundaries, and stateful networking primitives.
    * **Patterns & Tech:** Engineered as a unified **Helm Umbrella Chart** managing individual microservice dependencies via structured sub-charts. Enforces centralized configuration variables (`values.yaml`), maps localized Kubernetes Secrets, and establishes deterministic **Container Resource Limits** coupled with **Liveness and Readiness health probes**.

---

### ☁️ 3. Platform Layer (The Cloud Architecture)

Infrastructure as Code (IaC) architectures translate the abstract monorepo logic directly into an immutable, multi-zone AWS ecosystem.

* **Infrastructure as Code (`deploy/terraform/`)**
    * **Role:** Provides declarative, repeatable blueprinting of the remote cloud state.
    * **Patterns & Tech:** Divided into clean, reusable modules (`/vpc`, `/eks`, `/rds`, `/mq`). Implements a remote **S3 state backend with DynamoDB distributed state locking** to eliminate simultaneous write conflicts across engineering pipelines.
* **Network & Compute Topology (AWS VPC & EKS)**
    * **Role:** Secures application boundaries via strict network segmentation.
    * **Patterns & Tech:** Provisions an **AWS VPC** split into distinct public and private subnets. External internet traffic hits a secure Application Load Balancer (ALB), while **Managed AWS EKS Compute Nodes**, **RDS PostgreSQL**, and **Amazon MQ (RabbitMQ)** are isolated deep within **Private Subnets**. Implements **EKS IRSA** (IAM Roles for Service Accounts) to map native AWS IAM policies directly to Kubernetes Pods, eliminating the risk of hardcoded environment credentials.
