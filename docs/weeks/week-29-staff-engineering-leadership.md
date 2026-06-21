# Week 29 — Staff Engineering — Strategy, Leadership & 2027 Planning

**Week of:** December 21, 2026
**Estimated study time:** ~2 hours
**Tags:** `staff-engineering` `leadership` `architecture` `career`

---

## Overview

You have spent 28 weeks building a formidable technical foundation: OAuth 2.0 and OIDC, AI/LLM integration, Salesforce internals, Python performance, PostgreSQL at scale, GCP architecture, Kubernetes operations, data engineering pipelines, distributed systems observability, security hardening, reliability engineering, and advanced testing. Week 29 is not another topic to absorb — it is the lens through which all of that knowledge becomes staff-level impact. The difference between a senior engineer and a staff engineer is not more technical depth; it is the ability to translate technical depth into organizational outcomes.

Staff engineering is fundamentally about multiplying other engineers rather than adding your own output. When you write an RFC that aligns five teams around a migration strategy, you have done more than any single pull request could achieve. When you produce an ADR that documents why the integration platform chose PostgreSQL-backed queues over Kafka — capturing the constraints, the alternatives considered, and the decision rationale — you prevent that decision from being relitigated six months later when the original engineers are heads-down on something else. When you draw a C4 diagram of the CRM-EHR Integration Platform system and share it in onboarding, a new engineer understands in thirty minutes what took you six months to internalize through code archaeology. Documentation, diagrams, and decision records are force multipliers.

This week synthesizes everything you have learned into the practices, artifacts, and mental models that define staff engineering. You will study the five staff engineer archetypes and identify which fits your trajectory at the company. You will learn to write RFCs and design docs that actually get read and acted on. You will master ADRs as a living record of architectural intent. You will build C4 diagrams for the CRM-EHR Integration Platform system. You will practice making decisions under uncertainty — the defining skill of senior-and-above — and develop the techniques for driving alignment across teams when you have no formal authority. You will think through mentoring and engineering culture, and finally, you will design your 2027 learning and career plan using the full stack of what you have built.

By the end of this week you will be able to: articulate your staff engineering archetype and make a case for it to your manager; produce a publication-ready RFC for a proposed CRM-EHR Integration Platform architectural change; write an ADR for any significant technical decision; sketch a C4 Context and Container diagram of the CRM-EHR Integration Platform; apply structured decision-making frameworks under uncertainty; influence technical direction without direct authority; and craft a concrete, milestone-driven 2027 plan that moves you from senior to staff.

---

## 1. Staff Engineer Archetypes

Will Larson's *Staff Engineer* defines five archetypes that describe how staff engineers actually spend their time. Understanding which archetype fits you is the first step to intentional career development — trying to be the wrong archetype for your context is a common path to frustration and stagnation.

**The Tech Lead** guides the technical direction of a specific team. They are deeply embedded in day-to-day delivery, write significant code, and translate product requirements into technical roadmaps. This is the most common first staff role and the one most senior engineers step into naturally. At the integration platform, a Tech Lead would own the technical direction of the middleware-to-ETL integration layer, driving API versioning strategy and ensuring the `crm-middleware` and `etl-bde-pipeline` remain coherent over time.

**The Architect** works across teams on the shape of systems. They spend more time in design reviews, RFCs, and cross-team alignment than in implementation. They own the technical vision for a domain. An integration platform Architect would own the overall integration architecture: when is it time to move from the current PostgreSQL queue tables to an event-streaming model? How should Salesforce Apex triggers interact with middleware in a way that is resilient to middleware downtime? What is the right data model for bidirectional sync that prevents duplicate processing?

**The Solver** is deployed to the hardest, most ambiguous problems in the organization. They move between teams, are given few constraints, and are expected to unblock situations that no one else has been able to resolve. This archetype requires exceptional autonomy and tolerance for ambiguity. An integration platform Solver might be asked to diagnose why the `etl-bq-pipeline` produces inconsistent Account counts in BigQuery compared to Salesforce, an issue that spans Salesforce SOQL queries, Pentaho transforms, Python upsert logic, and BigQuery partitioning — and that no single team owns end-to-end.

**The Right Hand** operates as a force multiplier for an engineering executive. They extend the reach of a CTO or VP Engineering, handling strategic projects, organizational design, and technical due diligence. This archetype requires political savvy and executive communication skills alongside technical depth.

**The Community Builder** focuses on raising the technical capability of the entire engineering organization: through guilds, internal tech talks, mentoring programs, documentation initiatives, and hiring. Their leverage is cultural and social rather than directly technical.

| Archetype | Primary Leverage | Code % | Scope |
|---|---|---|---|
| Tech Lead | Team technical direction | 30–50% | One team |
| Architect | Cross-team system design | 10–20% | Domain / org |
| Solver | Ambiguous critical problems | 20–40% | Fluid |
| Right Hand | Executive amplification | 5–15% | Org-wide |
| Community Builder | Culture and capability | 5–20% | Org-wide |

