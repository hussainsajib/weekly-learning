# Week 21 — GCP Architecture & Core Services

**Week of:** October 26, 2026
**Estimated study time:** ~2 hours
**Tags:** `gcp` `cloud` `infrastructure`

---

## Overview

Google Cloud Platform organizes everything under a strict resource hierarchy — Organization → Folders → Projects → Resources. This hierarchy is not cosmetic; it is the foundation of every IAM binding, billing boundary, and network security decision you make. Understanding it deeply means you can reason about *why* a permission works (or doesn't), *where* a policy should live to minimize blast radius, and *how* your infrastructure scales without becoming a security audit nightmare.

For the integration platform, this matters concretely. The middleware, ETL containers, and BigQuery datasets all live in distinct GCP projects under the same organization. A misconfigured folder-level IAM binding can silently grant a service account in the BDE ETL project read access to production secrets — something that would sail past a code review entirely. The hierarchy is your first line of defense.

This week covers the full GCP architecture picture: resource hierarchy, networking (VPC, Cloud NAT, firewall rules), IAM and service accounts, compute options (Cloud Run vs. GKE vs. Compute Engine), data and messaging services (Cloud SQL, Memorystore, Pub/Sub), Workload Identity Federation, and the Vault vs. Secrets Manager trade-off. These are not independent topics — they form a single integrated system, and the goal is to understand how they compose.

By the end of this guide you should be able to justify every GCP architectural decision in the integration platform's stack: why GKE over Cloud Run for the middleware, why Workload Identity Federation over key-based service accounts, and why Vault sits alongside Secrets Manager rather than replacing it. These are staff-level conversations, and fluency here is a forcing function for moving from "I can deploy it" to "I can design it."

---

## 1. Resource Hierarchy: Org, Folders, Projects

The GCP resource hierarchy has four levels: **Organization** (your Google Workspace/Cloud Identity domain), **Folders** (groupings — typically by environment or business unit), **Projects** (the billing and API-enablement boundary), and **Resources** (VMs, buckets, Cloud SQL instances, etc.).

IAM policies are **additive and inherited downward**. A binding at the folder level grants that role to every project in the folder. There is no deny — only grant (unless you use IAM Deny policies, a newer feature). This means the safest place to put a binding is as low as possible: at the project or resource level.

```
Organization: the-company.com
├── Folder: Integration Platform Production
│   ├── Project: crm-middleware-prod
│   ├── Project: etl-pipeline-prod
│   └── Project: bq-analytics-prod
├── Folder: Integration Platform Staging
│   ├── Project: crm-middleware-staging
│   └── Project: etl-pipeline-staging
└── Folder: Integration Platform Dev
    ├── Project: crm-middleware-dev
    └── Project: etl-pipeline-dev
```

**gcloud: Create a folder and project**

```bash
# Create a folder under the org
gcloud resource-manager folders create \
  --display-name="Integration Platform Production" \
  --organization=123456789012

# Create a project inside the folder
gcloud projects create crm-middleware-prod \
  --folder=FOLDER_ID \
  --name="CRM Middleware Production"

# Link billing account
gcloud billing projects link crm-middleware-prod \
  --billing-account=BILLING_ACCOUNT_ID
```

**Terraform (CDKTF-style in HCL for clarity)**

```hcl
resource "google_folder" "platform_prod" {
  display_name = "Integration Platform Production"
  parent       = "organizations/123456789012"
}

resource "google_project" "middleware_prod" {
  name            = "CRM Middleware Production"
  project_id      = "crm-middleware-prod"
  folder_id       = google_folder.platform_prod.id
  billing_account = var.billing_account_id
}
```

**Common mistake:** Granting `roles/editor` at the folder level for "convenience" during initial setup and forgetting to remove it. An editor on the folder can read every secret, modify every firewall rule, and delete every Cloud SQL instance across all child projects. Always use project-scoped, resource-scoped bindings in production.

**Platform connection:** The integration platform's infra-manifests repo manages projects under the platform org. Keeping ETL and middleware in separate projects means a compromised ETL service account cannot touch middleware Cloud SQL credentials.

---

## 2. VPC, Cloud NAT, and Firewall Rules

A **VPC (Virtual Private Cloud)** in GCP is a global resource — one VPC can span all regions, and subnets are regional. This is different from AWS where VPCs are regional. GCP VPCs use **software-defined networking**: there are no hardware network appliances; rules are enforced at the VM/node level by the hypervisor.

**Subnets** are where you assign IP ranges. Use secondary ranges on GKE node subnets to carve out pod and service CIDRs without consuming primary address space.

```bash
# Create a VPC with a subnet for GKE
gcloud compute networks create platform-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

gcloud compute networks subnets create platform-gke-subnet \
  --network=platform-vpc \
  --region=us-east1 \
  --range=10.0.0.0/20 \
  --secondary-range=pods=10.4.0.0/14,services=10.0.16.0/20
```

**Cloud NAT** provides outbound internet access for resources without public IPs (GKE nodes, Cloud SQL private IPs). It is regional and attached to a Cloud Router.

```bash
gcloud compute routers create platform-router \
  --network=platform-vpc \
  --region=us-east1

gcloud compute routers nats create platform-nat \
  --router=platform-router \
  --region=us-east1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

**Firewall rules** in GCP are network-level (not instance-level). They apply to all instances in the VPC matching the target. Use **network tags** or **service accounts** as targets — service account targeting is more precise and doesn't rely on humans remembering to tag VMs.

```bash
# Allow GKE pods to reach Cloud SQL on port 5432
gcloud compute firewall-rules create allow-gke-to-cloudsql \
  --network=platform-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --source-service-accounts=gke-workload@crm-middleware-prod.iam.gserviceaccount.com \
  --target-tags=cloudsql-proxy \
  --rules=tcp:5432 \
  --action=ALLOW
```

**Common mistake:** Leaving the default `allow-internal` firewall rule (which allows all traffic between all instances in the VPC on all ports) in place for production. Audit and tighten ingress/egress rules to explicit allow lists.

**Platform connection:** The EHR system backend (EHR server on `ehr-server-prod`) is reached over VPC peering or VPN from the GKE cluster. Firewall rules on the platform VPC must explicitly allow egress on the EHR API port to the EHR server's IP range. Cloud NAT handles the return path for services without public IPs.

---

## 3. IAM and Service Accounts — Least Privilege

GCP IAM is **role-based**. Roles bundle permissions. There are three types: **Basic** (Owner/Editor/Viewer — never use in prod), **Predefined** (Google-managed, e.g. `roles/cloudsql.client`), and **Custom** (you define the exact permissions).

**Service accounts** are identities for workloads, not humans. A service account is both a principal (something that *has* permissions) and a resource (something that can *be* granted access to — the "act as" permission).

```bash
# Create a least-privilege service account for the middleware
gcloud iam service-accounts create platform-middleware-sa \
  --display-name="CRM Middleware Service Account" \
  --project=crm-middleware-prod

# Grant only Cloud SQL client access (not admin)
gcloud projects add-iam-policy-binding crm-middleware-prod \
  --member="serviceAccount:platform-middleware-sa@crm-middleware-prod.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Grant Pub/Sub publish access on a specific topic (resource-level, not project-level)
gcloud pubsub topics add-iam-policy-binding platform-sync-events \
  --project=crm-middleware-prod \
  --member="serviceAccount:platform-middleware-sa@crm-middleware-prod.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
```

**IAM conditions** let you add time-based or resource-based constraints to bindings — useful for temporary elevated access or restricting access to specific resource names.

```hcl
resource "google_project_iam_member" "middleware_sql" {
  project = "crm-middleware-prod"
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.middleware.email}"

  condition {
    title       = "only-prod-sql-instance"
    description = "Restrict to the production Cloud SQL instance"
    expression  = "resource.name == 'projects/crm-middleware-prod/instances/platform-postgres-prod'"
  }
}
```

**Common mistake:** Binding `roles/iam.serviceAccountUser` on a project rather than on a specific service account. This allows the grantee to impersonate *every* service account in the project, not just the intended one.

**Platform connection:** The middleware service account needs `roles/cloudsql.client` to authenticate via the Cloud SQL Auth Proxy. The ETL service account needs `roles/bigquery.dataEditor` on specific datasets, not the whole project. Review these bindings quarterly.

---

## 4. Cloud Run vs. GKE vs. Compute Engine

Choosing the right compute primitive is an architecture decision, not an ops decision. Here is how they compare for integration platform-style workloads:

| Dimension | Cloud Run | GKE | Compute Engine |
|---|---|---|---|
| Unit of deployment | Container (request-driven) | Pod (always-on or HPA) | VM |
| Scaling | 0 → N on requests, per-container | Node pool autoscaler + HPA | Managed instance groups |
| Networking | Serverless VPC connector or Direct VPC egress | Full VPC, native pod IPs | Full VPC |
| Workload Identity | Yes (via service account annotation) | Yes (via WIF) | Yes (VM SA) |
| Cold start | Yes (mitigated by min-instances) | No (pods stay warm) | No |
| Best for | Stateless APIs, event-driven, low-traffic | Long-running services, stateful workloads, complex networking | Legacy apps, GPU, specific OS requirements |
| Cost model | Per request + CPU/memory when active | Node VMs always running | VM always running |

For the integration platform's crm-middleware (FastAPI, long-running, Pub/Sub consumer, Alembic migrations): **GKE is the right choice**. Cloud Run would work for the API surface but struggles with background workers that consume Pub/Sub without a triggering HTTP request (Cloud Run scales to 0 and kills consumers). GKE keeps pods alive for the pull subscriber loop.

```yaml
# GKE Deployment for middleware
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crm-middleware
  namespace: platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: crm-middleware
  template:
    metadata:
      labels:
        app: crm-middleware
      annotations:
        # Workload Identity annotation — maps k8s SA to GCP SA
        iam.gke.io/gcp-service-account: platform-middleware-sa@crm-middleware-prod.iam.gserviceaccount.com
    spec:
      serviceAccountName: platform-middleware-ksa
      containers:
        - name: middleware
          image: us-east1-docker.pkg.dev/crm-middleware-prod/platform/middleware:latest
          ports:
            - containerPort: 8000
