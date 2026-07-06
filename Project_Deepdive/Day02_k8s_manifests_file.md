# Day 2 — Kubernetes Manifests Deep Dive
## Every File, Every Line, From Zero
### GitOps E-Commerce Project | Interview Preparation 2026

---

## 🎯 What You Will Learn Today

By end of Day 2 you will be able to:
- Explain every line of every deployment.yaml and service.yaml
- Explain what ClusterIP, LoadBalancer, Ingress means
- Explain how Traefik replaces NGINX (and WHY)
- Answer any manifest-related interview question confidently
- Know how to practice WITHOUT Dockerfile

---

## 📖 FIRST — Understand the 3 File Types in This Project

Every service in this project has exactly these files:

```
base/adservice/
├── deployment.yaml   ← "How to RUN the service"
└── service.yaml      ← "How to REACH the service"

base/frontend/
├── deployment.yaml   ← "How to RUN the frontend"
├── service.yaml      ← "How to REACH the frontend"
├── ingress.yaml      ← "How internet traffic ENTERS the cluster"
└── clusterissuer.yaml ← "How to get HTTPS certificate"
```

Simple analogy:
- `deployment.yaml` = Job description (what to run, how many, with what resources)
- `service.yaml` = Phone directory (how to call/reach that service)
- `ingress.yaml` = Building entrance (how outsiders enter)
- `clusterissuer.yaml` = SSL certificate authority (HTTPS lock in browser)

---

## 📄 PART 1 — deployment.yaml Explained Line by Line

We will use `adservice` as the first example. Every other service follows the same pattern.

---

### adservice/deployment.yaml — Every Line Explained

```yaml
apiVersion: apps/v1
```
**WHAT:** Which version of Kubernetes API we are using.
**WHY:** Kubernetes has multiple API versions. `apps/v1` is the stable version for Deployments.
**Analogy:** Like saying "I am using English language version 2024" — sets the rulebook.

---

```yaml
kind: Deployment
```
**WHAT:** The TYPE of Kubernetes object we are creating.
**WHY:** Kubernetes has many types — Deployment, Service, Pod, ConfigMap, Secret etc.
**Deployment means:** "Run this container. If it crashes, restart it automatically. If I want 3 copies, run 3 copies."
**Analogy:** Like telling HR: "I need a PERMANENT employee" (not a contractor = not a plain Pod).

---

```yaml
metadata:
  name: adservice
  labels:
    app: adservice
```
**WHAT:** Identity card of this Deployment.
- `name: adservice` → This Deployment's name is "adservice"
- `labels: app: adservice` → A tag/sticker on this object. Used for grouping and finding.

**WHY labels?** Later, the Service uses labels to find the right pods. Like a name tag at a conference.

---

```yaml
spec:
```
**WHAT:** Everything inside `spec:` is the SPECIFICATION — what you actually want.
**Analogy:** The spec is the recipe. Above was the recipe card name. Below is the actual ingredients.

---

```yaml
  selector:
    matchLabels:
      app: adservice
```
**WHAT:** The Deployment says "I manage pods that have the label `app: adservice`"
**WHY:** Kubernetes needs to know WHICH pods belong to this Deployment. It matches by label.
**Analogy:** A manager saying "I manage employees whose badge says Department: AdService."

---

```yaml
  template:
    metadata:
      labels:
        app: adservice
```
**WHAT:** The TEMPLATE is the blueprint for every pod this Deployment creates.
- Every pod created by this Deployment will have label `app: adservice`
- This label is how the Service finds these pods (selector matches label)

**Analogy:** A stamp that gets put on every employee's badge when they join the AdService department.

---

```yaml
    spec:
      serviceAccountName: adservice
```
**WHAT:** The pod runs with a Kubernetes ServiceAccount named "adservice"
**WHY:** ServiceAccounts control what permissions the pod has inside the cluster.
**Like:** An employee ID card that controls which rooms they can enter.
**Production importance:** We never use the "default" service account. Each service gets its own with minimum permissions (principle of least privilege).

---

