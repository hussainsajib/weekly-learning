# Week 18 — API Design — REST, gRPC & GraphQL

**Week of:** October 5, 2026
**Estimated study time:** ~2 hours
**Tags:** `api` `rest` `grpc` `graphql`

---

## Overview

API design is one of the highest-leverage skills a senior engineer can master. Unlike internal code, an API is a public contract: once consumers are wired to it, changes become expensive coordination problems. For engineers targeting the staff level, the question is rarely "how do I make an endpoint work?" but rather "how do I design a surface that survives years of evolution, heterogeneous consumers, and distributed teams?" This week covers the three dominant paradigms — REST, gRPC, and GraphQL — through that lens.

In the CRM-EHR Integration Platform ecosystem, this is not academic. The `crm-middleware` exposes v1 and v2 REST APIs consumed by two very different clients: Salesforce Apex callouts (strict JSON, no streaming, hard callout timeout at 120 seconds) and ETL pipelines (batch-oriented, schema-stable, latency-tolerant). Those two consumers have almost opposite needs, and the design decisions you make — versioning strategy, OpenAPI contract discipline, response shape — directly determine how painful the next breaking change will be.

REST is the lingua franca of web APIs but is frequently misimplemented. The Richardson Maturity Model gives a precise vocabulary for how "RESTful" an API actually is, and most production systems live at Level 2 rather than the theoretically ideal Level 3. gRPC trades human-readability for performance and strong typing; it excels at internal service-to-service calls and streaming workloads, but is a poor fit where the consumer is a browser or a platform like Salesforce. GraphQL inverts the query model: instead of the server deciding response shape, the client does — which solves over-fetching but introduces its own operational complexity (N+1 queries, authorization surface, caching challenges).

By the end of this week you will be able to articulate the right tool for a given API context, design an OpenAPI spec that serves as a contract between Apex and the middleware, apply DataLoader patterns to GraphQL, and use FastAPI's router/dependency system to manage a large multi-version API cleanly.

---

## 1. REST Maturity Levels and the Richardson Model

Leonard Richardson's maturity model defines four levels of REST adoption. Most engineers can recite "Level 2" without knowing what it costs to not be there.

| Level | Name | What it means | Platform relevance |
|-------|------|---------------|----------------|
| 0 | The Swamp of POX | Single URI, all verbs tunnel through POST | Pre-REST SOAP style |
| 1 | Resources | Multiple URIs, one per resource type | `/clients`, `/policies` — the integration platform is here minimum |
| 2 | HTTP Verbs | GET/POST/PUT/DELETE carry semantic meaning; status codes are meaningful | the integration platform v2 target |
| 3 | Hypermedia (HATEOAS) | Responses embed links to valid next actions | Rarely implemented in practice |

