# Week 24 — Observability — Logs, Metrics, Traces

**Week of:** November 16, 2026
**Estimated study time:** ~2 hours
**Tags:** `observability` `monitoring` `logging` `tracing`

---

## Overview

Observability is the property of a system that lets you understand its internal state from its external outputs. Unlike traditional monitoring — which answers "is the system up?" — observability answers "why is the system behaving the way it is?" This distinction matters enormously as systems grow in complexity. A single Salesforce trigger that kicks off an integration platform sync can fan out into dozens of downstream calls: the middleware API, the EHR system BDE backend, PostgreSQL queue tables, and Kubernetes sidecars. When something breaks silently — a record that never synced, a correlation ID that got dropped — you need more than an uptime check. You need to reconstruct the exact chain of events from structured evidence.

The three pillars of observability are logs, metrics, and traces. Logs are timestamped, structured records of discrete events. Metrics are numerical measurements sampled over time. Traces are causal chains of operations that span service boundaries, grouped under a single originating request. Each pillar answers different questions. Logs tell you what happened. Metrics tell you how much or how often. Traces tell you where time was spent and which service was responsible. A mature observability strategy uses all three in concert, with correlation IDs as the thread that ties them together.

For a Python/FastAPI stack on GKE with Datadog, the practical implementation involves three complementary tools: `structlog` for structured JSON log emission, OpenTelemetry (OTel) for vendor-neutral trace instrumentation, and the Datadog `ddtrace` library for APM integration. These are not competing choices — `ddtrace` can consume OTel trace context, and structlog can inject the active trace ID into every log line so that Datadog Log Management can correlate a single log entry directly to its parent span. The combination turns isolated log lines into navigable audit trails.

This week covers each pillar in depth, walks through FastAPI instrumentation patterns relevant to the crm-middleware, and examines alerting design principles to avoid the alert fatigue that undermines on-call effectiveness. By the end you should be able to instrument a new FastAPI endpoint so that every request emits a structured log, increments the right counters, and produces a distributed trace that surfaces in Datadog APM — all with the EHR sync correlation ID visible in every signal.

---

## 1. Structured Logging with structlog

Plain-text logs are hostile to machines. A log line like `INFO: synced account 001Xx0000012345` cannot be filtered, aggregated, or correlated without fragile regex. Structured logging treats each log event as a dictionary: every field has a name and a value that a log platform can index and query.

`structlog` is the Python standard for structured logging. It wraps the stdlib `logging` module with a processor pipeline that transforms each log call into a dictionary before serialization. The processors run in order, each receiving and optionally mutating the event dict.

```python
# crm-middleware/app/core/logging.py
import logging
import structlog
from structlog.contextvars import merge_contextvars, bind_contextvars, clear_contextvars

def configure_logging(log_level: str = "INFO") -> None:
    shared_processors = [
        merge_contextvars,                        # pull in request-scoped context vars
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]

    structlog.configure(
        processors=shared_processors + [
            structlog.stdlib.ProcessorFormatter.wrap_for_formatter,
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    formatter = structlog.stdlib.ProcessorFormatter(
        processor=structlog.processors.JSONRenderer(),
        foreign_pre_chain=shared_processors,
    )

    handler = logging.StreamHandler()
    handler.setFormatter(formatter)
    root_logger = logging.getLogger()
    root_logger.addHandler(handler)
    root_logger.setLevel(log_level)
```

With context vars, you bind request-scoped fields once per request and they appear in every subsequent log call within that request — no need to thread a logger object through every function.

```python
# In a FastAPI middleware
from structlog.contextvars import bind_contextvars, clear_contextvars
import uuid

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    clear_contextvars()
    correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))
    bind_contextvars(
        correlation_id=correlation_id,
        path=request.url.path,
        method=request.method,
    )
    response = await call_next(request)
    bind_contextvars(status_code=response.status_code)
    log = structlog.get_logger()
    log.info("request_complete")
    return response
```

Every log call anywhere in the call stack during this request will now carry `correlation_id`, `path`, `method`, and `status_code` automatically.

**Applied to the integration platform:** When a Salesforce trigger calls the middleware `/sync/account` endpoint, bind the Salesforce record ID and the EHR client ID immediately. Any downstream log — including PostgreSQL query logs forwarded via the Datadog agent — will carry those IDs, making it trivial to find every log line for a single sync attempt in Datadog Log Management with a single query: `@correlation_id:abc-123`.

