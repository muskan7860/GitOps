# Day 1 — What Is This Project & How Does It Work?
### GitOps E-Commerce Project | Interview Preparation 2026
### Role: Senior DevOps Engineer | Trainer: 10+ Years Experience

---

## 🎯 First Things First — What Will You Say in the Interview?

When interviewer says: **"Walk me through your project"**

You will say:

> *"I worked on a cloud-native e-commerce platform deployed on Kubernetes using GitOps principles. The application has 14 microservices written in multiple languages — Go, Python, Node.js, Java, and C#. Each service is independently containerized using Docker and deployed on Kubernetes. We used ArgoCD as our GitOps engine and Kustomize for managing environment-specific configurations. The entire deployment was automated — no one manually touched the cluster. Any change pushed to Git was automatically picked up by ArgoCD and applied to the cluster."*

That's your opening. Memorize this. Everything else builds on top of this.

---

## 🏪 What Is This Application? (First Standard Explanation)

Imagine you open **Amazon.com** on your browser.

You see products. You add to cart. You checkout. You get an email. Simple right?

But behind that simple website, there are **14 different small programs** running. Each program does ONE job only.

| What you do on website | Which service handles it |
|---|---|
| You open the website | **Frontend** shows you the page |
| You see products | **ProductCatalogService** gives the list |
| You see prices in your currency | **CurrencyService** converts it |
| You see "You may also like" | **RecommendationService** suggests |
| You see ads on the page | **AdService** shows relevant ads |
| You add item to cart | **CartService** saves it in Redis |
| You login | **AuthService** checks your password |
| You click "Place Order" | **CheckoutService** orchestrates everything |
| Payment is charged | **PaymentService** validates the card |
| Shipping is calculated | **ShippingService** gives quote |
| You get confirmation email | **EmailService** sends via Gmail |
| AI chat assistant | **ShoppingAssistantService** uses GPT-4o |
| Fake users for testing | **LoadGenerator** simulates traffic |
| AI product search database | **VectorDB** stores embeddings |

**Total: 14 services + 1 Redis = 15 running components**

Each one is a **separate Docker container** running inside Kubernetes.

---

## 🗺️ The Big Picture — How Traffic Flows

Think of it like a CITY.

- **LoadBalancer** = The main city gate. All visitors enter here.
- **Frontend** = The reception desk. Welcomes you and directs you.
- **Other services** = Different departments inside the building.
- **Redis** = A whiteboard where cart items are temporarily written.
- **AWS RDS** = The permanent filing cabinet (user accounts).
- **VectorDB** = The AI brain's memory.

```
YOU (browser)
     |
     | (type www.myshop.com)
     ▼
LoadBalancer (port 80)
     |
     ▼
FRONTEND (port 8080)
     |
     ├──► ProductCatalogService (port 3550) — "show me products"
     ├──► CurrencyService (port 7000) — "convert to INR"
     ├──► CartService (port 7070) — "add this to cart"
     │         └──► Redis (port 6379) — "save it here"
     ├──► RecommendationService (port 8080) — "suggest more"
     ├──► ShippingService (port 50051) — "how much to ship?"
     ├──► AdService (port 9555) — "show relevant ad"
     ├──► AuthService (port 8081) — "is this user logged in?"
     │         └──► AWS RDS PostgreSQL — "check user database"
     ├──► ShoppingAssistantService (port 80) — "AI chat"
     │         └──► OpenAI GPT-4o — "answer the question"
     │         └──► VectorDB (port 5432) — "search products by AI"
     │
     └──► CheckoutService (port 5050) — "PLACE ORDER"
               ├──► CartService — "get cart items"
               ├──► ProductCatalogService — "get product details"
               ├──► CurrencyService — "convert prices"
               ├──► ShippingService — "ship the order"
               ├──► PaymentService (port 50051) — "charge card"
               └──► EmailService (port 5000) — "send email"
                         └──► Gmail SMTP — "actual email sent"

LoadGenerator — constantly sends fake traffic to Frontend for testing
```

