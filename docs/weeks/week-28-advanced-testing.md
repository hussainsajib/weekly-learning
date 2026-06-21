# Week 28 — Advanced Testing Strategies

**Week of:** December 14, 2026
**Estimated study time:** ~2 hours
**Tags:** `testing` `quality` `tdd`

---

## Overview

Testing is the discipline that separates software you can ship with confidence from software you deploy and pray. For engineers targeting staff-level roles, the distinction is not just writing tests — it is knowing which kind of test to write, at what layer, and at what cost. The test pyramid is the mental model that organizes this thinking: many fast, cheap unit tests at the base; a smaller number of integration tests in the middle; a handful of slow end-to-end tests at the top. Violating the pyramid — an inverted pyramid with too many E2E tests and too few unit tests — is one of the most common causes of slow, flaky CI pipelines.

Property-based testing with Hypothesis pushes unit testing to its logical extreme by letting the framework generate thousands of inputs to your functions automatically, finding edge cases you would never think to write by hand. Mutation testing inverts the process: rather than asking "does this test pass?", it asks "if I subtly break the code, do the tests catch it?" Together, these two techniques measure the quality of your test suite itself, not just whether it runs green.

The crm-middleware is a canonical example of an integration-heavy system that is genuinely hard to test. It sits between Salesforce (Apex callouts), the EHR system backend, and a PostgreSQL queue. A pure unit test of the sync logic misses the contract between Salesforce and the middleware. A full E2E test against production the EHR system is slow and risky. The answer is a layered strategy: contract testing with Pact for the Apex→middleware interface, property-based testing for sync deduplication invariants, and Testcontainers to spin up a real PostgreSQL instance for queue-table logic — all running in CI without external dependencies.

Async code and ETL pipelines add another dimension. FastAPI's async request handlers, SQLAlchemy's async sessions, and pytest-asyncio introduce ordering and event-loop pitfalls that trip up engineers who treat async as just "faster sync". This week covers all these layers with concrete pytest code, Hypothesis strategies, Pact contract definitions, and Testcontainers fixtures — grounded throughout in the integration platform codebase patterns you actually work with.

---

## 1. The Test Pyramid and Where the Integration Platform Lives

The test pyramid, coined by Mike Cohn, defines three layers by speed and scope:

```
        /\
       /E2E\          few, slow, brittle
      /------\
     /  Integ  \      moderate, real I/O
    /------------\
   /    Unit       \  many, fast, isolated
  /------------------\
```

For the integration platform specifically, the layers map like this:

| Layer | Example | Tooling | Speed |
|---|---|---|---|
| Unit | Sync deduplication logic, field-mapping transforms | pytest + Hypothesis | <100 ms |
| Integration | Middleware endpoints + real PostgreSQL queue tables | pytest + Testcontainers | 1–10 s |
| Contract | Apex callout ↔ middleware JSON shape | Pact | seconds (no live EHR system) |
| E2E | Full Account sync from Salesforce sandbox to EHR QA | Manual / Cypress | minutes |

The goal is to push as much coverage as possible down the pyramid without sacrificing confidence. A contract test that runs in CI in 3 seconds gives you more day-to-day safety than an E2E test against the QA org that requires a live EHR DEV server.

**Common mistake:** Writing integration tests that call the real EHR BDE endpoint from CI. This makes CI dependent on network availability, EHR server uptime, and test data state. Use Pact consumer-driven contracts instead — the consumer (middleware) defines what it expects, and the provider (EHR SDK or a mock) verifies it without a live server.

---

## 2. Property-Based Testing with Hypothesis

Hypothesis generates inputs to your functions rather than requiring you to enumerate them. You describe the shape of valid input using strategies, and Hypothesis explores the space, shrinking failures to the minimal reproducing case.

### Basic setup

```python
# tests/test_sync_logic.py
from hypothesis import given, settings, assume
from hypothesis import strategies as st
from app.sync.deduplication import deduplicate_events

# Strategy: a list of sync event dicts with an 'external_id' and 'updated_at'
sync_event_strategy = st.lists(
    st.fixed_dictionaries({
        "external_id": st.text(min_size=1, max_size=64),
        "updated_at": st.datetimes(),
        "payload": st.dictionaries(st.text(), st.text()),
    }),
    min_size=0,
    max_size=50,
)

@given(events=sync_event_strategy)
@settings(max_examples=500)
def test_deduplication_is_idempotent(events):
    """Deduplicating twice produces the same result as deduplicating once."""
    once = deduplicate_events(events)
    twice = deduplicate_events(once)
    assert once == twice

@given(events=sync_event_strategy)
def test_deduplication_never_adds_events(events):
    """Deduplication can only remove events, never add them."""
    result = deduplicate_events(events)
    assert len(result) <= len(events)

@given(events=sync_event_strategy)
def test_deduplication_preserves_latest(events):
    """For each external_id, only the most-recent event survives."""
    result = deduplicate_events(events)
    seen_ids = [e["external_id"] for e in result]
    # No duplicate external_ids in output
    assert len(seen_ids) == len(set(seen_ids))
```