Level 2 is the practical target for most production APIs. Status codes matter: `200` for success, `201` for created, `204` for no-content deletes, `400` for client validation failures, `404` for missing resources, `409` for conflict (e.g., concurrent sync collision), `422` for unprocessable entity (FastAPI's default for Pydantic validation errors), and `500` only when something is genuinely unexpected.

```python
# FastAPI Level 2 example: integration platform client endpoint
from fastapi import APIRouter, HTTPException, status
from app.schemas.client import ClientCreate, ClientRead

router = APIRouter(prefix="/clients", tags=["clients"])

@router.post("/", response_model=ClientRead, status_code=status.HTTP_201_CREATED)
async def create_client(payload: ClientCreate, db: AsyncSession = Depends(get_db)):
    existing = await db.scalar(select(Client).where(Client.ehr_id == payload.ehr_id))
    if existing:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail="Client already exists")
    client = Client(**payload.model_dump())
    db.add(client)
    await db.commit()
    await db.refresh(client)
    return client
```

**Common mistake:** Returning `200 OK` for everything — including creation — and encoding success/failure in a custom `{"status": "error"}` body. This breaks any client that correctly inspects HTTP status codes, including Apex's `HttpResponse.getStatusCode()`.

---

## 2. API Versioning Strategies

Versioning is about managing change without breaking existing consumers. Three main strategies exist, each with real trade-offs.

**URI path versioning** (`/v1/clients`, `/v2/clients`) is explicit, easy to route in Kubernetes ingress rules, and easy for Apex to call. It is the strategy used in `crm-middleware` and is the right choice when consumers are external platforms you do not control.

**Header versioning** (`Accept: application/vnd.platform.v2+json`) keeps URIs clean but requires consumers to set custom headers — something Apex named credentials and standard `HttpRequest` can do but which adds configuration overhead.

**Query parameter versioning** (`/clients?version=2`) is convenient for browser exploration but semantically odd (versioning is not a filter) and can create caching problems.

For the integration platform middleware, URI versioning is correct. The challenge is managing divergence between v1 and v2 in a single FastAPI codebase without duplicating business logic.

```python
# app/api/v1/router.py
from fastapi import APIRouter
from app.api.v1 import clients, policies

router_v1 = APIRouter(prefix="/v1")
router_v1.include_router(clients.router)
router_v1.include_router(policies.router)

# app/api/v2/router.py
from fastapi import APIRouter
from app.api.v2 import clients, policies  # v2-specific schemas/handlers

router_v2 = APIRouter(prefix="/v2")
router_v2.include_router(clients.router)
router_v2.include_router(policies.router)

# app/main.py
app.include_router(router_v1)
app.include_router(router_v2)
```

The pattern for v1→v2 migration is to keep shared service-layer functions and only fork at the schema/serialization layer. If `ClientServiceV1` and `ClientServiceV2` call the same `ClientRepository`, business logic stays DRY.

```python
# Shared service, forked schemas
# app/services/client_service.py (shared)
async def get_client(ehr_id: str, db: AsyncSession) -> Client:
    return await db.scalar(select(Client).where(Client.ehr_id == ehr_id))

# v1 schema — flat, legacy field names
class ClientReadV1(BaseModel):
    ehr_client_id: str
    client_name: str

# v2 schema — nested, renamed fields
class ClientReadV2(BaseModel):
    id: str
    name: str
    address: AddressSchema
```

**Integration platform connection:** The v1 → v2 migration in the middleware is exactly this scenario. Apex callouts in Staging and PRD org still reference `/v1/` paths (hardcoded in named credentials or in Apex constants). Deprecating v1 safely means: (1) keep v1 alive while Apex is updated, (2) use the OpenAPI spec to communicate the deprecation timeline, (3) monitor v1 call volume in Datadog before cutting over.

**Common mistake:** Adding a new required field to an existing response schema without bumping the version. Apex classes that deserialize the response with strict JSON parsing will silently mismap or throw a `JSONException` at runtime.

---

## 3. Hypermedia and HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) is the Level 3 aspiration: responses include links describing valid next transitions, making the API self-documenting at runtime.

```json
{
  "id": "C-00123",
  "name": "Acme Corp",
  "_links": {
    "self": { "href": "/v2/clients/C-00123" },
    "policies": { "href": "/v2/clients/C-00123/policies" },
    "contacts": { "href": "/v2/clients/C-00123/contacts" }
  }
}
```

The honest assessment: HATEOAS is theoretically elegant but rarely implemented in full because most API clients are code-generated from specs rather than dynamically following links. Apex Apex callouts are pre-written — they will never "discover" the policies link at runtime.

Where HATEOAS-inspired thinking does pay off is in **pagination**. Including `next`, `prev`, `first`, `last` links in paginated responses is broadly adopted (GitHub API, JSON:API) and genuinely useful.

```python
from pydantic import BaseModel, AnyHttpUrl
from typing import Generic, TypeVar, List, Optional

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: List[T]
    total: int
    page: int
    page_size: int
    next: Optional[AnyHttpUrl] = None
    prev: Optional[AnyHttpUrl] = None
```

**Common mistake:** Implementing pagination with `offset`/`limit` and no link metadata, then discovering that consumers have to independently compute the next page offset — leading to off-by-one bugs in ETL pipelines that page through large result sets.

---

## 4. gRPC and Protobuf — When It Beats REST

gRPC is a high-performance RPC framework built on HTTP/2 and Protocol Buffers. Its advantages are concrete:

- **Strong typing enforced at compile time** via `.proto` schemas — no runtime JSON shape mismatches
- **Binary serialization** — Protobuf payloads are 3–10x smaller than JSON for equivalent data
- **Bidirectional streaming** — single long-lived connection, not request/response pairs
- **Code generation** — server stubs and client libraries generated from the same `.proto`

```protobuf
// client.proto
syntax = "proto3";

package platform.v1;

service ClientService {
  rpc GetClient (GetClientRequest) returns (ClientResponse);
  rpc StreamClients (StreamClientsRequest) returns (stream ClientResponse);
}

message GetClientRequest {
  string ehr_id = 1;
}

message ClientResponse {
  string id = 1;
  string name = 2;
  repeated PolicySummary policies = 3;
}

message PolicySummary {
  string policy_number = 1;
  string status = 2;
}
```

```python
# FastAPI + grpcio server stub (separate process)
import grpc
from concurrent import futures
import client_pb2_grpc, client_pb2

class ClientServicer(client_pb2_grpc.ClientServiceServicer):
    async def GetClient(self, request, context):
        client = await fetch_client(request.ehr_id)
        if not client:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            return client_pb2.ClientResponse()
        return client_pb2.ClientResponse(id=client.ehr_id, name=client.name)
```

**When gRPC beats REST in the integration platform context:** ETL pipelines (Python → middleware) could theoretically benefit from gRPC for large batch syncs — Protobuf binary payloads for 50,000 policy records would be meaningfully smaller. However, Salesforce Apex cannot make gRPC calls (no HTTP/2 support in platform callouts), so the external-facing API must remain REST. gRPC is appropriate for **internal** service-to-service calls, such as a future microservice split within the middleware.

| Factor | REST | gRPC |
|--------|------|------|
| Browser/Salesforce support | Yes | No |
| Human-readable debugging | Yes | Requires protoc/grpcurl |
| Streaming | SSE/WebSocket workarounds | Native |
| Schema evolution | Manual (OpenAPI) | Protobuf field numbers |
| Python ecosystem | Mature (FastAPI, httpx) | Good (grpcio, betterproto) |

**Common mistake:** Using gRPC for public APIs without a gRPC-Gateway or transcoding layer, then discovering that any browser-based tooling, Postman, or platform integration requires HTTP/JSON anyway.

---

## 5. GraphQL — Schema Design and the N+1 Problem

GraphQL gives clients precise control over response shape. A single `/graphql` endpoint replaces dozens of REST endpoints, and clients request only the fields they need.

```graphql
# Schema definition
type Query {
  client(id: ID!): Client
  clients(filter: ClientFilter, first: Int, after: String): ClientConnection!
}

type Client {
  id: ID!
  name: String!
  policies: [Policy!]!
  contacts: [Contact!]!
}

type Policy {
  policyNumber: String!
  effectiveDate: String!
  lines: [Line!]!
}
```

```python
# Strawberry (Python GraphQL) resolver
import strawberry
from typing import List

@strawberry.type
class Policy:
    policy_number: str
    effective_date: str

@strawberry.type
class Client:
    id: strawberry.ID
    name: str

    @strawberry.field
    async def policies(self) -> List[Policy]:
        # WARNING: This is the N+1 pattern — one DB query per client
        return await fetch_policies_for_client(self.id)

@strawberry.type
class Query:
    @strawberry.field
    async def clients(self) -> List[Client]:
        return await fetch_all_clients()  # Returns N clients
```

The N+1 problem: fetching 100 clients and then resolving `policies` for each triggers 101 database queries (1 for clients + 100 for policies). The solution is **DataLoader** — a batching and caching utility that collects all requested IDs within a single request tick and issues a single batched query.

```python
from strawberry.dataloader import DataLoader
from typing import List

async def load_policies_by_client_ids(client_ids: List[str]) -> List[List[Policy]]:
    # One query for all client IDs
    rows = await db.execute(
        select(Policy).where(Policy.client_id.in_(client_ids))
    )
    policies = rows.scalars().all()
    # Group by client_id and return in same order as input
    policy_map: dict[str, list] = {cid: [] for cid in client_ids}
    for p in policies:
        policy_map[p.client_id].append(p)
    return [policy_map[cid] for cid in client_ids]

# In context factory
policy_loader = DataLoader(load_fn=load_policies_by_client_ids)

@strawberry.type
class Client:
    id: strawberry.ID
    name: str

    @strawberry.field
    async def policies(self, info: strawberry.types.Info) -> List[Policy]:
        return await info.context["policy_loader"].load(self.id)
```

**Integration platform connection:** GraphQL would be a poor fit for the current Apex-facing API (Apex requires explicit JSON structures with known shapes). However, a GraphQL layer could be valuable for a future internal analytics API or an admin panel querying policy/client/contact data with flexible filtering.

**Common mistake:** Deploying GraphQL without depth limiting or query complexity analysis. A malicious or careless client can craft a deeply nested query (`client { policies { lines { endorsements { ... } } } }`) that triggers exponential resolver chains and takes down the server.

---

## 6. OpenAPI/Swagger Spec Design as a Contract

The OpenAPI spec is not just documentation — it is a **machine-readable contract** between the middleware and its consumers. For the integration platform, the Apex classes that call the middleware are hand-written to match the spec. When the spec drifts from the implementation, Apex callouts break in production.

FastAPI auto-generates an OpenAPI spec from your route definitions and Pydantic schemas. The discipline is in annotating it correctly.

```python
from fastapi import APIRouter, Query
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum

class PolicyStatus(str, Enum):
    ACTIVE = "active"
    CANCELLED = "cancelled"
    EXPIRED = "expired"

class PolicyRead(BaseModel):
    policy_number: str = Field(..., description="EHR policy number", example="POL-2026-00123")
    status: PolicyStatus = Field(..., description="Current policy status")
    effective_date: str = Field(..., description="ISO 8601 date", example="2026-01-01")
    expiration_date: Optional[str] = Field(None, description="ISO 8601 date, null if open-ended")

    model_config = {"json_schema_extra": {"example": {
        "policy_number": "POL-2026-00123",
        "status": "active",
        "effective_date": "2026-01-01",
        "expiration_date": "2027-01-01"
    }}}

router = APIRouter()

@router.get(
    "/policies/{policy_number}",
    response_model=PolicyRead,
    summary="Fetch a single policy by EHR policy number",
    description="Returns the full policy record. Used by Apex PolicyComponentController.",
    responses={
        404: {"description": "Policy not found in middleware"},
        409: {"description": "Sync conflict — policy modified concurrently"},
    },
    tags=["policies"],
    deprecated=False,
)
async def get_policy(
    policy_number: str,
    include_lines: bool = Query(False, description="Embed line items in response"),
):
    ...
```

**Contract stability practices:**
1. Pin the spec to a version tag in your CI pipeline — diff it on every PR to catch unintended breaking changes.
2. Use `response_model_exclude_none=True` consistently so Apex doesn't need to handle unexpected null fields.
3. Never rename a field in an existing version — add a new field in a minor release, rename only in the next major version.
4. Publish the spec at `/openapi.json` and `/docs` but also export a static copy to the repo so Apex developers can reference it offline.

```python
# Export spec to file during CI
import json
from app.main import app

with open("openapi-v2.json", "w") as f:
    json.dump(app.openapi(), f, indent=2)
```

**Common mistake:** Letting FastAPI infer the OpenAPI spec entirely from type hints without adding `description`, `example`, and `responses` annotations. The resulting spec is technically valid but useless as a contract — field meanings are ambiguous and Apex developers make wrong assumptions about nullable fields.

---

## 7. FastAPI Best Practices for Large APIs

A FastAPI app that starts as `main.py` with 10 routes becomes unmaintainable at 60+ routes without intentional structure. The patterns that scale:

**Router decomposition with shared dependencies:**

```
app/
  api/
    v1/
      __init__.py
      clients.py
      policies.py
      router.py       # aggregates all v1 routers
    v2/
      ...
      router.py
  core/
    config.py
    dependencies.py   # get_db, get_current_user, rate_limiter
  schemas/
    client.py
    policy.py
  services/
    client_service.py
    policy_service.py
  repositories/
    client_repo.py
    policy_repo.py
  main.py
```

**Dependency injection for auth and DB:**

```python
# app/core/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import async_session_factory

security = HTTPBearer()

async def get_db() -> AsyncSession:
    async with async_session_factory() as session:
        yield session

async def verify_api_key(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db)
) -> str:
    key = credentials.credentials
    if not await is_valid_api_key(key, db):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return key

# In router — applies to all routes in this router
router = APIRouter(
    prefix="/v2/clients",
    tags=["clients"],
    dependencies=[Depends(verify_api_key)],
)
```

**Background tasks for async EHR sync:**

```python
from fastapi import BackgroundTasks

@router.post("/contacts/", status_code=201)
async def create_contact(
    payload: ContactCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
):
    contact = await contact_service.create(payload, db)
    # Don't block the Apex callout on the EHR sync — Apex has a 120s timeout
    background_tasks.add_task(sync_contact_to_ehr, contact.id)
    return contact
```

**Pydantic v2 model patterns for the integration platform:**

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Optional
import re

class ContactCreate(BaseModel):
    first_name: str
    last_name: str
    email: Optional[str] = None
    phone: Optional[str] = None
    ehr_contact_id: Optional[str] = None

    @field_validator("email")
    @classmethod
    def validate_email_format(cls, v: Optional[str]) -> Optional[str]:
        if v and "@" not in v:
            raise ValueError("Invalid email format")
        return v

    @field_validator("phone")
    @classmethod
    def normalize_phone(cls, v: Optional[str]) -> Optional[str]:
        if v:
            return re.sub(r"\D", "", v)  # Strip non-digits for EHR compatibility
        return v

    @model_validator(mode="after")
    def require_contact_method(self) -> "ContactCreate":
        if not self.email and not self.phone:
            raise ValueError("At least one of email or phone is required")
        return self
```

**Common mistake:** Putting business logic directly in route handlers. When a service needs to be called from a background task, a CLI command, and an HTTP route, logic in the handler is impossible to reuse without the HTTP context. The service layer must be HTTP-agnostic.

---

## 8. API Error Design and Consistency

Error response shapes are part of the contract. An inconsistent error format forces consumers to write defensive parsing logic for every possible error shape.

```python
# app/core/errors.py
from pydantic import BaseModel
from typing import Optional, List

class ErrorDetail(BaseModel):
    field: Optional[str] = None
    message: str
    code: str  # machine-readable, stable across versions

class ErrorResponse(BaseModel):
    error: str          # human-readable summary
    details: List[ErrorDetail] = []
    request_id: str     # for Datadog tracing

# Register globally
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    details = [
        ErrorDetail(field=".".join(str(l) for l in e["loc"]), message=e["msg"], code="validation_error")
        for e in exc.errors()
    ]
    return JSONResponse(
        status_code=422,
        content=ErrorResponse(
            error="Request validation failed",
            details=details,
            request_id=request.headers.get("X-Request-ID", "unknown")
        ).model_dump()
    )
```

**Integration platform connection:** Apex callouts parse the error body to determine if a retry is warranted. A consistent `{"error": "...", "code": "..."}` shape lets Apex implement a single `parseErrorResponse` method rather than per-endpoint error parsing.

**Common mistake:** Letting FastAPI's default 422 Unprocessable Entity response (which has a Pydantic-specific nested `detail` array) reach Apex uncustomized. The Apex JSON parser expects your documented error shape, not FastAPI's internal schema.

---

## 9. Rate Limiting, Idempotency, and Retry Patterns

**Rate limiting** protects the middleware from runaway Apex trigger loops (a trigger fire that creates a record that fires another trigger — a common Salesforce footgun).

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@router.post("/clients/")
@limiter.limit("100/minute")
async def create_client(request: Request, payload: ClientCreate, ...):
    ...
```

**Idempotency keys** allow callers to safely retry without creating duplicates. This is critical for Apex callouts where the platform may retry on timeout.

```python
@router.post("/clients/", status_code=201)
async def create_client(
    payload: ClientCreate,
    idempotency_key: Optional[str] = Header(None, alias="X-Idempotency-Key"),
    db: AsyncSession = Depends(get_db),
):
    if idempotency_key:
        cached = await get_idempotency_cache(idempotency_key)
        if cached:
            return cached  # Return identical prior response

    client = await client_service.create(payload, db)

    if idempotency_key:
        await set_idempotency_cache(idempotency_key, client, ttl=86400)

    return client
```

**Common mistake:** Making write endpoints non-idempotent without documenting it, then discovering that Apex's retry-on-timeout behavior is creating duplicate records in both Salesforce and the EHR system.

---

## 10. Key Concepts Summary

```
API Design — Core Concepts
│
├── REST
│   ├── Richardson Maturity Model (L0 → L3)
│   ├── HTTP semantics (verbs, status codes)
│   ├── Versioning: URI path (preferred for the integration platform), header, query param
│   └── HATEOAS: useful for pagination links; rarely full Level 3
│
├── gRPC
│   ├── Protobuf: binary, strongly typed, schema-evolved via field numbers
│   ├── HTTP/2: multiplexed streams, lower latency than HTTP/1.1
│   ├── Best for: internal service-to-service, streaming pipelines
│   └── Not for: Salesforce Apex, browsers without transcoding proxy
│
├── GraphQL
│   ├── Single endpoint, client-driven queries
│   ├── N+1 problem: naive resolvers issue one query per object
│   ├── DataLoader: batches and deduplicates within request tick
│   └── Needs: depth limiting, complexity scoring, auth at field level
│
├── OpenAPI/FastAPI
│   ├── Auto-generated from Pydantic schemas + route decorators
│   ├── Annotate: description, example, response variants
│   ├── Pin spec in CI — diff on PR to catch breaking changes
│   └── Consistent error shape critical for Apex callout parsing
│
└── Cross-cutting Concerns
    ├── Idempotency keys: safe retries for Apex timeouts
    ├── Rate limiting: protect against Apex trigger loop storms
    ├── Background tasks: decouple EHR sync from Apex callout timeout
    └── Service/Repository layers: HTTP-agnostic, reusable across CLI/tasks
```

---

## Quiz — 20 Questions

### Questions

**1.** What are the four levels of the Richardson Maturity Model, and what distinguishes Level 2 from Level 3?

**2.** Which HTTP status code should the integration platform middleware return when an Apex callout tries to create a client that already exists in the middleware database?

**3.** You are adding a new `broker_code` field to the `/v2/clients/` response. The field is optional and new Apex code will use it, but old Apex code should not break. What is the correct approach?

**4.** Explain the N+1 query problem in GraphQL with a concrete example. How does DataLoader solve it?

**5.** Why is gRPC a poor fit for Salesforce Apex callouts, even though it has strong performance advantages over REST?

**6.** What is the difference between `response_model` and `response_model_exclude_none=True` in FastAPI, and why is the latter important for the integration platform?

**7.** An Apex trigger fires `POST /v2/contacts/` and the middleware times out (the EHR sync takes 130 seconds). Apex retries the callout. How should the middleware be designed to prevent duplicate contact records?

**8.** What is HATEOAS and why is it rarely implemented fully in practice, even for APIs targeting Level 3 REST maturity?

**9.** In Protobuf, you have a message with fields 1, 2, and 3. You need to remove field 2 in a new version. What is the correct way to handle this for backward compatibility?

**10.** Describe the FastAPI project structure pattern for managing v1 and v2 APIs in the same codebase without duplicating business logic.

**11.** What HTTP verb and status code should be used for a full replacement of a resource versus a partial update?

**12.** In the integration platform context, why is exporting the OpenAPI spec as a static JSON file and committing it to the repository valuable?

**13.** What is query complexity analysis in GraphQL and when would you need it?

**14.** A FastAPI route handler is doing: (1) validate input, (2) call the EHR system API, (3) write to database, (4) return response. An ETL batch job needs to call the same logic without HTTP. What refactor resolves this?

**15.** Explain the difference between URI path versioning, header versioning, and query parameter versioning. Which is most appropriate for the integration platform middleware and why?

**16.** What Pydantic v2 decorator replaces the `@validator` decorator from v1, and what is the key behavioral difference when it comes to the `mode` parameter?

**17.** A GraphQL client sends a query with 10 levels of nesting that causes the server to exhaust memory. What two mechanisms should be in place to prevent this?

**18.** In a FastAPI dependency, `yield` is used instead of `return` for database sessions. Why?

**19.** An Apex callout receives a `422 Unprocessable Entity` with FastAPI's default Pydantic error body. The Apex JSON parser fails. What is the correct fix?

**20.** Describe one scenario in the integration platform where gRPC would be a better choice than REST, and one where REST must be used regardless of gRPC's performance advantages.

---

### Answers

??? note "Reveal Answers"

    **1.** Level 0 uses a single URI with a single verb (POX/SOAP style). Level 1 introduces multiple resource URIs. Level 2 adds HTTP verb semantics (GET, POST, PUT, DELETE) and meaningful status codes — this is the practical target for most production APIs. Level 3 adds HATEOAS: responses embed hyperlinks describing valid next actions, making the API self-describing at runtime. The gap between L2 and L3 is significant in practice because most clients are code-generated from specs rather than dynamically following links, making full HATEOAS rarely worth the implementation cost.

    **2.** `409 Conflict` is the correct status code. `400 Bad Request` implies the input is malformed, which is not the case — the request is valid but conflicts with existing server state. `200 OK` or `201 Created` would be semantically wrong. Using `409` allows Apex's `HttpResponse.getStatusCode()` check to distinguish a duplicate-creation attempt from a validation failure, enabling different retry logic for each case.

    **3.** Add `broker_code` as an optional field (`Optional[str] = None`) to the existing `ClientReadV2` Pydantic schema without bumping the version. Optional additions (new nullable fields) are non-breaking changes under semver. Old Apex code that does not reference `broker_code` will simply ignore it when deserializing. Never add a required field to an existing response schema without a version bump, as it would force all consumers to update simultaneously.

    **4.** The N+1 problem occurs when fetching a list of N objects and then resolving a related field for each one with a separate query, resulting in N+1 total queries. For example: fetching 100 clients (1 query) then calling `fetch_policies(client.id)` inside each client resolver (100 more queries). DataLoader solves this by collecting all `client_id` values requested within the same event loop tick and issuing a single batched `SELECT ... WHERE client_id IN (...)`, then distributing results back to each resolver. This reduces N+1 to 2 queries regardless of the list size.

    **5.** Salesforce Apex platform callouts are limited to HTTP/1.1 over HTTPS. gRPC requires HTTP/2 for its multiplexing and streaming features. Without HTTP/2 support, Apex cannot establish a gRPC connection. Additionally, gRPC uses binary Protobuf framing that Apex's `HttpRequest`/`HttpResponse` classes cannot natively serialize or deserialize. A gRPC-HTTP transcoding proxy could bridge this, but adds operational complexity and latency, defeating gRPC's performance advantage.

    **6.** `response_model` tells FastAPI which Pydantic model to use for serializing and validating the response, providing automatic documentation and type safety. `response_model_exclude_none=True` additionally strips any fields whose value is `None` from the JSON output. For the integration platform this matters because Apex classes that strictly parse JSON may interpret an unexpected null field as an error or map it incorrectly. Excluding nulls produces a leaner, more predictable response that matches what the OpenAPI spec documents as "present when relevant."

    **7.** The middleware should support idempotency keys via an `X-Idempotency-Key` request header. On the first call, the middleware processes the request, stores the response in a short-lived cache (Redis or a database table) keyed by the idempotency key, and returns the response. On retry with the same key, the middleware returns the cached prior response without re-processing. Apex should generate a UUID per logical operation and include it as the idempotency key, ensuring that retries caused by the 120-second callout timeout do not create duplicate contacts in either the middleware or the EHR system.

    **8.** HATEOAS embeds hypermedia links in responses to describe valid state transitions, allowing a client to navigate the API without out-of-band documentation. It is rarely implemented fully because the vast majority of API clients — Apex callouts, code-generated SDKs, ETL pipeline consumers — are written against a known spec and will never dynamically follow response links at runtime. The implementation overhead (generating correct link URIs, handling link evolution across versions) is high relative to the practical benefit when consumers are not hypermedia-aware. Pagination links are the common partial adoption.

    **9.** In Protobuf, you must never reuse a field number, even for a removed field. The correct approach is to mark field 2 as `reserved` both by number and by name: `reserved 2; reserved "old_field_name";`. This prevents future `.proto` updates from accidentally reusing field number 2 with a different type, which would cause silent data corruption in clients that still parse the old binary format. The field is logically removed but its number is permanently tombstoned in the schema.

    **10.** The pattern is to split at the API layer (separate `api/v1/` and `api/v2/` router modules with different schemas) while sharing a single service and repository layer. `ClientServiceV1` and `ClientServiceV2` can both call `ClientRepository.get_by_ehr_id()` — the repository is version-agnostic. Only the request/response schemas and any version-specific field transformations live in the versioned layer. This keeps business logic DRY while allowing independent evolution of each API version's contract.

    **11.** Full replacement uses `PUT` and returns `200 OK` (or `204 No Content` if no body is returned). Partial update uses `PATCH` and also returns `200 OK` with the updated resource. The distinction matters because `PUT` semantically replaces the entire resource — any fields not included in the payload should revert to defaults or nulls. `PATCH` only modifies the provided fields. For the integration platform, EHR sync operations that send the full client record should use `PUT`; operations that update a single field (e.g., marking a policy as cancelled) should use `PATCH`.

    **12.** A static OpenAPI spec committed to the repository serves as a durable, version-controlled contract artifact. Apex developers can reference `openapi-v2.json` without running the middleware locally. CI pipelines can diff the spec on every pull request to catch unintended breaking changes (renamed fields, removed endpoints, changed types). The spec file can be tagged alongside a release to provide a precise historical record of what the API contract was at any given deployment — essential for debugging Apex callout failures in production after a middleware deployment.

    **13.** Query complexity analysis assigns a cost to each field in a GraphQL query and rejects queries that exceed a configured maximum total cost. This prevents denial-of-service from deeply nested or fan-out queries: a query that fetches clients, then each client's policies, then each policy's lines, then each line's endorsements could resolve exponentially. Without a complexity limit, a single malicious or careless query can exhaust database connections and memory. It is needed in any GraphQL API that is either public or accessible to non-trusted consumers — which in an internal API context still means defense against bugs in consumer code.

    **14.** Extract the logic into a service function that accepts a database session (or repository) and a data payload, with no dependency on `Request`, `Response`, or any FastAPI HTTP primitives. The route handler becomes a thin adapter: validate the HTTP input with Pydantic, call the service function, return the result. The ETL batch job calls the same service function directly with a database session from the shared session factory. This is the service/repository pattern and is the fundamental architectural requirement for testability and reuse across HTTP, CLI, and task contexts.

    **15.** URI path versioning embeds the version in the URL (`/v1/clients`), making it visible in logs, easy to route at the ingress/load-balancer level, and trivial to call from any HTTP client including Apex. Header versioning uses a custom `Accept` header and keeps URLs clean but requires consumers to set non-standard headers. Query parameter versioning appends `?version=2` but is semantically odd and complicates caching. URI path versioning is most appropriate for the integration platform because Salesforce named credentials and Apex `HttpRequest.setEndpoint()` are URL-based, Kubernetes ingress rules can route `/v1/` and `/v2/` to different service versions if needed, and it is unambiguous in access logs.

    **16.** In Pydantic v2, `@validator` is replaced by `@field_validator`. The key behavioral difference is the `mode` parameter: `mode="before"` runs the validator before Pydantic's own type coercion (equivalent to v1's `pre=True`), and `mode="after"` (the default) runs after coercion on the already-typed value. There is also `@model_validator` for cross-field validation, with `mode="before"` operating on raw dict input and `mode="after"` operating on the fully constructed model instance. The explicit `mode` parameter makes validation ordering unambiguous compared to v1's `pre`/`always`/`each_item` flags.

    **17.** Two mechanisms: (1) **Query depth limiting** — reject any query where the nesting depth exceeds a configured limit (e.g., 5 levels). This is a simple structural check before execution begins. (2) **Query complexity scoring** — assign a cost to each field resolver and reject queries whose total cost exceeds a threshold. Together these bound both structural depth and fan-out width. Some implementations also add **query timeout** as a third defense — killing any resolver that takes longer than a configured wall-clock duration — to handle cases where complexity scoring underestimates actual execution cost.

    **18.** Using `yield` in a FastAPI dependency creates a context manager that guarantees cleanup (session close/rollback) even if the route handler raises an exception. With `return`, the session object is handed to the handler but there is no mechanism to close it afterward — the caller would need to explicitly close it, which is error-prone. The `yield` pattern ensures the `finally` block (or the code after `yield` in the dependency) always runs after the route handler completes, whether successfully or with an exception, correctly closing the database session and returning the connection to the pool.

    **19.** The correct fix is to register a custom `RequestValidationError` exception handler on the FastAPI app that catches Pydantic's validation errors and re-serializes them into your documented error schema (e.g., `{"error": "...", "details": [...], "code": "validation_error"}`). The handler should return a `JSONResponse` with status `422` and a body that matches the `ErrorResponse` Pydantic model documented in your OpenAPI spec. This ensures every error response — validation errors, not-found errors, conflict errors — has the same shape that Apex's single `parseErrorResponse()` method can handle.

    **20.** A scenario where gRPC would be better: a future internal service split where a dedicated "EHR sync worker" microservice communicates with `crm-middleware` at high throughput to batch-sync thousands of policy records. The binary Protobuf payload would be significantly smaller than JSON, the strongly-typed schema enforced by the `.proto` file would prevent sync contract drift, and bidirectional streaming would allow the worker to push sync results back without polling. A scenario where REST must be used: any endpoint called directly by Salesforce Apex, including all current integration platform trigger handlers (`AccountTriggerHandler`, `ContactTriggerHandler`, etc.). Apex platform callouts are HTTP/1.1 only, with no gRPC support, making REST with JSON the only viable option regardless of performance considerations.
