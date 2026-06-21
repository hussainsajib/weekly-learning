# Week 10 — Salesforce Integration Patterns

**Week of:** August 10, 2026
**Estimated study time:** ~2 hours
**Tags:** `salesforce` `integration` `api`

---

## Overview

Salesforce is rarely an island. In mature enterprise deployments like the integration platform, it sits at the center of a web of integrations: inbound data from the EHR system sync jobs, outbound callouts to FastAPI middleware, bidirectional change events consumed by downstream ETL pipelines, and external systems pushing webhooks back in. Understanding the full menu of Salesforce integration patterns — and knowing when to reach for each one — is one of the clearest markers separating senior from staff-level engineers.

The CRM-EHR Integration Platform architecture embodies this complexity directly. An agent edits an `Account` in Salesforce, an Apex trigger fires, a `@future` callout hits the FastAPI middleware, the middleware enqueues a sync job, and the EHR system is updated within seconds. That single user gesture crosses three systems, two authentication boundaries, and at least two async execution contexts. Every piece of that chain has a Salesforce integration pattern behind it — REST callouts, named credentials, platform events for observability, and outbound messages for simpler one-way flows.

This week covers the building blocks: REST and SOAP APIs (authentication, limits, error handling), outbound messages vs. Apex callouts, Platform Events vs. Change Data Capture vs. Streaming API, inbound webhooks, and connected apps. You will also see how Canvas fits into the picture for embedded UIs. Throughout, the integration platform trigger→callout→middleware→EHR system pattern is used as the concrete anchor — every concept maps back to something you have already shipped.

By the end of this week you should be able to design a new integration surface on the integration platform — decide between a platform event and a callout, choose the right auth flow, implement limit-safe error handling, and explain the trade-offs to a product team in plain language. That is the staff-level move: turning pattern knowledge into architectural judgment.

---

## 1. REST and SOAP APIs: Authentication

Salesforce exposes two major API families. The REST API (JSON/HTTP) is the modern default and is what the CRM-EHR Integration Platform's FastAPI middleware uses. The SOAP API (XML/WSDL) is older but still prevalent in legacy insurance integrations — you may encounter it in EHR-adjacent systems. Both require OAuth 2.0 tokens on every request.

### OAuth 2.0 Flows in Salesforce

| Flow | Use Case | Integration Platform Relevance |
|---|---|---|
| Username-Password | Server-to-server (legacy) | Used in older ETL jobs; avoid for new work |
| JWT Bearer | Server-to-server (modern, no user) | Preferred for middleware → Salesforce calls |
| Web Server (Authorization Code) | User-delegated, interactive | Canvas apps, connected app installs |
| Device Flow | CLI/headless tools | Tooling, not production integrations |

The JWT Bearer flow is the gold standard for server-to-server. The client signs a JWT with a private key, posts it to `https://login.salesforce.com/services/oauth2/token`, and receives an access token scoped to a specific Salesforce user. No user interaction. No refresh token dance. The FastAPI middleware calling back into Salesforce (e.g., to write sync status) should use this flow.

```python
# FastAPI middleware: JWT Bearer token acquisition for Salesforce
import jwt, time, httpx
from cryptography.hazmat.primitives import serialization

def get_sf_access_token(
    consumer_key: str,
    private_key_pem: str,
    sf_username: str,
    instance_url: str = "https://login.salesforce.com",
) -> str:
    private_key = serialization.load_pem_private_key(
        private_key_pem.encode(), password=None
    )
    claim = {
        "iss": consumer_key,
        "sub": sf_username,
        "aud": instance_url,
        "exp": int(time.time()) + 300,  # 5-minute window
    }
    signed = jwt.encode(claim, private_key, algorithm="RS256")
    resp = httpx.post(
        f"{instance_url}/services/oauth2/token",
        data={"grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer", "assertion": signed},
    )
    resp.raise_for_status()
    return resp.json()["access_token"]
```

On the Apex side, Named Credentials abstract the token entirely. Instead of storing client secrets in custom settings, you point a callout at a Named Credential and let Salesforce manage token refresh transparently.

```apex
// Apex: callout using Named Credential — auth is fully delegated to Salesforce
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:CRM_Middleware/api/v2/accounts');
req.setMethod('POST');
req.setHeader('Content-Type', 'application/json');
req.setBody(JSON.serialize(payload));

HttpResponse res = new Http().send(req);
if (res.getStatusCode() != 200) {
    throw new CalloutException('Middleware error: ' + res.getStatus());
}
```