**Common mistake:** Choosing your archetype based on what you enjoy rather than what the organization needs right now. If your org desperately needs an Architect but you default to Tech Lead behaviors, you will be doing valuable work that is less than what you could contribute. Have an explicit conversation with your manager about which archetype creates the most leverage at the company in 2027.

**For Hussain:** Given the CRM-EHR Integration Platform system complexity and your five years of cross-cutting context (Salesforce, GCP, Python, ETL, PostgreSQL), you are positioned for the Architect or Solver archetype. The integration platform spans six repositories, three data backends, and multiple sync directions — that cross-cutting architectural awareness is rare and valuable.

---

## 2. Writing RFCs and Design Docs

An RFC (Request for Comments) is a structured proposal for a significant technical change. It creates alignment before implementation, surfaces hidden assumptions, invites expertise you do not have, and produces a shared artifact the team can reference during and after the work. A design doc is similar but often more implementation-focused and produced later in the process. The terms are used interchangeably in many organizations.

The goal of an RFC is not to prove you are right. It is to find out whether your proposal is right, and if not, to converge on something better. Write it with genuine intellectual humility. The best RFCs are ones where the author changes their mind in the comments section.

### RFC Template for the Integration Platform

```markdown
# RFC-XXXX: [Short title]

**Author:** [Name]
**Date:** [YYYY-MM-DD]
**Status:** Draft | In Review | Accepted | Superseded | Withdrawn
**Stakeholders:** [List teams and individuals who need to review]
**Decision deadline:** [Date by which a decision is needed]

---

## Summary

One paragraph. What are you proposing, and why?

## Motivation

What problem does this solve? What are the symptoms we observe today?
Include data where possible (e.g., error rates, latency percentiles, 
incident frequency, engineer hours spent on X per month).

## Detailed Design

How does the proposed solution work? Include:
- Architecture diagrams or data flow diagrams
- API changes (before/after)
- Data model changes
- Migration strategy
- Rollback plan

## Alternatives Considered

At least two alternatives. For each: what it is, why you considered it,
why you are NOT choosing it.

## Drawbacks

What are the downsides of this proposal? Be honest. Reviewers will find
them anyway — you earn trust by naming them first.

## Open Questions

What have you NOT resolved yet? What do you need reviewers to weigh in on?

## Implementation Plan

Phases, milestones, estimated effort, who owns each part.

## Success Metrics

How will you know the change succeeded? What do you measure, and when?
```

### Example: Integration Platform Middleware Async Processing RFC

**Motivation excerpt:** Currently, when a Salesforce Apex trigger fires on `Account` insert/update, the `AccountTriggerHandler.syncToEHR` future method calls `crm-middleware` synchronously and blocks the Salesforce transaction on a response. When middleware is degraded, Salesforce transactions time out, causing data loss and user-visible errors. In the past quarter, three incidents were caused by this tight coupling. The proposal is to introduce a reliable async handoff layer between Apex triggers and middleware processing.

**Alternatives to name:** (1) Salesforce Platform Events + middleware subscriber — decouples the Apex transaction, but introduces Platform Event limits and adds a new infrastructure component to operate. (2) Retry logic with exponential backoff in Apex future methods — reduces failures but does not eliminate the blocking coupling. (3) The proposed option: a Salesforce Outbound Message or Change Data Capture event written to a PostgreSQL queue table via a lightweight webhook, processed by the existing middleware queue worker — leverages infrastructure already in place.

**Common mistake:** Writing an RFC that is actually a design doc — fully specifying the implementation before anyone has reviewed whether the approach is right. Keep early RFCs short on implementation detail and long on motivation and alternatives. Save the detailed design for after the approach is validated.

---

## 3. Architecture Decision Records (ADRs)

An ADR captures a single significant architectural decision: the context in which it was made, the alternatives considered, the decision itself, and the consequences. Unlike RFCs, which are proposals seeking review, ADRs are records of decisions already made. They answer the question: "Why is the system built this way?"

ADRs have a lifecycle: Proposed → Accepted → Deprecated → Superseded. When a decision is later reversed, you do not delete the old ADR — you mark it Superseded and link to the new one. This preserves institutional memory across team turnover.

### ADR Format

```markdown
# ADR-XXXX: [Short title of decision]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXXX
**Deciders:** [Names / roles of people who made this decision]
**Technical story:** [Jira ticket or RFC this came from, if any]

---

## Context and Problem Statement

Describe the context and the specific problem or decision that needed
to be made. Include the forces at play: technical constraints, team
capabilities, timeline, cost, operational complexity.

## Decision Drivers

- [Force 1]
- [Force 2]
- [Force 3]

## Considered Options

- Option A: [Name]
- Option B: [Name]
- Option C: [Name]

## Decision Outcome

Chosen option: **Option X**, because [brief justification].

### Consequences

**Positive:**
- [Benefit 1]
- [Benefit 2]

**Negative:**
- [Trade-off 1]
- [Trade-off 2]

## Pros and Cons of Options

### Option A: [Name]
**Pro:** ...
**Con:** ...

### Option B: [Name]
**Pro:** ...
**Con:** ...
```