---

## 🔌 What is gRPC? (Zero Knowledge Explanation)

You saw many services have "gRPC" as protocol. What is this?

Normal way services talk: **HTTP REST**
- Like sending a letter. You write JSON. Other side reads JSON.
- Easy to understand. But a little slow.

gRPC way:
- Like a phone call. Faster. Strongly typed.
- Uses something called **Protocol Buffers (protobuf)** — a very compact format.
- Google invented it. Used for internal microservice communication.

**Rule in this project:**
- **Internal service-to-service** = gRPC (fast, efficient)
- **External (browser to frontend, frontend to auth)** = HTTP REST (browser understands HTTP)

**Interview Answer:**
> *"We used gRPC for all internal microservice communication because it is faster than REST, uses binary protocol buffers instead of JSON, and provides strong type safety. For browser-facing APIs like the frontend and auth service, we used HTTP REST since browsers natively understand HTTP."*

---

## 🏗️ The 3-Layer Architecture

This project has 3 layers. Think of it like a building:

```
┌─────────────────────────────────────┐
│  LAYER 3: EXTERNAL CONNECTIONS      │  ← AWS RDS, Gmail SMTP, OpenAI
│  (Things outside Kubernetes)        │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  LAYER 2: APPLICATION SERVICES      │  ← All 14 microservices
│  (Running inside Kubernetes)        │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  LAYER 1: DATA STORES               │  ← Redis, VectorDB (PostgreSQL)
│  (Databases inside Kubernetes)      │
└─────────────────────────────────────┘
```

**Layer 1 — Data Stores (inside cluster):**
- **Redis** — stores shopping cart data temporarily
- **VectorDB (PostgreSQL + pgvector)** — stores AI product search data

**Layer 2 — Application Services (inside cluster):**
- All 14 microservices running as Kubernetes pods

**Layer 3 — External Connections (outside cluster):**
- **AWS RDS PostgreSQL** — user login data (authservice connects to this)
- **Gmail SMTP** — sends real emails
- **OpenAI API** — AI shopping assistant brain

---

## 💾 Data Stores Explained (Zero Knowledge)

### Redis — The Whiteboard
- Super fast. Lives in memory.
- CartService writes: "User123 has: iPhone, Headphones"
- When pod restarts → **DATA IS LOST** ⚠️
- This is a known weakness in this project (uses `emptyDir` volume)
- **Interview Trap Question:** "Is this production ready?"
- **Your Answer:** *"No. Redis uses emptyDir which is ephemeral. In production we would use a PersistentVolumeClaim with a proper storage class, or a managed Redis service like AWS ElastiCache."*

### VectorDB — The AI Brain Memory
- PostgreSQL database with **pgvector extension**
- pgvector = PostgreSQL that can store and search AI embeddings
- Shopping Assistant service uploads product descriptions as vectors
- When you ask "show me red running shoes" → it does similarity search

### AWS RDS — The Permanent Filing Cabinet
- External to the cluster (not inside Kubernetes)
- AuthService stores usernames, bcrypt-hashed passwords, JWT tokens here
- Managed by AWS — backups, high availability handled by AWS

---

## 🔐 Security — How This Project Is Secured

Every pod in this project runs with strict security rules. Think of it like:

**Normal Docker container** = Like giving someone a master key to the house.
**This project** = Like giving someone a key only to one room, and even in that room, they cannot write anything.

```yaml
# What every pod has:
runAsNonRoot: true          # Pod cannot run as root (admin)
runAsUser: 1000             # Runs as a normal user, user ID 1000
allowPrivilegeEscalation: false  # Cannot gain more power than it started with
readOnlyRootFilesystem: true     # Cannot write to its own filesystem
capabilities.drop: [ALL]         # All Linux special powers removed
```

**Interview Answer:**
> *"We implemented Pod Security Context on all deployments. Containers run as non-root user with ID 1000, privilege escalation is disabled, root filesystem is read-only, and all Linux capabilities are dropped. This follows the principle of least privilege — each service has only the minimum permissions it needs to function."*