**Common mistake:** Hardcoding the instance URL (`https://na1.salesforce.com/...`) in callout endpoints. Instances migrate. Always use `https://[MyDomain].my.salesforce.com` or a Named Credential so the URL survives instance migrations automatically.

---

## 2. API Limits and Error Handling

Salesforce enforces per-org, per-user, and per-transaction limits. Ignoring them in integration code causes throttling, data loss, or silent failures that are painful to debug at 2 AM.

### Key Limit Dimensions

| Limit Type | Scope | Default (Enterprise) | Integration Platform Impact |
|---|---|---|---|
| Daily API calls | Org/24h | 1,000 × user licenses | ETL batch jobs consume this fast |
| Concurrent REST requests | Org | 25 | Bulk sync bursts can hit this |
| Callouts per transaction | Apex transaction | 100 | Trigger loops must be bulkified |
| Callout timeout | Per callout | 120 seconds | Middleware must respond promptly |
| `@future` calls per transaction | Apex | 50 | Bulk DML on Accounts can exhaust this |

The `@future` limit is especially relevant to the integration platform. When an agent does a bulk import of 200 Accounts, the trigger fires 200 times in a single transaction. If each trigger invocation calls `@future`, only the first 50 enqueue — the rest are silently dropped unless you bulkify with a single `@future` call that accepts a `List<Id>`.

```apex
// Integration platform pattern: bulkified @future callout — one HTTP call per batch, not per record
public class AccountSyncHandler {
    @future(callout=true)
    public static void syncToMiddleware(List<Id> accountIds) {
        List<Account> accounts = [
            SELECT Id, Name, APP__EHR_Client_Id__c, BillingStreet
            FROM Account
            WHERE Id IN :accountIds
        ];
        String payload = JSON.serialize(accounts);

        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:CRM_Middleware/api/v2/accounts/bulk');
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        req.setBody(payload);
        req.setTimeout(60000); // 60s; middleware must respond within 120s hard limit

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() >= 400) {
            // Log to platform event for async error tracking
            APP__Sync_Error__e evt = new APP__Sync_Error__e(
                APP__Record_Ids__c = String.join(new List<Id>(accountIds), ','),
                APP__Error_Message__c = res.getBody(),
                APP__HTTP_Status__c = res.getStatusCode()
            );
            EventBus.publish(evt);
        }
    }
}
```

**Error handling hierarchy:** (1) HTTP 4xx — client error, log and alert, do not retry without fix. (2) HTTP 429 / 503 — rate limit or transient, exponential back-off, retry in middleware queue. (3) HTTP 5xx — server error, retry with jitter, alert if persistent. (4) Callout timeout — treat as 503, retry.

**Common mistake:** Swallowing non-200 responses silently. Always publish a Platform Event or write to a custom error log object when a callout fails. Silent failures cause data divergence between Salesforce and the EHR system that takes hours to reconcile.

---

## 3. Outbound Messages vs. Apex Callouts

These are the two native mechanisms for Salesforce to push data out to an external system. They have very different operational characteristics.

### Outbound Messages

Outbound Messages are declaratively configured (Workflow Rules or Process Builder) and send a SOAP payload to a listener URL when a record is saved. Salesforce guarantees at-least-once delivery with automatic retries for up to 24 hours if the endpoint is unavailable.

- No Apex code required
- Fixed SOAP/XML format (limited field control)
- Delivery receipt required: listener must return an `<Ack>` or Salesforce retries
- Cannot make conditional logic or multi-step transformations

### Apex Callouts

Callouts are code-driven HTTP/REST calls from Apex. They offer full control over payload, headers, and logic, but require careful limit management. They are synchronous within the transaction (or async via `@future`, Queueable, or Batch).

| Dimension | Outbound Message | Apex Callout |
|---|---|---|
| Protocol | SOAP XML | Any (REST, SOAP, GraphQL) |
| Configuration | Declarative | Code |
| Retry | Automatic (24h) | Manual / queue-based |
| Limit exposure | None | Governor limits apply |
| Payload control | Low | Full |
| Integration platform usage | Not used | Primary pattern |

The integration platform uses Apex callouts exclusively because the EHR middleware API is REST/JSON and the integration logic requires field mapping, conditional routing (insert vs. update vs. delete), and error publication — none of which outbound messages can handle.

**When to use outbound messages:** Simple, low-volume, one-way notifications to a SOAP listener you do not control. Example: notifying a legacy insurance rating engine when an Opportunity closes.