### Example: ADR-0003 — PostgreSQL Queue Tables vs. Kafka for Integration Platform Sync

**Context:** The integration platform requires a reliable message handoff between Salesforce Apex triggers and the `etl-bde-pipeline` ETL pipeline, with exactly-once semantics for critical objects (Account, Contact, Opportunity, Policy). The team evaluated event streaming (Kafka/GCP Pub/Sub) against the existing PostgreSQL-backed queue table approach.

**Decision:** Retain PostgreSQL queue tables. Context: the engineering team has deep PostgreSQL expertise; the current middleware already manages these tables via SQLAlchemy; operational complexity of adding Kafka or Pub/Sub to a 3-person team is high; message volumes (peak ~500 events/hour) are well within PostgreSQL's capabilities; the existing approach supports transactional writes from Apex via middleware API, which simplifies exactly-once guarantees.

**When to revisit:** If message volume exceeds 50,000 events/hour, or if multiple independent consumers need to subscribe to the same event stream, the Kafka option should be re-evaluated.

**Common mistake:** Writing ADRs only for decisions that worked out well. The most valuable ADRs are for decisions that were painful, or decisions that the team almost got wrong. Document the near-misses — they are the most instructive records for future engineers and your future self.

Store ADRs in the repository they govern. For the integration platform, a natural home is `crm-middleware/docs/adr/` for middleware decisions, and a top-level `docs/adr/` in `infra-manifests` for infrastructure decisions.

---

## 4. C4 Architecture Diagrams

The C4 model (Simon Brown) provides four levels of abstraction for architecture diagrams: Context, Container, Component, and Code. The first two are the most valuable for communication; the latter two are closer to implementation documentation.

**Level 1 — Context:** Shows the system in relation to its users and external systems. One box per major system. Audience: non-technical stakeholders, new engineers.

**Level 2 — Container:** Shows the major deployable/runnable units inside the system and how they communicate. Audience: engineers and architects.

**Level 3 — Component:** Shows the internal structure of a single container. Audience: the team building that container.

**Level 4 — Code:** Class diagrams, entity-relationship diagrams. Rarely worth maintaining manually; generate from code.

### CRM-EHR Integration Platform C4 Context Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│              CRM-EHR Integration Platform                        │
│                    [Software System]                            │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐   ┌──────────────────┐   ┌───────────────────┐
│  Salesforce  │   │  EHR System      │   │  Google BigQuery  │
│   [CRM]      │   │  [Backend]       │   │  [Analytics DW]   │
│              │   │                  │   │                   │
│ sfdc-crm-    │   │ EHR              │   │ BQ Analytics      │
│ package      │   │ REST API         │   │ Projects          │
│ (Apex)       │   │                  │   │                   │
└──────────────┘   └──────────────────┘   └───────────────────┘
       ▲                    ▲
       │   [Insurance       │   [Policy/
       │    Producers]      │    Client data]
       │                    │
┌──────────────┐   ┌──────────────────┐
│  Agency      │   │  ETL Workers     │
│  Producers   │   │  (BDE + BQ)      │
│  [Users]     │   │  [Processes]     │
└──────────────┘   └──────────────────┘
```

### CRM-EHR Integration Platform C4 Container Diagram (ASCII)

```
┌──────────────────────── CRM-EHR Integration Platform ───────────────────────┐
│                                                                              │
│  ┌─────────────────┐     ┌──────────────────────┐                           │
│  │  sfdc-crm-      │     │  crm-middleware       │                           │
│  │  package        │────▶│  [FastAPI / Python]   │                           │
│  │  [Apex Triggers │ API │  PostgreSQL queue     │                           │
│  │   + Aura/LWC]   │     │  tables, REST API     │                           │
│  └─────────────────┘     │  v1 + v2, admin panel │                           │
│         │                └──────────┬────────────┘                           │
│         │                           │                                         │
│         │                    ┌──────┴──────┐                                 │
│         │                    │ PostgreSQL  │                                 │
│         │                    │ [Database]  │                                 │
│         │                    └──────┬──────┘                                 │
│         │                           │                                         │
│  ┌──────▼──────────────────────┐    │                                         │
│  │  etl-bde-pipeline           │◀───┘                                        │
│  │  [Pentaho PDI + Python]     │                                              │
│  │  Sync / Migrate / Reconcile │                                              │
│  └──────────────┬──────────────┘                                              │
│                 │                                                              │
│  ┌──────────────▼──────────────┐   ┌──────────────────────────────┐          │
│  │  EHR Backend                │   │  etl-bq-pipeline             │          │
│  │  [EHR REST API]             │   │  [Pentaho + Python + BQ SQL] │          │
│  └─────────────────────────────┘   └──────────────┬───────────────┘          │
│                                                    │                          │
│  ┌─────────────────────────────────────────────────▼──────────────────────┐  │
│  │  infra-manifests [CDKTF + Helm / GKE]                                  │  │
│  │  Deploys: middleware, etl-bde, ehr-sdk as K8s services                 │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

