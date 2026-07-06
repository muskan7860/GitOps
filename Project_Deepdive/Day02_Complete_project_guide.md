# Day 1 — Complete Project Architecture
## GitOps E-Commerce | Interview Preparation 2026
### Your Trainer: Senior DevOps Engineer (10+ Years Experience)

---

## 🎯 WHY Are We Doing This Project?

**Simple answer:**
Because this project teaches you EVERYTHING a senior DevOps engineer does in real life:
- How to containerize applications using Docker
- How to deploy on Kubernetes
- How to automate deployments using GitOps (ArgoCD)
- How to manage multiple environments (Dev vs Production)
- How to handle security, networking, databases, AI services

**When interviewer asks "Why did you choose this project?"**
> *"This project is a real-world simulation of how top companies like Amazon, Flipkart, and Google deploy their applications. It covers the complete DevOps lifecycle — from containerization with Docker, to orchestration with Kubernetes, to automated GitOps deployment with ArgoCD. It gave me hands-on experience with 14 microservices, multiple databases, cloud infrastructure, and production-grade security practices."*

---

## 🏪 WHAT Is This Project?

**First Standard Explanation:**

Imagine you are shopping on **Amazon.in**.

You open the website → you see products → you add to cart → you pay → you get an email.

That entire website is NOT one big program. It is **14 small programs working together.**

Each small program = one **microservice.**

Each microservice:
- Does ONE job only
- Runs in its own Docker container
- Is deployed independently on Kubernetes
- Can be updated without stopping others

This is called **Microservices Architecture.**

---

## 🗂️ WHO Does WHAT — All 14 Services Explained

| # | Service Name | Language | What It Does | Simple Analogy |
|---|---|---|---|---|
| 1 | **frontend** | Go | The website you see. Shows products, cart, checkout page | The shop front |
| 2 | **authservice** | Python | Login, Register, JWT token. Checks who you are | Security guard at door |
| 3 | **productcatalogservice** | Go | Shows list of products from products.json file | Product shelf |
| 4 | **cartservice** | C# | Saves what you added to cart. Uses Redis | Shopping basket |
| 5 | **redis-cart** | Redis | Temporary memory for cart data | Whiteboard |
| 6 | **checkoutservice** | Go | The BOSS. Coordinates all services when you place order | Manager |
| 7 | **paymentservice** | Node.js | Validates your credit card. Returns transaction ID | Cashier |
| 8 | **currencyservice** | Node.js | Converts prices (USD to INR etc) | Currency counter |
| 9 | **shippingservice** | Go | Calculates shipping cost. Creates tracking ID | Delivery counter |
| 10 | **emailservice** | Python | Sends order confirmation email via Gmail | Post office |
| 11 | **recommendationservice** | Python | "You may also like..." feature | Helpful salesperson |
| 12 | **adservice** | Java | Shows relevant ads on the page | Advertisement board |
| 13 | **shoppingassistantservice** | Python | AI chatbot using OpenAI GPT-4o | AI assistant |
| 14 | **loadgenerator** | Python | Fake users for testing. Simulates real traffic | Dummy customers |
| 15 | **vectordb** | PostgreSQL | AI brain memory. Stores product embeddings | AI's notebook |

---

## 🔌 HOW Do Services Talk to Each Other?

**There are 2 types of communication in this project:**

### Type 1 — HTTP REST
**What:** Normal way of talking. Like sending a text message. Uses JSON format.
**Where used:** Browser → Frontend, Frontend → AuthService, Frontend → ShoppingAssistant
**Why:** Browsers understand HTTP natively.

### Type 2 — gRPC
**What:** A faster, smarter way of talking. Uses binary format (not JSON). Google invented it.
**Where used:** Frontend → all backend services, CheckoutService → all downstream services
**Why:** Much faster than REST. Strongly typed (no mistakes). Perfect for internal service calls.