**Common mistake:** Using Outbound Messages for high-volume or complex integrations and then discovering the SOAP listener and retry queue are impossible to debug. Apex callouts with explicit error logging are far more observable.

---

## 4. Platform Events vs. Change Data Capture vs. Streaming API

These three mechanisms all deliver real-time data from Salesforce to external consumers via a pub/sub model. They are often confused but serve different purposes.

### Platform Events

Developer-defined event schema (`__e` suffix). Published explicitly in Apex (`EventBus.publish()`), Flow, or via REST. Subscribers receive events within seconds. Retention: 72 hours. Replay from any position using `replayId`.

Use case: custom application events — sync errors, workflow milestones, audit signals. In the integration platform, publishing `APP__Sync_Error__e` events lets the middleware subscribe and trigger alerting without polling.

### Change Data Capture (CDC)

Salesforce-generated events fired automatically on any DML to subscribed objects. No Apex code needed. Captures full change context: `changeType` (CREATE/UPDATE/DELETE/UNDELETE), changed fields only, and header metadata. Retention: 72 hours.

Use case: replicate Salesforce record changes to an external store (e.g., BigQuery). The etl-bde-pipeline pipeline could subscribe to CDC events for `APP__Policy__c` instead of polling the REST API on a schedule.

### Streaming API (PushTopic)

Older mechanism. Defines a SOQL query; Salesforce pushes matching records when they change. Being deprecated in favor of CDC. Avoid for new work.

### Decision Matrix

| Need | Pattern |
|---|---|
| Custom event (workflow signal, error) | Platform Event |
| Replicate record changes externally | CDC |
| Legacy SOQL-based subscription | PushTopic (avoid) |
| External system publishes into Salesforce | Platform Event via REST |

```python
# FastAPI middleware subscribing to integration platform Platform Events via Salesforce Streaming API
# Uses aiosfstream (asyncio CometD client)
import asyncio
from aiosfstream import SalesforceStreamingClient

async def listen_for_sync_errors(access_token: str, instance_url: str):
    async with SalesforceStreamingClient(
        consumer_key="...",
        consumer_secret="...",
        username="...",
        password="...",
    ) as client:
        await client.subscribe("/event/APP__Sync_Error__e")
        async for message in client:
            payload = message["data"]["payload"]
            record_ids = payload["APP__Record_Ids__c"]
            error_msg = payload["APP__Error_Message__c"]
            # Trigger re-queue logic or alert
            await handle_sync_error(record_ids, error_msg)
```

**Common mistake:** Assuming CDC events contain the full record. CDC only ships changed fields plus the record ID and `changeType`. If you need unchanged field values for downstream processing, you must query Salesforce after receiving the event.

---

## 5. Inbound Webhooks to Salesforce

External systems pushing data into Salesforce — the reverse of callouts. Salesforce has no native webhook listener endpoint, so you must build one using a Site (public unauthenticated REST endpoint), a Connected App with an external-facing Apex REST class, or the Platform Events REST API.

### Pattern 1: Apex REST Class (authenticated)

Best for middleware → Salesforce pushes where you control both sides. The middleware authenticates with JWT Bearer and POSTs to a custom Apex REST endpoint.

```apex
@RestResource(urlMapping='/webhook/ehr-sync-result/*')
global class EHRSyncResultWebhook {
    @HttpPost
    global static void handleResult() {
        RestRequest req = RestContext.request;
        RestResponse res = RestContext.response;

        Map<String, Object> body = (Map<String, Object>) JSON.deserializeUntyped(req.requestBody.toString());
        String ehrClientId = (String) body.get('ehr_client_id');
        String status = (String) body.get('status');
        String errorMsg = (String) body.get('error_message');

        List<Account> accounts = [
            SELECT Id, APP__Sync_Status__c, APP__Last_EHR_Error__c
            FROM Account
            WHERE APP__EHR_Client_Id__c = :ehrClientId
            LIMIT 1
        ];

        if (!accounts.isEmpty()) {
            accounts[0].APP__Sync_Status__c = status;
            accounts[0].APP__Last_EHR_Error__c = errorMsg;
            update accounts;
        }

        res.statusCode = 200;
        res.responseBody = Blob.valueOf('{"ok": true}');
    }
}
```

