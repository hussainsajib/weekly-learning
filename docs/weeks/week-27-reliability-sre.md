# Week 27 — Reliability Engineering & SRE Practices

**Week of:** December 7, 2026
**Estimated study time:** ~2 hours
**Tags:** `sre` `reliability` `devops` `incidents`

---

## Overview

Reliability engineering is the discipline of building systems that remain trustworthy under real-world load, partial failure, and human error. Google's Site Reliability Engineering (SRE) model formalized this into a set of practices — SLOs, error budgets, blameless post-mortems, and toil reduction — that have since become the baseline expectation for senior engineers at any company running production services. For a staff-track engineer, knowing *how to define* reliability targets is as important as knowing how to hit them.

The CRM-EHR Integration Platform is a production integration platform where reliability has a precise, business-visible consequence: when the middleware goes down or degrades, Salesforce and the EHR system fall out of sync, creating data drift that takes hours of reconciliation work to repair. That context makes SRE practices immediately relevant rather than abstract. Every concept in this week's material maps to something you either own today (SLO definition for EHR sync lag, incident response, Alembic migrations) or will own as you move toward a staff role (cross-team reliability contracts, chaos program, deployment strategy).

This guide covers the full SRE toolbox in practical depth: SLIs/SLOs/SLAs and how they chain together, error budgets as a policy instrument, chaos engineering at the GKE level, graceful degradation patterns for FastAPI, incident response runbooks, blameless post-mortem culture, and zero-downtime deployment and database migration strategies. Each section includes working code, configuration, or templates grounded in your stack (Python 3.13, FastAPI, GKE, Datadog, Alembic, Helm).

By the end of this week you should be able to write a defensible SLO document for the integration platform middleware, draft a Datadog SLO monitor in Terraform, run a controlled chaos experiment against a GKE deployment, write an incident runbook, and execute a zero-downtime Alembic migration on Cloud SQL — all without causing a production incident in the process.

---

## 1. SLIs, SLOs, and SLAs — The Reliability Contract Stack

**SLI (Service Level Indicator):** A quantitative metric that measures a specific aspect of service behavior as experienced by users. Good SLIs are user-facing, measurable in near-real-time, and directly correlated with pain when they degrade.

**SLO (Service Level Objective):** A target range or threshold applied to an SLI over a rolling window. An SLO is an internal commitment — a contract between the engineering team and the rest of the organization about what "good enough" looks like.

**SLA (Service Level Agreement):** A contractual commitment to an external party (customer, partner) that typically bundles one or more SLOs with financial or operational consequences for breach.

The key insight is that SLAs should always be *looser* than your SLOs, which should be looser than what your system actually achieves. If your middleware's real availability is 99.95%, your SLO might target 99.9%, and your SLA might commit to 99.5%. That gap is your operational runway.

For the integration platform, the most meaningful SLIs are:

| SLI | Measurement | Why it matters |
|-----|-------------|----------------|
| Sync lag (P95) | Time from EHR system event to Salesforce record update | Data drift; sales reps see stale policy data |
| API success rate | Non-5xx responses / total requests (rolling 5m) | Sync jobs depend on middleware uptime |
| Queue depth | Pending records in middleware queue tables | Leading indicator of backlog/debt |
| Migration execution time | Alembic `upgrade head` wall-clock duration | Downtime risk during deploys |

**Integration platform SLO definition example (YAML, stored in repo):**

```yaml
# crm-middleware/slos/ehr-sync-slo.yaml
service: crm-middleware
owner: platform-team
window: 30d

slos:
  - name: ehr_sync_lag_p95
    description: >
      95th-percentile latency from EHR system event creation to
      Salesforce record upsert must stay below 60 seconds.
    sli:
      type: latency
      metric: integration.sync.lag_seconds
      aggregation: p95
      source: datadog
    target: 60  # seconds
    budget_policy: freeze_deploys_at_5pct

  - name: middleware_availability
    description: >
      HTTP 2xx+3xx response rate for /api/v2/* endpoints,
      measured over the trailing 30-day window.
    sli:
      type: availability
      good_events: "http.status_code:(2* OR 3*)"
      total_events: "http.status_code:*"
      source: datadog
    target_pct: 99.9
    budget_policy: freeze_deploys_at_10pct
```

**Datadog SLO monitor (Terraform snippet):**

```hcl
resource "datadog_service_level_objective" "middleware_availability" {
  name        = "CRM Middleware Availability — 30d"
  type        = "metric"
  description = "HTTP success rate for crm-middleware /api/v2/* endpoints"

  query {
    numerator   = "sum:trace.fastapi.request.hits{http.status_code:2*,service:crm-middleware}.as_count()"
    denominator = "sum:trace.fastapi.request.hits{service:crm-middleware}.as_count()"
  }

  thresholds {
    timeframe = "30d"
    target    = 99.9
    warning   = 99.95
  }

  tags = ["service:crm-middleware", "env:prod", "team:platform"]
}
```

> **Common mistake:** Teams define SLOs on infrastructure metrics (CPU, memory) rather than user-facing behavior. CPU at 80% is not an SLI; a customer experiencing a 30-second sync lag is. Always anchor SLIs to observable user/system outcomes.