**Simple Analogy:**
- HTTP REST = Sending letters (readable, but slow)
- gRPC = Talking on phone (fast, direct, efficient)

**Interview Answer:**
> *"We used gRPC for all internal microservice communication because it uses Protocol Buffers — a binary format that is 3-10x faster than JSON. It also gives us strong type safety. For browser-facing interfaces, we used HTTP REST since browsers natively understand HTTP but not gRPC."*

---

## 🗺️ HOW Traffic Flows — Step by Step

### Step 1: User Opens Website
```
You type: www.myshop.com
     ↓
Request goes to: LoadBalancer (port 80)
     ↓
LoadBalancer sends to: Frontend (port 8080)
     ↓
Frontend shows you the homepage
```

### Step 2: You Browse Products
```
Frontend asks: productcatalogservice:3550 → "give me product list"
Frontend asks: currencyservice:7000 → "convert price to my currency"
Frontend asks: recommendationservice:8080 → "what to recommend?"
     ↓
recommendationservice asks: productcatalogservice:3550 → "full product list please"
Frontend asks: adservice:9555 → "which ad to show?"
```

### Step 3: You Login
```
Frontend asks: authservice:8081 → "verify this user"
     ↓
authservice checks: AWS RDS PostgreSQL (external database) → "is this user registered?"
     ↓
authservice returns: JWT Token (like a temporary pass)
     ↓
Frontend stores token in browser cookie
```

### Step 4: You Add to Cart
```
Frontend asks: cartservice:7070 → "add iPhone to cart for User123"
     ↓
cartservice writes to: redis-cart:6379 → "User123: [iPhone]"
```

### Step 5: You Click "Place Order" — THE MOST IMPORTANT FLOW
```
Frontend calls: checkoutservice:5050 → "PlaceOrder()"

CheckoutService then does 10 steps IN ORDER:
Step 1: cartservice:7070       → "get cart items for User123"
Step 2: productcatalogservice:3550 → "get details of those items"
Step 3: currencyservice:7000   → "convert all prices to user's currency"
Step 4: shippingservice:50051  → "calculate shipping cost"
Step 5: paymentservice:50051   → "charge the credit card"
Step 6: shippingservice:50051  → "ship the order, give tracking ID"
Step 7: emailservice:5000      → "send confirmation email"
         ↓
         emailservice calls: smtp.gmail.com:587 → actual email sent
Step 8: cartservice:7070       → "clear the cart"
Step 9: return order ID to Frontend
Step 10: Frontend shows "Order Confirmed!" page
```

---

## 💾 WHERE Is Data Stored?

This project has **3 separate databases**. Each one is isolated from the others.

### Database 1 — AWS RDS PostgreSQL (External)
**Who uses it:** AuthService only
**What is stored:** User accounts, bcrypt-hashed passwords, JWT tokens
**Where it runs:** OUTSIDE the Kubernetes cluster, on AWS cloud
**Why external:** User data is critical. AWS RDS gives automatic backups, high availability, multi-AZ failover.

```
authservice (inside cluster) ──────► AWS RDS PostgreSQL (outside cluster)
                                      "users" table: email, password_hash, created_at
```

**Interview Answer:**
> *"We chose AWS RDS for user authentication data because it's mission-critical. RDS provides automated backups, point-in-time recovery, Multi-AZ deployment for high availability, and automatic failover. Losing user account data would be catastrophic, so we didn't risk storing it on ephemeral in-cluster storage."*

### Database 2 — Redis (Inside Cluster)
**Who uses it:** CartService only
**What is stored:** Shopping cart items per user
**Where it runs:** INSIDE the Kubernetes cluster as a pod
**Important:** Uses a Kubernetes PersistentVolumeClaim in production for data durability

```
cartservice ──► redis-cart:6379
                "User123" → ["iPhone:1", "Headphones:1"]
                "User456" → ["Laptop:2"]
```