```python
# FastAPI middleware: posting sync result back to Salesforce Apex REST endpoint
async def post_sync_result_to_salesforce(
    ehr_client_id: str,
    status: str,
    error_message: str | None,
    sf_token: str,
    instance_url: str,
):
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{instance_url}/services/apexrest/webhook/ehr-sync-result/",
            headers={"Authorization": f"Bearer {sf_token}", "Content-Type": "application/json"},
            json={"ehr_client_id": ehr_client_id, "status": status, "error_message": error_message},
            timeout=30.0,
        )
        resp.raise_for_status()
```

### Pattern 2: Platform Events REST API (unauthenticated producer)

If the external system cannot authenticate (e.g., a third-party webhook), you can expose a thin proxy (FastAPI or Cloud Run function) that receives the raw webhook, validates an HMAC signature, and republishes as a Platform Event via the Salesforce REST API.

**Common mistake:** Exposing an Apex REST class via a Salesforce Site (unauthenticated) without HMAC validation. Any system on the internet can POST to it. Always validate a shared secret or IP allowlist.

---

## 6. The Salesforce → Middleware → EHR System Pattern in the Integration Platform

This is the architecture you own. It is worth dissecting every layer to understand where failures propagate and how to make each boundary resilient.

```
Salesforce Agent (DML)
    │
    ▼
Apex Trigger (before/after insert/update/delete)
    │  bulkify, collect Ids
    ▼
@future callout (async, one HTTP call per batch)
    │  Named Credential → CRM_Middleware
    ▼
FastAPI Middleware (/api/v2/accounts/bulk)
    │  deserialize, validate, map to EHR schema
    ▼
PostgreSQL queue table (app_sync_queue)
    │  worker polls, retries on failure
    ▼
EHR System REST API (/clients POST/PUT)
    │
    ▼
Middleware POSTs result back to Salesforce Apex REST endpoint
    │
    ▼
Account.APP__Sync_Status__c updated
```

### Failure Points and Mitigations

| Layer | Failure Mode | Mitigation |
|---|---|---|
| Apex trigger | Governor limit (callout count) | Bulkify: single `@future` per batch |
| `@future` | Async context, no rollback | Log failures to Platform Event |
| Named Credential | Token expiry | Salesforce auto-refreshes; no manual handling needed |
| FastAPI | Unhandled exception | Return 500, Apex publishes error event |
| PostgreSQL queue | Worker crash | Poison-pill detection, dead-letter queue |
| EHR System | 4xx client error | Do not retry; alert and log; fix mapping |
| EHR System | 5xx / timeout | Exponential back-off in worker |
| Result callback | Salesforce unavailable | Retry with back-off from middleware |

**Common mistake:** Treating the `@future` call as fire-and-forget with no error path. The `@future` method runs in a separate execution context — if it throws or the HTTP call fails, the original DML transaction has already committed. You must publish a Platform Event or write to an error log from within the `@future` method itself.

---

## 7. Connected Apps and Canvas

### Connected Apps

A Connected App is the OAuth 2.0 client registration in Salesforce. Every integration that calls the Salesforce API must have one. Key settings:

- **OAuth Scopes:** Principle of least privilege. The middleware's connected app should have `api` and `refresh_token` only — not `full`.
- **IP Restrictions:** Allowlist the middleware's egress IPs. Reduces blast radius if the key is compromised.
- **Certificate-based auth:** For JWT Bearer, upload the public key certificate to the connected app. The private key stays in the middleware's secret manager (Vault in the integration platform's case).
- **Refresh Token Policy:** For server-to-server JWT flows, refresh tokens are not issued — the JWT is re-signed on each call. Set "Refresh Token Policy" to "Immediately expire refresh token" to enforce this.

### Canvas Apps

Canvas is a mechanism for embedding an external web application inside the Salesforce UI (in a page layout, Visualforce page, or Community). The external app receives a signed request containing the user's identity, org ID, and record context — no separate login needed.

Canvas authentication flow:
1. Salesforce renders the Canvas iframe.
2. Salesforce POSTs a signed JSON payload (`signed_request`) to the Canvas app URL.
3. The Canvas app verifies the HMAC-SHA256 signature using the Connected App's consumer secret.
4. The app extracts the access token and user context from the payload.

```python
# FastAPI Canvas app: verify signed_request from Salesforce
import hmac, hashlib, base64, json

def verify_canvas_request(signed_request: str, consumer_secret: str) -> dict:
    encoded_sig, encoded_payload = signed_request.split(".", 1)
    expected_sig = hmac.new(
        consumer_secret.encode(),
        encoded_payload.encode(),
        hashlib.sha256,
    ).digest()
    provided_sig = base64.b64decode(encoded_sig + "==")  # pad for safety

    if not hmac.compare_digest(expected_sig, provided_sig):
        raise ValueError("Canvas signature verification failed")

    payload_json = base64.b64decode(encoded_payload + "==").decode()
    return json.loads(payload_json)
```