**Common mistake:** Logging sensitive data (PII fields like SSN or DOB from the EHR system) at DEBUG level and forgetting that debug logs are enabled in staging. Always log EHR response payloads at TRACE level (or suppress entirely) and scrub known PII fields in a structlog processor before the JSON renderer runs.

---

## 2. Metrics: Counters, Gauges, and Histograms

A metric is a numerical measurement with a name, a value, a timestamp, and a set of labels (dimensions). The three fundamental metric types behave differently and answer different questions.

| Type | Behavior | Resets? | Best for |
|------|----------|---------|----------|
| Counter | Monotonically increasing integer | On restart | Request counts, error counts, bytes sent |
| Gauge | Arbitrary current value, up or down | N/A | Queue depth, active connections, memory usage |
| Histogram | Samples values into buckets; tracks count + sum | On restart | Latency distributions, payload sizes |

Prometheus is the de-facto metrics format for Kubernetes workloads. Your FastAPI app exposes a `/metrics` endpoint; Prometheus scrapes it on a configurable interval; Datadog's Kubernetes integration can scrape Prometheus endpoints and forward metrics to Datadog automatically.

```python
# crm-middleware/app/metrics.py
from prometheus_client import Counter, Gauge, Histogram, CollectorRegistry, generate_latest
from fastapi import APIRouter
from fastapi.responses import PlainTextResponse

registry = CollectorRegistry()

SYNC_REQUESTS_TOTAL = Counter(
    "platform_sync_requests_total",
    "Total sync requests to the middleware",
    ["object_type", "direction", "status"],  # labels
    registry=registry,
)

EHR_API_LATENCY = Histogram(
    "platform_ehr_api_duration_seconds",
    "Duration of outbound EHR API calls",
    ["endpoint", "method"],
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
    registry=registry,
)

QUEUE_DEPTH = Gauge(
    "platform_sync_queue_depth",
    "Number of records pending sync in the middleware queue",
    ["object_type"],
    registry=registry,
)

metrics_router = APIRouter()

@metrics_router.get("/metrics", response_class=PlainTextResponse)
def metrics_endpoint():
    return generate_latest(registry)
```

Usage in a sync handler:

```python
with EHR_API_LATENCY.labels(endpoint="/clients", method="PUT").time():
    response = await ehr_client.put_client(client_payload)

SYNC_REQUESTS_TOTAL.labels(
    object_type="Account",
    direction="sf_to_ehr",
    status="success" if response.ok else "error",
).inc()
```

**Prometheus scrape config for GKE** (annotate the Pod):

```yaml
# In infra-manifests/kubernetes/values.staging.yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: "/metrics"
  prometheus.io/port: "8000"
```

**Common mistake:** Using high-cardinality values as label dimensions — for example, using the Salesforce record ID as a label. Each unique label combination creates a new time series. With millions of accounts, this explodes Prometheus storage and query latency. Labels should be low-cardinality categories: object type, endpoint path, status code.

---

## 3. Prometheus Scraping and the Datadog Agent

Prometheus scraping is a pull model: the Prometheus server (or Datadog agent acting as a Prometheus collector) periodically fetches the `/metrics` endpoint from each target. The Datadog agent running as a DaemonSet on GKE can be configured to auto-discover annotated Pods and scrape their Prometheus metrics, then forward them to Datadog as custom metrics.

The scrape interval is a key tuning parameter. A 15-second interval gives reasonable resolution for latency histograms while keeping cardinality manageable. For queue depth gauges where you want near-real-time alerting, 15s is usually sufficient — Datadog alerting evaluates metrics at 1-minute granularity by default anyway.

```yaml
# Datadog agent autodiscovery annotation on the middleware Pod
annotations:
  ad.datadoghq.com/middleware.checks: |
    {
      "openmetrics": {
        "init_config": {},
        "instances": [{
          "openmetrics_endpoint": "http://%%host%%:8000/metrics",
          "namespace": "platform",
          "metrics": ["platform_sync_requests_total", "platform_ehr_api_duration_seconds", "platform_sync_queue_depth"]
        }]
      }
    }
```

This surfaces metrics in Datadog under `platform.sync_requests_total`, `platform.ehr_api_duration_seconds.*`, and `platform.sync_queue_depth` with all original Prometheus labels preserved as Datadog tags.