**Interview Answer:**
> *"We use Redis for cart storage because it's an in-memory database — blazing fast for read-write operations on cart data. In production, we back Redis with a PersistentVolumeClaim so cart data survives pod restarts. For a large-scale production system, we would migrate to AWS ElastiCache Redis for managed availability and automatic failover."*

### Database 3 — VectorDB — PostgreSQL + pgvector (Inside Cluster)
**Who uses it:** ShoppingAssistantService only
**What is stored:** AI product embeddings (mathematical representations of products)
**Where it runs:** INSIDE the Kubernetes cluster

**What is an embedding? (First Standard):**
- Normal database stores: "Product name: Red Running Shoes, Size: 10"
- Vector database stores: [0.23, 0.87, 0.45, 0.91 ...] — a list of 1000+ numbers that represents what the product MEANS
- When you ask "comfortable shoes for gym" — it searches for vectors closest to your question
- This is called **semantic search** — finding meaning, not just matching words

```
ShoppingAssistantService
    ↓ user asks: "show me red shoes for running"
    ↓ converts question to vector: [0.23, 0.87, ...]
    ↓ searches vectordb for similar vectors
    ↓ finds: "Red Adidas Running Shoes" (closest match)
    ↓ calls OpenAI GPT-4o to form a nice answer
    ↓ returns: "I found these perfect running shoes for you!"
```

---

## 🌍 WHERE Does This Run — Environments

### Environment 1 — MicroK8s (Your ThinkPad — Development)
**What:** A lightweight Kubernetes that runs on a single machine
**Why:** Free, fast to set up, perfect for learning and testing
**Config location:** `overlays/microk8s/kustomization.yaml`
**Settings:** 1 replica per service, image tag `:1`

### Environment 2 — AWS EKS (Amazon Cloud — Production)
**What:** Amazon's fully managed Kubernetes service
**Why:** High availability, auto-scaling, AWS integrations
**Config location:** `overlays/eks/kustomization.yaml`
**Settings:** 2 replicas for key services, AWS NLB Load Balancer, IRSA for IAM

**Interview Answer:**
> *"We used a Kustomize base-overlay pattern. The base directory contains all common Kubernetes manifests shared across environments. For local development on MicroK8s, we have an overlay with single replicas and lightweight settings. For production on AWS EKS, we have a separate overlay with 2 replicas for high availability, AWS NLB annotations for the load balancer, and IRSA — IAM Roles for Service Accounts — for secure AWS resource access without hardcoding credentials."*

---

## 🔐 HOW Is Security Handled?

### Security Layer 1 — Pod Security Context
Every single pod in this project runs with strict rules:

```yaml
# What this means in simple words:
runAsNonRoot: true
# → The container cannot run as root (admin/superuser)
# → Like saying: employees cannot have admin access to the building

runAsUser: 1000
# → The container runs as user ID 1000 (a normal user)
# → Like an employee badge with limited access

allowPrivilegeEscalation: false
# → The container cannot gain more power than it started with
# → Like: you cannot promote yourself to manager

readOnlyRootFilesystem: true
# → The container cannot write to its own disk
# → Like: you can read files but cannot change them

capabilities.drop: [ALL]
# → All special Linux powers are removed
# → Like: no special tools allowed inside the building
```

**Interview Answer:**
> *"We implemented pod-level security hardening on all deployments following the principle of least privilege. Containers run as non-root user ID 1000, privilege escalation is disabled, root filesystems are read-only, and all Linux capabilities are dropped. This significantly reduces the attack surface — even if an attacker gets into a container, they cannot escalate privileges or write malicious files to the filesystem."*

### Security Layer 2 — Kubernetes Secrets
Sensitive information (passwords, API keys) is NEVER hardcoded in YAML files.