C4 diagrams should be maintained as code (Structurizr DSL, Mermaid, or PlantUML) and committed alongside the architecture they describe. A diagram that lives in a PowerPoint deck is a diagram that will not be updated.

**Common mistake:** Creating C4 diagrams at the wrong level for the audience. Context diagrams shown to engineers bore them; Component diagrams shown to executives confuse them. Always state which level you are presenting and who the intended audience is.

---

## 5. Making Decisions Under Uncertainty

The defining skill gap between senior and staff engineers is not technical — it is the ability to make good decisions with incomplete information, on a timeline that does not permit waiting for certainty. Senior engineers often stall because they want more data. Staff engineers develop frameworks for deciding when they have enough data.

**The reversibility heuristic:** Jeff Bezos's "Type 1 / Type 2" framework is useful here. Type 1 decisions are irreversible or very costly to undo (choosing a primary database engine, defining an API contract that external systems depend on, selecting a vendor). Type 2 decisions are easily reversible (which feature to build first, internal refactoring approaches, tooling choices with no external interface). Apply extreme rigor to Type 1; apply speed and experimentation to Type 2. Many decisions engineers treat as Type 1 are actually Type 2.

**The 70% information rule:** If you have 70% of the information you would ideally want, that is often enough to make a good decision. The cost of waiting for the remaining 30% frequently exceeds the cost of making a slightly imperfect decision and correcting it. The exception is Type 1 decisions with catastrophic failure modes.

**Pre-mortem:** Before committing to a decision, run a 10-minute pre-mortem: "Imagine it is 12 months from now and this decision turned out to be a disaster. What happened?" This surfaces failure modes that forward-looking analysis misses because it bypasses optimism bias.

**Integration platform application:** When deciding whether to add a new sync direction (e.g., BigQuery → Salesforce writeback), a pre-mortem might surface: data quality issues in BigQuery propagating into the system of record; Salesforce API limits being consumed by a high-volume reverse sync; engineers spending months building and operating a system that business stakeholders later deprioritize. Each of these is either a decision driver or a condition that must be met before proceeding.

**Documenting decisions in the open:** Make your reasoning visible. When you make a technical call in Slack or a meeting, follow up with a written summary of the decision, the alternatives you considered, and your reasoning. This is proto-ADR behavior — it builds the habit and creates accountability without requiring a formal process.

**Common mistake:** Confusing analysis with progress. Producing a 20-page analysis document that exhaustively covers every consideration — but recommends no decision and sets no deadline — is a failure mode common among senior engineers moving toward staff. Analysis is an input to a decision, not a substitute for one.

---

## 6. Driving Alignment Without Authority

Staff engineers rarely have direct reports, yet they are expected to drive significant technical changes across multiple teams. This requires influence skills that are different from management skills and that are rarely taught in technical career paths.

**Understand before proposing.** The most effective way to build buy-in is to talk to stakeholders before you have a solution. Learn their constraints, their concerns, and their history with the problem space. Engineers who jump straight to proposing solutions frequently encounter resistance that would have been preemptively addressed had they done 30 minutes of listening first.

**Make the status quo expensive.** The default organizational answer to any proposal for change is "not now." Alignment happens when stakeholders perceive the cost of inaction as greater than the cost of the change. Use data: incident history, engineer-hours spent on manual processes, error rates, customer impact. For the integration platform, quantifying the operational cost of the current synchronous Apex-to-middleware coupling — in terms of incidents, SLA violations, and engineer time spent on recovery — makes the case for an async architecture far more compelling than technical elegance alone.

**Find your allies early.** In any significant technical change, there are engineers and leads on affected teams who will be natural advocates. Find them early, share your draft RFC with them before it is public, and incorporate their feedback. They become co-owners rather than reviewers, and they will advocate internally on their teams.

**Give people a face-saving path.** If your proposal implicitly criticizes decisions that were made by people who are still in the room, those people will resist not because your proposal is wrong, but because accepting it feels like admitting failure. Acknowledge the wisdom of the original decision in its original context. Explain why the context has changed. This is not spin — it is usually accurate, and it removes the ego obstacle from a technical conversation.

**Common mistake:** Treating alignment as a one-time event rather than an ongoing process. Getting a decision accepted in an RFC is not the same as having alignment. Plans change, people forget, new engineers join who were not part of the original discussion. Staff engineers continuously invest in re-communicating the "why" behind architectural decisions, especially during implementation when the original context is no longer fresh.

---

## 7. Mentoring and Engineering Culture

Mentoring is one of the highest-leverage activities available to a staff engineer, and one of the most underinvested in. A senior engineer you invest six months in mentoring can become self-sufficient for the next decade. The return on that investment compounds.