---

## 2. Error Budgets — Reliability as a Policy Instrument

An error budget is the complement of your SLO: if your 30-day availability target is 99.9%, your error budget is 0.1% of total request volume — roughly 43 minutes of downtime equivalent per month. The budget is consumed by any event that degrades service below target: incidents, risky deploys, flaky dependencies.

Error budgets change the conversation from "should we ship this?" to "do we have budget to absorb the risk of shipping this?" That framing makes reliability a shared concern between product and engineering, not a tax that ops pays alone.

**Budget consumption tracking in Python (simplified):**

```python
# crm-middleware/app/core/slo.py
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class ErrorBudget:
    slo_target_pct: float          # e.g. 99.9
    window_days: int               # e.g. 30
    total_requests: int
    failed_requests: int

    @property
    def allowed_failures(self) -> int:
        allowed_pct = (100 - self.slo_target_pct) / 100
        return int(self.total_requests * allowed_pct)

    @property
    def remaining_pct(self) -> float:
        remaining = max(0, self.allowed_failures - self.failed_requests)
        if self.allowed_failures == 0:
            return 0.0
        return (remaining / self.allowed_failures) * 100

    @property
    def is_frozen(self) -> bool:
        """Freeze deploys when less than 5% budget remains."""
        return self.remaining_pct < 5.0


def check_deploy_gate(budget: ErrorBudget) -> tuple[bool, str]:
    if budget.is_frozen:
        return False, (
            f"Deploy blocked: error budget {budget.remaining_pct:.1f}% remaining "
            f"({budget.failed_requests}/{budget.allowed_failures} failures used). "
            "Stabilize the service before shipping new changes."
        )
    return True, f"Deploy allowed: {budget.remaining_pct:.1f}% budget remaining."
```

**Budget policies (typical tiers):**

| Remaining budget | Policy |
|-----------------|--------|
| > 50% | Ship freely; low-risk changes |
| 10–50% | Require extra review; prefer canary deploys |
| 5–10% | Incident post-mortem required before next deploy |
| < 5% | Feature freeze; reliability work only |
| 0% | All hands on reliability; escalate to leadership |

**Integration platform application:** After the March 2026 Cloud SQL failover incident consumed 40% of the quarterly error budget in a single event, the team should have enforced a two-week feature freeze and mandated chaos testing of the failover path before the next release. Error budget policy makes that decision automatic rather than political.

> **Common mistake:** Setting a single error budget for the entire service. Break it down: one budget for the sync pipeline, one for the admin API, one for migrations. A slow batch job eating sync budget is invisible if you only track aggregate availability.

---

## 3. Chaos Engineering Basics

Chaos engineering is the practice of deliberately injecting failure into a system to validate that it behaves correctly under adversarial conditions. The goal is to find weaknesses before production traffic does.

The canonical approach (from Netflix's Chaos Monkey and Principles of Chaos Engineering):
1. Define a hypothesis: "The middleware will continue processing the sync queue if one of three GKE pods is killed."
2. Measure steady-state: Record baseline SLI values before the experiment.
3. Inject failure: Kill a pod, partition a network, degrade a dependency.
4. Observe: Does the system return to steady-state within the SLO window?
5. Fix or accept: Either harden the system or update the runbook to reflect the real behavior.

**GKE pod kill experiment using kubectl (manual):**

```bash
# Identify running middleware pods
kubectl get pods -n integration-prod -l app=crm-middleware

# Kill one pod — GKE should reschedule within ~30s
kubectl delete pod crm-middleware-7d9f8b-xkp2v -n integration-prod

# Watch recovery
kubectl get pods -n integration-prod -l app=crm-middleware -w
```

**Chaos experiment using Chaos Toolkit (Python DSL):**

```json
{
  "title": "CRM middleware survives single pod failure",
  "description": "Kill one middleware pod; assert sync lag P95 stays under 60s",
  "steady-state-hypothesis": {
    "title": "Sync lag within SLO",
    "probes": [
      {
        "type": "probe",
        "name": "check-sync-lag",
        "tolerance": {"type": "range", "range": [0, 60]},
        "provider": {
          "type": "http",
          "url": "http://crm-middleware-internal/metrics/sync_lag_p95"
        }
      }
    ]
  },
  "method": [
    {
      "type": "action",
      "name": "kill-one-pod",
      "provider": {
        "type": "process",
        "path": "kubectl",
        "arguments": "delete pod -n integration-prod -l app=crm-middleware --field-selector=status.phase=Running --sort-by=.metadata.creationTimestamp -o name | head -1 | xargs kubectl delete -n integration-prod"
      },
      "pauses": {"after": 30}
    }
  ],
  "rollbacks": []
}
```

**GKE-specific chaos scenarios relevant to the integration platform:**

| Scenario | Inject via | Expected behavior |
|----------|-----------|-------------------|
| Pod OOM kill | Resource limit reduction | Queue drain resumes after reschedule |
| Cloud SQL connection exhaustion | `pgbench` saturation test | Connection pool rejects and returns 503 |
| EHR system timeout | `tc netem` delay on egress | Retry with exponential backoff; alert fires |
| Node pool drain | `kubectl drain <node>` | Pod rescheduled to healthy node; no queue gap |

> **Common mistake:** Running chaos experiments in production without a rollback plan and an on-call engineer watching dashboards. Always have a "stop" procedure documented before starting. Start in staging.

---

## 4. Graceful Degradation

Graceful degradation means a system continues to provide reduced — but acceptable — service when a dependency fails, rather than failing completely. For the integration platform, the most important degradation scenarios are: EHR system unavailable, Cloud SQL read replica lag, and Salesforce API rate limiting.

**FastAPI dependency health with fallback:**

```python
# crm-middleware/app/api/deps.py
from fastapi import HTTPException, status
from app.core.ehr_client import EhrClient, EhrUnavailableError
from app.core.cache import get_cached_policy

async def get_policy_with_fallback(policy_id: str) -> dict:
    """
    Try EHR system first; fall back to last-known-good cache on transient errors.
    Raises 503 only if cache is also stale (> 4h old).
    """
    try:
        return await EhrClient().get_policy(policy_id)
    except EhrUnavailableError:
        cached = await get_cached_policy(policy_id)
        if cached and cached.age_seconds < 14400:  # 4h TTL
            cached.metadata["degraded"] = True
            return cached.data
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail={
                "error": "ehr_unavailable",
                "message": "EHR system is unreachable and cache is stale. Retry later.",
                "retry_after": 60,
            }
        )
```

**Circuit breaker pattern (using `circuitbreaker` library):**

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30, expected_exception=EhrUnavailableError)
async def call_ehr_api(endpoint: str, payload: dict) -> dict:
    return await EhrClient().post(endpoint, payload)