```

**Common mistake:** Using Cloud Run for a Pub/Sub pull subscriber without setting `min-instances=1`. The service will scale to 0 between messages and miss messages queued during the cold-start window.

---

## 5. Cloud SQL — Configuration and Best Practices

Cloud SQL is GCP's managed relational database service. For the integration platform, it runs PostgreSQL. Key configuration decisions:

- **Private IP only** — no public IP on production instances. Connect via Cloud SQL Auth Proxy or Private Service Connect.
- **Deletion protection** — always enabled in prod to prevent accidental `terraform destroy` from deleting the DB.
- **Automated backups + PITR** — point-in-time recovery requires binary logging (for MySQL) or WAL archiving (for PostgreSQL). Enable it.
- **Maintenance window** — schedule it during off-peak hours (Sunday 2–4 AM).

```bash
gcloud sql instances create platform-postgres-prod \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-16384 \
  --region=us-east1 \
  --no-assign-ip \
  --network=projects/crm-middleware-prod/global/networks/platform-vpc \
  --enable-google-private-path \
  --backup-start-time=03:00 \
  --enable-point-in-time-recovery \
  --deletion-protection \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=2
```

**Cloud SQL Auth Proxy** is the recommended connection method from GKE. It handles IAM authentication and TLS without requiring a VPN or firewall rule to the Cloud SQL IP.

```yaml
# Sidecar pattern for Cloud SQL Auth Proxy in GKE
- name: cloud-sql-proxy
  image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.11.4
  args:
    - "--structured-logs"
    - "--port=5432"
    - "crm-middleware-prod:us-east1:platform-postgres-prod"
  securityContext:
    runAsNonRoot: true
  resources:
    requests:
      memory: "64Mi"
      cpu: "50m"