```bash
# Bad practice (NEVER do this):
env:
  - name: GMAIL_PASSWORD
    value: "mypassword123"   # ← Anyone who reads YAML file sees this!

# Good practice (what this project does):
env:
  - name: GMAIL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: emailservice-secret
        key: GMAIL_APP_PASSWORD   # ← Pulled from Kubernetes Secret at runtime
```

**Secrets created before deployment:**
1. `postgres-secret` — VectorDB password
2. `authservice-secret` — JWT signing key
3. `shopping-assistant-secrets` — OpenAI API key + database URL
4. `emailservice-secret` — Gmail address + app password

---

## 🤖 WHAT Is GitOps and WHY Did We Use It?

### Old Way (Without GitOps):
```
Developer writes code
     ↓
Developer SSHs into server
     ↓
Developer manually runs: kubectl apply -f deployment.yaml
     ↓
Hope it works. No audit trail. If someone changes something manually, nobody knows.
```

### New Way (GitOps with ArgoCD):
```
Developer writes code
     ↓
Developer pushes YAML to GitHub
     ↓
ArgoCD (running inside cluster) automatically detects the change
     ↓
ArgoCD runs: kustomize build overlays/microk8s/ (or eks/)
     ↓
ArgoCD applies the result to Kubernetes
     ↓
Cluster is always in sync with Git. Automatic. Zero manual work.
```

### WHY GitOps?

| Problem (Old Way) | Solution (GitOps) |
|---|---|
| Nobody knows who changed what | Every change is a Git commit — full audit trail |
| Manual mistakes in production | Automated — same process every time |
| "It works on my machine" | Git is the single source of truth |
| No rollback | Git rollback = instant cluster rollback |
| Manual kubectl = human error | ArgoCD applies automatically = no human error |

### ArgoCD Key Settings in This Project:
- **Auto-sync: ON** — ArgoCD checks Git every 3 minutes automatically
- **Prune: true** — If you DELETE a file from Git, ArgoCD deletes from cluster too
- **Self-heal: true** — If someone manually changes cluster, ArgoCD REVERTS it back to Git state

**Interview Answer:**
> *"We chose GitOps with ArgoCD because it solves the core problem of configuration drift. In traditional deployments, someone might manually kubectl edit a deployment in production, and nobody knows. With ArgoCD self-heal enabled, any manual change to the cluster is automatically reverted to match the Git state. Git becomes the single source of truth. Every deployment is a Git commit — fully auditable, reviewable, and rollbackable."*

---

## 📦 WHAT Is Kustomize and WHY Did We Use It?

**First Standard Explanation:**

Imagine you are baking a cake.

The **base recipe** is: flour, sugar, eggs, butter. Same for all cakes.

But for **birthday cake** you add: sprinkles, candles, extra frosting.
And for **wedding cake** you add: white frosting, 5 layers, flowers.

Same base. Different additions for different occasions.

That's Kustomize.

```
base/frontend/deployment.yaml    ← Base recipe (same for all)
     +
overlays/microk8s/kustomization.yaml  ← "For dev: 1 replica, tag :1"
     =
Final YAML for MicroK8s (what ArgoCD applies to your ThinkPad cluster)

base/frontend/deployment.yaml    ← Same base recipe
     +
overlays/eks/kustomization.yaml  ← "For production: 2 replicas, AWS NLB"
     =
Final YAML for EKS (what ArgoCD applies to AWS cluster)
```

**Interview Answer:**
> *"We used Kustomize for environment-specific configuration management. It follows DRY principle — Don't Repeat Yourself. We write base Kubernetes manifests once and maintain environment differences in overlay files. For MicroK8s development, our overlay sets 1 replica and lightweight resource limits. For AWS EKS production, the overlay sets 2 replicas for high availability, adds AWS NLB annotations for the load balancer, and adds IRSA annotations for IAM integration. No duplication of YAML files."*

---

## 🏗️ App of Apps Pattern — WHY One App.yaml Per Service?