```

**Degradation levels for the integration platform sync pipeline:**

```
Level 0 (Normal):   Full sync, all objects, real-time
Level 1 (Degraded): Write-through cache; skip non-critical objects (Categories, Structures)
Level 2 (Minimal):  Queue writes locally; batch sync when EHR system recovers
Level 3 (Read-only): Serve stale data from cache; disable all writes to EHR system
Level 4 (Down):     Return 503; alert PagerDuty; start incident
```

> **Common mistake:** Returning stale data silently without indicating degradation to callers. Always include a `X-Degraded: true` response header or a `degraded` field in the response body so downstream systems (and Datadog dashboards) can detect the degraded state.

---

## 5. Incident Response Runbooks

A runbook is a documented, step-by-step procedure for diagnosing and resolving a specific class of incident. Good runbooks reduce mean-time-to-resolve (MTTR) by eliminating the cognitive load of figuring out what to check during a high-stress outage.

**Integration platform incident runbook template:**

```markdown
# Runbook: Integration Platform Sync Queue Backlog (ALERT: integration.queue.depth > 500)

**Severity:** SEV-2
**Owner:** Platform Team
**Last updated:** 2026-12-07
**PagerDuty policy:** integration-platform-oncall

---

## Symptoms
- Datadog alert: `integration.queue.depth` exceeds 500 for 5+ minutes
- Salesforce reps report stale policy/contact data
- Sync lag P95 > 120s (2× SLO threshold)

## Immediate triage (< 5 min)

1. **Check middleware pod health:**
   ```bash
   kubectl get pods -n integration-prod -l app=crm-middleware
   kubectl logs -n integration-prod -l app=crm-middleware --tail=100 | grep ERROR
   ```

2. **Check EHR system connectivity:**
   ```bash
   kubectl exec -n integration-prod deploy/crm-middleware -- \
     curl -sf http://ehr-server-prod/ehr-sdk/health || echo "EHR UNREACHABLE"
   ```

3. **Check Cloud SQL connection pool:**
   ```bash
   kubectl exec -n integration-prod deploy/crm-middleware -- \
     python -c "from app.db.session import engine; print(engine.pool.status())"
   ```

4. **Check queue table directly:**
   ```sql
   SELECT status, COUNT(*), MIN(created_at), MAX(created_at)
   FROM sync_queue
   WHERE status IN ('pending', 'failed')
   GROUP BY status;
   ```

## Decision tree

```
Queue > 500?
├── Are middleware pods running? NO → restart deploy, page on-call
├── EHR system unreachable? YES → activate Level 2 degradation, notify EHR team
├── DB connection pool exhausted? YES → increase pool size, restart pods
└── All healthy but queue growing? → check ETL job; check rate limiting on Salesforce API
```

## Escalation
- 15 min no progress → escalate to Senior Engineer
- 30 min no progress → escalate to Engineering Manager + notify Account team
- Data drift confirmed → run reconciliation job: `python scripts/reconcile.py --hours 4`

## Resolution
- Confirm queue depth returns to < 50 within 15 min of fix
- Update Datadog incident timeline
- File post-mortem issue within 24h
```

> **Common mistake:** Runbooks that describe the system architecture rather than the diagnostic steps. A runbook is not documentation — it is a decision tree for a stressed engineer at 2 AM. Keep it command-first, explanation-second.

---

## 6. Blameless Post-Mortem Culture

A post-mortem (or "incident review") is a structured analysis of what went wrong, conducted after the incident is resolved. The "blameless" modifier is critical: the goal is to understand system and process failures, not to assign individual fault. Blaming individuals suppresses future incident reporting and drives problems underground.

**The blameless principle** rests on the assumption that engineers act rationally given the information and tools they had at the time. If an engineer made a "bad" decision, ask: Why did the system make that decision seem correct? What monitoring, tooling, or process was absent?

**Integration platform post-mortem template:**

```markdown
# Post-Mortem: Integration Platform Sync Outage — 2026-11-14