**Common mistake:** Using string equality (`==`) instead of `hmac.compare_digest()` to compare Canvas signatures. String equality is vulnerable to timing attacks. Always use a constant-time comparison function.

---

## 8. Apex Callout Patterns: Queueable vs. `@future` vs. Batch

Not all async callout contexts are equal. Choosing the wrong one creates subtle bugs.

| Context | Callout Support | Chaining | Limit Notes |
|---|---|---|---|
| `@future(callout=true)` | Yes | No | 50/transaction; no parameter objects |
| `Queueable` + `AllowsCallouts` | Yes | Yes (chain) | 1 callout job queued per transaction from trigger |
| `Batchable` (`Database.AllowsCallouts`) | Yes | No chaining | 1 HTTP call per `execute()` scope; use scope=1 for one-per-record |
| Synchronous Apex | Yes (if not in trigger) | N/A | Only from non-trigger, non-future context |

For the integration platform's bulk Account sync, `@future` works well for moderate volumes. If the integration needs to chain calls (e.g., create client in the EHR system, then fetch the assigned EHR ID, then update Salesforce), `Queueable` with chaining is the right pattern.

```apex
// Queueable callout with chaining: create EHR client, then write back EHR ID
public class EHRClientCreateJob implements Queueable, Database.AllowsCallouts {
    private List<Id> accountIds;

    public EHRClientCreateJob(List<Id> accountIds) {
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext ctx) {
        // Step 1: POST to middleware
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:CRM_Middleware/api/v2/accounts/bulk');
        req.setMethod('POST');
        req.setBody(buildPayload(accountIds));
        HttpResponse res = new Http().send(req);

        if (res.getStatusCode() == 202) {
            // Step 2: Chain a job to poll for completion
            String jobId = (String) ((Map<String, Object>) JSON.deserializeUntyped(res.getBody())).get('job_id');
            System.enqueueJob(new EHRClientPollJob(accountIds, jobId));
        }
    }

    private String buildPayload(List<Id> ids) {
        List<Account> accs = [SELECT Id, Name, APP__EHR_Client_Id__c FROM Account WHERE Id IN :ids];
        return JSON.serialize(accs);
    }
}
```

**Common mistake:** Calling `System.enqueueJob()` from within a `@future` method. This throws a `System.AsyncException` — you cannot enqueue a Queueable from a `@future` context. If you need chaining, start with Queueable from the trigger.

---

## 9. Middleware Design for Salesforce Integration

The FastAPI middleware is not just a proxy — it is where the integration's resilience lives. Key design principles:

### Idempotency

Every endpoint that Salesforce calls must be idempotent. Salesforce `@future` methods can be invoked multiple times if a transaction is retried or a developer re-runs a trigger. The middleware must handle duplicate payloads gracefully.

```python
# FastAPI: idempotent account upsert using Salesforce ID as natural key
@router.post("/api/v2/accounts/bulk", status_code=202)
async def bulk_upsert_accounts(
    payload: list[SalesforceAccountSchema],
    db: AsyncSession = Depends(get_db),
):
    for account in payload:
        existing = await db.execute(
            select(SyncQueue).where(SyncQueue.sf_record_id == account.id)
        )
        record = existing.scalar_one_or_none()
        if record:
            # Update existing queue entry — do not duplicate
            record.payload = account.model_dump()
            record.status = "pending"
            record.updated_at = datetime.utcnow()
        else:
            db.add(SyncQueue(
                sf_record_id=account.id,
                object_type="Account",
                payload=account.model_dump(),
                status="pending",
            ))
    await db.commit()
    return {"queued": len(payload)}
```

### Payload Validation

Do not pass raw Salesforce JSON to the EHR system. The middleware's job is to validate, transform, and enrich. Use Pydantic models to enforce the EHR API's contract and surface mapping errors early.

### Circuit Breaker

If the EHR system returns 5xx errors for 10 consecutive calls, stop sending for 60 seconds and alert. This prevents the queue from growing unboundedly while the EHR system is down and avoids hammering a recovering service.

**Common mistake:** Setting the middleware request timeout too high (e.g., 300 seconds). Salesforce's hard callout limit is 120 seconds. If the middleware takes longer than 120 seconds to respond (even with a 202 Accepted), the Apex callout throws a `System.CalloutException`. Always respond with 202 immediately and process asynchronously.