**Mentoring vs. sponsoring.** Mentoring is advice and guidance. Sponsoring is using your political capital to create opportunities for someone: recommending them for a high-visibility project, advocating for their promotion, nominating them to give a talk at an internal tech summit. Staff engineers should do both. Sponsorship is rarer and often more impactful, especially for engineers from underrepresented groups.

**Technical mentoring on the integration platform.** Given the system's complexity, a high-impact mentoring contribution is building the onboarding materials and architectural understanding that took you months to acquire. A new engineer joining the integration platform team needs to understand: the bidirectional sync model; why queue tables exist in `crm-middleware`; the distinction between the BDE and BQ ETL pipelines; how Apex triggers flow through to the EHR system. The C4 diagrams you draw this week, the ADRs you write, and the RFC templates you establish are all mentoring infrastructure — they scale your knowledge beyond the 1:1 conversations you can have.

**Engineering culture.** Culture is the set of behaviors that are modeled, rewarded, and tolerated. Staff engineers shape culture by what they do visibly, not by what they say. If you want a culture where engineers write design docs before coding, write design docs before coding — visibly, and comment thoughtfully on others'. If you want a culture of blameless post-mortems, write the blameless post-mortem after the next incident you are part of and share it openly.

The five cultural attributes most consistently associated with high-performing engineering organizations (from the DORA research and Google's Project Aristotle): **psychological safety** (people can raise concerns without punishment); **learning culture** (incidents are analyzed, not blamed); **high deployment frequency** (fast feedback loops); **clear mission** (engineers understand why their work matters); and **technical excellence** (code quality and architecture are treated as first-class concerns).

**Common mistake:** Mentoring people in your own image. The goal is not to produce more engineers who think and work exactly like you do — it is to help each person develop in the direction that is authentic to their strengths. Ask mentees what they want to work on and where they feel stuck, rather than assigning them the projects you find interesting.

---

## 8. Designing the 2027 Learning and Career Plan

You have completed 29 weeks of structured, deep technical study. The habit infrastructure is built. Week 29 is the moment to convert that infrastructure into a deliberate 2027 plan. Here is a framework for doing so.

### Step 1 — Self-Assessment Against Staff Criteria

Most organizations define staff engineer criteria across four dimensions. Assess yourself honestly (1–5) on each:

| Dimension | Description | Self-Score |
|---|---|---|
| Technical depth | Expert in 1–2 domains; able to debug at any layer | /5 |
| Technical breadth | Conversant across the full stack; knows enough to collaborate | /5 |
| System design | Produces correct, well-documented designs for complex systems | /5 |
| Impact and influence | Changes outcomes for teams beyond your own | /5 |
| Execution | Ships significant projects; manages complexity and risk | /5 |

After 28 weeks covering your full stack, your technical depth and breadth scores should be materially higher than they were in January 2026. The dimensions most staff engineers find hardest to demonstrate — and where a deliberate 2027 plan adds the most value — are **impact/influence** and **system design**. These require opportunities, not just knowledge.

### Step 2 — Identify Your 2027 Leverage Points

What are the two or three integration platform projects or initiatives in 2027 where you could have staff-level impact? Consider:

- Is there a major architectural change under consideration (async sync, new API version, migration off a legacy component)?
- Is there a reliability initiative where you could lead the observability and SLO work?
- Is there an onboarding or documentation gap where you could build the knowledge infrastructure?
- Is there a cross-team initiative (BDE ETL + BQ ETL convergence, unified monitoring, Kubernetes upgrade) where you could be the technical anchor?

Choose two or three such initiatives and write them as OKRs:

```
Objective: Establish integration platform async processing architecture
  KR1: RFC accepted by stakeholders by Q1 2027
  KR2: Proof-of-concept deployed to QA environment by Q2 2027
  KR3: Production rollout complete with < 0.1% event loss rate by Q3 2027

Objective: Build integration platform architectural knowledge base
  KR1: C4 diagrams (Context + Container) committed to all six repos by Q1
  KR2: 10 ADRs written for existing architectural decisions by Q2
  KR3: New engineer onboarding time reduced from ~4 weeks to ~2 weeks
```

### Step 3 — Identify Skill Gaps for 2027

Your 29-week plan covered the technical stack comprehensively. The skills most likely to be your limiting factor in the staff transition are non-technical:

- **Written communication:** RFCs, design docs, incident write-ups, architecture summaries for non-technical audiences.
- **Facilitation:** Running architecture review meetings, technical decision sessions, pre-mortems, and blameless post-mortems.
- **Stakeholder management:** Identifying stakeholders, managing their expectations, communicating technical trade-offs in business terms.
- **Scope management:** Knowing when a project has grown too large and needs to be cut, and having the credibility to make that call.

Build these into your 2027 plan as explicitly as you would build in a study week on Kubernetes.

### Step 4 — Establish Your 2027 Learning Rhythm

The 29-week study habit is your most valuable asset. Protect it. In 2027, the rhythm should shift: fewer weeks of structured learning, more weeks of producing — writing that RFC, that ADR, that post-mortem, that architecture review. Learning in 2027 is validated by artifacts that make the organization smarter, not just you.

A suggested 2027 cadence:
- **Weekly:** One hour of deliberate reading or study (papers, books, deep technical posts)
- **Monthly:** One artifact — an ADR, a design doc excerpt, a post-mortem, a runbook
- **Quarterly:** One significant written proposal or architectural analysis shared beyond your immediate team

**Common mistake:** Treating the staff promotion as the end goal rather than a milestone. The promotion is an organization's acknowledgement that you are already operating at staff level. Focus on operating at staff level in 2027 — the promotion follows from that.

---

## 9. Key Concepts Summary

```
Staff Engineering — Synthesis
├── Archetypes
│   ├── Tech Lead (team depth)
│   ├── Architect (cross-team systems)
│   ├── Solver (ambiguous hard problems)
│   ├── Right Hand (executive leverage)
│   └── Community Builder (culture & capability)
│
├── Artifacts
│   ├── RFC
│   │   ├── Problem → Alternatives → Design → Metrics
│   │   └── Goal: alignment before implementation
│   ├── ADR
│   │   ├── Context → Options → Decision → Consequences
│   │   └── Goal: institutional memory across turnover
│   └── C4 Diagram
│       ├── L1 Context → L2 Container → L3 Component
│       └── Goal: shared mental model at correct abstraction
│
├── Decision-Making
│   ├── Reversibility heuristic (Type 1 vs. Type 2)
│   ├── 70% information rule
│   └── Pre-mortem
│
├── Influence
│   ├── Understand before proposing
│   ├── Make status quo expensive (with data)
│   ├── Find allies early
│   └── Give face-saving paths
│
├── Culture & Mentoring
│   ├── Model the behaviors you want
│   ├── Mentor + sponsor (different)
│   └── Knowledge infrastructure = scaled mentoring
│
└── 2027 Plan
    ├── Self-assess against staff criteria
    ├── Identify leverage points (OKRs)
    ├── Close non-technical skill gaps
    └── Shift from consuming to producing
```

---

## Quiz — 20 Questions

### Questions

**1.** What are the five staff engineer archetypes, and which two seem most aligned with a cross-cutting integration platform like the CRM-EHR Integration Platform?

**2.** What is the primary purpose of an RFC, and how does it differ from a design doc?

**3.** An ADR has been Accepted for using PostgreSQL queue tables in the integration platform. A year later, message volume has grown 100x and you want to move to Kafka. What do you do with the existing ADR?

**4.** What are the four levels of the C4 model, and which two are most commonly used for communication outside the immediate engineering team?

**5.** Explain the "Type 1 / Type 2 decision" framework and give one example of each from the integration platform context.

**6.** What is the 70% information rule, and when should you NOT apply it?

**7.** You have proposed an RFC to introduce async event processing in the integration platform. A senior engineer on the middleware team is resistant. They originally designed the synchronous coupling you are proposing to replace. What technique helps address this resistance without escalating?

**8.** What is the difference between mentoring and sponsoring?

**9.** Name three of the five DORA/Project Aristotle cultural attributes associated with high-performing engineering teams.

**10.** You are writing an RFC for the integration platform's migration to async trigger processing. What data would you include in the Motivation section to make the case compelling?

**11.** An ADR you wrote six months ago is now being ignored by engineers who were not at the company when it was written. What should you have done differently, and what can you do now?

**12.** Where should ADRs be stored in the integration platform repository structure?

**13.** What is a pre-mortem, and how does it differ from a post-mortem?

**14.** You are asked to present the integration platform architecture to a group of new engineering executives who have no Salesforce or EHR system background. Which C4 level do you use, and why?

**15.** A staff engineer is evaluated partly on "impact beyond your immediate team." Give two concrete examples of staff-level impact in the integration platform context.

**16.** The C4 diagrams you draw are committed to the repository as Mermaid or Structurizr DSL rather than as PNG screenshots. Why does this matter?

**17.** What is the most common failure mode when senior engineers try to move into staff-level decision-making?

**18.** What is the difference between "alignment" as a one-time event and alignment as an ongoing process, and why does the distinction matter for staff engineers?

**19.** You are designing your 2027 OKRs for the staff transition. You identify four possible leverage points. How do you decide which two or three to focus on?

**20.** After 29 weeks of structured study, what is the single most important shift in how you spend your time as you move from senior to staff engineer?

---

### Answers

??? note "Reveal Answers"

    **1.** The five archetypes are Tech Lead, Architect, Solver, Right Hand, and Community Builder. For the CRM-EHR Integration Platform specifically, the Architect and Solver archetypes are most naturally aligned. The integration platform spans six repositories, three data backends, two ETL pipelines, and a complex bidirectional sync model — it benefits most from someone who can own cross-cutting architectural decisions (Architect) and who can be deployed to diagnose the ambiguous multi-system failures that no single team fully owns (Solver). The Tech Lead archetype is valuable too, but its leverage is more limited to a single team boundary.

    **2.** An RFC is a structured proposal seeking stakeholder input and alignment *before* significant implementation work begins. Its primary purpose is to surface hidden assumptions, invite expertise, and build shared ownership of the direction. A design doc is typically produced *after* an approach has been validated — it documents the implementation design in detail for the engineers building it. RFCs tend to be higher-level and more focused on "what and why"; design docs tend to be more detailed on "how." In practice many teams use the terms interchangeably, but the key distinction is timing: RFCs are for alignment, design docs are for implementation guidance.

    **3.** You do not delete or edit the original ADR. You update its status to "Superseded by ADR-XXXX" and create a new ADR documenting the new decision — including the context that has changed (100x message volume growth), the alternatives considered, and the reasoning for moving to Kafka. The original ADR remains in the repository as a permanent record. This is one of the core principles of ADRs: they are append-only. Future engineers can trace the entire decision history, including why the system was built the way it was at each stage.

    **4.** The four C4 levels are Context (system in its environment), Container (major deployable units), Component (internals of a container), and Code (class-level detail). Context and Container are the two most commonly used for communication outside the immediate engineering team. Context diagrams work for non-technical stakeholders and executive briefings; Container diagrams work for engineers on adjacent teams who need to understand how systems integrate without diving into internal implementation. Component and Code diagrams are primarily useful for engineers actively building or debugging a specific container.

    **5.** Type 1 decisions are effectively irreversible or extremely costly to undo and warrant rigorous deliberation. Type 2 decisions are easily reversible and should be made quickly with the information available. In the integration platform: choosing PostgreSQL as the middleware database when the platform was first built is a Type 1 decision — migrating off it would require coordinated changes across middleware, ETL, deployment manifests, and operational runbooks. Choosing whether to add a new custom field to the `APP__Policy__c` object for a new feature is a Type 2 decision — it can be removed or renamed later with manageable cost.

    **6.** The 70% information rule states that if you have roughly 70% of the information you would ideally want, that is often enough to make a good decision and proceed. You should NOT apply it when: the decision is Type 1 (effectively irreversible), when the downside risk is catastrophic (e.g., data loss affecting production EHR records), or when the cost of gathering the remaining 30% of information is low and the timeline permits it. The rule exists to counter the common failure mode of analysis paralysis — not to encourage recklessness on high-stakes decisions.

    **7.** The most effective technique is to give the resistant engineer a face-saving path by acknowledging the wisdom of the original design in its original context. You might say: "The synchronous design made sense when the system was smaller and middleware reliability was not a constraint — it kept things simple and easy to debug. What's changed is that middleware is now a shared dependency with more teams and more failure modes." This reframes the conversation from "your design was wrong" to "the context has changed." You should also have shared the RFC draft with this engineer *before* it went public and incorporated their technical concerns into the design — co-ownership removes the ego obstacle entirely.

    **8.** Mentoring is providing guidance, advice, feedback, and technical coaching to help someone develop their skills and navigate challenges. Sponsoring is using your own reputation and political capital to create concrete opportunities for someone: recommending them for a high-visibility project, advocating for their promotion in calibration discussions, nominating them for a conference talk, or connecting them to influential people. Mentoring helps someone grow; sponsoring creates the opportunities that demonstrate and accelerate that growth. Staff engineers should do both, and should be aware that sponsorship is rarer and often more impactful — especially for engineers from underrepresented groups who may receive mentoring but be systematically under-sponsored.

    **9.** Any three of: psychological safety (people can raise concerns, ask questions, and admit mistakes without fear of punishment), learning culture / blameless post-mortems (incidents are analyzed for system improvement rather than individual blame), high deployment frequency (fast feedback loops between writing code and observing its behavior in production), clear mission alignment (engineers understand why their work matters and how it connects to organizational goals), and technical excellence as a first-class concern (code quality, architecture, and operational health are treated as important as feature delivery). These attributes are from the DORA State of DevOps research and Google's Project Aristotle.

    **10.** Compelling Motivation sections for the integration platform async processing RFC would include: the number of incidents in the past quarter caused by middleware degradation propagating back to Salesforce transaction failures; the average engineer-hours spent per incident on recovery and data reconciliation; the number of data loss events (records dropped or duplicated) attributable to the synchronous coupling; the customer-visible impact (which insurance producers experienced errors or delays); and the trend line (is this getting worse as the system scales?). Concrete data transforms a proposal from "this would be architecturally nicer" into "this is costing us X hours and Y customer incidents per quarter."

    **11.** Two things should have been done differently: first, ADRs should be stored in the repository alongside the code they govern, not in a separate wiki or shared document that can be overlooked; second, they should be referenced from the relevant code (e.g., a comment in the queue table model pointing to `docs/adr/0003-postgresql-queue.md`). What you can do now: add links to the ADR in the codebase, include a summary of key ADRs in the team's onboarding documentation, and review relevant ADRs as part of the architecture discussion at the start of any project that touches the affected system.

    **12.** ADRs should be stored in the repository that the decision most directly governs: `crm-middleware/docs/adr/` for middleware architecture decisions (database choices, API versioning strategy, queue design); `infra-manifests/docs/adr/` for infrastructure decisions (Kubernetes configuration, Vault integration, monitoring stack); a top-level shared `docs/adr/` for cross-repository decisions that span multiple systems (e.g., the decision to use GCP as the cloud provider for all integration platform services). Storing ADRs in the repo ensures they are versioned alongside the code and are accessible to engineers who have cloned the repo.

    **13.** A pre-mortem is a prospective exercise conducted *before* a project or decision is executed: you imagine that the project has already failed and ask "what went wrong?" This surfaces failure modes that forward-looking analysis misses because it bypasses optimism bias. A post-mortem is a retrospective analysis conducted *after* an incident or project failure to understand root causes and prevent recurrence. The key difference is timing and purpose: pre-mortems improve decisions before they are made; post-mortems improve systems and processes after failure. Both are valuable — pre-mortems are underused because they require imagining failure when the team is optimistic about a plan.

    **14.** You use the C4 Level 1 Context diagram. Executive stakeholders with no Salesforce or EHR system background do not need to understand that `crm-middleware` is a FastAPI application backed by PostgreSQL — they need to understand that the integration platform connects your CRM (Salesforce) with the EHR system, feeds an analytics warehouse (BigQuery), and is used by insurance producers. The Context diagram shows the system's purpose and its relationship to external actors without any implementation detail. Showing a Container diagram to this audience adds noise without adding understanding and risks a meeting that derails into implementation questions.

    **15.** Two concrete examples: (1) Writing and driving acceptance of an RFC that aligns the middleware team, the ETL team, and Salesforce developers on an async processing architecture — this changes the direction of three teams and prevents a class of production incidents for years. (2) Building the C4 diagrams and ADR library for the integration platform and integrating them into onboarding, reducing new engineer ramp time from four weeks to two — this multiplies the productivity of every engineer who joins the team going forward. Both examples demonstrate impact that extends well beyond any individual pull request or feature.

    **16.** Diagrams stored as code (Mermaid, Structurizr DSL, PlantUML) are version-controlled, diffable, reviewable in pull requests, and can be regenerated from source. PNG screenshots in a repository become stale and uneditable — when the architecture changes, someone must open a separate tool, redraw the diagram, export a new image, and commit it. Diagram-as-code integrates into the same review workflow as code changes: when an engineer adds a new container to the integration platform system, the PR includes the updated Mermaid source and the reviewer can see exactly what changed. This is the same principle as infrastructure-as-code: the source of truth should be the thing that is versioned and reviewed.

    **17.** The most common failure mode is confusing analysis with progress — producing exhaustive documents that explore every consideration in depth but defer the actual decision, set no deadline, and make no recommendation. This often stems from the senior engineer's discomfort with the ambiguity inherent in decisions that cannot be fully validated in advance. Staff engineering requires developing comfort with the fact that you will sometimes be wrong, and that reversible decisions made at 70% information are better than paralysis waiting for 100%. The corollary failure mode is writing a recommendation but not following up to ensure it is actually acted on — analysis without accountability.

    **18.** Alignment as a one-time event means getting a decision accepted in an RFC meeting or design review and considering the work done. Alignment as an ongoing process recognizes that team membership changes, people forget, context evolves, and new engineers join who were not part of the original decision. Staff engineers continuously reinvest in communicating the "why" behind architectural decisions — through documentation, onboarding conversations, code comments pointing to ADRs, and architecture discussions at the start of new projects. This matters because the most common reason technical decisions get relitigated or circumvented is not that people disagree with them — it is that they have forgotten or never knew the reasoning.

    **19.** Prioritize based on three criteria: organizational impact (which initiative, if it succeeds, creates the most value for the platform and its users?), visibility (which initiative will demonstrate staff-level work to the people making promotion decisions?), and your unique leverage (where does your specific combination of context — 29 weeks of full-stack study, five years on the integration platform, knowledge of all six repositories — create an advantage that no one else has?). Initiatives where all three criteria align are the highest-priority choices. Avoid the trap of optimizing only for visibility — high-visibility projects that you are not uniquely positioned to succeed at are worse choices than lower-visibility projects where your leverage is decisive.

    **20.** The single most important shift is moving from consuming knowledge to producing artifacts. As a senior engineer, learning is primarily personal: you study, you build skills, you deliver features. As a staff engineer, learning is organizational: you write the RFC that makes three teams smarter; you draw the diagram that makes the architecture legible to the next engineer who joins; you write the ADR that prevents a bad decision from being made again; you mentor the engineer who then solves problems you will never have to touch. Your value compounds through the artifacts and people you invest in, not through what you personally ship. After 29 weeks of deep study, you have the knowledge. 2027 is the year to convert it into force multiplication.