```

**Common mistake:** Connecting to Cloud SQL using the public IP with an authorized network CIDR. This works but bypasses IAM authentication, exposes the DB to the internet, and doesn't rotate credentials. Use the Auth Proxy.

**Platform connection:** The middleware's Cloud SQL connection goes through the sidecar proxy. The proxy authenticates using the GKE pod's Workload Identity (the GCP service account annotated on the Kubernetes service account). No key file is needed.

---

## 6. Memorystore and Pub/Sub

**Memorystore** is GCP's managed Redis (and Memcached). The integration platform could use it for API response caching, rate limiting, or session storage. It lives inside your VPC — no public endpoint.

```bash
gcloud redis instances create platform-cache \
  --size=2 \
  --region=us-east1 \
  --network=projects/crm-middleware-prod/global/networks/platform-vpc \
  --redis-version=redis_7_0 \
  --tier=STANDARD_HA
```

Connect from Python using `redis-py` with the private IP:

```python
import redis

r = redis.Redis(
    host="10.0.20.5",  # Memorystore private IP
    port=6379,
    decode_responses=True,
)
r.set("sync_lock:account:001XX000003GYn2", "1", ex=300)
```

**Pub/Sub** is GCP's fully managed message queue. The integration platform uses it to decouple Salesforce trigger events from the middleware processing pipeline. Key concepts:

- **Topics** — logical channels (e.g., `platform-sync-events`)
- **Subscriptions** — consumers pull from or GCP pushes to (e.g., `platform-middleware-sub`)
- **Acknowledgment deadline** — if a message isn't acked within the deadline (default 10s, max 600s), it's redelivered
- **Dead-letter topics** — messages that exceed `max_delivery_attempts` are forwarded here for inspection

```bash
gcloud pubsub topics create platform-sync-events --project=crm-middleware-prod