```yaml
      terminationGracePeriodSeconds: 5
```
**WHAT:** When Kubernetes wants to STOP this pod, it gives it 5 seconds to finish what it's doing before force-killing it.
**WHY:** Without this, if a request is being processed and pod gets killed, that request fails. 5 seconds = graceful shutdown.
**Analogy:** Like telling an employee "you have 5 minutes to wrap up before you leave."

---

```yaml
      securityContext:
        fsGroup: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        runAsUser: 1000
```
**WHAT:** POD-LEVEL security rules. Applied to the entire pod (all containers inside it).

| Line | Meaning | Why Important |
|---|---|---|
| `runAsNonRoot: true` | Pod cannot run as root/admin | If hacked, attacker has no admin power |
| `runAsUser: 1000` | Run as user ID 1000 (normal user) | Like a regular employee, not a superuser |
| `runAsGroup: 1000` | Run as group ID 1000 | File ownership control |
| `fsGroup: 1000` | All files/volumes owned by group 1000 | Pod can read/write its own files |

**Interview Answer:**
> *"We set pod-level security context to enforce non-root execution. The container runs as user ID 1000 — a non-privileged user. This means even if someone exploits the container, they cannot perform root-level operations on the host node. This is a Kubernetes security best practice and follows the principle of least privilege."*

---

```yaml
      containers:
      - name: server
```
**WHAT:** List of containers inside this pod. This pod has ONE container named "server".
**Note:** A pod CAN have multiple containers (sidecar pattern). But most services here have just one.

---

```yaml
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: true
```
**WHAT:** CONTAINER-LEVEL security rules. More specific than pod-level.

| Line | Meaning | Interview Point |
|---|---|---|
| `allowPrivilegeEscalation: false` | Cannot gain more power than it started with | Prevents sudo attacks |
| `capabilities.drop: [ALL]` | Removes ALL Linux special powers | No network sniffing, no raw sockets |
| `privileged: false` | Not a privileged container | Cannot access host devices |
| `readOnlyRootFilesystem: true` | Cannot write to its own disk | Malware cannot write files |

**Interview Answer:**
> *"At container level, we drop all Linux capabilities and set the root filesystem to read-only. This means even if the container is compromised, an attacker cannot install tools, write scripts, or modify system files inside the container. Combined with non-root execution, this significantly reduces the attack surface."*

---