Look at the repo root. There is one `app.yaml` inside EACH service folder:
- `adservice/app.yaml`
- `frontend/app.yaml`
- `cartservice/app.yaml`
- ... and so on

**WHY? First Standard Explanation:**

Imagine a school. The principal (ArgoCD) manages 14 classrooms (services).

**Bad way:** Principal personally manages every student in every classroom.
**Good way:** Each classroom has a monitor (app.yaml). Principal only manages the monitors.

This is the **App of Apps pattern.**

The root `argocd-app.yaml` is the PARENT. It contains 15 ArgoCD Application objects — one per service.

Each Application object tells ArgoCD:
- "Watch THIS folder in Git"
- "Deploy to THIS namespace"
- "Use THIS kustomize overlay"

**Benefit:** Each microservice can be deployed, updated, or rolled back **independently** without affecting other services.

**Interview Answer:**
> *"We implemented the App of Apps pattern in ArgoCD. Each microservice has its own ArgoCD Application resource pointing to its specific directory in the GitOps repository. This gives us independent deployment control — we can deploy, rollback, or pause a single service without touching others. The root argocd-app.yaml acts as the parent application that manages all child applications."*

---

## 🧠 AI Features — ShoppingAssistant Explained Simply

This is the most modern part of this project. Great to mention in interviews.

**What it does:**
User uploads a photo of their room and asks: "What furniture would look good here?"

**How it works:**
```
Step 1: User sends photo + question to ShoppingAssistantService
Step 2: Service sends photo to OpenAI GPT-4o → "describe what's in this room"
Step 3: OpenAI returns: "Modern minimalist room, needs white furniture"
Step 4: Service converts "white furniture" into a vector (list of numbers)
Step 5: Service searches VectorDB → "find products with similar vectors"
Step 6: VectorDB returns: [Product ID 1, Product ID 3, Product ID 7]
Step 7: Service calls ProductCatalog → "give details of these products"
Step 8: OpenAI formats a nice response: "Here are 3 perfect items for your room..."
Step 9: User sees AI-curated product recommendations
```

**Tech used:**
- OpenAI GPT-4o = The brain
- LangChain = The framework that connects everything
- pgvector = The AI memory (vector similarity search)
- Kubernetes Secrets = Securely stored API keys

---

## 📊 Resource Allocation — WHY Each Service Gets Different CPU/Memory

In production, we must tell Kubernetes exactly how much CPU and RAM each service needs.

**Two values always defined:**

| Term | Simple Meaning | Analogy |
|---|---|---|
| **Request** | Minimum guaranteed resources | "I need at least this much" |
| **Limit** | Maximum allowed resources | "You cannot take more than this" |

**Example — Frontend:**
```yaml
resources:
  requests:
    cpu: 100m      # 100 millicores = 0.1 CPU core minimum
    memory: 64Mi   # 64 megabytes minimum
  limits:
    cpu: 200m      # Cannot use more than 0.2 CPU core
    memory: 128Mi  # Cannot use more than 128 megabytes
```

**WHY this matters in interview:**
> *"Resource requests and limits are critical for cluster stability. Without limits, one misbehaving service can consume all cluster resources and crash other services — this is called the Noisy Neighbor problem. With requests, Kubernetes knows how to schedule pods on nodes efficiently. With limits, it enforces boundaries."*

---

## 🔍 Observability — OpenTelemetry

Every service in this project has tracing built in using **OpenTelemetry**.

**What is tracing? (First Standard):**

When you click "Place Order", that request touches 10 different services. Something goes wrong and order fails. How do you find WHERE it failed?

Tracing = Like a GPS tracker on the request. It records every service the request visited, how long it spent there, and where it failed.

```
Request ID: abc123
  → Frontend (2ms) ✅
  → CheckoutService (5ms) ✅
  → CartService (3ms) ✅
  → PaymentService (ERROR after 1000ms) ❌  ← found the problem!
```