gcloud pubsub subscriptions create platform-middleware-sub \
  --topic=platform-sync-events \
  --ack-deadline=60 \
  --max-delivery-attempts=5 \
  --dead-letter-topic=projects/crm-middleware-prod/topics/platform-sync-dlq \
  --project=crm-middleware-prod
```

**Common mistake:** Not setting a dead-letter topic. A message that causes a processing exception will be retried indefinitely until the retention period expires (7 days default), clogging the queue and potentially causing out-of-order processing for subsequent messages.

---

## 7. Workload Identity Federation

**Workload Identity Federation (WIF)** allows workloads running *outside* GCP (GitHub Actions, on-prem, other clouds) to authenticate as GCP service accounts without a service account key file. For workloads *inside* GKE, the analogous feature is **GKE Workload Identity**.

**GKE Workload Identity** binds a Kubernetes ServiceAccount (KSA) to a GCP ServiceAccount (GSA). The GKE metadata server intercepts token requests from pods and returns short-lived GCP tokens for the mapped GSA.

```bash
# Enable Workload Identity on the GKE cluster
gcloud container clusters update platform-cluster \
  --workload-pool=crm-middleware-prod.svc.id.goog \
  --region=us-east1

# Create the Kubernetes ServiceAccount
kubectl create serviceaccount platform-middleware-ksa \
  --namespace=platform

# Bind the KSA to the GSA
gcloud iam service-accounts add-iam-policy-binding \
  platform-middleware-sa@crm-middleware-prod.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:crm-middleware-prod.svc.id.goog[platform/platform-middleware-ksa]"

# Annotate the KSA
kubectl annotate serviceaccount platform-middleware-ksa \
  --namespace=platform \
  iam.gke.io/gcp-service-account=platform-middleware-sa@crm-middleware-prod.iam.gserviceaccount.com
```

**For GitHub Actions (WIF to GCP):**

```yaml
# .github/workflows/deploy.yml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
    service_account: "github-deploy-sa@crm-middleware-prod.iam.gserviceaccount.com"
```

```bash
# Set up the WIF pool and provider
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --project=crm-middleware-prod

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri=https://token.actions.githubusercontent.com \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --project=crm-middleware-prod
```

**Common mistake:** Creating service account keys instead of using Workload Identity, then committing the key file to the repo. Service account keys never expire by default, are hard to rotate, and represent a persistent credential. WIF tokens are short-lived (1 hour) and auto-rotated.

**Platform connection:** Every GKE workload (middleware, ETL sidecar) should use Workload Identity to access Cloud SQL, Pub/Sub, and Secrets Manager. No JSON key files should exist in the integration platform's Kubernetes secrets. The infra-manifests repo should enforce this in Helm chart values.

---

## 8. Secrets Manager vs. Vault

The integration platform uses both Vault (HashiCorp) and GCP Secrets Manager. Understanding the trade-off helps you decide where a secret should live.

| Dimension | GCP Secrets Manager | HashiCorp Vault |
|---|---|---|
| Auth model | GCP IAM (service accounts, WIF) | Multiple: AppRole, Kubernetes, LDAP, GCP, JWT |
| Secret types | Opaque binary/string, versioned | KV, database credentials (dynamic), PKI, transit encryption |
| Dynamic secrets | No | Yes — generates per-request DB credentials with TTL |
| Audit log | Cloud Audit Logs | Vault audit log (separate, configurable) |
| Cross-cloud | GCP only | Multi-cloud, on-prem |
| Operator overhead | None (managed) | Requires cluster management, HA, unsealing |
| Kubernetes integration | Secrets Store CSI Driver or env injection | Vault Agent Injector or VSO (Vault Secrets Operator) |
| Cost | Per secret version + access | Self-hosted infra cost |

**When to use Secrets Manager:** GCP-native secrets (Cloud SQL passwords, API keys for GCP services), secrets accessed by Cloud Run or Cloud Functions, simple key-value secrets with no rotation requirement.

**When to use Vault:** Dynamic database credentials (Vault generates a unique user/password per pod, auto-expires), cross-cloud secrets (EHR API keys used by both GKE and the on-prem ETL Pentaho jobs), PKI certificate issuance, encryption-as-a-service.

```bash
# GCP Secrets Manager: create and access a secret
echo -n "supersecretpassword" | gcloud secrets create platform-db-password \
  --data-file=- \
  --project=crm-middleware-prod \
  --replication-policy=user-managed \
  --locations=us-east1