**Duration:** 47 minutes (14:22–15:09 UTC)
**Severity:** SEV-2
**Impact:** ~1,200 Salesforce records fell out of sync; 3 account managers affected
**Author:** [on-call engineer]
**Reviewed by:** [team lead, +1 senior engineer]
**Status:** Draft / In Review / Closed

---

## Summary
A misconfigured Alembic migration added a NOT NULL column without a default,
causing all middleware pods to crash-loop on startup after the 14:20 deploy.

## Timeline
| Time (UTC) | Event |
|-----------|-------|
| 14:20 | Deploy triggered; Alembic `upgrade head` ran |
| 14:21 | Pods begin crash-loop: `sqlalchemy.exc.IntegrityError` on startup |
| 14:22 | PagerDuty alert fires: pod restart count > 5 |
| 14:28 | On-call acknowledges; checks logs |
| 14:35 | Root cause identified: missing column default |
| 14:50 | Hotfix migration deployed (`alembic downgrade -1`) |
| 15:09 | All pods healthy; queue backlog cleared |

## Root cause
Migration `2026_11_14_add_ehr_ref_id.py` added `ehr_ref_id VARCHAR NOT NULL`
without a server-side default. Cloud SQL applied the DDL; existing rows violated
the constraint; SQLAlchemy model validation raised on startup.

## Contributing factors
- No pre-deploy migration dry-run in CI (`alembic upgrade --sql` not run)
- Staging database was empty; constraint violation not triggered in staging
- No smoke test after deploy verifying pod readiness before traffic shift

## What went well
- PagerDuty alert fired within 60 seconds of first crash
- Runbook for crash-loop clearly identified log pattern
- Rollback procedure (`alembic downgrade`) was documented and fast

## Action items
| Action | Owner | Due |
|--------|-------|-----|
| Add `alembic upgrade --sql` to CI pipeline | [engineer] | 2026-11-21 |
| Seed staging DB with representative row count | [engineer] | 2026-11-28 |
| Add post-deploy pod readiness smoke test to Helm hook | [engineer] | 2026-11-28 |

## What we won't fix (and why)
- Manual review of migrations: CI check is sufficient; adding process gates
  without tooling creates toil without reliability improvement.
```

> **Common mistake:** Post-mortems that list action items but never close them. Assign a single owner, a concrete due date, and a Jira ticket. Review open items in the next team meeting. An unclosed action item from a previous post-mortem that reoccurs is an organizational failure, not just a technical one.

---

## 7. CI/CD Deployment Strategies — Blue-Green and Canary

Zero-downtime deployment is a prerequisite for maintaining error budgets. The two main strategies for stateless services like the integration platform middleware are blue-green and canary.

**Blue-Green Deployment:**
Two identical environments (blue = current, green = new). Traffic switches atomically from blue to green. Rollback is instant: switch traffic back.

```yaml
# kubernetes/crm-middleware/blue-green-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: crm-middleware
  namespace: integration-prod
spec:
  selector:
    app: crm-middleware
    slot: green   # Toggle between "blue" and "green" for cutover
  ports:
    - port: 80
      targetPort: 8000
```

```bash
# Deploy green slot
helm upgrade crm-middleware ./charts/crm-middleware \
  --set image.tag=v2.14.0 \
  --set slot=green \
  --set replicaCount=3 \
  -n integration-prod

# Verify green is healthy
kubectl rollout status deployment/crm-middleware-green -n integration-prod

# Cut over: patch service selector
kubectl patch service crm-middleware -n integration-prod \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# Monitor for 10 min, then decommission blue
kubectl delete deployment crm-middleware-blue -n integration-prod
```

**Canary Deployment (Helm-based weight split):**

```yaml
# values.canary.yaml — deploy 10% canary alongside stable
canary:
  enabled: true
  weight: 10
  image:
    tag: v2.15.0-rc1
stable:
  weight: 90
  image:
    tag: v2.14.0