---

## 10. Key Concepts Summary

```
Salesforce Integration Patterns
├── Outbound (SF → External)
│   ├── Apex Callout (@future / Queueable / Batch)
│   │   ├── Named Credential (auth abstraction)
│   │   ├── Bulkify: List<Id> per call
│   │   └── Error → Platform Event
│   └── Outbound Message (SOAP, declarative, auto-retry)
│       └── Use only for simple SOAP endpoints
│
├── Inbound (External → SF)
│   ├── Apex REST (@RestResource)
│   │   └── JWT Bearer auth from middleware
│   ├── Platform Event REST publish
│   │   └── No Salesforce auth needed in producer
│   └── Canvas (signed_request, iframe embed)
│
├── Event / Change Streaming
│   ├── Platform Events (__e)
│   │   ├── Developer-defined schema
│   │   └── Explicit publish (Apex / REST)
│   ├── Change Data Capture (CDC)
│   │   ├── Auto-generated on DML
│   │   └── Changed fields + changeType only
│   └── PushTopic (deprecated → avoid)
│
├── Auth Patterns
│   ├── JWT Bearer (server-to-server, preferred)
│   ├── Username-Password (legacy, avoid)
│   └── Web Server / Auth Code (user-delegated, Canvas)
│
└── Integration Platform Architecture
    ├── Trigger → bulkified @future callout
    ├── Named Credential → FastAPI middleware
    ├── Middleware → PostgreSQL queue → EHR system
    ├── EHR result → Apex REST webhook → Account update
    └── Errors → Platform Event → middleware alert
```

---

## Quiz — 20 Questions

### Questions

**1.** In the JWT Bearer OAuth flow, what does Salesforce validate to authenticate the client application (not the user)?

**2.** The integration platform's `AccountTriggerHandler` calls a `@future(callout=true)` method. A batch DML operation inserts 200 Accounts in one transaction. How many `@future` invocations actually execute, and why?

**3.** What is the key operational difference between Outbound Messages and Apex callouts regarding delivery guarantees?

**4.** A Platform Event subscriber receives a `APP__Sync_Error__e` event. The `replayId` is 4500. What does this number enable?

**5.** Change Data Capture fires when an `APP__Policy__c` record is updated. The event arrives at the middleware subscriber. What fields are guaranteed to be present in the payload, and what is NOT included?

**6.** Why must the FastAPI middleware respond to an Apex callout within 120 seconds, even if the actual work takes longer?

**7.** Describe the Canvas authentication flow. What does the external Canvas app receive from Salesforce, and how does it verify it?

**8.** What is the difference between `@future(callout=true)` and a Queueable implementing `Database.AllowsCallouts`, and when would you choose Queueable in the integration platform?

**9.** An Apex REST endpoint at `/services/apexrest/webhook/ehr-sync-result/` receives POST requests from the FastAPI middleware. What authentication mechanism should the middleware use, and why not anonymous?

**10.** What does `hmac.compare_digest()` protect against compared to the `==` operator when validating a Canvas `signed_request`?

**11.** A new engineer proposes using a PushTopic to stream `APP__Policy__c` changes to the BigQuery ETL pipeline. What is the preferred modern alternative and why?

**12.** The integration platform middleware's `/api/v2/accounts/bulk` endpoint receives the same `sf_record_id` twice in quick succession (Salesforce retry). What design property prevents a duplicate EHR API call?

**13.** Named Credentials in Apex callouts handle authentication transparently. What happens when the OAuth access token expires mid-operation?

**14.** A Platform Event is published from an Apex trigger. The trigger's DML transaction is later rolled back due to a validation error. Is the Platform Event also rolled back?

**15.** The integration platform circuit breaker stops sending requests to the EHR system after 10 consecutive 5xx errors. From the Salesforce side, what happens to the sync queue during the 60-second pause?

**16.** What OAuth scope should the integration platform middleware's Connected App have, and why not `full`?

**17.** An Apex `@future` method finishes executing after the original transaction commits. A `System.CalloutException` is thrown inside the `@future` method. What is the impact on the original Salesforce record?

**18.** Describe two ways to publish a Salesforce Platform Event from an external system (not Apex).

**19.** In the integration platform sync architecture, why is PostgreSQL used as a queue between the middleware and the EHR system, rather than calling the EHR system directly in the FastAPI request handler?

**20.** A Queueable job chains to a second Queueable. The first job makes a callout to the middleware. Can the second chained job also make a callout?