gcloud secrets versions access latest \
  --secret=platform-db-password \
  --project=crm-middleware-prod
```

```python
# Access Secrets Manager from Python (uses ADC / Workload Identity automatically)
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = "projects/crm-middleware-prod/secrets/platform-db-password/versions/latest"
response = client.access_secret_version(request={"name": name})
password = response.payload.data.decode("utf-8")
```

**Vault dynamic DB credentials (PostgreSQL):**

```bash
# Configure Vault PostgreSQL secrets engine
vault secrets enable database

vault write database/config/platform-postgres \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@10.0.0.5:5432/platform_db" \
  allowed_roles="middleware-role" \
  username="vault-admin" \
  password="$VAULT_ADMIN_PASSWORD"

vault write database/roles/middleware-role \
  db_name=platform-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

**Common mistake:** Using Secrets Manager for secrets that should be dynamically generated (DB credentials). A static DB password in Secrets Manager means all pods share one credential — a compromised pod exposes credentials that work until someone manually rotates them. Vault's dynamic credentials limit the blast radius to one TTL window (e.g., 1 hour).

**Platform connection:** The integration platform's current stack uses Vault for secrets injected into GKE pods via the Vault Agent Injector. Secrets Manager is appropriate for secrets consumed by Cloud Run or Cloud Functions (where Vault Agent doesn't run). For Cloud SQL passwords used by the middleware, Vault's dynamic PostgreSQL credentials would be a significant security improvement over the current static password approach.

---

## 9. Putting It Together — Platform Architecture Walkthrough

Here is how all the pieces connect in the integration platform's production architecture:

```
Salesforce (Apex Trigger)
    │ HTTPS POST
    ▼
crm-middleware (GKE, platform namespace)
    │ KSA → GSA via Workload Identity
    ├── Cloud SQL Auth Proxy (sidecar) → Cloud SQL PostgreSQL (private IP, platform-vpc)
    ├── Pub/Sub publisher → platform-sync-events topic
    └── Vault Agent (sidecar) → Vault cluster (GKE, vault namespace)

etl-bde-pipeline (GKE, separate project)
    │ KSA → GSA via Workload Identity
    ├── Pub/Sub subscriber → platform-sync-events
    ├── Egress via Cloud NAT → EHR system backend (ehr-server-prod, on-prem or GCP VPC peer)
    └── Vault Agent → Vault cluster

etl-bq-pipeline (GKE, separate project)
    │ KSA → GSA via Workload Identity
    ├── BigQuery client → BigQuery datasets (IAM: roles/bigquery.dataEditor on specific datasets)
    └── Vault Agent → Vault cluster
```

Firewall rules enforce that the ETL project's service accounts can reach Cloud SQL (via Auth Proxy) and Pub/Sub (via GCP APIs) but cannot reach the middleware's internal admin endpoints. VPC Service Controls provide an additional perimeter to prevent data exfiltration from BigQuery.

---

## 10. Key Concepts Summary

```
GCP Resource Hierarchy
├── Organization (the-company.com)
│   └── IAM policies inherited by all below
├── Folders (Integration Platform Prod / Staging / Dev)
│   └── Billing and policy boundaries
├── Projects (crm-middleware-prod, etl-pipeline-prod, ...)
│   └── API enablement, service accounts, quotas
└── Resources (GKE clusters, Cloud SQL, Pub/Sub topics, ...)
    └── Resource-level IAM bindings (most granular)

Networking
├── VPC (global, custom subnet mode)
│   ├── Subnets (regional, primary + secondary ranges for GKE)
│   ├── Firewall rules (network tags or SA targets, no stateful inspection)
│   └── VPC Peering / VPN (to EHR server)
└── Cloud NAT (outbound for private instances, via Cloud Router)

IAM
├── Roles: Basic (avoid) → Predefined → Custom (most granular)
├── Service Accounts: workload identities (not humans)
│   ├── Workload Identity (GKE pod → GCP SA, no key files)
│   └── WIF (GitHub Actions / external → GCP SA)
└── Principle: bind at resource level, not project/folder level

Compute
├── Cloud Run: stateless, event-driven, scales to 0
├── GKE: long-running, complex networking, stateful
└── Compute Engine: VMs, GPU, legacy

Data & Messaging
├── Cloud SQL: managed PostgreSQL/MySQL, private IP, Auth Proxy
├── Memorystore: managed Redis, VPC-internal
└── Pub/Sub: async messaging, topics/subscriptions, dead-letter topics

Secrets
├── Secrets Manager: GCP-native, simple KV, IAM auth
└── Vault: dynamic credentials, multi-cloud, PKI, transit encryption
```