---

## 🌍 Environments — Where Does This Run?

This project can run in TWO places:

| Environment | Where | Purpose | Config Location |
|---|---|---|---|
| **MicroK8s** | Your laptop/ThinkPad | Development & Testing | `overlays/microk8s/` |
| **EKS** | AWS Cloud | Production | `overlays/eks/` |

**MicroK8s** = Single machine Kubernetes. Like a mini version of the real thing. Perfect for your ThinkPad.
**EKS** = Amazon's managed Kubernetes service. Used for real production.

The beauty of Kustomize: **same base code, different settings per environment.**

---

## 📁 Repository Structure (What Every Folder Means)

```
GitOps/
│
├── base/                    ← THE FOUNDATION
│   ├── adservice/
│   │   ├── deployment.yaml  ← "Run adservice with this image"
│   │   └── service.yaml     ← "Expose adservice on port 9555"
│   ├── frontend/
│   │   ├── deployment.yaml  ← "Run frontend"
│   │   ├── service.yaml     ← "Expose frontend"
│   │   ├── ingress.yaml     ← "Route internet traffic to frontend"
│   │   └── clusterissuer.yaml ← "Get SSL/HTTPS certificate"
│   └── kustomization.yaml   ← "Master list of all services"
│
├── overlays/
│   ├── microk8s/
│   │   └── kustomization.yaml ← "For laptop: 1 replica, tag :1"
│   └── eks/
│       └── kustomization.yaml ← "For AWS: 2 replicas, NLB annotations"
│
├── argocd-app.yaml          ← "ArgoCD, please watch these 14 services"
├── kustomization.yaml       ← "Root level Kustomize"
│
├── adservice/app.yaml       ← ArgoCD Application for adservice
├── frontend/app.yaml        ← ArgoCD Application for frontend
└── ... (one app.yaml per service)
```

---

## 🎤 "Walk Me Through Your Project" — Full Interview Answer

Memorize this. Practice saying it out loud.

> *"I worked on a cloud-native e-commerce microservices platform deployed on Kubernetes. The application consists of 14 microservices written in Go, Python, Node.js, Java, and C#. Each service is containerized using Docker and deployed independently on Kubernetes.*
>
> *For deployment automation, we used GitOps with ArgoCD. This means our Git repository is the single source of truth — any change pushed to Git is automatically detected by ArgoCD and applied to the cluster. Nobody manually runs kubectl in production. If someone makes a manual change to the cluster, ArgoCD detects the drift and automatically reverts it.*
>
> *For configuration management across environments, we used Kustomize with a base-overlay pattern. The base folder has common Kubernetes manifests shared across all environments. The overlays folder has environment-specific customizations — for local development we used MicroK8s overlay with single replicas, and for AWS production we used the EKS overlay with 2 replicas and AWS NLB annotations.*
>
> *The services communicate internally using gRPC for performance and type safety. The frontend is the single entry point, exposed via a Kubernetes LoadBalancer service. Authentication uses JWT tokens stored in AWS RDS PostgreSQL. Shopping cart data uses Redis. We also have an AI-powered shopping assistant backed by OpenAI GPT-4o and a PostgreSQL pgvector database for semantic product search.*
>
> *Security was enforced at the pod level — all containers run as non-root, with read-only root filesystems and all Linux capabilities dropped."*

---

## ❓ Interview Questions — Day 1 Topics

### Basic Questions (They always ask these)

**Q1: What is this application?**
A: E-commerce microservices platform with 14 services. Frontend in Go, services in Python/Node.js/Java/C#. Each service does one job independently.

**Q2: How many services are there and what do they do?**
A: 14 services + Redis = 15 components. (List them — Frontend, Auth, ProductCatalog, Cart, Checkout, Payment, Currency, Shipping, Email, Recommendation, Ad, ShoppingAssistant, LoadGenerator, VectorDB)