**Interview Answer:**
> *"We implemented distributed tracing using OpenTelemetry across all 14 services. This gives us end-to-end visibility into request flows. When an order fails, instead of checking logs in 14 different services one by one, we can trace the exact request journey and immediately identify which service caused the failure and how long each step took."*

---

## 🎤 "Walk Me Through Your Project" — FULL INTERVIEW SCRIPT

**Speak this out loud 5 times. This is your most important answer.**

> *"I worked on a production-grade, cloud-native e-commerce platform deployed on Kubernetes using GitOps principles.*
>
> *The application has 14 microservices written in multiple languages — Go, Python, Node.js, C#, and Java. Each service is containerized using Docker and deployed independently on Kubernetes. The frontend is the single entry point, exposed via a Kubernetes LoadBalancer service on port 80. Internally, all services communicate using gRPC for performance and type safety, except browser-facing endpoints which use HTTP REST.*
>
> *For deployment automation, we used GitOps with ArgoCD. Our GitHub repository is the single source of truth. Any YAML change pushed to the main branch is automatically detected by ArgoCD and applied to the cluster. We enabled auto-sync, prune, and self-heal — so the cluster always matches Git state, and any manual changes are automatically reverted.*
>
> *For configuration management, we used Kustomize with a base-overlay pattern. The base directory has common manifests. The overlays directory has environment-specific customizations — MicroK8s overlay for local development with single replicas, and EKS overlay for production with 2 replicas and AWS NLB integration.*
>
> *We implemented the App of Apps pattern in ArgoCD — each microservice has its own ArgoCD Application for independent deployment control.*
>
> *For data storage, we have three isolated stores — AWS RDS PostgreSQL for user authentication, Redis for shopping cart with PersistentVolumeClaim backing, and PostgreSQL with pgvector extension for AI-powered product search.*
>
> *Security was enforced at the pod level — non-root execution, read-only root filesystem, all Linux capabilities dropped, and all secrets stored in Kubernetes Secrets never hardcoded in YAML.*
>
> *We also built an AI-powered shopping assistant using OpenAI GPT-4o, LangChain, and pgvector for semantic product search.*
>
> *Observability was built in using OpenTelemetry distributed tracing across all services."*

---

## ❓ Interview Questions & Answers — Day 1

### Basic Questions

**Q: How many services are in this project?**
A: 14 microservices plus Redis = 15 running components in the cluster.

**Q: What languages are used and why?**
A: Go for high-throughput services like frontend, checkout, shipping — Go is fast and memory efficient. Python for ML/AI services like recommendation, email, shopping assistant — Python has best ML libraries. Node.js for payment and currency — lightweight and fast for I/O. Java for adservice — mature ecosystem for enterprise patterns. C# for cartservice — .NET has excellent gRPC support.

**Q: How do services discover each other?**
A: Through Kubernetes DNS. Every Kubernetes Service gets a stable DNS name inside the cluster. For example, cartservice is reachable at `cartservice:7070` from any pod. These addresses are injected as environment variables into each pod at startup.

**Q: What is the checkout flow?**
A: 10 steps — get cart → get products → convert currency → get shipping quote → charge payment → ship order → send email → clear cart → return order ID → show confirmation.

**Q: How are secrets managed?**
A: All sensitive data — API keys, passwords, database URLs — are stored as Kubernetes Secrets. Pods reference secrets via `secretKeyRef` in environment variable definitions. Secrets are never hardcoded in YAML files or Docker images.

### Scenario Questions

**Scenario 1: "Order placed but user never received confirmation email."**
> *"I would debug in 3 steps. First, kubectl logs on emailservice pod — check if it connected to Gmail SMTP on port 587. Second, verify the emailservice-secret exists with GMAIL_ADDRESS and GMAIL_APP_PASSWORD. Third, check the port configuration — emailservice container listens on 8080 but the Kubernetes Service exposes port 5000. CheckoutService calls emailservice:5000. If that Service mapping is wrong, requests never reach the container."*