**Common mistake:** Forgetting the `namespace` field in the OpenMetrics instance config. Without it, Datadog ingests the raw Prometheus metric names, which can collide with existing metrics and are harder to search. Always namespace your custom metrics.

---

## 4. Distributed Tracing and OpenTelemetry

A trace represents the end-to-end journey of a single request through your distributed system. It is composed of spans — named, timed operations. The root span covers the entire request; child spans cover sub-operations (database queries, outbound HTTP calls, message queue pops). Every span carries a `trace_id` (same for all spans in a trace), a `span_id` (unique per span), and a `parent_span_id` that encodes the causal relationship.

OpenTelemetry (OTel) is the CNCF standard for trace instrumentation. It provides a vendor-neutral API and SDK; exporters forward spans to any backend (Datadog, Jaeger, Zipkin, etc.) without changing application code.

```python
# crm-middleware/app/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

def configure_tracing(service_name: str = "crm-middleware") -> None:
    provider = TracerProvider()
    exporter = OTLPSpanExporter(
        endpoint="http://datadog-agent:4317",  # Datadog agent OTLP receiver
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    # Auto-instrument FastAPI, HTTPX (for EHR API calls), and SQLAlchemy
    FastAPIInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()
    SQLAlchemyInstrumentor().instrument()
```

After `configure_tracing()` is called at startup, every incoming HTTP request automatically gets a root span, and every outbound EHR API call and SQLAlchemy query gets a child span — all linked under one `trace_id`.

For custom spans around business logic:

```python
tracer = trace.get_tracer("platform.sync")

async def sync_account_to_ehr(sf_account_id: str, payload: dict) -> dict:
    with tracer.start_as_current_span("sync_account_to_ehr") as span:
        span.set_attribute("sf.account_id", sf_account_id)
        span.set_attribute("ehr.endpoint", "/clients")

        existing = await ehr_client.get_client(sf_account_id)
        span.set_attribute("ehr.client_exists", existing is not None)

        if existing:
            result = await ehr_client.put_client(sf_account_id, payload)
        else:
            result = await ehr_client.post_client(payload)

        span.set_attribute("ehr.response_status", result.status_code)
        return result.json()
```

**Common mistake:** Creating spans but forgetting to set meaningful attributes. A span named `sync_account_to_ehr` with no attributes is far less useful than one that carries `sf.account_id`, `ehr.client_id`, `ehr.endpoint`, and `ehr.response_status`. Span attributes are what make traces searchable and filterable in Datadog APM.

---

## 5. Trace Context Propagation

Trace context propagation is how spans in different services get linked to the same trace. When Service A calls Service B, it injects the current trace context (trace ID + parent span ID) into the HTTP headers. Service B extracts the context and creates a child span, preserving the causal link.

The W3C `traceparent` header is the standard format:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^
              version  trace-id (128-bit hex)        parent-span-id   flags
```

OpenTelemetry's `HTTPXClientInstrumentor` handles injection automatically for outbound HTTPX calls. For the integration platform scenario where a Salesforce Flow or Apex trigger initiates the call, the originating `X-Correlation-ID` from Salesforce should be mapped to a span attribute so you can correlate across the Salesforce → Middleware → EHR system boundary even though Salesforce does not speak W3C traceparent.

```python
@app.middleware("http")
async def trace_correlation_middleware(request: Request, call_next):
    span = trace.get_current_span()
    sf_correlation_id = request.headers.get("X-Salesforce-Request-ID")
    if sf_correlation_id and span.is_recording():
        span.set_attribute("sf.request_id", sf_correlation_id)
        span.set_attribute("sf.object_type", request.headers.get("X-Object-Type", "unknown"))
    return await call_next(request)
```

Now in Datadog APM you can search for `@sf.request_id:your-id` and find the exact trace for that Salesforce operation.

**Common mistake:** Propagating trace context over message queues or async jobs without explicit extraction. If the middleware writes a record to a PostgreSQL queue table and a background worker reads it later, the trace context is lost unless you serialize the `traceparent` header value into a column and extract it when processing. Use OTel's `inject`/`extract` APIs with a dict carrier to handle this.

---

## 6. Datadog APM and Log Correlation

Datadog APM is built on ddtrace, which can ingest OTel spans via the OTLP receiver on the Datadog agent. The key feature that makes Datadog especially powerful is automatic log correlation: when `ddtrace` is active, it injects `dd.trace_id` and `dd.span_id` into the Python logging context. structlog can pick these up and embed them in every JSON log line.

```python
# Add dd trace context to every structlog event
import ddtrace
from ddtrace import tracer as dd_tracer