```

```python
# Datadog monitor for canary error rate
# Alert if canary error rate > 2× stable error rate
query = """
(
  sum:trace.fastapi.request.errors{service:crm-middleware,version:v2.15.0-rc1}.as_rate()
  /
  sum:trace.fastapi.request.hits{service:crm-middleware,version:v2.15.0-rc1}.as_rate()
) > 2 * (
  sum:trace.fastapi.request.errors{service:crm-middleware,version:v2.14.0}.as_rate()
  /
  sum:trace.fastapi.request.hits{service:crm-middleware,version:v2.14.0}.as_rate()
)
"""
```

> **Common mistake:** Running blue-green without ensuring the database schema is backward-compatible with both versions simultaneously. If the green deployment requires a new column that blue-era code doesn't know about, the cutover is not zero-downtime — it's a schema flip with a risk window. Solve this with the expand/contract migration pattern covered in Section 8.

---

## 8. Zero-Downtime Database Migrations with Alembic

Database migrations are the highest-risk part of any deploy. A migration that takes an exclusive table lock, or that adds a NOT NULL column without a default, can cause an outage even if the application code is correct.

The **expand/contract pattern** solves this:
1. **Expand:** Add the new column as nullable (no lock, no downtime). Deploy.
2. **Backfill:** Populate existing rows in batches (no lock held across all rows).
3. **Constrain:** Add NOT NULL constraint and index after backfill completes. Deploy.
4. **Contract:** Remove the old column once all code references are gone. Deploy.

**Integration platform example — adding `ehr_ref_id` safely:**

```python
# alembic/versions/2026_12_07_001_expand_ehr_ref_id.py
"""expand: add ehr_ref_id nullable

Revision ID: a1b2c3d4e5f6
"""
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Step 1: Add nullable — no exclusive lock, no downtime
    op.add_column(
        'sync_queue',
        sa.Column('ehr_ref_id', sa.String(64), nullable=True)
    )
    # Partial index — only locks rows being indexed, not the whole table
    op.create_index(
        'ix_sync_queue_ehr_ref_id',
        'sync_queue',
        ['ehr_ref_id'],
        postgresql_where=sa.text('ehr_ref_id IS NOT NULL')
    )

def downgrade():
    op.drop_index('ix_sync_queue_ehr_ref_id', 'sync_queue')
    op.drop_column('sync_queue', 'ehr_ref_id')
```

```python
# scripts/backfill_ehr_ref_id.py — run as a one-off job, NOT in migration
"""
Backfill ehr_ref_id in batches of 1000 to avoid long-running locks.
Safe to run while production traffic is live.
"""
import asyncio
from sqlalchemy import text
from app.db.session import async_engine

BATCH_SIZE = 1000

async def backfill():
    async with async_engine.begin() as conn:
        while True:
            result = await conn.execute(text("""
                UPDATE sync_queue
                SET ehr_ref_id = ehr_id::text
                WHERE ehr_ref_id IS NULL
                  AND id IN (
                    SELECT id FROM sync_queue
                    WHERE ehr_ref_id IS NULL
                    LIMIT :batch
                    FOR UPDATE SKIP LOCKED
                  )
                RETURNING id
            """), {"batch": BATCH_SIZE})
            updated = result.rowcount
            print(f"Backfilled {updated} rows")
            if updated < BATCH_SIZE:
                break
            await asyncio.sleep(0.1)  # yield to avoid starving other queries

asyncio.run(backfill())
```

```python
# alembic/versions/2026_12_07_002_constrain_ehr_ref_id.py
"""constrain: make ehr_ref_id NOT NULL after backfill

Revision ID: b2c3d4e5f6a7
"""
from alembic import op
import sqlalchemy as sa

def upgrade():
    # PostgreSQL NOT VALID skips re-validating existing rows (fast!)
    # Then validate separately — shares lock mode, does not block reads/writes
    op.execute(
        "ALTER TABLE sync_queue "
        "ADD CONSTRAINT sync_queue_ehr_ref_id_not_null "
        "CHECK (ehr_ref_id IS NOT NULL) NOT VALID"
    )
    op.execute(
        "ALTER TABLE sync_queue "
        "VALIDATE CONSTRAINT sync_queue_ehr_ref_id_not_null"
    )

def downgrade():
    op.execute(
        "ALTER TABLE sync_queue "
        "DROP CONSTRAINT sync_queue_ehr_ref_id_not_null"
    )
```

**Migration safety checklist:**

| Check | Command | Pass condition |
|-------|---------|---------------|
| Preview DDL | `alembic upgrade --sql head` | No `ALTER TABLE ... NOT NULL` without default |
| Lock timeout | Set `lock_timeout = 3s` in session | Migration aborts rather than blocking |
| Staging test | Run on staging with prod-scale row count | Completes in < 30s |
| Rollback test | `alembic downgrade -1` on staging | Completes cleanly |

> **Common mistake:** Running `op.alter_column(..., nullable=False)` directly in SQLAlchemy/Alembic. This generates `ALTER COLUMN ... SET NOT NULL` which acquires an `ACCESS EXCLUSIVE` lock and rewrites the table on older Postgres versions. Always use the `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` two-step on Cloud SQL (Postgres 14+).

---

## 9. Observability for Reliability — Datadog SLO Dashboards

Reliability without observability is guesswork. Every SLO needs a corresponding dashboard that shows current status, burn rate, and projected budget exhaustion.

**Error budget burn rate alert (Datadog monitor):**

```python
# terraform/datadog/monitors/slo_burn_rate.tf equivalent in Python SDK
import datadog

datadog.initialize(api_key="...", app_key="...")