---

## Quiz — 20 Questions

### Questions

**1.** A developer binds `roles/editor` to a service account at the folder level "to give it access to all projects." What is the risk, and what is the correct approach?

**2.** You have a GKE pod that needs to write to Cloud SQL. What is the recommended authentication mechanism, and why should you avoid service account key files?

**3.** What is the difference between a Cloud NAT and a VPC firewall rule? Can one substitute for the other?

**4.** Your FastAPI middleware service on GKE consumes Pub/Sub messages in a background thread. A colleague suggests migrating it to Cloud Run to save costs. What is the key problem with this suggestion?

**5.** Explain GKE Workload Identity. What are the three resources you need to configure for it to work?

**6.** A Cloud SQL instance has a public IP and an authorized network CIDR of `0.0.0.0/0`. Describe two security problems with this configuration.

**7.** What is the purpose of a Pub/Sub dead-letter topic? How do you configure the maximum delivery attempts before a message is forwarded to it?

**8.** You need to allow a GitHub Actions workflow to deploy to GKE without storing a service account key in GitHub Secrets. What GCP feature enables this, and what is the key OIDC concept involved?

**9.** Compare Vault dynamic database credentials to a static password stored in Secrets Manager for a Cloud SQL connection. When does Vault's approach provide a concrete security advantage?

**10.** Your VPC has the default `allow-internal` firewall rule. Explain what this rule does and why it is risky in a multi-service production environment.

**11.** You have a service account with `roles/bigquery.dataEditor` bound at the project level. A new dataset containing PII is added to the project. What access does the service account automatically have, and how should you redesign the binding?

**12.** What is Private Service Connect and how does it differ from VPC Peering for connecting to a Cloud SQL instance?

**13.** A Cloud Run service needs to access Memorystore Redis. What networking feature is required, and what is the alternative introduced in newer Cloud Run versions?

**14.** Your Pub/Sub subscriber is taking 90 seconds to process some messages, but the acknowledgment deadline is 60 seconds. What happens, and how do you fix it?

**15.** You need to create a custom IAM role that allows a service account to only publish to Pub/Sub topics but not create or delete them. List the exact permissions you would include.

**16.** What is the `iam.gke.io/gcp-service-account` annotation on a Kubernetes ServiceAccount, and what must be configured on the GCP side for it to work?

**17.** Describe the resource hierarchy binding that would be required if you want the ETL service account to access BigQuery datasets in a *different project* (the BQ analytics project).

**18.** Cloud SQL PITR (Point-in-Time Recovery) is enabled but you discover the oldest available recovery point is only 3 days ago, not the expected 7 days. What are two possible causes?

**19.** You are designing a system where the on-prem Pentaho ETL job needs to write to a Cloud SQL database in the integration platform's VPC. The job runs on a server that is not in GCP. What are two connectivity options?

**20.** Explain the trade-off between `--subnet-mode=auto` and `--subnet-mode=custom` when creating a GCP VPC. Which should you use for a production GKE cluster and why?

---

### Answers