```yaml
        image: manojkrishnappa/adservice:09c365a668171f643477a72b08bccfc4da6fcbdf
```
**WHAT:** The Docker image to run. Format is `dockerhub-username/image-name:tag`
- `manojkrishnappa` = DockerHub username (the coach's account)
- `adservice` = image name
- `09c365a668171f643477a72b08bccfc4da6fcbdf` = the exact version (Git commit SHA)

**WHY use commit SHA as tag?**
- Using `latest` tag is dangerous — you don't know which version is running
- Using commit SHA = 100% reproducible — you always know exact version
- This is a production best practice called **image immutability**

**Interview Answer:**
> *"We use Git commit SHAs as image tags instead of 'latest'. This gives us complete reproducibility — every deployment is pinned to an exact version. If something breaks, we know exactly which code is running. 'Latest' tag is considered an anti-pattern in production because it's not reproducible."*

---

```yaml
        ports:
        - containerPort: 9555
```
**WHAT:** The port the container listens on INSIDE the pod.
**Important:** This is just documentation for Kubernetes. The container actually needs to listen on this port in its code. Kubernetes doesn't enforce it — it's informational.

---

```yaml
        env:
        - name: PORT
          value: "9555"
```
**WHAT:** Environment variables injected into the container at startup.
**WHY:** The adservice application reads the PORT environment variable to know which port to listen on. This makes the container configurable without rebuilding the image.
**Analogy:** Like telling an employee "your desk extension number is 9555."

---

```yaml
        resources:
          requests:
            cpu: 200m
            memory: 180Mi
          limits:
            cpu: 300m
            memory: 300Mi
```
**WHAT:** CPU and memory boundaries for this container.

| Term | Meaning | Analogy |
|---|---|---|
| `requests.cpu: 200m` | Needs at least 0.2 CPU cores | "I need at least 2 hours of meeting room time" |
| `requests.memory: 180Mi` | Needs at least 180 MB RAM | "I need at least 180MB of desk space" |
| `limits.cpu: 300m` | Cannot use more than 0.3 CPU | "You cannot book more than 3 hours" |
| `limits.memory: 300Mi` | Cannot use more than 300MB | "You cannot take more than 300MB of space" |

**What is 200m CPU?**
- 1 CPU core = 1000 millicores (m)
- 200m = 0.2 of one CPU core
- If your machine has 4 cores = 4000m total

**WHY both requests AND limits?**
- `requests` = used for SCHEDULING — Kubernetes finds a node with enough free resources
- `limits` = used for ENFORCEMENT — container gets killed (OOMKilled) if it exceeds memory limit

**Interview Answer:**
> *"We define resource requests and limits for every container. Requests tell the Kubernetes scheduler how much resource the pod needs — the scheduler only places the pod on a node that has enough free capacity. Limits enforce a ceiling — if a container exceeds its memory limit, Kubernetes kills it with an OOMKilled event. This prevents one misbehaving service from stealing resources from others — the Noisy Neighbor problem."*

---

```yaml
        readinessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          grpc:
            port: 9555
        livenessProbe:
          initialDelaySeconds: 20
          periodSeconds: 15
          grpc:
            port: 9555
```
**WHAT:** Health checks that Kubernetes runs automatically.

**Two types of probes:**

| Probe | Question It Asks | What Happens on Failure |
|---|---|---|
| `readinessProbe` | "Is this pod READY to receive traffic?" | Pod is removed from Service endpoints. No traffic sent. |
| `livenessProbe` | "Is this pod ALIVE?" | Pod is KILLED and restarted automatically. |

**Parameters explained:**
- `initialDelaySeconds: 20` = Wait 20 seconds after container starts before first check (app needs time to boot)
- `periodSeconds: 15` = Check every 15 seconds
- `grpc: port: 9555` = Make a gRPC health check call to port 9555

**For adservice (Java/gRPC):** Uses gRPC health check protocol
**For frontend (HTTP):** Uses HTTP GET to `/_healthz`
**For authservice (HTTP):** Uses HTTP GET to `/health`
**For Redis:** Uses TCP socket check on port 6379

**Interview Answer:**
> *"We implement both readiness and liveness probes on every service. The readiness probe ensures traffic is only sent to pods that are fully initialized — a pod that is starting up but not ready won't receive requests. The liveness probe detects if a pod has entered a deadlock or broken state and automatically restarts it. The initialDelaySeconds gives the application time to complete startup before health checks begin."*

---

### authservice/deployment.yaml — Special Points

**Point 1: Secret as Environment Variable**
```yaml
# adservice uses plain env value:
- name: PORT
  value: "9555"

# emailservice uses SECRET:
- name: GMAIL_ADDRESS
  valueFrom:
    secretKeyRef:
      name: emailservice-secret    # ← name of the Kubernetes Secret
      key: GMAIL_ADDRESS           # ← key inside that Secret
```
**WHY:** Gmail password must NEVER be written as plain text in YAML. It's stored in a Kubernetes Secret and only injected at runtime.

**Point 2: ServiceAccount with AWS IAM (IRSA)**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: authservice-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::508262720940:role/AuthServiceRole
```
**WHAT:** This ServiceAccount has an AWS IAM Role attached to it.
**WHY:** AuthService needs to connect to AWS RDS. Instead of hardcoding AWS credentials, we use IRSA — IAM Roles for Service Accounts.
**How it works:**
- The pod gets a temporary AWS token automatically
- It can access RDS without any hardcoded username/password
- Token refreshes automatically

**Interview Answer:**
> *"We used IRSA — IAM Roles for Service Accounts — for the authservice. This eliminates the need to store AWS credentials in the pod. The Kubernetes ServiceAccount is annotated with an IAM Role ARN. When the pod starts, it automatically receives a temporary AWS token from the metadata service. This follows AWS security best practices — no long-lived credentials, automatic rotation."*

---

### emailservice — The Port Mismatch (Important Interview Trap!)

```yaml
# Deployment — container listens on 8080
ports:
- containerPort: 8080

# Service — exposes port 5000, routes to 8080
ports:
- name: grpc
  port: 5000        # ← CheckoutService calls emailservice:5000
  targetPort: 8080  # ← but traffic actually goes to container port 8080
```

**WHY this exists:** The email service application code was written to listen on 8080. But for logical grouping/versioning reasons, the team decided to expose it externally on 5000. Kubernetes Service handles the translation.

**How Kubernetes Service port mapping works:**
```
CheckoutService calls: emailservice:5000
      ↓ Kubernetes Service receives on port 5000
      ↓ Kubernetes forwards to: container port 8080
      ↓ Email service container receives on 8080
```

**Interview Trap:**
Interviewer: "CheckoutService connects to emailservice:5000 but the container listens on 8080. Is this a bug?"
Your Answer: *"No, this is intentional. Kubernetes Service acts as a proxy. The Service's `port` (5000) is what other services use to connect. The `targetPort` (8080) is where the actual container listens. The Service translates between them. This is a common pattern when you want to expose a service on a different port than the application uses internally."*

---

### shoppingassistantservice — envFrom (Bulk Secret Loading)

```yaml
# Other services load secrets one by one:
env:
- name: GMAIL_ADDRESS
  valueFrom:
    secretKeyRef:
      name: emailservice-secret
      key: GMAIL_ADDRESS

# ShoppingAssistant loads ALL secrets at once:
envFrom:
- secretRef:
    name: shopping-assistant-secrets
```
**WHAT:** `envFrom` loads EVERY key from a Secret as environment variables in one go.
**WHY:** ShoppingAssistant needs 4+ secrets (OPENAI_API_KEY, LANGCHAIN_API_KEY, DATABASE_URL, COLLECTION_NAME). Writing each one separately is verbose. `envFrom` is cleaner.

---

### vectordb — ConfigMap for Database Init

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vectordb-init-scripts
data:
  init.sql: |
    CREATE EXTENSION IF NOT EXISTS vector;
    CREATE TABLE IF NOT EXISTS langchain_pg_collection (...);
    CREATE TABLE IF NOT EXISTS langchain_pg_embedding (...);
```
**WHAT:** A ConfigMap stores non-sensitive configuration data. Here it stores SQL scripts.
**WHY:** When PostgreSQL starts for the first time, it automatically runs scripts in `/docker-entrypoint-initdb.d/`. We mount this ConfigMap there.

```yaml
volumeMounts:
- name: init-scripts
  mountPath: /docker-entrypoint-initdb.d   # ← PostgreSQL reads from here on first boot
volumes:
- name: init-scripts
  configMap:
    name: vectordb-init-scripts            # ← Mount our ConfigMap here
```

**What the SQL does:**
- `CREATE EXTENSION IF NOT EXISTS vector` → Enables pgvector (the AI vector math extension)
- Creates tables for LangChain to store product embeddings

**Interview Answer:**
> *"We use a ConfigMap to store database initialization SQL scripts. The ConfigMap is mounted as a volume at /docker-entrypoint-initdb.d — PostgreSQL's standard location for init scripts. On first startup, PostgreSQL automatically runs these scripts, which enables the pgvector extension and creates the tables LangChain needs for vector similarity search."*

---

## 📄 PART 2 — service.yaml Explained Line by Line

### Three Service Types Used in This Project

```
ClusterIP    → Internal only. Only pods inside cluster can reach it.
LoadBalancer → External. Gets a public IP from cloud provider.
(Ingress     → External via HTTP/HTTPS routing. Not a Service type but works with Services.)
```

---

### Type 1: ClusterIP (Used by ALL backend services)

```yaml
# adservice/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: adservice
  labels:
    app: adservice
spec:
  type: ClusterIP        # ← Internal only
  selector:
    app: adservice       # ← Find pods with label app=adservice
  ports:
  - name: grpc
    port: 9555           # ← Port this Service listens on (what callers use)
    targetPort: 9555     # ← Port on the actual pod
```

**How ClusterIP works:**
```
Frontend pod wants to talk to adservice
Frontend calls: adservice:9555
     ↓
Kubernetes DNS resolves "adservice" to a virtual IP (ClusterIP)
     ↓
Kubernetes routes to one of the adservice pods
     ↓
Pod receives on port 9555
```

**Why ClusterIP for backend services?**
- They should NOT be accessible from the internet
- Only other services inside the cluster need them
- Security: least exposure

---

### Type 2: LoadBalancer (Used ONLY for frontend)

```yaml
# frontend/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer     # ← Gets a real public IP from cloud
  selector:
    app: frontend
  ports:
  - name: http
    port: 80             # ← Public port (what internet users hit)
    targetPort: 8080     # ← Container port
```

**How LoadBalancer works:**
```
User types: www.myshop.com (resolves to public IP)
     ↓
Cloud LoadBalancer (AWS ELB / GCP LB) receives on port 80
     ↓
Forwards to: frontend pod port 8080
```

**In the project there's ALSO an Ingress — why both?**
- The LoadBalancer Service gives us the external IP entry point
- The Ingress adds routing rules on top (domain-based, path-based, HTTPS)
- The Ingress Controller (NGINX/Traefik) itself is exposed via a LoadBalancer Service

---

## 📄 PART 3 — Ingress + ClusterIssuer Explained

### ingress.yaml — Line by Line

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```
- `ssl-redirect: "true"` → Automatically redirect HTTP → HTTPS
- `cert-manager.io/cluster-issuer` → Tell cert-manager to get SSL certificate using "letsencrypt-prod"

```yaml
spec:
  ingressClassName: nginx    # ← Use NGINX Ingress Controller
  tls:
  - hosts:
    - itkannadigaru.in       # ← Domain to secure
    secretName: frontend-tls # ← Store the SSL cert in this Secret
  rules:
  - host: itkannadigaru.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80     # ← Route all traffic to frontend Service port 80
```

**How it works:**
```
User visits: https://itkannadigaru.in
     ↓
NGINX Ingress Controller receives (port 443)
     ↓
Checks TLS certificate (stored in frontend-tls Secret)
     ↓
Decrypts HTTPS → HTTP internally
     ↓
Routes "/" path to frontend Service port 80
     ↓
Frontend Service routes to frontend pod port 8080
```

---

### clusterissuer.yaml — Getting FREE HTTPS Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory  # ← Let's Encrypt server
    email: manojdevopstest@gmail.com                         # ← For certificate notifications
    privateKeySecretRef:
      name: letsencrypt-prod-account-key                     # ← Store account key here
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx                            # ← Use NGINX to solve challenge
```

**How Let's Encrypt certificate works (simple):**
```
Step 1: cert-manager asks Let's Encrypt: "Give me certificate for itkannadigaru.in"
Step 2: Let's Encrypt says: "Prove you own that domain"
Step 3: cert-manager creates a temporary file at: http://itkannadigaru.in/.well-known/acme-challenge/
Step 4: Let's Encrypt checks that file exists (HTTP-01 challenge)
Step 5: If file found → domain ownership proven → certificate issued FREE
Step 6: Certificate stored in frontend-tls Secret
Step 7: NGINX uses this certificate for HTTPS
```

**Interview Answer:**
> *"We used cert-manager with Let's Encrypt for automatic TLS certificate provisioning. cert-manager runs as a controller inside the cluster. When it sees the Ingress annotation pointing to our ClusterIssuer, it automatically requests a certificate from Let's Encrypt using the HTTP-01 ACME challenge. cert-manager creates the challenge file, Let's Encrypt validates domain ownership, and the certificate is automatically stored as a Kubernetes Secret. cert-manager also handles automatic renewal 30 days before expiry."*

---

## 🔄 PART 4 — TRAEFIK Instead of NGINX (Modern Approach)

### WHY We Switch to Traefik

The project currently uses `ingressClassName: nginx` — this refers to the community `kubernetes/ingress-nginx` which was **retired in March 2026**.

For your practice and interview, we use **Traefik** as the replacement.

| Feature | NGINX (retired community) | Traefik |
|---|---|---|
| Status | ❌ Retired March 2026 | ✅ Actively maintained |
| Dashboard | No built-in dashboard | ✅ Beautiful built-in dashboard |
| Auto-discovery | Manual config | ✅ Auto-discovers Kubernetes services |
| Let's Encrypt | Via cert-manager | ✅ Built-in Let's Encrypt support |
| Learning curve | Medium | Easy |
| MicroK8s support | Available | ✅ Available as addon |

### How to Enable Traefik on MicroK8s

```bash
# Enable Traefik in MicroK8s
microk8s enable community
microk8s enable traefik

# Verify Traefik is running
microk8s kubectl get pods -n traefik
```

### Updated ingress.yaml for Traefik

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    # cert-manager still works with Traefik!
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: traefik    # ← Changed from nginx to traefik
  tls:
  - hosts:
    - itkannadigaru.in
    secretName: frontend-tls
  rules:
  - host: itkannadigaru.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

**Only TWO changes from NGINX to Traefik:**
1. `ingressClassName: nginx` → `ingressClassName: traefik`
2. Replace nginx annotations with traefik annotations

**Interview Answer:**
> *"The community ingress-nginx controller was retired in March 2026. For our project, we migrated to Traefik as the Ingress Controller. Traefik is actively maintained, has native Kubernetes support, built-in Let's Encrypt integration, and provides a dashboard for traffic visibility. The migration only required changing the ingressClassName in our Ingress manifests from 'nginx' to 'traefik' and updating the annotations — our cert-manager ClusterIssuer and TLS configuration remained unchanged."*

---

## 📄 PART 5 — kustomization.yaml Explained

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - adservice/deployment.yaml
  - adservice/service.yaml
  - authservice/deployment.yaml
  - authservice/service.yaml
  - cartservice/deployment.yaml
  - cartservice/service.yaml
  ... (all services)
```

**WHAT:** This is the MASTER LIST. It tells Kustomize: "When someone runs `kubectl apply -k base/`, apply ALL these files."

**WHY not just `kubectl apply -f` on each file?**
- 28+ files = 28+ commands manually. Error-prone.
- `kubectl apply -k base/` = ONE command applies ALL 28 files
- Kustomize also allows overlays on top (Day 3 topic)

---

## 📄 PART 6 — No Dockerfile? Here is the Answer

### WHY No Dockerfile in This Repo

This is a **GitOps CD (Continuous Delivery) repository.** It only contains Kubernetes manifests.

The Dockerfiles live in SEPARATE application code repositories.

**Two-Repo GitOps Pattern:**

```
Repo 1: Application Code Repo (has Dockerfile)
├── src/
│   └── main.go
├── Dockerfile          ← Build image here
└── Jenkinsfile         ← CI: build, test, push image to DockerHub

Repo 2: GitOps Repo (THIS repo — only YAMLs)
├── base/
│   └── adservice/
│       ├── deployment.yaml   ← references the image built by Repo 1
│       └── service.yaml
└── overlays/
```

**The Flow:**
```
Developer pushes code to Repo 1
     ↓
Jenkins (CI) builds Docker image
     ↓
Jenkins pushes image to DockerHub
     ↓
Jenkins updates deployment.yaml in Repo 2 with new image tag
     ↓
ArgoCD detects change in Repo 2
     ↓
ArgoCD applies new deployment.yaml to cluster
     ↓
Cluster pulls new image and deploys
```

**WHY separate repos?**
- Clean separation: code changes vs deployment changes
- Different teams can own each repo
- Rollback deployment without touching code
- Security: CI server only has write access to GitOps repo, not production cluster

---

### HOW to Practice WITHOUT Dockerfile

You have 3 options for practice:

**Option A — Use the existing images (Recommended for now)**
The images are already on DockerHub under `manojkrishnappa/`. Just apply the YAMLs and the images download automatically.
```bash
cd ~/GitOps
microk8s kubectl apply -k overlays/microk8s/
```
Everything runs. No Dockerfile needed for practice.

**Option B — Write a sample Dockerfile for one service**
We will do this on Day 5. You don't need to BUILD the image to understand it.

**Option C — Use a simple Python app**
Create a simple Flask app, write a Dockerfile for it, build it, push to your DockerHub, update a deployment.yaml to use your image, and deploy. This proves you understand the full flow.

---

## ❓ INTERVIEW QUESTIONS — Kubernetes Manifests

### Basic Questions

**Q: What is a Deployment in Kubernetes?**
> *"A Deployment is a Kubernetes object that manages a set of identical pods. It ensures the desired number of replicas is always running. If a pod crashes, the Deployment automatically creates a new one. It also handles rolling updates — replacing old pods with new ones gradually without downtime."*

**Q: What is the difference between a Pod and a Deployment?**
> *"A Pod is a single running instance of a container. If it crashes, it's gone. A Deployment is a controller that manages pods — it maintains the desired replica count, handles restarts, and manages updates. In production, you never create bare pods. You always use Deployments."*

**Q: What is the difference between ClusterIP, NodePort, and LoadBalancer?**
> *"ClusterIP is internal only — accessible only from within the cluster. We use this for all backend microservices that shouldn't be exposed to the internet. NodePort exposes the service on a static port on every node — useful for development but not production. LoadBalancer provisions a cloud load balancer with a public IP — we use this for the frontend service that needs internet access."*

**Q: What is an Ingress and why use it instead of LoadBalancer?**
> *"Without Ingress, every service that needs external access needs its own LoadBalancer, which means a separate cloud load balancer and IP address for each service — expensive. Ingress uses ONE LoadBalancer (the Ingress Controller) and routes traffic based on domain names and URL paths. One LoadBalancer, multiple services. Much more cost-effective."*

**Q: What is readinessProbe vs livenessProbe?**
> *"readinessProbe determines if a pod is ready to receive traffic. If it fails, the pod is removed from the Service endpoints — requests stop going to it. livenessProbe determines if a pod is alive. If it fails, Kubernetes kills and restarts the pod. readinessProbe protects users from bad responses. livenessProbe recovers stuck pods."*

**Q: What are resource requests and limits?**
> *"Requests are the minimum resources guaranteed to the container — used by the scheduler to find a suitable node. Limits are the maximum resources allowed — if a container exceeds its memory limit, it gets OOMKilled (Out Of Memory killed). Requests ensure the pod has what it needs. Limits prevent one pod from starving others."*

**Q: What is a ServiceAccount and why does each service have one?**
> *"A ServiceAccount is a Kubernetes identity for pods. It controls what the pod can do inside the cluster — which APIs it can call, which resources it can access. We give each service its own ServiceAccount following the principle of least privilege. This way, if one service is compromised, the attacker cannot use that service's identity to affect other services."*

**Q: What is IRSA and why does authservice use it?**
> *"IRSA — IAM Roles for Service Accounts — is an AWS feature that links a Kubernetes ServiceAccount to an AWS IAM Role. The authservice uses it to securely access AWS RDS without storing any AWS credentials in the pod or environment variables. The pod automatically receives a temporary AWS token. This follows AWS security best practices — no long-lived credentials, automatic rotation, no secrets to manage."*

**Q: What is a ConfigMap? How is it used in vectordb?**
> *"A ConfigMap stores non-sensitive configuration data as key-value pairs. For vectordb, we store the database initialization SQL script in a ConfigMap. We then mount it as a volume at /docker-entrypoint-initdb.d — PostgreSQL's standard init script directory. On first startup, PostgreSQL runs the SQL script, which enables the pgvector extension and creates the tables needed for AI vector search."*

**Q: What is cert-manager and ClusterIssuer?**
> *"cert-manager is a Kubernetes controller that automates TLS certificate management. A ClusterIssuer is a cert-manager resource that defines HOW to get certificates — in our case, from Let's Encrypt using the HTTP-01 ACME challenge. When cert-manager sees an Ingress with the cert-manager annotation, it automatically requests, provisions, and renews TLS certificates. No manual certificate management needed."*

### Scenario Questions

**Scenario 1: "Pod is in CrashLoopBackOff. What do you do?"**
> *"First, kubectl describe pod <podname> to see Events section — it shows WHY the pod is crashing. Then kubectl logs <podname> --previous to see the last logs before crash. Common causes: wrong image tag, missing environment variable, missing Secret, container running as root but security context says non-root. Fix the root cause in the YAML, push to Git, let ArgoCD apply."*

**Scenario 2: "Pod is Running but traffic is not reaching it."**
> *"Check readinessProbe — if it's failing, the pod is Running but marked NotReady, so Service removes it from endpoints. kubectl describe pod and look at readiness probe status. Also kubectl get endpoints <servicename> — if empty, no pods are ready. Fix could be: wrong probe path, wrong port, app not fully started yet (increase initialDelaySeconds)."*

**Scenario 3: "A new service deployment is causing all pods to OOMKilled."**
> *"OOMKilled means the container exceeded its memory limit. Check kubectl describe pod <podname> — it will show OOMKilled in Last State. Solution: increase the memory limit in deployment.yaml resources section. Also check if there's a memory leak in the application. For immediate fix: increase limits. Long term: profile the application memory usage."*

**Scenario 4: "How do you update an image tag in production (GitOps way)?"**
> *"I never touch the cluster directly. I update the image tag in deployment.yaml in the GitOps repository, commit and push to main branch. ArgoCD detects the change within 3 minutes (or immediately if webhook is configured) and applies the new deployment. Kubernetes does a rolling update — gradually replacing old pods with new ones, maintaining zero downtime."*

**Scenario 5: "How do you prove this project is production-grade?"**
> *"Several indicators: All containers run non-root with read-only filesystems and dropped capabilities. Resource requests and limits are defined on every container. Readiness and liveness probes ensure only healthy pods receive traffic. Secrets are managed via Kubernetes Secrets, never hardcoded. IRSA is used for AWS access — no static credentials. TLS is enforced via cert-manager and Let's Encrypt. GitOps with ArgoCD ensures cluster state always matches Git. Each service has its own ServiceAccount following principle of least privilege."*

---

## 🗣️ "Walk Me Through the Kubernetes Setup" — Interview Script

> *"Our Kubernetes setup follows a layered approach. At the base layer, each of the 14 microservices has a Deployment and a Service. The Deployment defines how many replicas to run, which Docker image to use, resource limits, health checks, and security context. All containers run as non-root user with read-only root filesystem and all Linux capabilities dropped.*
>
> *For networking, backend services use ClusterIP — they're only accessible within the cluster. The frontend uses a LoadBalancer Service for external access. On top of that, we have an Ingress managed by Traefik — it handles domain-based routing and TLS termination. TLS certificates are automatically provisioned by cert-manager from Let's Encrypt.*
>
> *Secrets like API keys and database passwords are never hardcoded. They're stored as Kubernetes Secrets and injected into containers at runtime via secretKeyRef. The authservice uses IRSA for AWS RDS access — no static credentials anywhere.*
>
> *The entire deployment is managed through Kustomize. The base/kustomization.yaml lists all 28+ manifest files. A single `kubectl apply -k base/` applies everything. Environment-specific differences live in overlays — MicroK8s for development, EKS for production."*

---

## 📝 Practice Commands — Run on MicroK8s Today

```bash
# 1. Apply the full project
cd ~/GitOps
microk8s kubectl apply -k overlays/microk8s/

# 2. Check all pods are running
microk8s kubectl get pods

# 3. Check all services
microk8s kubectl get services

# 4. Check a specific deployment
microk8s kubectl describe deployment adservice

# 5. Check pod security
microk8s kubectl get pod <adservice-pod-name> -o yaml | grep -A 10 securityContext

# 6. Check resource usage
microk8s kubectl top pods

# 7. Check logs of a service
microk8s kubectl logs -l app=adservice

# 8. Check endpoints (is service routing correctly?)
microk8s kubectl get endpoints

# 9. Check events (great for debugging)
microk8s kubectl get events --sort-by='.lastTimestamp'

# 10. Describe a service
microk8s kubectl describe service frontend
```

---

## 🗓️ What's Next

| Day | Topic |
|---|---|
| Day 2 ✅ | K8s Manifests — deployment.yaml, service.yaml, ingress, ClusterIssuer |
| Day 3 | Kustomize — overlays/microk8s vs overlays/eks line by line |
| Day 4 | ArgoCD — argocd-app.yaml, App of Apps, sync/prune/self-heal |
| Day 5 | Dockerfile — write from scratch for Python, Go, Node services |
| Day 6 | Jenkinsfile — full CI pipeline from scratch |

---

*Save this file to: `Interview_Preparation_2026/gitops_project/Day2_K8s_Manifests.md`*
*Read tonight. Practice the interview answers OUT LOUD.*
*Tomorrow: Day 3 — Kustomize overlays deep dive.*