monitor = {
    "name": "Integration Platform SLO Burn Rate — 1h Fast Burn",
    "type": "query alert",
    "query": """
        (
          1 - (
            sum:trace.fastapi.request.hits{http.status_code:2*,service:crm-middleware}
              .as_rate().rollup(sum, 3600)
            /
            sum:trace.fastapi.request.hits{service:crm-middleware}
              .as_rate().rollup(sum, 3600)
          )
        ) / 0.001 > 14.4
    """,
    # 14.4× burn rate = consuming 30-day budget in 50 hours
    "message": (
        "CRM middleware is burning its error budget at 14.4× the sustainable rate.\n"
        "At this rate the 30-day budget will be exhausted in ~50 hours.\n"
        "@pagerduty-integration-platform @slack-integration-alerts"
    ),
    "tags": ["service:crm-middleware", "slo:availability"],
    "options": {
        "thresholds": {"critical": 14.4, "warning": 6.0},
        "notify_no_data": False,
        "evaluation_delay": 60,
    }
}
```

**Synthetic monitor for integration platform sync pipeline health (Datadog):**

```python
# Runs every 5 minutes from us-east-1, eu-west-1, ap-southeast-1
synthetic_test = {
    "name": "CRM Middleware — Sync Health Check",
    "type": "api",
    "subtype": "http",
    "config": {
        "request": {
            "method": "GET",
            "url": "https://crm-middleware.prod.internal/health/sync",
            "timeout": 10,
        },
        "assertions": [
            {"type": "statusCode", "operator": "is", "target": 200},
            {"type": "responseTime", "operator": "lessThan", "target": 5000},
            {"type": "body", "operator": "contains", "target": '"status":"ok"'},
        ],
    },
    "locations": ["aws:us-east-1", "aws:eu-west-1"],
    "options": {"tick_every": 300, "min_failure_duration": 60},
}
```

> **Common mistake:** Alerting only on the SLO breach threshold. By the time the SLO fires, the budget is already consumed. Use multi-window burn rate alerts (1h fast burn + 6h slow burn) that fire while budget is still available to act.

---

## 10. Key Concepts Summary

```
Reliability Engineering & SRE
│
├── Measurement
│   ├── SLI — what you measure (sync lag P95, availability rate)
│   ├── SLO — internal target (99.9% availability over 30d)
│   └── SLA — external contract (looser than SLO)
│
├── Policy
│   ├── Error Budget — (100% - SLO%) of total events
│   ├── Budget burn rate — speed of budget consumption
│   └── Freeze policy — deploy gate when budget < 5%
│
├── Resilience
│   ├── Chaos engineering — hypothesis → inject → observe → fix
│   ├── Graceful degradation — levels 0→4, fallback cache, circuit breaker
│   └── Runbooks — decision tree for on-call; command-first format
│
├── Culture
│   ├── Blameless post-mortems — system failure, not human failure
│   ├── Action items — owner + due date + ticket, reviewed weekly
│   └── Toil reduction — automate repetitive operational work
│
└── Deployment Safety
    ├── Blue-green — atomic cutover, instant rollback
    ├── Canary — weighted traffic split, error rate comparison
    └── Zero-downtime migrations
        ├── Expand — add nullable column (no lock)
        ├── Backfill — batch UPDATE with SKIP LOCKED
        └── Constrain — ADD CONSTRAINT NOT VALID + VALIDATE