**Q3: What communication protocol is used between services?**
A: gRPC for internal service-to-service (faster, binary, strongly typed). HTTP REST for browser-facing APIs.

**Q4: What databases are used?**
A: Three data stores — AWS RDS PostgreSQL for user authentication, Redis for shopping cart (ephemeral), and PostgreSQL with pgvector for AI product embeddings.

**Q5: Is this project production ready?**
A: Partially. The security contexts, resource limits, and GitOps automation are production-grade. But Redis uses emptyDir (data lost on restart), and some EKS patches are commented out. We would need to add PersistentVolumeClaim for Redis and activate EKS overlays for true production.

### Scenario Questions

**Scenario 1:** *"User complains they added items to cart but items disappeared."*
Your Answer: *"This is a known architecture issue. Redis uses emptyDir volume which is ephemeral — when the redis-cart pod restarts for any reason (OOM kill, node restart, deployment update), all cart data is lost. The fix is to replace emptyDir with a PersistentVolumeClaim backed by a StorageClass, or migrate to a managed Redis service like AWS ElastiCache."*

**Scenario 2:** *"Order confirmation emails are not being received."*
Your Answer: *"I would check three things. First, kubectl logs on the emailservice pod — is it connecting to Gmail SMTP on port 587? Second, check if the emailservice-secret exists in the cluster with GMAIL_ADDRESS and GMAIL_APP_PASSWORD. Third, there is a known port mismatch — the container listens on 8080 but the Kubernetes Service exposes port 5000. If the Service or CheckoutService config is wrong, emails will silently fail."*

**Scenario 3:** *"The shopping assistant is returning wrong product recommendations."*
Your Answer: *"I would check the VectorDB — if product embeddings are not properly initialized, the pgvector similarity search returns irrelevant results. The vectordb has an init.sql ConfigMap that runs on first startup to enable the pgvector extension. I would also verify the shopping-assistant-secrets are correctly set with OPENAI_API_KEY and DATABASE_URL pointing to vectordb:5432."*

---

## 📝 Homework Before Day 2

Run these commands on your ThinkPad and read the output:

```bash
# 1. Clone the repo (if not done)
git clone https://github.com/muskan7860/GitOps.git
cd GitOps

# 2. Look at the base structure
ls base/

# 3. Read one deployment file
cat base/frontend/deployment.yaml

# 4. Read one service file
cat base/frontend/service.yaml

# 5. Read the kustomization
cat base/kustomization.yaml
```

Paste the output of step 3 and 4 in our next session. We will read every single line together.

---

## 🗓️ Full Study Plan (17 Days to July 15th)

| Day | Topic |
|---|---|
| Day 1 (Today) | ✅ Project Architecture & Big Picture |
| Day 2 | Kubernetes Manifests — deployment.yaml & service.yaml line by line |
| Day 3 | Kustomize — base vs overlays, how EKS differs from MicroK8s |
| Day 4 | ArgoCD — App of Apps, sync, prune, self-heal, drift detection |
| Day 5 | Dockerfile — how to write for each service (frontend, python, node) |
| Day 6 | Jenkins CI Pipeline — Jenkinsfile from scratch |
| Day 7 | Networking — LoadBalancer, Ingress, ClusterIssuer, TLS/HTTPS |
| Day 8 | Security — Pod Security Context, Secrets, RBAC |
| Day 9 | AWS EKS Architecture — how this runs on AWS |
| Day 10 | Monitoring & Observability — OpenTelemetry in this project |
| Day 11 | Troubleshooting Scenarios — 20 real interview scenarios |
| Day 12 | Mock Interview Round 1 — Architecture & Design Questions |
| Day 13 | Mock Interview Round 2 — Scenario & Troubleshooting Questions |
| Day 14 | Weak Areas Revision |
| Day 15 | Final Mock Interview — Full Simulation |

---

*Day 1 Complete. Save this file. Re-read it tonight before sleeping.*
*Tomorrow: Day 2 — We open deployment.yaml and service.yaml and read every single line.*