### Stateful testing with RuleBasedStateMachine

For the integration platform queue table, which behaves like a state machine (events enqueued → processing → done/failed), Hypothesis's `RuleBasedStateMachine` is a natural fit:

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, initialize, invariant
from app.queue.manager import QueueManager

class QueueStateMachine(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()
        self.manager = QueueManager(dsn="sqlite:///:memory:")  # swap for Testcontainers in integration
        self.enqueued_count = 0
        self.processed_count = 0

    @rule(payload=st.dictionaries(st.text(), st.text()))
    def enqueue(self, payload):
        self.manager.enqueue(payload)
        self.enqueued_count += 1

    @rule()
    def process_one(self):
        event = self.manager.dequeue_one()
        if event:
            self.processed_count += 1

    @invariant()
    def processed_never_exceeds_enqueued(self):
        assert self.processed_count <= self.enqueued_count

TestQueueStateMachine = QueueStateMachine.TestCase
```

**Common mistake:** Using `st.text()` without `min_size=1` for fields that are used as dictionary keys or database identifiers. Hypothesis will generate empty strings, which can expose real bugs — but also expose missing input validation that should be caught at the API layer, not the deduplication layer. Use `assume()` to filter inputs that violate preconditions rather than suppressing them.

---

## 3. Contract Testing with Pact

Pact is a consumer-driven contract testing framework. The consumer (e.g., the Apex callout or the ETL job) defines what it expects from the provider (the middleware API), and the provider verifies those expectations independently — without a live consumer present.

### Why this matters for the integration platform

The Apex `AccountTriggerHandler` makes HTTP callouts to crm-middleware. If a middleware developer renames a JSON field (say, `client_id` → `clientId`), Salesforce deployments break silently until someone manually tests the integration. Pact catches this in CI.

### Consumer side (Python client simulating Apex behavior)

```python
# tests/contract/test_account_consumer.py
import pytest
from pact import Consumer, Provider

PACT_DIR = "tests/contract/pacts"

@pytest.fixture(scope="session")
def pact():
    p = Consumer("ApexAccountTrigger").has_pact_with(
        Provider("CRMMiddleware"),
        pact_dir=PACT_DIR,
        log_dir="logs/pact",
    )
    p.start_service()
    yield p
    p.stop_service()

def test_post_account_creates_client(pact):
    expected_body = {
        "client_id": "SF_ACCOUNT_001",
        "name": "Acme Corp",
        "status": "active",
    }

    (
        pact
        .given("no existing client with id SF_ACCOUNT_001")
        .upon_receiving("a POST request to create a client")
        .with_request(
            method="POST",
            path="/api/v2/clients",
            headers={"Content-Type": "application/json"},
            body={
                "salesforce_id": "SF_ACCOUNT_001",
                "name": "Acme Corp",
            },
        )
        .will_respond_with(
            status=201,
            headers={"Content-Type": "application/json"},
            body=expected_body,
        )
    )

    with pact:
        import httpx
        response = httpx.post(
            f"{pact.uri}/api/v2/clients",
            json={"salesforce_id": "SF_ACCOUNT_001", "name": "Acme Corp"},
        )
        assert response.status_code == 201
        assert response.json()["client_id"] == "SF_ACCOUNT_001"
```

### Provider side (FastAPI middleware verification)

```python
# tests/contract/test_account_provider.py
import pytest
from pact import Verifier

def test_provider_honors_apex_contract():
    verifier = Verifier(
        provider="CRMMiddleware",
        provider_base_url="http://localhost:8000",  # real FastAPI app in test
    )
    output, _ = verifier.verify_with_broker(
        broker_url="http://pact-broker:9292",
        publish_verification_results=True,
        provider_version="2.1.0",
    )
    assert output == 0
```

**Pact workflow summary:**

| Step | Who | What |
|---|---|---|
| Write consumer test | Apex team / ETL team | Defines expected request/response |
| Generate pact file | Consumer CI | JSON contract artifact |
| Publish to Pact Broker | Consumer CI | Central contract registry |
| Verify against provider | Middleware CI | FastAPI app replays interactions |
| Gate deployments | Both CIs | "Can I Deploy" check before merge |

**Common mistake:** Writing Pact tests that are too specific — asserting exact values in responses rather than using matchers. If the response body contains a generated UUID, asserting `"id": "abc-123"` will always fail in provider verification. Use `Like`, `EachLike`, and `Regex` matchers to assert shape, not exact values.

---

## 4. Mutation Testing

Mutation testing answers the question: "How good are my tests, really?" It works by automatically introducing small bugs (mutations) into the source code — flipping `>` to `>=`, removing a `return`, changing `and` to `or` — and then checking whether your test suite fails. A mutation that does not cause any test to fail is a "surviving mutant", which indicates a gap in test coverage.

### Running mutmut on the middleware

```bash
# Install
pip install mutmut

# Run against sync module only (full codebase is too slow for CI)
mutmut run --paths-to-mutate app/sync/ --tests-dir tests/unit/

# Show surviving mutants
mutmut results

# Show the diff for a specific surviving mutant
mutmut show 42
```

### Interpreting results

```
Mutation score: 73%   (73 killed / 100 total mutants)
Surviving mutants: 27
```

A score above 80% is generally considered healthy for business logic. For sync deduplication and field-mapping code — which is the heart of integration platform correctness — aim for 90%+.

### Example: catching a surviving mutant

Suppose `deduplicate_events` contains:

```python
# Original
if event["updated_at"] >= existing["updated_at"]:
    registry[event["external_id"]] = event
```

A mutant changes `>=` to `>`. If your tests only check that deduplication works on strictly newer events, they will not catch the case where equal timestamps should keep the existing event. The surviving mutant tells you to add a test:

```python
def test_deduplication_keeps_existing_on_equal_timestamp():
    t = datetime(2026, 6, 1, 12, 0, 0)
    events = [
        {"external_id": "X", "updated_at": t, "payload": {"v": "1"}},
        {"external_id": "X", "updated_at": t, "payload": {"v": "2"}},
    ]
    result = deduplicate_events(events)
    # first event wins on tie (or last — pick a policy and test it)
    assert len(result) == 1
```

**Common mistake:** Running mutation testing on the entire codebase in CI on every commit. Mutation testing is slow — O(n × test suite time) where n is the number of mutants. Run it nightly or on PRs that touch critical paths, not on every push.

---

## 5. Testcontainers for Real PostgreSQL Tests

Testcontainers spins up real Docker containers during tests and tears them down afterward. For the integration platform, this means your integration tests run against a real PostgreSQL database — not SQLite, not mocks — so you catch dialect-specific behavior, constraint violations, and index performance issues that mocks hide.

### Fixture setup with pytest

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.db.base import Base

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    engine = create_engine(postgres_container.get_connection_url())
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db_engine):
    Session = sessionmaker(bind=db_engine)
    session = Session()
    yield session
    session.rollback()
    session.close()
```

### Testing integration platform queue table logic

```python
# tests/integration/test_queue_table.py
import pytest
from datetime import datetime, UTC
from app.models.queue import SyncEvent
from app.queue.manager import QueueManager

def test_enqueue_and_dequeue_preserves_order(db_session):
    manager = QueueManager(session=db_session)

    events = [
        SyncEvent(external_id=f"SF_{i}", object_type="Account", payload={"n": i})
        for i in range(5)
    ]
    for e in events:
        manager.enqueue(e)

    dequeued = [manager.dequeue_one() for _ in range(5)]
    assert [e.external_id for e in dequeued] == [f"SF_{i}" for i in range(5)]

def test_failed_event_is_retryable(db_session):
    manager = QueueManager(session=db_session)
    event = SyncEvent(external_id="SF_FAIL", object_type="Contact", payload={})
    manager.enqueue(event)

    dequeued = manager.dequeue_one()
    manager.mark_failed(dequeued, reason="EHR timeout")

    # Failed events with retry_count < MAX_RETRIES should be re-enqueued
    retryable = manager.get_retryable_events()
    assert any(e.external_id == "SF_FAIL" for e in retryable)

def test_duplicate_external_id_does_not_double_enqueue(db_session):
    manager = QueueManager(session=db_session)
    event = SyncEvent(external_id="SF_DUP", object_type="Account", payload={"v": 1})
    manager.enqueue(event)
    manager.enqueue(event)  # should be idempotent

    count = db_session.query(SyncEvent).filter_by(external_id="SF_DUP").count()
    assert count == 1
```

### Async SQLAlchemy variant

```python
# tests/integration/test_async_queue.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

@pytest_asyncio.fixture(scope="session")
async def async_engine(postgres_container):
    url = postgres_container.get_connection_url().replace(
        "postgresql://", "postgresql+asyncpg://"
    )
    engine = create_async_engine(url)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest.mark.asyncio
async def test_async_enqueue(async_engine):
    async_session = sessionmaker(async_engine, class_=AsyncSession, expire_on_commit=False)
    async with async_session() as session:
        manager = AsyncQueueManager(session=session)
        await manager.enqueue(SyncEvent(external_id="ASYNC_1", object_type="Account", payload={}))
        event = await manager.dequeue_one()
        assert event.external_id == "ASYNC_1"
```

**Common mistake:** Using `scope="session"` for the database session fixture (not the container fixture). This causes tests to share state — a rollback in one test affects the next. Use `scope="function"` for sessions and `scope="session"` for the container and engine only.

---

## 6. Testing Async Code with pytest-asyncio

FastAPI's async request handlers, background tasks, and event hooks require an async test runner. `pytest-asyncio` provides this, but has several footguns.

### Configuration

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # automatically treat async test functions as async
```

### Testing a FastAPI endpoint

```python
# tests/integration/test_clients_endpoint.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c

async def test_create_client_returns_201(client):
    response = await client.post(
        "/api/v2/clients",
        json={"salesforce_id": "SF_TEST_001", "name": "Test Corp"},
    )
    assert response.status_code == 201
    assert response.json()["client_id"] == "SF_TEST_001"

async def test_create_client_duplicate_returns_409(client):
    payload = {"salesforce_id": "SF_DUP_001", "name": "Dupe Corp"}
    await client.post("/api/v2/clients", json=payload)
    response = await client.post("/api/v2/clients", json=payload)
    assert response.status_code == 409
```

### Testing background tasks

FastAPI's `BackgroundTasks` run after the response is sent, which makes them invisible to standard request-response tests. Use `anyio` task groups or mock `BackgroundTasks`:

```python
from unittest.mock import AsyncMock, patch

async def test_sync_task_is_enqueued_on_create(client):
    with patch("app.routers.clients.enqueue_sync_task", new_callable=AsyncMock) as mock_enqueue:
        response = await client.post(
            "/api/v2/clients",
            json={"salesforce_id": "SF_BG_001", "name": "BG Corp"},
        )
        assert response.status_code == 201
        mock_enqueue.assert_called_once_with("SF_BG_001")
```

**Common mistake:** Mixing sync and async fixtures. A `scope="session"` async fixture requires `asyncio_mode = "auto"` and a session-scoped event loop. Without explicit event loop configuration, pytest-asyncio creates a new event loop per test, causing session-scoped async fixtures to fail. Set `asyncio_mode = "auto"` globally and use `@pytest.fixture(scope="session", loop_scope="session")` for session-scoped async fixtures in pytest-asyncio 0.23+.

---

## 7. Testing ETL Pipelines

ETL pipelines are notoriously hard to test because their value is in transforming data correctly at scale, but tests must be fast and isolated. The strategy: unit-test transforms, integration-test the full pipeline against Testcontainers, and use snapshot testing for complex output shapes.

### Unit testing a Salesforce → BigQuery field transform

```python
# tests/unit/test_account_transform.py
from hypothesis import given
import hypothesis.strategies as st
from etl.transforms.account import transform_account_to_bq_row

@given(
    name=st.text(min_size=1, max_size=255),
    sf_id=st.from_regex(r"001[A-Z0-9]{15}", fullmatch=True),
    annual_revenue=st.one_of(st.none(), st.floats(min_value=0, max_value=1e12)),
)
def test_transform_never_drops_required_fields(name, sf_id, annual_revenue):
    account = {"Id": sf_id, "Name": name, "AnnualRevenue": annual_revenue}
    row = transform_account_to_bq_row(account)
    assert "salesforce_id" in row
    assert "account_name" in row
    assert row["salesforce_id"] == sf_id

def test_transform_nulls_annual_revenue_when_missing():
    account = {"Id": "001ABC123ABC12345", "Name": "Test"}
    row = transform_account_to_bq_row(account)
    assert row.get("annual_revenue") is None
```

### Integration test: full ETL pipeline with Testcontainers

```python
# tests/integration/test_etl_pipeline.py
import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.core.container import DockerContainer
from etl.pipeline import AccountSyncPipeline

@pytest.fixture(scope="module")
def pipeline(postgres_container):
    """Wire up the ETL pipeline against a real PostgreSQL source."""
    return AccountSyncPipeline(
        source_dsn=postgres_container.get_connection_url(),
        bq_project="test-project",
        bq_dataset="test_dataset",
        dry_run=True,  # capture output rows without writing to BQ
    )

def test_pipeline_processes_all_staged_accounts(pipeline, db_session):
    # Seed source table
    db_session.execute(
        "INSERT INTO staged_accounts (salesforce_id, name) VALUES ('SF_ETL_001', 'ETL Corp')"
    )
    db_session.commit()

    result = pipeline.run()
    assert result.rows_processed == 1
    assert result.rows_failed == 0
    assert result.output_rows[0]["salesforce_id"] == "SF_ETL_001"
```

**Common mistake:** Testing ETL pipelines by asserting on log output rather than on the actual transformed data. Logs are presentation, not behavior. Assert on the rows written, the records updated, and the error counts — not on whether the logger emitted the right string.

---

## 8. Testing Salesforce Integrations

Apex callouts to external services cannot hit real endpoints during unit tests — Salesforce enforces this with `System.CalloutException`. The standard pattern is to implement `HttpCalloutMock` and inject it in tests. On the Python side, the pattern is to mock the HTTP layer that receives Apex's requests.

### Mocking inbound Apex callouts in Python tests

When crm-middleware receives a callout from Apex, you test the middleware endpoint independently. The question is: does the middleware correctly handle the JSON shape that Apex actually sends?

```python
# tests/unit/test_apex_callout_handler.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

# This is the exact JSON shape the Apex AccountTriggerHandler sends
APEX_ACCOUNT_PAYLOAD = {
    "SalesforceId": "001ABC123ABC12345",
    "Name": "Acme Insurance",
    "BillingCity": "Chicago",
    "BillingState": "IL",
    "APP__EHR_Client_Id__c": None,
    "APP__Sync_Status__c": "Pending",
}

async def test_middleware_accepts_apex_account_shape(client):
    response = await client.post("/api/v2/clients", json=APEX_ACCOUNT_PAYLOAD)
    assert response.status_code in (200, 201)
    assert "client_id" in response.json()

async def test_middleware_returns_ehr_id_for_apex_to_write_back(client):
    response = await client.post("/api/v2/clients", json=APEX_ACCOUNT_PAYLOAD)
    data = response.json()
    # Apex reads this field and writes it back to APP__EHR_Client_Id__c
    assert "ehr_client_id" in data
    assert isinstance(data["ehr_client_id"], str)
```

### Testing Apex itself with HttpCalloutMock

For completeness, the Apex side (not Python) uses this pattern:

```apex
// AccountTriggerHandlerTest.cls
@IsTest
private class AccountTriggerHandlerTest {
    @IsTest
    static void testSyncToEHR_CreatesClient() {
        Test.setMock(HttpCalloutMock.class, new CRMMiddlewareMock(201, '{"ehr_client_id":"EHR_001"}'));
        Account acc = new Account(Name = 'Test Corp', BillingCity = 'Chicago');
        insert acc;
        // Assert APP__EHR_Client_Id__c was set by the trigger
        acc = [SELECT APP__EHR_Client_Id__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals('EHR_001', acc.APP__EHR_Client_Id__c);
    }
}
```

**Common mistake:** Writing Apex mock responses that are more complete or well-formed than what the real middleware actually returns. Over time, the mock drifts from reality, and the Apex tests pass while the real integration is broken. Pact contracts solve this: the middleware's verified Pact interactions become the source of truth for the mock response shape.

---

## 9. Test Organization and CI Strategy

A well-organized test suite scales to large codebases without becoming a maintenance burden. Use pytest marks to separate test layers:

```python
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: fast, no I/O",
    "integration: requires Docker (Testcontainers)",
    "contract: Pact consumer/provider tests",
    "e2e: requires live external services",
]
```

```python
# tests/integration/test_queue_table.py
import pytest

@pytest.mark.integration
def test_enqueue_and_dequeue_preserves_order(db_session):
    ...
```

### CI pipeline design

```yaml
# .github/workflows/test.yml (excerpt)
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: pytest -m unit --tb=short -q

  integration:
    runs-on: ubuntu-latest
    services:
      docker:
        image: docker:dind
    steps:
      - run: pytest -m integration --tb=short

  contract:
    runs-on: ubuntu-latest
    steps:
      - run: pytest -m contract
      - run: pact-broker can-i-deploy --pacticipant CRMMiddleware --version ${{ github.sha }}

  mutation:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - run: mutmut run --paths-to-mutate app/sync/
```

**Common mistake:** Running all test layers in a single `pytest` invocation without marks or parallel workers. This serializes Testcontainers startup (which is slow) with fast unit tests, making the total CI time proportional to the slowest layer. Use `pytest-xdist` with `-n auto` for unit tests and run integration tests in a separate job.

---

## 10. Key Concepts Summary

```
Advanced Testing Strategies
├── Test Pyramid
│   ├── Unit (fast, isolated, most numerous)
│   ├── Integration (real I/O, Testcontainers)
│   ├── Contract (Pact, no live provider needed)
│   └── E2E (few, slow, high confidence)
│
├── Property-Based Testing (Hypothesis)
│   ├── @given + strategies
│   ├── Invariant assertions (idempotency, monotonicity)
│   ├── Shrinking (minimal failing example)
│   └── RuleBasedStateMachine (queue, state machines)
│
├── Contract Testing (Pact)
│   ├── Consumer defines expectations
│   ├── Provider verifies without live consumer
│   ├── Pact Broker as contract registry
│   └── "Can I Deploy" gate in CI
│
├── Mutation Testing (mutmut)
│   ├── Introduces bugs automatically
│   ├── Kills = test caught the mutation
│   ├── Survivors = gaps in test suite
│   └── Run on PRs, not every commit
│
├── Testcontainers
│   ├── Real PostgreSQL (no SQLite dialect gaps)
│   ├── Session-scoped container, function-scoped session
│   └── Async SQLAlchemy + asyncpg variant
│
├── Async Testing (pytest-asyncio)
│   ├── asyncio_mode = "auto"
│   ├── AsyncClient + ASGITransport for FastAPI
│   └── Background task mocking
│
└── Integration Platform Applications
    ├── Pact: Apex callout ↔ middleware JSON contract
    ├── Hypothesis: deduplication invariants + queue state machine
    ├── Testcontainers: queue table logic + ETL source tables
    └── CI layers: unit → integration → contract → mutation
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the test pyramid, and why does an inverted pyramid cause CI problems?

**2.** In Hypothesis, what is "shrinking" and why is it valuable?

**3.** What does `@given(events=st.lists(...))` do differently from writing a parametrized test with a fixed list of inputs?

**4.** What is consumer-driven contract testing and how does it differ from a shared integration test environment?

**5.** In Pact, what is the difference between using exact values and using matchers like `Like()` in consumer tests?

**6.** What is a "surviving mutant" in mutation testing, and what does it indicate about your test suite?

**7.** Why should mutation testing not run on every CI commit for large codebases?

**8.** What advantage does Testcontainers provide over using an in-memory SQLite database for PostgreSQL integration tests?

**9.** In the Testcontainers pytest fixture pattern, why should the container and engine use `scope="session"` but the database session use `scope="function"`?

**10.** What is `asyncio_mode = "auto"` in pytest-asyncio and what problem does it solve?

**11.** Why can't Apex unit tests make real HTTP callouts, and what mechanism does Salesforce provide instead?

**12.** How does Pact prevent mock drift between the Apex `HttpCalloutMock` responses and the real middleware behavior?

**13.** What invariant properties can Hypothesis test for a deduplication function that a fixed set of example-based tests might miss?

**14.** Describe how you would structure CI for the crm-middleware to balance speed (fast feedback) against coverage (real I/O tests).

**15.** What is the `RuleBasedStateMachine` in Hypothesis and what kind of system is it suited for?

**16.** When testing a FastAPI background task with `BackgroundTasks`, why does a standard response assertion miss whether the task ran, and how do you fix it?

**17.** What pytest mark strategy allows you to run unit tests in isolation without triggering Docker container startups?

**18.** In ETL pipeline testing, why is asserting on log output a poor substitute for asserting on transformed data?

**19.** What is the "Can I Deploy" check in the Pact workflow, and at what point in CI should it gate a deployment?

**20.** How does property-based testing complement mutation testing? Can one replace the other?

---

### Answers

??? note "Reveal Answers"

    **1.** The test pyramid organizes tests into three layers: unit tests at the base (many, fast, isolated), integration tests in the middle (fewer, real I/O), and E2E tests at the top (few, slow). An inverted pyramid — where most tests are E2E or integration — causes CI to be slow and flaky because E2E tests depend on live services, network availability, and external state that changes unpredictably. The result is tests that fail for reasons unrelated to code changes, eroding trust in CI and causing teams to ignore failures or disable tests.

    **2.** Shrinking is Hypothesis's process of reducing a failing input to its minimal form after it finds a counterexample. When Hypothesis generates a large random input that causes a test to fail, it systematically simplifies the input — removing list elements, shortening strings, reducing numbers — until it finds the smallest input that still reproduces the failure. This is valuable because it presents the developer with a clear, understandable counterexample rather than a complex random one, making the bug much easier to diagnose and fix.

    **3.** `@given(events=st.lists(...))` instructs Hypothesis to generate hundreds or thousands of different lists, varying in length, element values, and edge cases (empty lists, single elements, lists with duplicate keys), and to run the test function once for each generated input. A parametrized test with a fixed list only tests the specific cases the developer thought to include. Hypothesis explores the input space systematically and is much more likely to discover edge cases — like empty inputs, duplicate external IDs, or boundary timestamp values — that fixed examples miss.

    **4.** Consumer-driven contract testing means the consumer of an API (e.g., the Apex callout or ETL job) defines the exact requests it will make and responses it needs, and these expectations become a contract artifact that the provider (the middleware) verifies independently. Unlike a shared integration test environment — where both consumer and provider must be running simultaneously, in compatible versions, with shared test data — Pact decouples verification: the consumer test generates a contract file, and the provider verifies it against its own running instance, with no consumer present. This eliminates environment coupling and makes integration tests runnable in any CI environment.

    **5.** Using exact values in Pact consumer tests (e.g., `"id": "abc-123"`) means the provider must return exactly that value to pass verification. This is almost always wrong for dynamic fields like generated IDs, timestamps, or UUIDs, because the provider will return a different value each time. Matchers like `Like("abc-123")` assert that the field is present and has the same type (string) without requiring an exact match. `EachLike({...})` asserts that a field is an array where each element matches a given shape. Using matchers makes contracts robust and focused on API shape rather than specific data values.

    **6.** A surviving mutant is a code mutation (e.g., `>=` changed to `>`, a condition removed, a return value changed) that did not cause any test to fail. It indicates that your test suite does not cover the behavior that the mutation affects — you have a gap in coverage for that particular code path or boundary condition. Surviving mutants are actionable: each one points to a specific line of code and a specific missing test. A high mutation score (many killed mutants) indicates a test suite that is sensitive to behavioral changes; a low score indicates tests that execute code without actually asserting on its behavior.

    **7.** Mutation testing is computationally expensive: for each mutant, it re-runs the entire test suite. If a codebase has 1,000 mutants and the test suite takes 30 seconds, a full mutation run takes roughly 8 hours. Running this on every commit would block developers waiting for CI feedback. Instead, mutation testing is best run nightly, weekly, or scoped to files changed in a pull request (using `--paths-to-mutate` targeting the diff). This provides the quality signal without blocking the development loop.

    **8.** SQLite and PostgreSQL have different SQL dialects, different constraint enforcement behavior, different index strategies, and different handling of concurrent writes. Code that works perfectly against SQLite can fail against PostgreSQL due to dialect-specific syntax (e.g., `ON CONFLICT DO UPDATE`), stricter foreign key enforcement, different NULL handling in indexes, or transaction isolation differences. Testcontainers runs a real PostgreSQL container, so your integration tests catch these issues before deployment. This is especially important for integration platform queue table logic, which uses PostgreSQL-specific features like advisory locks and `FOR UPDATE SKIP LOCKED`.

    **9.** The container and engine are expensive to create — starting a Docker container takes several seconds, and creating a SQLAlchemy engine runs initial connection setup. Reusing them across the entire test session is efficient. The database session, however, must be isolated per test: each test should start with a clean transaction and roll it back at the end so that data inserted by one test does not affect another. Using `scope="function"` for the session ensures every test gets a fresh, empty transaction, making tests order-independent and preventing data pollution.

    **10.** `asyncio_mode = "auto"` tells pytest-asyncio to automatically detect and run `async def` test functions as coroutines without requiring the `@pytest.mark.asyncio` decorator on every individual test. Without this setting, async test functions that are missing the decorator are silently collected and executed as synchronous functions — they return a coroutine object immediately without awaiting it, and the test passes trivially without ever running the actual async code. Setting `asyncio_mode = "auto"` prevents this silent false-pass footgun and reduces boilerplate.

    **11.** Salesforce's platform sandbox enforces a rule that Apex code executed during tests cannot make real outbound HTTP callouts, throwing `System.CalloutException` if attempted. This is by design — it prevents tests from depending on external service availability and avoids unintended side effects in external systems. Salesforce provides the `HttpCalloutMock` interface as the replacement: developers implement the interface to return a controlled `HttpResponse`, register it with `Test.setMock()`, and the Apex HTTP classes use the mock during test execution instead of making a real network request.

    **12.** Pact creates a verified link between the mock and reality. The consumer test generates a pact file that records the exact request/response interaction, including the JSON shape the consumer expects. The provider (the real middleware) runs a verification step in CI that replays every interaction from the pact file against the actual running FastAPI application and asserts the responses match. If a middleware developer changes the response schema — renaming `ehr_client_id` to `ehrClientId` — the provider verification fails in middleware CI, preventing the breaking change from being deployed. The Pact Broker's "Can I Deploy" gate then blocks the Salesforce deployment until the consumer and provider are compatible.

    **13.** Hypothesis can test invariants that hold for all valid inputs: deduplication is idempotent (running it twice gives the same result as once), deduplication never increases the number of events, each external ID appears at most once in the output, and the surviving event for a given ID is always the one with the latest `updated_at` timestamp. These invariants are much harder to verify exhaustively with fixed examples because edge cases like exactly two events with the same ID and equal timestamps, or a list of 50 events where 49 have the same ID, require explicit thought to include. Hypothesis generates these cases automatically.

    **14.** A well-structured CI for crm-middleware uses separate jobs per layer: a `unit` job that runs fast (`pytest -m unit`) with no Docker, completing in under a minute; an `integration` job that uses Testcontainers (requires Docker-in-Docker) and runs in 2–5 minutes; a `contract` job that runs Pact consumer and provider verification in parallel; and a nightly `mutation` job scoped to changed files. The `unit` job runs on every push and is the primary developer feedback loop. The `integration` and `contract` jobs run on pull requests. This structure means most commits get feedback in under 2 minutes while still having real-database and contract coverage on every PR.

    **15.** `RuleBasedStateMachine` is a Hypothesis class for testing stateful systems. You define rules (methods decorated with `@rule`) that represent valid state transitions, and an invariant (decorated with `@invariant`) that must hold after every rule application. Hypothesis generates sequences of rule applications — essentially random but valid state machine paths — and checks the invariant after each step. It is suited for systems that are naturally stateful: queues (enqueue/dequeue/fail/retry), caches (set/get/evict), event sourcing systems, and any system where the order of operations matters. For the integration platform's sync queue, it can verify that the queue never delivers an event more times than it was enqueued.

    **16.** FastAPI's `BackgroundTasks` run after the HTTP response has been sent to the client. A test that only asserts `response.status_code == 201` never waits for the background task to complete and has no visibility into whether it ran, what it did, or whether it failed. To test background task behavior, either mock the task function with `unittest.mock.AsyncMock` and assert it was called with the correct arguments (testing that the endpoint correctly schedules the task), or use `anyio` task groups to await the task directly in a unit test (testing the task's logic itself). Both approaches give you deterministic, fast tests without relying on timing or thread scheduling.

    **17.** pytest marks combined with the `-m` flag allow selective test execution. By decorating Testcontainers-dependent tests with `@pytest.mark.integration`, you can run only unit tests with `pytest -m unit`, which never triggers Docker container creation. The `pyproject.toml` `markers` configuration registers the marks and suppresses the "unknown mark" warning. For local development, engineers can run `pytest -m unit` for a fast feedback loop during coding and `pytest -m "unit or integration"` before pushing. CI jobs use the same marks to define what each job runs without maintaining separate test directories.

    **18.** Log output is a side effect of implementation, not a specification of behavior. If the ETL transform function logs "processed 5 rows" but actually wrote corrupted data to the output table, a log-based assertion would pass while the real failure is invisible. Logs can also change without changing behavior — adding a debug log line would break log-based tests unnecessarily. Assertions on transformed data — the actual rows in the output table, the count of processed records, the field values in the BigQuery rows — are direct assertions on what the ETL is supposed to do. They are stable under refactoring and fail precisely when the behavior changes.

    **19.** The "Can I Deploy" check queries the Pact Broker to determine whether a specific version of a pacticipant (consumer or provider) is compatible with all the other pacticipants it integrates with in a target environment. It should gate deployments immediately before the actual deployment step — after all tests pass but before `kubectl apply` or `helm upgrade` runs. For the integration platform, it would check: "Is this version of CRMMiddleware compatible with all known consumers (ApexAccountTrigger, ETL jobs) that are currently deployed in the target environment?" If a breaking change was introduced and the consumer has not yet been updated, "Can I Deploy" returns false and the deployment is blocked.

    **20.** Property-based testing and mutation testing are complementary and neither can replace the other. Property-based testing is a test-writing technique: it generates inputs to verify that your code satisfies invariants. Mutation testing is a test-quality measurement technique: it verifies that your tests are sensitive to behavioral changes. You can have a property-based test that is poorly written — testing a trivially true invariant — and mutation testing would reveal that surviving mutants are not caught by it. Conversely, mutation testing tells you there is a gap but does not tell you what input to use; property-based testing is a powerful way to fill that gap once the survivor points you at the right code. Use Hypothesis to build high-quality assertions, and mutmut to verify that those assertions actually exercise the full behavioral space.