def add_datadog_trace_context(logger, method, event_dict):
    """structlog processor that injects dd.trace_id and dd.span_id."""
    span = dd_tracer.current_span()
    if span:
        # Datadog uses decimal trace IDs in logs for correlation
        event_dict["dd.trace_id"] = str(span.trace_id & 0xFFFFFFFFFFFFFFFF)
        event_dict["dd.span_id"] = str(span.span_id)
        event_dict["dd.service"] = "crm-middleware"
        event_dict["dd.env"] = "staging"
    return event_dict
```

Add `add_datadog_trace_context` to the `shared_processors` list in `configure_logging()`. Now every log line carries `dd.trace_id`, and Datadog Log Management will show a "View Trace" link directly from any log entry, jumping you to the exact APM trace that was active when that log was emitted.

For the integration platform EHR sync, this means: you find a log line `"ehr_client_error": "timeout after 5s"`, click "View Trace", and immediately see the full waterfall — the FastAPI handler span, the SQLAlchemy SELECT span that fetched the queue record, and the HTTPX span to `ehr-server-prod/clients` that timed out — all in one view without leaving Datadog.

**Common mistake:** Using OTel's 128-bit trace IDs (as a 32-character hex string) in logs while Datadog Log Correlation expects 64-bit decimal trace IDs. The two formats are incompatible. When using ddtrace alongside OTel, always use `dd_tracer.current_span().trace_id & 0xFFFFFFFFFFFFFFFF` (the lower 64 bits) and convert to decimal string for the `dd.trace_id` log field.

---

## 7. Alerting Design and Avoiding Alert Fatigue

Alert fatigue is the state where on-call engineers begin ignoring or dismissing alerts because too many are noisy, flapping, or irrelevant. It is one of the most dangerous failure modes in operations: the alert that gets dismissed at 2 AM is the one that was real.

Good alert design starts with a taxonomy:

| Alert type | Property | Response time |
|------------|----------|---------------|
| Page (SEV-1) | Customer impact, SLO breach imminent | Immediate |
| Ticket (SEV-3) | Degraded state, no immediate customer impact | Next business day |
| Dashboard / informational | Trend worth watching | No response required |

For the integration platform, alerts should be tied to SLO breach risk, not to symptom thresholds. Instead of "alert when EHR API latency > 2s", prefer "alert when the 95th-percentile EHR API latency has exceeded 2s for 10 consecutive minutes" — because a single slow call is noise; a sustained degradation is a problem.

```
# Datadog monitor: integration platform EHR sync lag SLO
Monitor type: Metric Alert
Query: avg(last_10m):avg:platform.sync_queue_depth{object_type:Account,env:staging} > 500

Alert: "Account sync queue depth > 500 for 10m — possible EHR API backpressure"
Warning: 200
Recovery: 50

Notification: @pagerduty-platform-oncall (critical), @slack-platform-eng (warning)
```

For error rate monitors, use anomaly detection rather than static thresholds when the baseline fluctuates (e.g., sync volume is much lower on weekends):

```
# Anomaly detection for sync error rate
avg(last_30m):anomalies(
  sum:platform.sync_requests_total{status:error}.as_rate() /
  sum:platform.sync_requests_total{}.as_rate(),
  "agile", 3
) >= 1
```

**Alert fatigue reduction checklist:**
- Every alert must have a runbook URL in the notification body.
- Every SEV-1 page must be tied to an SLO, not an arbitrary threshold.
- After any false positive, update the threshold or add a filter — do not dismiss and move on.
- Run a monthly alert audit: any alert that paged and was immediately acknowledged-without-action three times in a row should be downgraded or eliminated.

**Common mistake:** Creating an alert for every metric that exists. Start with the golden signals (latency, traffic, errors, saturation) for each service boundary, then add specifics only when an incident reveals a gap. More alerts does not mean better coverage — it means more noise.

---

## 8. The OpenTelemetry Collector for GKE

In a Kubernetes environment, it is best practice to deploy an OTel Collector as a DaemonSet or Deployment rather than having each app pod export spans directly to Datadog. The Collector receives spans from all pods, batches them, and forwards to Datadog (or any backend). This decouples your application code from the observability backend and enables sampling decisions at the Collector layer.

```yaml
# Simplified OTel Collector config (infra-manifests)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000
  resource:
    attributes:
      - key: deployment.environment
        value: staging
        action: upsert