??? note "Reveal Answers"

    **1.** The risk is that `roles/editor` at the folder level grants the service account broad read/write permissions across all projects in the folder — including the ability to modify firewall rules, read all secrets, and delete resources in every project. The correct approach is to identify the minimum set of permissions the service account actually needs, create bindings at the project or resource level for each specific permission (e.g., `roles/cloudsql.client` on the specific project), and never use `roles/editor` or `roles/owner` for service accounts in production.

    **2.** The recommended mechanism is **GKE Workload Identity**: annotate the Kubernetes ServiceAccount with the GCP service account email, bind the KSA to the GSA via `roles/iam.workloadIdentityUser`, and add the Cloud SQL Auth Proxy as a sidecar. Service account key files should be avoided because they are long-lived credentials (no expiry by default), difficult to rotate across all pods, and if committed to a repo or baked into a container image they create a persistent, hard-to-remediate breach. Workload Identity tokens are short-lived (1 hour) and automatically rotated by GKE's metadata server.

    **3.** Cloud NAT handles **outbound traffic** — it allows instances without public IPs to initiate connections to the internet by translating their private IPs to a NAT IP. VPC firewall rules control **which traffic is allowed in or out** of instances based on protocol, port, source, and destination. They are complementary: a firewall rule allows a connection to leave the VPC, and Cloud NAT translates the source IP. Removing Cloud NAT does not block traffic — it just means private instances have no route to the internet. Removing a permissive egress firewall rule blocks traffic regardless of Cloud NAT.

    **4.** The key problem is that Cloud Run scales to 0 when there is no incoming HTTP traffic. A Pub/Sub pull subscriber runs as a background loop — it is not triggered by HTTP requests. When Cloud Run scales to 0, the subscriber loop stops, and messages accumulate in the queue unprocessed. You could mitigate this with `min-instances=1` (keeping one instance always running), but at that point the cost savings disappear, and you have added complexity without benefit. GKE is the right home for long-running background workers.

    **5.** GKE Workload Identity is a mechanism that maps a Kubernetes ServiceAccount (KSA) to a GCP ServiceAccount (GSA), allowing pods to authenticate as the GSA without a key file. The three required configurations are: (1) Enable the workload pool on the GKE cluster (`--workload-pool=PROJECT.svc.id.goog`); (2) Grant the KSA the `roles/iam.workloadIdentityUser` role on the GSA with the member string `serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]`; (3) Annotate the Kubernetes ServiceAccount with `iam.gke.io/gcp-service-account=GSA_EMAIL`. The pod's service account must be set to the annotated KSA in the pod spec.

    **6.** First, a public IP makes the database reachable from the internet — any vulnerability in the PostgreSQL server or its authentication mechanism is now exposed globally. Second, the authorized network `0.0.0.0/0` means *any* IP address can attempt to connect, reducing the security to only the database password as a control. The correct configuration is no public IP (private IP only), accessed via Cloud SQL Auth Proxy, which uses IAM-based authentication and TLS without opening a firewall port.

    **7.** A dead-letter topic is a Pub/Sub topic where messages are forwarded after exceeding the maximum delivery attempts. It prevents a poison-pill message from blocking the subscription indefinitely — the message is moved aside so subsequent messages can be processed. You configure it with `--dead-letter-topic=TOPIC_NAME` and `--max-delivery-attempts=N` (between 5 and 100) on the subscription. The Pub/Sub service account for the subscription's project also needs `roles/pubsub.publisher` on the dead-letter topic and `roles/pubsub.subscriber` on the dead-letter subscription.

    **8.** **Workload Identity Federation** enables GitHub Actions to authenticate as a GCP service account. The OIDC concept involved is **token exchange**: GitHub Actions generates a signed OIDC JWT from GitHub's identity provider (`token.actions.githubusercontent.com`), and GCP's WIF service validates that token against the configured OIDC provider, then issues a short-lived GCP access token for the mapped service account. You configure a Workload Identity Pool and OIDC Provider in GCP, then grant the GitHub repository's subject the `roles/iam.workloadIdentityUser` role on the target GSA.

    **9.** With static Secrets Manager credentials, all pods and all users of that DB share one username/password. If the password leaks — through logs, a compromised pod, a misconfigured env var — the attacker has persistent access until someone manually rotates the password and updates every consumer. Vault's dynamic credentials generate a *unique* username/password per request with a TTL (e.g., 1 hour). A compromised credential is useless after the TTL expires, and rotating is just "stop issuing credentials from the old root account." The concrete advantage is limiting the blast radius of a compromised pod to one TTL window rather than indefinitely.

    **10.** The default `allow-internal` firewall rule allows all TCP, UDP, and ICMP traffic between *all instances* in the VPC on *all ports*. In a multi-service production environment this means a compromised pod in the ETL project can freely connect to the middleware's internal admin port, a Cloud SQL port, Memorystore, or any other service in the VPC — all without any additional privilege escalation. The correct approach is to delete the default allow-internal rule and replace it with explicit allow rules scoped to specific source service accounts or tags and specific destination ports.

    **11.** The service account automatically gains `dataEditor` access to the new PII dataset, because the binding is at the project level and applies to all current and future datasets. The redesign is to remove the project-level binding and replace it with dataset-level bindings: `roles/bigquery.dataEditor` granted only on the specific datasets the service account legitimately needs to write. For the PII dataset, a separate, more restricted service account should be used if access is needed at all, with additional audit logging enabled.

    **12.** **VPC Peering** connects two VPCs at the network level, allowing all instances in both VPCs to communicate using internal IPs — but it is not transitive and requires non-overlapping CIDR ranges. **Private Service Connect** is a newer, more secure model where a consumer project connects to a specific service endpoint (e.g., Cloud SQL) via a private IP endpoint in their own VPC, without full network-level peering. PSC is preferred because it is uni-directional (the producer VPC cannot initiate connections back), avoids CIDR conflicts, and supports cross-project and cross-organization connectivity for managed services.

    **13.** Cloud Run requires a **Serverless VPC Access connector** (a small managed VM-based connector in your VPC) to reach VPC-internal resources like Memorystore. The newer alternative is **Direct VPC Egress**, which gives Cloud Run services an IP in your VPC subnet directly without requiring a connector VM, reducing latency and cost. You configure it with `--network` and `--subnet` flags on the Cloud Run service.

    **14.** Messages whose processing time exceeds the acknowledgment deadline are treated as unacknowledged and redelivered to another subscriber (or the same one). This means the message will be processed twice, causing duplicate side effects (duplicate writes to Cloud SQL, duplicate API calls to the EHR system). The fix is to increase the acknowledgment deadline to 300 or 600 seconds to cover the longest expected processing time, or to use **acknowledgment deadline extension** — the subscriber calls `modifyAckDeadline` during processing to extend the deadline before it expires, keeping the message checked out until processing completes.

    **15.** The custom role should include: `pubsub.topics.publish` (to publish messages), `pubsub.topics.get` (to retrieve topic metadata, required by the client library), and optionally `pubsub.snapshots.seek` if snapshots are used. It should explicitly exclude: `pubsub.topics.create`, `pubsub.topics.delete`, `pubsub.topics.update`, `pubsub.subscriptions.create`, `pubsub.subscriptions.delete`. The predefined `roles/pubsub.publisher` already scopes correctly to publish + get at the topic level and is the simpler choice unless you need to restrict further.

    **16.** The `iam.gke.io/gcp-service-account` annotation on a Kubernetes ServiceAccount tells GKE's metadata server which GCP service account this KSA maps to. When a pod using this KSA requests a token from the metadata server, the server returns a short-lived GCP access token for the annotated GSA. For this to work, the GCP side must be configured with an IAM binding: the principal `serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA_NAME]` must have `roles/iam.workloadIdentityUser` on the target GSA. Without this binding, the annotation is ignored and token requests will fail with a permission error.

    **17.** The ETL service account lives in `etl-pipeline-prod`. The BigQuery datasets live in `bq-analytics-prod`. You need a cross-project IAM binding: on the *BigQuery project* (`bq-analytics-prod`), grant `roles/bigquery.dataEditor` (or dataset-level access) to the ETL service account from `etl-pipeline-prod`. The format is `serviceAccount:etl-sa@etl-pipeline-prod.iam.gserviceaccount.com`. Dataset-level bindings are preferred over project-level to restrict access to specific datasets. The ETL project also needs `roles/bigquery.jobUser` on its own project to run BigQuery jobs.

    **18.** Two possible causes: (1) The Cloud SQL instance was recreated or restored from backup within the last 7 days — PITR log chain starts from the last full restore point, not the instance creation date. (2) The transaction log storage is full or write volume is very high, causing older log files to be purged earlier than the 7-day retention window. Check the `database/disk/bytes_used` metric in Cloud Monitoring for log growth rate, and verify the `backupRetentionSettings.retainedBackups` and `backupRetentionSettings.retentionUnit` settings on the instance.

    **19.** Two options: (1) **Cloud VPN** — establish an IPsec VPN tunnel between the on-prem network and the integration platform's VPC. The Pentaho server gets a route to the Cloud SQL private IP over the tunnel. This is straightforward but adds VPN infrastructure. (2) **Cloud SQL Auth Proxy running on the on-prem server** — the proxy authenticates via a service account key file (or WIF if the on-prem server can obtain OIDC tokens) and tunnels the connection over HTTPS to Cloud SQL's public endpoint with TLS. This avoids VPN but requires a service account key on the on-prem machine, which is a credential management concern. For the on-prem Pentaho jobs, Cloud VPN is the better long-term answer; the Auth Proxy with a key file is an acceptable temporary solution with compensating controls (key rotation, audit logging).

    **20.** `--subnet-mode=auto` creates one subnet per region automatically with pre-assigned CIDR ranges — convenient but inflexible. You cannot control the CIDR ranges, which causes problems when ranges conflict with on-prem networks or when you need secondary ranges for GKE pod/service CIDRs. `--subnet-mode=custom` gives you full control over subnet CIDRs, regions, and secondary ranges. For a production GKE cluster you must use `custom` mode: GKE requires explicitly defined secondary ranges for pod IPs (e.g., `/14`) and service IPs (e.g., `/20`), and you need to ensure those ranges do not overlap with your on-prem network (important for VPN connectivity to the EHR server). Custom mode is the production standard.