```

---

## Quiz — 20 Questions

### Questions

**1.** What is the difference between an SLI and an SLO, and why does the distinction matter?

**2.** Your integration platform middleware has a 99.9% availability SLO over 30 days. How many minutes of "downtime equivalent" does your error budget represent?

**3.** What is the expand/contract migration pattern and why is it necessary for zero-downtime deploys?

**4.** A colleague runs `op.alter_column('sync_queue', 'ehr_ref_id', nullable=False)` in an Alembic migration. What PostgreSQL behavior does this trigger and what is the risk?

**5.** Describe the three-tier reliability contract: SLI → SLO → SLA. Why should SLAs always be looser than SLOs?

**6.** You observe the integration platform sync lag P95 has been 45 seconds for the past hour (SLO: < 60s). Should you be concerned? What would make you more confident in your answer?

**7.** What is error budget burn rate, and what does a 14.4× burn rate mean in terms of budget exhaustion time for a 30-day window?

**8.** Your chaos experiment kills one of three middleware pods. The queue depth spikes from 50 to 400 over 3 minutes before returning to baseline at 60. Did the system pass the experiment? What would you improve?

**9.** What is the purpose of `FOR UPDATE SKIP LOCKED` in the backfill script? What problem does it solve?

**10.** Explain the blue-green deployment model. What is the single most important prerequisite for it to be truly zero-downtime?

**11.** What is the difference between a blameless post-mortem and a traditional root-cause analysis that names responsible individuals?

**12.** You are writing a runbook for "integration platform queue depth > 500." What are the first three diagnostic commands an on-call engineer should run?

**13.** What is `ALTER TABLE ... ADD CONSTRAINT ... NOT VALID` followed by `VALIDATE CONSTRAINT`? Why is this safer than a direct `NOT NULL` alter?

**14.** You have 4% of your monthly error budget remaining and it's day 22 of a 30-day cycle. What policy should you enforce?

**15.** Describe two integration-platform-specific SLIs that are more meaningful than generic infrastructure metrics like CPU or memory.

**16.** What is a canary deployment and how do you decide when to promote a canary to stable?

**17.** What is graceful degradation Level 2 in the integration platform context, and how does it protect the EHR sync pipeline during an outage?

**18.** What is the primary risk of defining SLOs on infrastructure metrics (CPU, memory) rather than user-facing behavior?

**19.** Why should the backfill script (Step 2 of expand/contract) be run as a separate job rather than inside the Alembic migration?

**20.** What is toil in the SRE context, and give one example of toil in integration platform operations that could be automated?

---

### Answers

??? note "Reveal Answers"

    **1.** An SLI (Service Level Indicator) is a raw metric that measures observable system behavior — for example, the fraction of HTTP requests returning 2xx, or the P95 latency of sync operations. An SLO (Service Level Objective) is a target applied to an SLI over a time window — for example, "the success rate SLI must stay above 99.9% over any 30-day rolling window." The distinction matters because SLIs are facts about what the system is doing, while SLOs are commitments about what it *should* do. Conflating them leads to vague targets ("we want low latency") that cannot be operationalized as alerts or error budgets.

    **2.** A 99.9% SLO leaves a 0.1% error budget. Over 30 days there are 43,200 minutes total; 0.1% of that is 43.2 minutes. This is the maximum downtime-equivalent your service can experience in a month before breaching the SLO. It applies to any form of service degradation below target — not just complete outages — so a partial degradation that affects 50% of requests counts as consuming half the equivalent time per minute.

    **3.** The expand/contract pattern splits a schema change into three safe phases: first, add the new column as nullable (no locking DDL); second, backfill existing rows in small batches; third, add the NOT NULL constraint using PostgreSQL's two-step `ADD CONSTRAINT NOT VALID` + `VALIDATE CONSTRAINT`. It is necessary for zero-downtime deploys because a single-step `ALTER TABLE ... ADD COLUMN ... NOT NULL` takes an `ACCESS EXCLUSIVE` lock that blocks all reads and writes on the table until it completes — on a large table in production this can cause seconds to minutes of effective downtime.

    **4.** SQLAlchemy's `alter_column(..., nullable=False)` generates `ALTER TABLE sync_queue ALTER COLUMN ehr_ref_id SET NOT NULL`. In PostgreSQL this acquires an `ACCESS EXCLUSIVE` lock and performs a full table scan to verify no nulls exist. On a large table under production load this blocks all concurrent reads and writes for the duration of the scan, which can range from seconds to minutes. The risk is a sync outage and error budget consumption during what appears to be a routine deploy.

    **5.** The chain works as follows: SLIs are measured facts about system behavior; SLOs are internal targets placed on those SLIs; SLAs are external commitments made to customers or partners, backed by SLOs. SLAs must be looser than SLOs because there must be a buffer for real-world variation — if your SLO is 99.9% and you commit an SLA to 99.9%, any month where you barely meet your SLO will breach the SLA. The buffer between SLO and SLA represents the operational margin that protects the business from contractual consequences during normal engineering variance.

    **6.** A sync lag of 45 seconds is within the SLO (< 60s), so no breach is occurring — but you should be monitoring the trend. If lag has been climbing from 20s to 45s over the past hour, you are likely approaching a breach and should investigate the cause now rather than after the SLO fires. More useful context would be: is the EHR queue growing? Is one object type disproportionately slow? Is a recent deploy correlated with the increase? The metric alone is insufficient without a rate-of-change view.

    **7.** Error budget burn rate is the ratio of the current error rate to the sustainable error rate that would exactly consume the budget over the SLO window. A 14.4× burn rate on a 30-day window means the service is consuming its monthly budget at 14.4 times the safe pace, which will exhaust the entire budget in approximately 30 / 14.4 ≈ 2.08 days, or roughly 50 hours. This is the "fast burn" threshold in Google's SRE alerting model and should trigger an immediate page rather than waiting for the SLO to breach.

    **8.** The system arguably passed the hypothesis (sync resumed; lag did not breach the 60s SLO if the spike lasted only 3 minutes) but the queue spike to 400 reveals a weakness: the remaining two pods were not sized to absorb the traffic of three pods immediately. Two improvements to consider: increase `resources.requests` so the scheduler places pods on nodes with sufficient headroom, and add a `PodDisruptionBudget` ensuring at least two pods are always available during voluntary disruptions.

    **9.** `FOR UPDATE SKIP LOCKED` acquires a row-level lock on each row selected for update but skips any rows that are already locked by another transaction. This is critical for a batched backfill running concurrently with production traffic: without it, multiple backfill workers (or the backfill and a production write) would block each other, creating lock contention that degrades response times. With `SKIP LOCKED`, each worker takes an exclusive batch of rows and they proceed in parallel without waiting on each other.

    **10.** Blue-green deployment runs two identical environments — blue (current) and green (new). Traffic is switched atomically by changing a load balancer or Kubernetes Service selector. Rollback is instant: switch the selector back. The single most important prerequisite for true zero-downtime is **backward-compatible database schema**: both the blue and green versions of the application code must be able to operate correctly against the same database schema simultaneously, because during the cutover window both versions may be live. Any schema change that breaks the old code makes blue-green unsafe.

    **11.** A blameless post-mortem starts from the assumption that engineers acted rationally given the tools, information, and environment available to them at the time. When something goes wrong, the question is not "who made the mistake" but "what about the system, process, or tooling led a rational person to take the action that caused the failure?" Traditional root-cause analyses that name individuals suppress future incident reporting — engineers avoid disclosing mistakes if they expect punishment — which means systemic problems go unfixed and recur. Blameless culture produces better action items because it targets the conditions that made failure possible, not the person who happened to trigger it.

    **12.** The first three commands in the integration platform queue depth runbook should be: (1) `kubectl get pods -n integration-prod -l app=crm-middleware` — confirm pods are running and not crash-looping; (2) `kubectl logs -n integration-prod -l app=crm-middleware --tail=100 | grep ERROR` — check for application errors like database connection failures or EHR API timeouts; (3) a direct SQL query against the sync_queue table: `SELECT status, COUNT(*) FROM sync_queue WHERE status IN ('pending','failed') GROUP BY status` — to understand whether records are pending (not yet processed) or failed (attempted and errored), which points to different root causes.

    **13.** `ADD CONSTRAINT ... NOT VALID` adds the constraint to the table metadata so new inserts and updates must satisfy it, but it explicitly does not re-scan existing rows to verify them. This means it completes almost instantly without blocking. `VALIDATE CONSTRAINT` then performs the scan — but it holds only a `SHARE UPDATE EXCLUSIVE` lock, which allows concurrent reads and writes to proceed normally. The split approach is safer than a direct `SET NOT NULL` because it never blocks production traffic for the duration of a full table scan, which on a large Cloud SQL table could take minutes.

    **14.** With 4% budget remaining and 8 days left in the cycle, you should enforce a **feature freeze** (deploy only reliability fixes and critical security patches). At current consumption rate, the budget will likely be exhausted before the cycle ends, and any additional incidents will breach the SLO. The appropriate policy actions are: (a) require a post-mortem for whatever caused the budget burn to reach 96%; (b) staff a reliability sprint for the remaining week; (c) notify stakeholders that feature velocity is halted until reliability is restored.

    **15.** Two integration-platform-specific SLIs more meaningful than CPU/memory: (1) **Sync lag P95** — the 95th-percentile time between an EHR system event being created and the corresponding Salesforce record being updated. This directly measures the user-visible pain of data drift. (2) **Queue failure rate** — the fraction of sync_queue records that transition to `failed` status within their first processing attempt. This measures the reliability of the Salesforce-to-EHR data path, which is the core business function of the platform, independent of whether infrastructure is technically "healthy."

    **16.** A canary deployment routes a small fraction of production traffic (typically 1–10%) to the new version while the majority stays on the stable version. The promotion decision is based on comparing error rate, latency, and custom business metrics between canary and stable over a observation window (typically 10–30 minutes). The canary is promoted to stable when: (a) its error rate is not statistically higher than stable; (b) P95 latency has not regressed; and (c) any custom SLI checks (e.g., sync queue depth not growing) pass. Automated promotion gates in Argo Rollouts or Flagger can make this decision programmatically.

    **17.** Level 2 degradation in the integration platform context means the middleware stops attempting real-time EHR API calls and instead queues all write operations locally in the database, batching them for replay when EHR connectivity is restored. Reads may still be served from a cache with a staleness indicator. This protects the sync pipeline by preventing cascading failures: rather than failing all in-flight Salesforce-triggered operations with 503s (which would cause Apex trigger failures and lost events), the system absorbs the writes safely and reconciles them when the dependency recovers. Data drift is bounded and controlled rather than unbounded.

    **18.** Defining SLOs on CPU and memory creates a false sense of reliability: the infrastructure can be "green" while users experience degraded service. Conversely, CPU at 80% might be perfectly acceptable during a batch job with no user-facing impact. Infrastructure metrics are leading indicators of potential problems, not direct measures of user experience. If you page on high CPU, you create alert fatigue from false positives. If your SLO only covers infrastructure, you will miss incidents where the service is responding slowly or incorrectly despite healthy-looking infrastructure — exactly the kind of subtle degradation that causes data drift in the integration platform.

    **19.** Running the backfill inside the Alembic migration is dangerous for two reasons. First, Alembic migrations run inside a transaction — a backfill that processes millions of rows holds an open transaction for minutes or hours, blocking other DDL operations and consuming database connection resources. Second, if the backfill fails partway through, the entire migration rolls back, leaving the schema unchanged and requiring the engineer to diagnose and re-run in a stressful deploy situation. A separate script gives you fine-grained control: it can be paused, resumed, monitored, and retried in isolation without touching the schema state.

    **20.** Toil in the SRE context is operational work that is manual, repetitive, automatable, tactical (reactive rather than strategic), and scales linearly with service growth rather than remaining constant. Google's SRE model targets keeping toil below 50% of an engineer's time. One concrete integration platform example: manually running `python scripts/reconcile.py --hours 4` after every incident to repair data drift. This is manual (requires SSH access or kubectl exec), repetitive (happens after most SEV-2 incidents), automatable (could be triggered automatically when queue depth returns to normal after an incident), and scales with incident frequency. Automating it via a post-recovery Kubernetes Job would eliminate this toil entirely.