---

### Answers

??? note "Reveal Answers"

    **1.** In the JWT Bearer flow, the client signs a JWT with its RSA private key and Salesforce validates the signature against the public key certificate uploaded to the Connected App. There is no client secret exchange — the cryptographic proof is the signed JWT itself. Salesforce also validates the `iss` (Consumer Key), `sub` (Salesforce username), `aud` (login URL), and `exp` (expiry within 5 minutes) claims. This makes JWT Bearer more secure than username-password because no long-lived secret is transmitted over the wire.

    **2.** Only 50 of the 200 invocations execute. Salesforce enforces a limit of 50 `@future` calls per Apex transaction. The remaining 150 are silently discarded — no error is thrown for the dropped calls. The fix is to bulkify: collect all 200 Account IDs in the trigger handler and pass them as a single `List<Id>` to one `@future` invocation that builds a single bulk HTTP payload.

    **3.** Outbound Messages provide Salesforce-managed automatic retry for up to 24 hours if the endpoint is unreachable or does not return an `<Ack>`. Apex callouts have no automatic retry — if the HTTP call fails, the error must be caught in Apex code and retried manually (e.g., via re-enqueue to a Queueable or publishing a Platform Event for a retry worker). The integration platform's callout pattern relies on the middleware's queue for durability, not Salesforce's retry mechanism.

    **4.** The `replayId` is a monotonically increasing pointer in the Platform Event bus that enables a subscriber to replay events from a specific position. If the middleware process restarts, it can re-subscribe starting from `replayId` 4500 to receive all events it missed during the downtime, rather than starting from -1 (newest only) or -2 (all retained). Retention is 72 hours, so events older than that cannot be replayed regardless of `replayId`.

    **5.** CDC guarantees the record's `Id`, `changeType` (CREATE/UPDATE/DELETE/UNDELETE), changed field names and their new values, and header metadata (org ID, transaction key, sequence number). What is NOT included is the full record — unchanged fields are absent from the payload. If the downstream system needs the full record state (e.g., to build a complete BigQuery row), it must query Salesforce after receiving the event using the record ID from the payload.

    **6.** Salesforce enforces a hard 120-second timeout on all Apex callouts, regardless of any timeout value set on the `HttpRequest`. If the middleware has not returned a response within 120 seconds, Salesforce throws a `System.CalloutException` in the Apex execution context. The standard pattern is to respond immediately with `HTTP 202 Accepted` and a job ID, then process asynchronously. The actual EHR sync happens in the middleware's background worker, and results are pushed back via the Apex REST webhook endpoint.

    **7.** Canvas authentication begins when Salesforce renders the Canvas iframe and POSTs a `signed_request` parameter to the Canvas app URL. The `signed_request` consists of a base64-encoded HMAC-SHA256 signature and a base64-encoded JSON payload, separated by a dot. The Canvas app splits on the dot, recomputes the HMAC using the Connected App's consumer secret, and compares the signatures using a constant-time function. If valid, the app decodes the payload to extract the Salesforce access token, user ID, org ID, and record context — no separate login required.

    **8.** `@future` is simpler but cannot chain (you cannot enqueue another job from inside it) and accepts only primitive or primitive-collection parameters. Queueable with `Database.AllowsCallouts` supports chaining — a Queueable can call `System.enqueueJob()` to spawn a follow-up job — and accepts complex object parameters. In the integration platform, Queueable is preferred when the sync requires two sequential HTTP calls: for example, POST to the EHR system to create the client (getting an assigned EHR ID back), then write that EHR ID back to Salesforce. A single `@future` cannot do this because it cannot chain.

    **9.** The middleware should use the JWT Bearer OAuth 2.0 flow to obtain a Salesforce access token and include it as a `Bearer` token in the `Authorization` header. Anonymous (Site) access should not be used because there is no way to enforce per-caller identity or scope, any system on the internet could POST forged sync results, and it violates least-privilege security. With JWT Bearer, the middleware's Connected App is the authenticated caller, and Salesforce can enforce IP restrictions and scope limits.

    **10.** `hmac.compare_digest()` performs a constant-time comparison, meaning it always takes the same amount of time regardless of how many bytes match. The `==` operator short-circuits on the first mismatched byte and returns `False` immediately. An attacker can exploit this timing difference to progressively reconstruct the correct HMAC one byte at a time (timing attack). For Canvas signature validation — where a compromise lets an attacker forge Salesforce user identities — constant-time comparison is a required security control.

    **11.** The preferred modern alternative is Change Data Capture (CDC). CDC is automatically generated by Salesforce for any subscribed standard or custom object without requiring SOQL queries or manual PushTopic definitions. CDC includes `changeType` metadata, supports 72-hour replay, and scales to high-volume DML without the per-query result-set limits that PushTopics have. Salesforce has also signaled that PushTopics are on a deprecation path and will eventually be retired.

    **12.** Idempotency — the middleware uses the `sf_record_id` as the natural key in the `SyncQueue` table. On receiving a duplicate payload, the middleware finds the existing queue row and updates it (payload and status) rather than inserting a new row. The EHR worker processes one queue row per record, so the duplicate callout results in one EHR API call, not two. This is critical because Salesforce `@future` methods can be invoked multiple times in error-retry scenarios.

    **13.** Salesforce Named Credentials handle token refresh transparently. When the access token expires, Salesforce automatically re-authenticates using the configured OAuth flow (e.g., re-signing a new JWT if using JWT Bearer) before sending the outbound callout. The Apex code does not need to implement any token refresh logic — it simply re-issues the callout to the Named Credential endpoint and the platform handles auth. This is one of the primary reasons to use Named Credentials over manually managing tokens in custom settings.

    **14.** No — this is a critical Platform Events behavior that trips up many engineers. Platform Events published from within an Apex transaction are **not** rolled back when the transaction fails. They are committed to the event bus immediately at the point `EventBus.publish()` is called, independent of the outer DML transaction's outcome. This makes them useful for emitting audit or error signals that must survive transaction rollbacks, but it also means you can receive "orphan" events for records that were never actually saved.

    **15.** During the 60-second circuit-breaker pause, new sync requests from Salesforce continue to arrive at the middleware and are written to the PostgreSQL queue as usual (the middleware still accepts inbound callouts). The queue grows during the pause. When the circuit closes again, the worker drains the backlog in order. The Salesforce-side `Account.APP__Sync_Status__c` remains in "pending" or "error" state, and any SLAs for EHR sync latency are violated for records queued during the pause. This is expected and preferable to sending requests to a failing EHR instance.

    **16.** The Connected App should have the `api` scope (enables REST API access) and, if refresh tokens are needed, `refresh_token`. It should not have `full` because that grants every permission the authenticated user has, including destructive operations (delete all records, modify profiles). Least-privilege OAuth scopes limit the blast radius if the client credentials are compromised — an attacker with only `api` scope cannot modify org metadata, create users, or access billing.

    **17.** The original DML transaction has already committed — the Account record is saved in Salesforce. The `@future` method's `CalloutException` has no effect on the original record. This is the fundamental danger of `@future`: the sync is detached from the DML, so a failed callout leaves the Account saved in Salesforce but not propagated to the EHR system, creating silent data divergence. This is why the integration platform's `@future` method publishes a `APP__Sync_Error__e` Platform Event on failure, which the middleware subscribes to for alerting and potential retry orchestration.

    **18.** First, via the Salesforce REST API: an authenticated external system can POST to `/services/data/vXX.0/sobjects/YourEvent__e/` with a JSON payload matching the event's schema. The system must have a valid OAuth access token. Second, via the Platform Events Pub/Sub API (gRPC-based, newer): external systems can connect using the gRPC Pub/Sub API endpoint and publish events with higher throughput and lower latency than the REST API. This is particularly useful for high-volume event streams from systems like the integration platform middleware.

    **19.** Calling the EHR system directly in the FastAPI request handler (synchronously within the inbound callout from Salesforce) would tie the Salesforce callout's 120-second timeout to the EHR system's response time. The EHR system is a third-party system with variable latency. A slow or unavailable EHR server would cause the middleware to miss the 120-second window, resulting in Salesforce `CalloutException` errors even when the middleware itself is healthy. The PostgreSQL queue decouples the two async boundaries, allows independent retry logic for EHR failures, enables rate limiting to respect the EHR system's API constraints, and provides a durable audit log of every sync attempt.

    **20.** Yes — chained Queueable jobs can each make callouts independently, as long as each Queueable class implements `Database.AllowsCallouts`. Callout governor limits (100 callouts per transaction) reset for each Queueable execution context. The key restriction is that only one Queueable can be enqueued per transaction from a trigger context, but once in the Queueable chain, each job is its own transaction and can enqueue another job and make callouts freely (subject to per-transaction limits). This is the correct pattern for the integration platform "create client in the EHR system, then write EHR ID back to Salesforce" multi-step flow.