exporters:
  datadog:
    api:
      key: ${DATADOG_API_KEY}
      site: datadoghq.com

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [datadog]
```

For the crm-middleware, each pod sends spans to `http://otel-collector:4317`. The Collector adds the `deployment.environment` resource attribute, which Datadog surfaces as the `env` tag — enabling environment-scoped APM views.

**Common mistake:** Using head-based sampling (drop at the collector before trace completion) without considering that it will drop error traces proportionally to their occurrence rate. For low-traffic error scenarios (e.g., an EHR API 500 that fires once per hour), head-based sampling at 10% will drop 90% of error traces. Use tail-based sampling with the `tail_sampling` processor to always keep error and slow traces.

---

## 9. End-to-End Integration Platform Observability: Tracing a Sync Request

Putting all the pieces together for a complete EHR sync trace:

```
Salesforce Trigger (AccountTriggerHandler.syncToEHR)
  │
  │  HTTP POST /sync/account
  │  Headers: X-Salesforce-Request-ID: sf-req-abc123
  │           traceparent: (not set — Salesforce doesn't emit OTel)
  ▼
FastAPI root span: POST /sync/account          [trace_id: aabbcc...]
  ├── Middleware: bind sf.request_id=sf-req-abc123
  ├── SQLAlchemy span: SELECT FROM sync_queue   [parent: root]
  │     attribute: db.statement, db.row_count
  ├── Custom span: sync_account_to_ehr         [parent: root]
  │   ├── HTTPX span: GET ehr-server-prod/clients/...   [parent: custom]
  │   │     attribute: http.status_code=200
  │   └── HTTPX span: PUT ehr-server-prod/clients/...   [parent: custom]
  │         attribute: http.status_code=204
  └── SQLAlchemy span: UPDATE sync_queue        [parent: root]
        attribute: db.statement

Every span carries: service=crm-middleware, env=staging, version=2.3.1
Every log line carries: dd.trace_id, dd.span_id, correlation_id, sf.request_id
```

In Datadog APM this renders as a flame chart. A 4-second total request breaks down as: 50ms queue SELECT, 200ms EHR GET, 3.5s EHR PUT, 50ms queue UPDATE. The EHR PUT is the bottleneck. You can click it, see the full `sf.request_id` and `sf.account_id` attributes, then pivot to Logs to see the structured log lines emitted during that PUT — including any retry attempts or error messages from the EHR system.

---

## 10. Key Concepts Summary