**Scenario 2: "Cart items disappeared after deployment."**
> *"This is a data persistence issue. If Redis pod restarted during deployment, and we are using ephemeral storage, all cart data is lost. The fix is to ensure Redis uses a PersistentVolumeClaim backed by a StorageClass that survives pod restarts. Long term, for production scale, we would use AWS ElastiCache which gives us managed Redis with automatic replication and failover."*

**Scenario 3: "ArgoCD shows OutOfSync. What do you do?"**
> *"OutOfSync means the cluster state does not match the Git state. First, I check the ArgoCD UI to see which resources are out of sync. If it is due to a legitimate change someone made in Git, I click Sync to apply it. If someone manually changed the cluster and self-heal is on, ArgoCD should auto-revert it. If auto-revert is not happening, I check ArgoCD logs for errors. I would never manually fix the cluster — I fix the Git and let ArgoCD apply."*

**Scenario 4: "How would you roll back a bad deployment?"**
> *"In GitOps, rollback is simply reverting the Git commit. I would run git revert on the bad commit and push to main. ArgoCD detects the change within 3 minutes and applies the previous YAML — which uses the previous image tag. The cluster rolls back automatically. No kubectl rollout undo needed."*

**Scenario 5: "The AI shopping assistant is slow. What do you check?"**
> *"Three areas. First, OpenAI API latency — if GPT-4o is taking 5+ seconds, that is an external dependency issue, not ours. Second, VectorDB query performance — if similarity search is slow, the pgvector index might not be created. Third, shoppingassistantservice resource limits — if CPU limit is too low (we have 500m limit), the container might be throttled. I would check kubectl top pod and increase limits in the overlay."*

---

## 📝 Homework Before Day 2

Run these on your ThinkPad:

```bash
# Clone the repo
git clone https://github.com/muskan7860/GitOps.git
cd GitOps

# Read the frontend deployment line by line
cat base/frontend/deployment.yaml

# Read the frontend service
cat base/frontend/service.yaml

# Read the kustomize base list
cat base/kustomization.yaml

# Read the MicroK8s overlay
cat overlays/microk8s/kustomization.yaml
```

Paste these 4 files here tomorrow. We will read EVERY SINGLE LINE together from zero.

---

## 🗓️ 15-Day Master Plan

| Day | Topic | Files We Cover |
|---|---|---|
| Day 1 ✅ | Architecture + Big Picture | ARCHITECTURE.md |
| Day 2 | Kubernetes Manifests | deployment.yaml + service.yaml |
| Day 3 | Kustomize Deep Dive | base/kustomization.yaml + overlays |
| Day 4 | ArgoCD Deep Dive | argocd-app.yaml + app.yaml per service |
| Day 5 | Dockerfile — Writing from Scratch | Dockerfile for frontend, python, node |
| Day 6 | Jenkinsfile — CI Pipeline | Jenkinsfile for build + push + update |
| Day 7 | Ingress + Networking | ingress.yaml + clusterissuer.yaml |
| Day 8 | Security Deep Dive | securityContext + Secrets + RBAC |
| Day 9 | AWS EKS Architecture | overlays/eks/ + AWS services |
| Day 10 | Observability | OpenTelemetry + tracing concepts |
| Day 11 | Troubleshooting Scenarios | 20 real interview scenarios |
| Day 12 | Mock Interview Round 1 | Architecture + Design |
| Day 13 | Mock Interview Round 2 | Scenario + Troubleshooting |
| Day 14 | Weak Area Revision | Whatever needs more practice |
| Day 15 | Final Mock Interview | Full simulation |

---

*Save this file to your GitHub repo under `Interview_Preparation_2026/gitops_project/Day1_Architecture.md`*

*Read it tonight. Speak the interview answers out loud 3 times each.*

*Tomorrow: Day 2 — We open deployment.yaml and read every line from zero.*