```
OBSERVABILITY
├── LOGS (what happened)
│   ├── structlog — structured JSON emission
│   ├── Context vars — bind request fields once, appear everywhere
│   ├── Processors — transform event dicts (add timestamps, inject trace IDs)
│   └── Datadog Log Management — index, search, correlate
│
├── METRICS (how much / how often)
│   ├── Counter — monotonically increasing (requests, errors)
│   ├── Gauge — current value (queue depth, connections)
│   ├── Histogram — distribution (latency buckets, payload size)
│   ├── Prometheus — pull-based scrape from /metrics endpoint
│   └── Datadog agent — autodiscovery scrape → custom metrics
│
├── TRACES (where time was spent)
│   ├── Span — named, timed operation with attributes
│   ├── Trace — tree of spans under one trace_id
│   ├── OpenTelemetry — vendor-neutral instrumentation API/SDK
│   ├── W3C traceparent — standard context propagation header
│   ├── OTel Collector — batching, sampling, backend routing
│   └── Datadog APM — flame charts, service map, error tracking
│
└── CORRELATION (tying it all together)
    ├── dd.trace_id in logs — click log → view trace in APM
    ├── sf.request_id span attribute — Salesforce → middleware link
    ├── correlation_id context var — appears in every log line
    └── Alerting — golden signals, SLO-based, runbook-linked
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between observability and traditional monitoring, and why does the distinction matter for distributed systems?

**2.** Name the three pillars of observability and describe what question each one is best suited to answer.

**3.** What is a structlog processor, and how does the processor pipeline differ from traditional Python logging formatters?

**4.** How do structlog context variables (`bind_contextvars`) work, and why are they preferable to passing a bound logger through every function call?

**5.** Explain the three Prometheus metric types — Counter, Gauge, and Histogram — and give a concrete integration platform example for each.

**6.** Why is using a Salesforce record ID as a Prometheus metric label a bad idea, and what should be used instead?

**7.** Describe the Prometheus pull scrape model and how the Datadog agent can be configured to autodiscover and scrape FastAPI pods on GKE.

**8.** What is a distributed trace, and how does parent/child span relationships encode causality across service calls?

**9.** What is the W3C `traceparent` header format, and which fields does it carry?

**10.** How does OpenTelemetry's `HTTPXClientInstrumentor` enable automatic trace context propagation in outbound HTTP calls?

**11.** Describe the problem of trace context loss in async workflows, and explain how to preserve trace context when writing to a PostgreSQL queue table.

**12.** What is the `dd.trace_id` field in Datadog log correlation, and why must it be a 64-bit decimal string rather than the full 128-bit OTel hex trace ID?

**13.** How does the OTel Collector's tail-based sampling differ from head-based sampling, and when should you use each?

**14.** What are the four golden signals, and which Datadog metric types (Counter/Gauge/Histogram) would you use to measure each for the crm-middleware?

**15.** Define alert fatigue and describe two concrete practices that reduce it in a Datadog alerting setup.

**16.** What is the difference between a SEV-1 page and a SEV-3 ticket in an alerting taxonomy, and how should thresholds differ between them?

**17.** Explain why anomaly detection monitors are more appropriate than static threshold monitors for integration platform sync error rates.

**18.** In the integration platform observability architecture described in section 9, which span would you examine first if the total request time was 8 seconds, and how would you use span attributes to investigate?

**19.** What does the `resource` processor do in the OTel Collector configuration, and why is the `deployment.environment` attribute important in Datadog?

**20.** A new engineer adds a histogram metric `platform_payload_size_bytes` with Salesforce object type as one label and the full JSON payload hash as another label. What problem will this cause, and how would you fix it?

---

### Answers

??? note "Reveal Answers"

    **1.** Traditional monitoring asks "is the system up?" by checking predefined thresholds on known symptoms — a ping succeeds, a process is running, CPU is below 80%. Observability goes further: it asks "why is the system behaving this way?" using logs, metrics, and traces to reconstruct internal state from external outputs. The distinction matters for distributed systems because failures rarely manifest as simple binary up/down states. A Salesforce sync might be "up" (the middleware responds with 200) while silently dropping records because of a subtle EHR API state mismatch. Observability surfaces that silent failure through correlated signals that monitoring alone cannot detect.

    **2.** The three pillars are logs, metrics, and traces. Logs answer "what happened?" — they are timestamped records of discrete events with contextual detail. Metrics answer "how much or how often?" — they are numerical measurements over time that enable trend analysis and alerting. Traces answer "where did the time go, and which service was responsible?" — they are causal chains of operations spanning multiple services, linked under a single originating request. Effective observability uses all three in conjunction, with a shared correlation identifier as the thread.

    **3.** A structlog processor is a callable that accepts `(logger, method, event_dict)` and returns a (potentially modified) `event_dict`. The pipeline is a sequential list of processors; each one transforms the event dictionary before the final renderer (e.g., `JSONRenderer`) serializes it. Unlike traditional formatters, which operate on a pre-formatted string, structlog processors work on structured data — so a processor can add fields (like timestamps or trace IDs), rename fields, scrub sensitive values, or entirely drop events based on arbitrary logic. This makes the pipeline composable and testable in isolation.

    **4.** Context variables (`bind_contextvars`) use Python's `contextvars.ContextVar` under the hood, which is coroutine-safe in asyncio. When you call `bind_contextvars(correlation_id="abc")` at the start of a request, that binding is automatically available to every `structlog.get_logger()` call on any code path within the same async context — including deeply nested utility functions that have no knowledge of the request. Passing a bound logger explicitly would require every function signature to accept a logger argument, which is invasive and brittle. Context vars give you ambient request context without coupling.

    **5.** A Counter is a monotonically increasing integer — reset only on restart. Integration platform example: `platform_sync_requests_total` labeled by `object_type` and `status` counts every sync attempt. A Gauge holds an arbitrary current value that can go up or down. Integration platform example: `platform_sync_queue_depth` labeled by `object_type` reflects how many records are waiting to sync right now. A Histogram samples values into pre-defined buckets and tracks total count and sum. Integration platform example: `platform_ehr_api_duration_seconds` with buckets at 0.05s through 10s measures how long EHR API calls take so you can compute p50, p95, p99 latency.

    **6.** Salesforce record IDs are unique per record — there can be millions of Account IDs. Each unique label value combination in Prometheus creates a new time series in memory and on disk. Using record IDs as labels would create a new time series per Account sync, making cardinality explode into the millions, which exhausts Prometheus memory, slows queries to a crawl, and can crash the Datadog agent. Instead, use low-cardinality categorical labels like `object_type` (Account, Contact, Opportunity), `status` (success, error, retry), and `endpoint` (/clients, /contacts).

    **7.** Prometheus uses a pull model: a Prometheus server (or Prometheus-compatible scraper like the Datadog agent) sends HTTP GET requests to each target's `/metrics` endpoint on a configurable interval (typically 15–60 seconds). The Datadog agent on GKE uses Kubernetes autodiscovery: it watches for Pods with specific annotations (`prometheus.io/scrape: "true"`, `prometheus.io/path`, `prometheus.io/port`) or with explicit `ad.datadoghq.com/<container>.checks` annotations. When a matching Pod is found, the agent adds it to its scrape list automatically — no manual configuration needed when Pods scale or restart.

    **8.** A distributed trace is a collection of spans — named, timed operations — linked by a shared `trace_id`. Each span has a unique `span_id` and a `parent_span_id` pointing to the span that initiated it, forming a tree (or directed acyclic graph). The root span represents the initial request; child spans represent sub-operations like database queries, outbound HTTP calls, or function-level processing. This tree encodes causality: by looking at the parent/child relationships, you can reconstruct exactly which operation triggered which, and attribute total request latency to specific sub-operations across any number of microservices.

    **9.** The W3C `traceparent` header has the format `version-trace_id-parent_span_id-flags`. The `version` is always `00`. The `trace_id` is a 128-bit value encoded as 32 lowercase hexadecimal characters, unique to the entire distributed trace. The `parent_span_id` is a 64-bit value (16 hex chars) identifying the current span being propagated — the receiving service treats this as the parent of its new span. The `flags` byte is currently used for the sampling bit: `01` means the trace was sampled, `00` means not sampled.

    **10.** `HTTPXClientInstrumentor().instrument()` monkey-patches the HTTPX client at the library level. When any HTTPX request is made, the instrumentation automatically creates a child span for that outbound call, records the HTTP method, URL, and response status as span attributes, and injects the current trace context into the request headers as a `traceparent` header. The receiving service (if also OTel-instrumented) extracts this header and creates its own child spans under the same trace. This happens transparently — application code using `httpx.AsyncClient` does not need to be modified.

    **11.** When a middleware handler writes a sync request to a PostgreSQL queue table and a background worker later reads it, the original trace context is not automatically carried across the boundary — the worker starts a new, disconnected trace. To preserve context, serialize the `traceparent` header value into a dedicated column (e.g., `trace_context VARCHAR(55)`) when inserting the record. When the worker picks up the record, extract the trace context using `opentelemetry.propagate.extract({'traceparent': row.trace_context})` and use it as the parent context for the worker's new span. This links the worker's trace back to the original request.

    **12.** Datadog APM internally uses 64-bit trace IDs. OpenTelemetry generates 128-bit trace IDs (32 hex chars). When ddtrace operates alongside OTel, it uses the lower 64 bits of the OTel trace ID as its own trace ID. Datadog Log Management correlates a log entry to a trace by matching the `dd.trace_id` field (decimal string of the lower 64 bits) against the APM trace store. If you embed the full 128-bit hex OTel trace ID in logs instead, Datadog cannot match it, and the "View Trace" link will not appear. The correct approach is `str(span.trace_id & 0xFFFFFFFFFFFFFFFF)` converted to a decimal string.

    **13.** Head-based sampling makes the keep/drop decision at the start of a trace, before any spans have been completed. It is computationally cheap but blind — it drops traces based on probability without knowing whether a trace will contain errors or be slow. Tail-based sampling buffers all spans for a trace and makes the keep/drop decision after the trace is complete, allowing rules like "always keep traces with an error span" or "always keep traces slower than 2 seconds." Use head-based sampling for high-volume, homogeneous traffic where you just need a statistical sample. Use tail-based sampling when you need guaranteed capture of error and slow traces, even at low volumes — which is the correct choice for integration platform EHR sync where errors are rare but operationally critical.

    **14.** The four golden signals are latency, traffic, errors, and saturation. Latency (how long requests take) is best measured with a Histogram (`platform_ehr_api_duration_seconds`) so you can compute percentiles. Traffic (request volume) is best measured with a Counter (`platform_sync_requests_total`) tracked as a rate over time. Errors (failed requests) are also a Counter (`platform_sync_requests_total` with `status=error` label) expressed as a ratio of total traffic. Saturation (how full the system is) is best measured with a Gauge (`platform_sync_queue_depth`) — a growing queue indicates the sync pipeline is saturated relative to EHR API throughput.

    **15.** Alert fatigue occurs when on-call engineers receive so many alerts (especially false positives or low-signal ones) that they begin habitually acknowledging without investigating, increasing the risk of missing a real incident. Two concrete practices to reduce it: First, require every SEV-1 page to link to a runbook with explicit diagnostic steps — if you can't write a runbook for an alert, the alert is not yet well-defined enough to page someone at 2 AM. Second, conduct monthly alert audits: any monitor that fired and was acknowledged-without-action three or more times in the past month should be demoted to a ticket or eliminated. The burden of proof is on keeping an alert, not on removing it.

    **16.** A SEV-1 page represents confirmed or imminent customer impact — an EHR sync backlog that will breach the SLA, authentication failures preventing any Salesforce users from syncing, or a complete middleware outage. It demands an immediate human response. A SEV-3 ticket represents a degraded state with no immediate customer impact — elevated error rate on a single object type, a single EHR API call taking longer than usual, a non-critical background job that failed. It requires investigation by the next business day. Thresholds for SEV-1 should be conservative (few false positives are worth tolerating) while SEV-3 thresholds can be more sensitive since the cost of a false positive is lower.

    **17.** Integration platform sync error rates are not constant: they are near-zero during nights and weekends when Salesforce users are not active, and higher during business hours. A static threshold like "error rate > 2%" would never fire on a Saturday (because volume is too low to generate 2%) and might fire spuriously on a Monday morning spike that is actually normal. Anomaly detection builds a model of expected behavior from historical data, accounting for time-of-day and day-of-week seasonality. It alerts when the current error rate deviates from the expected pattern by more than N standard deviations — so a 0.5% error rate at 2 AM (abnormal) triggers while a 1.8% rate during a Monday morning rush (normal) does not.

    **18.** With an 8-second total request, you would first look at the waterfall in Datadog APM to identify which span contributes most to the total duration. Based on the architecture in section 9, the EHR PUT span is the most likely culprit (it was 3.5s in the example; scaled up it could dominate an 8s total). Click on that span and examine its attributes: `http.status_code` (was it a retry after a 429 or 503?), `http.url` (confirm it is the correct EHR endpoint), and any custom attributes like `ehr.client_exists` (unexpected GET-before-PUT overhead?). Then pivot to the correlated logs for that span using `dd.trace_id` to find any timeout messages, retry attempt counts, or EHR system error response bodies that explain the slowdown.

    **19.** The `resource` processor in the OTel Collector adds or modifies resource attributes on all spans passing through the pipeline. Resource attributes describe the environment where the span was generated — the service, host, Kubernetes cluster, and environment. Adding `deployment.environment=staging` as a resource attribute means every span exported to Datadog will carry the `env:staging` tag. This is critical in Datadog because the environment tag gates APM Service Catalog views, SLO filters, and dashboard template variables. Without it, all spans from all environments would be mixed together in APM, making it impossible to view staging and production metrics independently.

    **20.** Adding the full JSON payload hash as a label creates unbounded cardinality — each unique payload produces a distinct hash, and because payloads vary per record, the number of unique label combinations equals the number of sync operations, potentially millions. This causes Prometheus memory exhaustion, crashes the scrape target, and makes any aggregation on that metric meaningless. The fix is to remove the payload hash label entirely. If payload size distribution matters, the histogram buckets already segment size ranges without needing cardinality. If you need to debug a specific large payload, use a log entry with the full payload (at an appropriate log level) rather than encoding payload identity into a metric label.
