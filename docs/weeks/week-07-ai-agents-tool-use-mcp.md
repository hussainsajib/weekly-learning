# Week 7 — AI Agents, Tool Use & MCP

**Week of:** July 20, 2026
**Estimated study time:** ~2 hours
**Tags:** `ai` `agents` `mcp` `tools`

---

## Overview

For the first six weeks of this plan you have studied how to talk to an LLM — how to authenticate, prompt, retrieve context, and call the Claude API. This week the framing shifts: instead of you driving the conversation, the model drives itself. An *agent* is an LLM embedded in a loop that can take actions, observe results, and decide what to do next. That sounds simple, but the implications are profound. The model is no longer a stateless function that returns text; it is a participant in a stateful, multi-step process that can read databases, call APIs, write files, and orchestrate other models.

This matters directly to your work on the integration platform. The middleware you maintain is essentially a deterministic agent built from human-authored if/else logic: receive a Salesforce trigger event, decide which EHR system endpoints to hit, handle errors, retry. An AI agent could, in principle, take a natural-language task ("sync all Accounts created in the last hour that don't have a matching EHR client") and produce a correct EHR sync plan, execute it step by step, and report on anomalies — without you writing the orchestration code. Whether you build that or not, you need to understand the machinery so you can evaluate where agents genuinely help versus where they introduce failure modes that deterministic code avoids.

This week covers the four major pillars: (1) agentic reasoning patterns like ReAct and plan-and-execute, (2) the tool/function-calling mechanism that lets models take actions, (3) the Model Context Protocol (MCP), which standardizes how tools are exposed to agents, and (4) orchestration choices — LangChain, raw Anthropic SDK, and custom loops. You will end the week with a concrete mental model for when to use each pattern and hands-on Python code grounded in your actual stack.

After this week you should be able to design an agent loop from scratch using the Anthropic SDK, write an MCP server that exposes your FastAPI middleware as a tool, understand the tradeoffs between single-agent and multi-agent architectures, and critically evaluate when agentic approaches make things better and when they make things worse.

---

## 1. What Makes Something an Agent?

The word "agent" is overloaded. A marketing chatbot is sometimes called an agent. So is a fully autonomous system that operates a data center. To reason clearly, it helps to define a spectrum.

**The autonomy spectrum:**

```
Stateless LLM call → Chatbot → Tool-augmented model → Agent → Multi-agent system
      (no loop)       (memory)    (one tool call)       (loop)      (delegation)
```

The key ingredient that distinguishes an agent from a chatbot is the *action-observation loop*: the model produces an action (call a tool, write to memory, delegate to another model), receives an observation (the tool's response), and then continues reasoning. The loop repeats until the model decides it is done or a halt condition fires.

**The minimal agent loop in pseudocode:**

```python
messages = [{"role": "user", "content": task}]

while True:
    response = llm.call(messages, tools=available_tools)

    if response.stop_reason == "end_turn":
        # Model is done
        print(response.content[-1].text)
        break

    if response.stop_reason == "tool_use":
        # Model wants to call a tool
        tool_call = extract_tool_call(response)
        result = execute_tool(tool_call)
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": [
            {"type": "tool_result", "tool_use_id": tool_call.id, "content": result}
        ]})
```

This is the skeleton of every agent, regardless of framework. LangChain, LangGraph, CrewAI — all of them are different ways of filling in this skeleton. Understanding the skeleton means you can debug any framework and can build your own when the framework gets in your way.

**What an agent is NOT:**

- A single LLM call that returns a long response is not an agent.
- A chatbot with conversation history is not an agent (no external actions).
- A workflow with hard-coded API calls and no LLM decisions is not an agent; it is a script.

**Common mistake:** Calling any LLM-powered feature an "agent" and then applying agentic architecture (loops, tool registries, multi-step orchestration) to problems that a single well-prompted LLM call would solve in 100ms with no failure modes. Agents add complexity, latency, cost, and non-determinism. Use them when the task genuinely requires multiple steps where each step depends on the result of the previous one and those steps cannot be predetermined.

---

## 2. Agentic Reasoning Patterns

### ReAct (Reason + Act)

ReAct is the most widely used agentic pattern. The model interleaves *thought* (chain-of-thought reasoning about what to do next) with *action* (tool calls). The original paper (Yao et al., 2022) showed this outperforms pure chain-of-thought on multi-step tasks because the model can ground its reasoning in real observations rather than hallucinating intermediate facts.

**ReAct trace example for the integration platform:**

```
Task: "Find all Salesforce Accounts that were updated in the last 24 hours
       but do not have a corresponding EHR client. Return a summary."

Thought: I need to query Salesforce for recently updated Accounts. I'll use
         the query_salesforce tool with a SOQL query.
Action: query_salesforce(soql="SELECT Id, Name, APP__EHR_Client_Id__c
        FROM Account WHERE LastModifiedDate = LAST_N_DAYS:1")
Observation: [{id: "001...", name: "Acme Corp", ehr_client_id: None}, ...]

Thought: I have 14 accounts. I need to filter those where ehr_client_id is
         null. That's 3 accounts. I should verify against the EHR system directly.
Action: query_ehr_clients(client_ids=["001...", "002...", "003..."])
Observation: No clients found for those IDs.

Thought: Confirmed — 3 accounts need sync. I'll format the summary.
Action: (none — end_turn)
Answer: "3 accounts require EHR sync: Acme Corp, Beta LLC, Gamma Inc.
         None have a corresponding EHR client record."
```

With the Anthropic SDK, thoughts appear as `text` blocks before `tool_use` blocks in the assistant message content array. You do not need to do anything special to get ReAct-style reasoning — Claude does this naturally when given tools and a multi-step task. What you do need to do is pass the full message history including tool results so the model can observe and continue.

### Plan-and-Execute

ReAct is reactive — the model figures out the next step only after seeing the previous result. Plan-and-execute separates the planning phase from the execution phase.

```
Phase 1 (Plan):   LLM generates a numbered plan: Step 1, Step 2, Step 3...
Phase 2 (Execute): Each step is executed, often by a separate "executor" model
                   or by the same model with the plan injected as context.
```

**When to prefer plan-and-execute over ReAct:**

| Scenario | ReAct | Plan-and-Execute |
|----------|-------|-----------------|
| Short tasks (2-4 steps) | Better | Overkill |
| Long tasks (10+ steps) | Tends to drift | Better: plan is a contract |
| Steps can parallelize | Poor fit | Good: plan exposes parallelism |
| User needs to approve before execution | Awkward | Natural: show plan, get approval |
| Debugging failures | Hard: trace is long | Easier: which step failed? |

For the integration platform, a plan-and-execute architecture makes sense for a "reconcile all Accounts in org" job: the planning step generates a list of batches; the execution step processes each batch independently, which can even run in parallel with `asyncio.gather`.

### Reflection and Self-Critique

Reflection agents add a third step: after executing and observing, the model explicitly evaluates whether the result is correct or whether it should retry with a different approach.

```python
# Simplified reflection loop
for attempt in range(max_retries):
    result = execute_agent(task, tools)
    critique = llm.call([
        {"role": "user", "content": f"Task: {task}\nResult: {result}\n"
         "Is this result correct and complete? If not, what is wrong?"}
    ])
    if "looks correct" in critique.lower():
        return result
    # Otherwise loop with critique injected into next attempt
```

Reflection is expensive (doubles or triples your LLM calls) but dramatically reduces hallucination on tasks where correctness is verifiable. In integration platform terms: after an agent generates a sync plan, a reflection step could verify that every referenced field name actually exists in your Salesforce schema before any EHR API calls are made.

**Common mistake:** Running reflection in an unbounded loop. Always set a `max_retries` guard (typically 2-3). An agent stuck in a reflection loop is one of the most expensive bugs you can ship.

---

## 3. Tool / Function Calling in Depth

Tool calling is the mechanism that gives agents the ability to act. The model doesn't actually execute tools — it outputs a structured JSON object describing which tool to call and with what arguments. Your code executes the tool and feeds the result back.

### The Tool Definition

In the Anthropic SDK, tools are defined as a list of JSON Schema objects passed to the `tools` parameter:

```python
import anthropic
import json

client = anthropic.Anthropic()

# Define a tool that queries the crm-middleware
tools = [
    {
        "name": "query_middleware_accounts",
        "description": (
            "Query the crm-middleware API for Salesforce Account records. "
            "Returns a list of accounts matching the given filters. "
            "Use this when you need to look up account data."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "updated_after": {
                    "type": "string",
                    "description": "ISO 8601 datetime. Only return accounts updated after this time.",
                },
                "has_ehr_client": {
                    "type": "boolean",
                    "description": "If false, return only accounts without an EHR client mapping.",
                },
                "limit": {
                    "type": "integer",
                    "description": "Maximum number of records to return. Default 50.",
                    "default": 50,
                },
            },
            "required": [],
        },
    },
    {
        "name": "trigger_ehr_sync",
        "description": (
            "Trigger an EHR sync for a specific Salesforce Account ID. "
            "This calls the middleware /sync endpoint which queues the sync job."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "account_id": {
                    "type": "string",
                    "description": "Salesforce Account ID (18-character).",
                },
                "force": {
                    "type": "boolean",
                    "description": "If true, sync even if no changes detected.",
                    "default": False,
                },
            },
            "required": ["account_id"],
        },
    },
]
```

**The description field is load-bearing.** The model decides which tool to call (and when) almost entirely based on the description. A vague description like "gets accounts" leads to the model calling the tool at wrong times or with wrong arguments. Write descriptions as if you are writing documentation for a human developer who has never seen your API.

### Handling Tool Calls

```python
import httpx
from typing import Any

async def execute_tool(tool_name: str, tool_input: dict[str, Any]) -> str:
    """Execute a tool call and return the result as a string."""
    async with httpx.AsyncClient(base_url="http://localhost:8000") as http:
        if tool_name == "query_middleware_accounts":
            params = {k: v for k, v in tool_input.items() if v is not None}
            resp = await http.get("/api/v2/accounts", params=params)
            resp.raise_for_status()
            data = resp.json()
            return json.dumps(data, indent=2)

        elif tool_name == "trigger_ehr_sync":
            resp = await http.post(
                f"/api/v2/accounts/{tool_input['account_id']}/sync",
                json={"force": tool_input.get("force", False)},
            )
            resp.raise_for_status()
            return json.dumps(resp.json(), indent=2)

        else:
            return json.dumps({"error": f"Unknown tool: {tool_name}"})
```

### The Full Async Agent Loop

```python
import asyncio
import anthropic

async def run_agent(task: str) -> str:
    client = anthropic.Anthropic()
    messages = [{"role": "user", "content": task}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        # Add assistant's response to history
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            # Find the final text response
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return "(no text response)"

        if response.stop_reason == "tool_use":
            # Process all tool calls in this response (there can be multiple)
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = await execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })

            # Feed all results back in a single user message
            messages.append({"role": "user", "content": tool_results})

        else:
            # stop_reason is "max_tokens" or something unexpected
            raise RuntimeError(f"Unexpected stop_reason: {response.stop_reason}")


# Usage
result = asyncio.run(run_agent(
    "Find all Salesforce Accounts updated in the last 24 hours "
    "that don't have an EHR client. Trigger sync for each one."
))
print(result)
```

**Parallel tool calls:** Claude will sometimes return multiple `tool_use` blocks in a single response when the tool calls are independent. The loop above handles this correctly — it collects all results and sends them back together. Do not send results one at a time; this breaks the conversation structure.

**Common mistake:** Forgetting to append the assistant's message to `messages` before appending the tool results. The conversation must be `user → assistant → user (tool results) → assistant → ...`. If you skip the assistant message, the API will reject the request with a validation error about alternating roles.

---

## 4. Model Context Protocol (MCP)

### The Problem MCP Solves

Without MCP, every agent framework invents its own tool format. LangChain tools look different from Anthropic SDK tools, which look different from OpenAI function calling. If you build a tool — say, a tool that queries your crm-middleware — you have to rewrite it for each framework. This is the connector problem: N frameworks × M tools = N×M integrations.

MCP (Model Context Protocol) is an open protocol created by Anthropic in late 2024 that standardizes the interface between *hosts* (applications like Claude Code, Claude Desktop, or your custom agent) and *servers* (processes that expose tools, resources, and prompts).

```
┌─────────────────────────────────────────────────────────────┐
│  MCP Host (Claude Code / your FastAPI agent / Claude.ai)    │
│                                                             │
│  ┌─────────────┐    JSON-RPC 2.0     ┌──────────────────┐  │
│  │  MCP Client │◄──────────────────►│   MCP Server     │  │
│  └─────────────┘   (stdio or SSE)   │                  │  │
│                                     │  • Tools         │  │
│                                     │  • Resources     │  │
│                                     │  • Prompts       │  │
│                                     └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### MCP Architecture

**Transport layer:** MCP servers communicate via JSON-RPC 2.0. Two transports are supported:
- `stdio` — the host spawns the server as a subprocess and communicates via stdin/stdout. This is the most common for local tools.
- `SSE (Server-Sent Events)` — the server is a remote HTTP service. Used for shared or cloud-hosted tools.

**Three capability types:**

| Capability | What it is | Integration Platform Example |
|-----------|------------|-------------|
| **Tools** | Functions the model can call | `sync_account`, `query_ehr_clients` |
| **Resources** | Read-only data the model can read | middleware config, crm-middleware OpenAPI spec |
| **Prompts** | Reusable prompt templates | "Reconcile Accounts prompt" |

### Writing an MCP Server for the crm-middleware

The `mcp` Python package (from Anthropic) makes writing servers straightforward:

```python
# crm_middleware_mcp_server.py
import asyncio
import json
import httpx
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Initialize the MCP server
app = Server("crm-middleware")
MIDDLEWARE_BASE = "http://localhost:8000"


@app.list_tools()
async def list_tools() -> list[Tool]:
    """Declare all tools this server exposes."""
    return [
        Tool(
            name="query_accounts",
            description=(
                "Query crm-middleware for Salesforce Account records. "
                "Supports filtering by sync status, last modified date, "
                "and EHR client mapping status."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "sync_status": {
                        "type": "string",
                        "enum": ["pending", "synced", "error", "all"],
                        "description": "Filter by sync status. Default: 'all'.",
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Max records to return (1-200). Default: 50.",
                    },
                },
            },
        ),
        Tool(
            name="get_sync_queue",
            description=(
                "Return the current EHR sync queue — jobs waiting to be "
                "processed, currently running, and recently failed."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "status": {
                        "type": "string",
                        "enum": ["pending", "running", "failed", "all"],
                    }
                },
            },
        ),
        Tool(
            name="trigger_account_sync",
            description=(
                "Trigger an immediate EHR sync for a Salesforce Account. "
                "Use this only when the user explicitly wants to start a sync."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "account_id": {
                        "type": "string",
                        "description": "18-character Salesforce Account ID.",
                    },
                    "force_full_sync": {
                        "type": "boolean",
                        "description": "Re-sync all fields, not just changed ones.",
                        "default": False,
                    },
                },
                "required": ["account_id"],
            },
        ),
    ]


@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Execute a tool call and return the result."""
    async with httpx.AsyncClient(base_url=MIDDLEWARE_BASE, timeout=30.0) as client:
        try:
            if name == "query_accounts":
                params = {
                    "status": arguments.get("sync_status", "all"),
                    "limit": arguments.get("limit", 50),
                }
                resp = await client.get("/api/v2/accounts", params=params)
                resp.raise_for_status()
                return [TextContent(type="text", text=json.dumps(resp.json(), indent=2))]

            elif name == "get_sync_queue":
                params = {"status": arguments.get("status", "all")}
                resp = await client.get("/api/v2/sync/queue", params=params)
                resp.raise_for_status()
                return [TextContent(type="text", text=json.dumps(resp.json(), indent=2))]

            elif name == "trigger_account_sync":
                resp = await client.post(
                    f"/api/v2/accounts/{arguments['account_id']}/sync",
                    json={"force": arguments.get("force_full_sync", False)},
                )
                resp.raise_for_status()
                return [TextContent(type="text", text=json.dumps(resp.json(), indent=2))]

            else:
                return [TextContent(type="text", text=f"Unknown tool: {name}")]

        except httpx.HTTPStatusError as e:
            return [TextContent(
                type="text",
                text=json.dumps({
                    "error": f"HTTP {e.response.status_code}",
                    "detail": e.response.text,
                })
            )]


async def main():
    async with stdio_server() as streams:
        await app.run(*streams, app.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
```

### Registering the Server with Claude Code

Add to your project's `.claude/settings.json` or `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "crm-middleware": {
      "command": "python",
      "args": ["C:/Works/crm-middleware/crm_middleware_mcp_server.py"],
      "env": {
        "MIDDLEWARE_BASE_URL": "http://localhost:8000"
      }
    }
  }
}
```

After restarting Claude Code, you will see `crm-middleware` in the MCP server list and can use its tools directly in conversation — the same tools your custom agents call, but now also available to Claude Code itself.

**Common mistake:** Returning errors as exceptions from `call_tool`. When an MCP tool throws an uncaught exception, the host often shows a generic error with no details. Always catch exceptions inside `call_tool` and return them as `TextContent` with a JSON error payload. This gives the model actionable information it can reason about.

---

## 5. Multi-Agent Systems

### When Multiple Agents Help

Single-agent systems hit limits when:
1. **Context window pressure** — a task requires processing more tokens than fit in one context window.
2. **Parallelism** — subtasks are independent and could run concurrently.
3. **Specialization** — different subtasks benefit from different system prompts, tool sets, or even different models.
4. **Quality checks** — a second agent reviewing the first agent's output catches errors the first agent missed.

**A practical integration platform multi-agent example:**

```
Orchestrator Agent
├── Planner (Claude Sonnet 4.5 — cheap, fast planning)
│   └── Produces: list of Account batches to reconcile
├── Batch Workers (Claude Haiku 3.5 × N — fast, cheap execution)
│   └── Each worker: query accounts, check EHR system, flag mismatches
└── Reviewer Agent (Claude Opus 4.5 — expensive, thorough)
    └── Validates batch worker outputs, produces final report
```

```python
import asyncio
import anthropic
from dataclasses import dataclass

client = anthropic.Anthropic()


@dataclass
class BatchTask:
    batch_id: int
    account_ids: list[str]


async def run_batch_worker(task: BatchTask) -> dict:
    """Process one batch of accounts — uses a cheap, fast model."""
    response = client.messages.create(
        model="claude-haiku-3-5",  # fast and cheap for batch work
        max_tokens=1024,
        system=(
            "You are an integration platform sync auditor. You check whether Salesforce accounts "
            "have corresponding EHR client records. Be concise and factual."
        ),
        tools=tools,  # query_middleware_accounts, etc.
        messages=[{
            "role": "user",
            "content": (
                f"Check these account IDs for EHR sync status: {task.account_ids}. "
                "Return a JSON object with keys: synced, missing, errors."
            ),
        }],
    )
    # (simplified — real version would run the tool loop)
    return {"batch_id": task.batch_id, "result": response.content[-1].text}


async def orchestrate_reconciliation(account_ids: list[str]) -> str:
    """Orchestrate a full reconciliation across all accounts."""
    # Split into batches of 20
    batches = [
        BatchTask(i, account_ids[i:i+20])
        for i, _ in enumerate(range(0, len(account_ids), 20))
    ]

    # Run all batches concurrently
    batch_results = await asyncio.gather(
        *[run_batch_worker(batch) for batch in batches]
    )

    # Reviewer synthesizes results
    review_response = client.messages.create(
        model="claude-opus-4-5",  # thorough review
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": (
                f"Review these batch reconciliation results and produce "
                f"a final summary report:\n{batch_results}"
            ),
        }],
    )
    return review_response.content[-1].text
```

### When Multiple Agents Hurt

Multi-agent systems are not free. Every inter-agent communication is:
- **Latency**: sequential agent calls chain their latencies.
- **Cost**: each agent call bills tokens independently.
- **Fragility**: errors propagate and amplify between agents.
- **Debugging complexity**: a bug could be in any agent's reasoning, any tool, or the orchestration layer.

**The multi-agent smell test:**

| If you're doing this... | Consider instead... |
|------------------------|---------------------|
| Chaining agents where each adds a tiny refinement | One agent with a richer prompt |
| Delegating to sub-agents because you don't trust the main agent | Better tool descriptions and system prompt |
| Using multi-agent for a task that always takes the same N steps | Deterministic code with LLM calls at specific points |
| Building an agent to route between other agents | A `match` statement in Python |

**Common mistake:** Building a "manager agent" that does nothing but decompose tasks and delegate to specialized agents, where the real work is so small each specialized agent returns in one LLM call. The manager adds latency and cost with no benefit. If the subtasks are small and well-defined, just call them directly.

---

## 6. Orchestration: LangChain vs. Raw SDK vs. Custom

The orchestration question is where engineers waste the most time arguing. Here is a direct comparison.

### LangChain / LangGraph

**What it is:** A framework that provides pre-built agent executors, tool integrations, memory classes, and a graph-based workflow engine (LangGraph).

**When it wins:**
- You need to build a complex stateful workflow (LangGraph is genuinely good for multi-step graphs with cycles).
- You want ready-made integrations (hundreds of pre-built tools for databases, APIs, etc.).
- Your team is already familiar with it.

**When it loses:**
- You need to understand exactly what's happening (the abstraction layers make debugging painful).
- You're working with Claude specifically — many LangChain abstractions were designed for OpenAI and have impedance mismatch.
- Your prompts need to be precisely controlled (LangChain injects its own formatting).
- Production reliability matters — LangChain has a history of breaking changes.

### Raw Anthropic SDK

**What it is:** Calling `client.messages.create()` directly with your own agent loop.

**When it wins:**
- You understand exactly what's happening.
- You need precise control over prompts, token counting, and error handling.
- You're building a production system where reliability and debuggability matter.
- Your task has a simple structure (linear loop or shallow branching).

**When it loses:**
- You need a complex graph (loops within loops, parallel branches that merge).
- You need memory persistence across many sessions.
- Your team wants to move fast and doesn't want to write boilerplate.

### Custom Orchestration Layer

Build your own thin abstraction over the raw SDK. This is the approach that scales best for production systems.

```python
# platform_agent/core.py — a minimal, production-grade agent runner
from __future__ import annotations

import asyncio
import logging
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable

import anthropic

logger = logging.getLogger(__name__)


@dataclass
class AgentConfig:
    model: str = "claude-sonnet-4-5"
    max_tokens: int = 4096
    max_iterations: int = 10
    system_prompt: str = ""


ToolHandler = Callable[[str, dict[str, Any]], Awaitable[str]]


class PlatformAgent:
    """
    Minimal agent runner for integration platform operations.
    Wraps the Anthropic SDK with production concerns:
    iteration limits, structured logging, error handling.
    """

    def __init__(
        self,
        config: AgentConfig,
        tools: list[dict],
        tool_handler: ToolHandler,
    ) -> None:
        self._client = anthropic.Anthropic()
        self._config = config
        self._tools = tools
        self._handler = tool_handler

    async def run(self, task: str) -> str:
        messages: list[dict] = [{"role": "user", "content": task}]
        iterations = 0

        logger.info("agent_start", extra={"task": task[:100]})

        while iterations < self._config.max_iterations:
            iterations += 1
            logger.info("agent_iteration", extra={"iteration": iterations})

            response = self._client.messages.create(
                model=self._config.model,
                max_tokens=self._config.max_tokens,
                system=self._config.system_prompt,
                tools=self._tools,
                messages=messages,
            )

            messages.append({"role": "assistant", "content": response.content})

            if response.stop_reason == "end_turn":
                result = next(
                    (b.text for b in response.content if hasattr(b, "text")), ""
                )
                logger.info("agent_done", extra={"iterations": iterations})
                return result

            if response.stop_reason == "tool_use":
                tool_results = []
                for block in response.content:
                    if block.type == "tool_use":
                        logger.info(
                            "tool_call",
                            extra={"tool": block.name, "input": block.input},
                        )
                        result = await self._handler(block.name, block.input)
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result,
                        })
                messages.append({"role": "user", "content": tool_results})

            else:
                raise RuntimeError(
                    f"Unexpected stop_reason after {iterations} iterations: "
                    f"{response.stop_reason}"
                )

        raise RuntimeError(
            f"Agent exceeded max_iterations ({self._config.max_iterations})"
        )
```

This ~80-line class is the core of a production agent. It handles the loop, the iteration limit, and structured logging. Everything else (tools, prompts, the actual business logic) is injected as dependencies. This is far more maintainable than importing LangChain.

**Common mistake:** Not implementing an iteration limit. An agent in a bad state can loop forever, burning tokens and money. Always set `max_iterations` and treat hitting it as an error that gets alerted, not silently swallowed.

---

## 7. Tool Design Principles

Good tools make the difference between an agent that works and one that hallucinates its way to wrong answers. Here are the principles, with integration platform-grounded examples.

### Principle 1: One Tool, One Purpose

A tool named `manage_salesforce` that can create, read, update, and delete is a bad tool. The model can't easily reason about when to use it. Split into `query_accounts`, `create_account`, `update_account`.

### Principle 2: Safe by Default, Destructive by Explicit Request

```python
# BAD: Tool that overwrites without confirmation
{
    "name": "sync_account",
    "description": "Sync a Salesforce account to the EHR system.",
    ...
}

# GOOD: Separate read-only and write tools
{
    "name": "preview_account_sync",
    "description": "Show what changes would be made if this account were synced to the EHR system. Does NOT make any changes.",
    ...
},
{
    "name": "execute_account_sync",
    "description": "Execute an EHR sync for this account. THIS MAKES REAL API CALLS to the EHR system. Only call this after previewing and confirming with the user.",
    ...
}
```

### Principle 3: Return Structured Data, Not Prose

Tool results should be JSON, not natural language summaries. The model is better at reasoning over structured data.

```python
# BAD tool result
return "Found 3 accounts: Acme, Beta, Gamma. Two are synced, one has errors."

# GOOD tool result
return json.dumps({
    "total": 3,
    "accounts": [
        {"id": "001...", "name": "Acme Corp", "sync_status": "synced"},
        {"id": "002...", "name": "Beta LLC", "sync_status": "synced"},
        {"id": "003...", "name": "Gamma Inc", "sync_status": "error",
         "error": "EHR client ID mismatch"},
    ]
})
```

### Principle 4: Include Context in Error Messages

```python
# BAD
return json.dumps({"error": "Not found"})

# GOOD
return json.dumps({
    "error": "Account not found",
    "account_id": account_id,
    "suggestion": (
        "Check that the ID is 18 characters and starts with '001'. "
        "Use query_accounts to find accounts by name if needed."
    )
})
```

### Principle 5: Idempotency for Write Tools

Any tool that modifies state should be idempotent. If the agent calls it twice with the same arguments, the second call should succeed without duplicating data.

```python
# In your FastAPI middleware endpoint for sync:
@router.post("/api/v2/accounts/{account_id}/sync")
async def trigger_sync(account_id: str, body: SyncRequest, db: AsyncSession = Depends(get_db)):
    # Check for existing pending job — don't create duplicate
    existing = await db.scalar(
        select(SyncJob)
        .where(SyncJob.account_id == account_id)
        .where(SyncJob.status == "pending")
    )
    if existing:
        return {"job_id": existing.id, "status": "already_queued"}

    job = SyncJob(account_id=account_id, force=body.force)
    db.add(job)
    await db.commit()
    return {"job_id": job.id, "status": "queued"}
```

**Common mistake:** Tools that return different data shapes depending on success vs. error. The model learns your tool's schema from examples in the conversation. If the schema changes on error, the model may misinterpret error responses as success.

---

## 8. Prompt Engineering for Agents

System prompts for agents are different from system prompts for chatbots. They need to:

1. **Define the agent's identity and scope** — what it is, what it can and cannot do.
2. **Establish safety constraints** — what it must never do (production writes without confirmation, etc.).
3. **Set the reasoning style** — explicit instructions to think before acting.
4. **Describe the tools** — even though tools have their own descriptions, the system prompt can explain the relationship between tools.

```python
PLATFORM_AGENT_SYSTEM_PROMPT = """
You are an Integration Platform Operations Agent — an AI assistant with access to the crm-middleware API.
You help engineers investigate and resolve sync issues between Salesforce and the EHR system.

CAPABILITIES:
- Query Salesforce accounts, contacts, and opportunities via the middleware
- Check sync queue status and job history
- Preview what changes a sync would make (read-only)
- Trigger syncs (write operation — requires explicit confirmation)

CONSTRAINTS:
- Always use preview_account_sync before execute_account_sync
- Never trigger syncs for more than 10 accounts in a single run without explicit user approval
- If a tool returns an error, explain it to the user in plain language before retrying
- Do not guess at account IDs — always query to find them

REASONING APPROACH:
- Before taking any action, think through what information you already have and what you need
- When a task is ambiguous, ask one clarifying question rather than proceeding with assumptions
- After completing a task, summarize what you did and the outcome

ENVIRONMENT: Production middleware at http://localhost:8000. Real API calls affect real data.
"""
```

---

## 9. Key Concepts Summary

```
AI AGENTS, TOOL USE & MCP
│
├── AGENT FUNDAMENTALS
│   ├── Agent = LLM + action-observation loop
│   ├── Not an agent: single call, chatbot, hard-coded script
│   └── Iteration limit is mandatory
│
├── REASONING PATTERNS
│   ├── ReAct: interleave thought + action (most common)
│   ├── Plan-and-Execute: plan first, execute in parallel
│   └── Reflection: self-critique loop (expensive, use sparingly)
│
├── TOOL / FUNCTION CALLING
│   ├── Tool = JSON Schema definition + your execution code
│   ├── Description is the most important field
│   ├── Handle parallel tool calls (multiple tool_use blocks)
│   ├── Always append assistant message before tool results
│   └── Design: one purpose, safe defaults, structured output, idempotent
│
├── MODEL CONTEXT PROTOCOL (MCP)
│   ├── Standard protocol: host ↔ server via JSON-RPC 2.0
│   ├── Capabilities: tools, resources, prompts
│   ├── Transports: stdio (local) or SSE (remote)
│   ├── Solves N×M connector problem
│   └── Error: always return TextContent, never raise exceptions
│
├── MULTI-AGENT SYSTEMS
│   ├── When to use: context overflow, parallelism, specialization
│   ├── When NOT to use: simple tasks, debugging complexity, cost
│   └── Each agent gets its own system prompt and tool set
│
└── ORCHESTRATION
    ├── LangChain/LangGraph: good for complex graphs, bad for control
    ├── Raw SDK: best for production, full control, debuggable
    └── Custom thin wrapper: best of both worlds
```

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** What is the minimum requirement for something to be called an agent rather than a chatbot?

**2.** In the Anthropic SDK, what is the `stop_reason` value that indicates the model wants to call a tool?

**3.** You have an agent that calls `trigger_account_sync` but sometimes calls it twice for the same account. What two mitigations should you apply?

**4.** Why is the `description` field in a tool definition more important than the `input_schema`?

**5.** In the Anthropic messages API, what is the correct message sequence after the model returns a `tool_use` response?

**6.** What does MCP stand for, and what problem does it solve?

**7.** You are writing an MCP server. A tool call to your middleware raises an `httpx.HTTPStatusError`. What should your `call_tool` handler do?

**8.** What are the three capability types in MCP?

**9.** Describe the difference between ReAct and plan-and-execute. When would you choose plan-and-execute for a reconciliation job on the integration platform?

**10.** What is the "reflection" pattern in agents, and what is its main cost?

**11.** Claude returns two `tool_use` blocks in a single response. How many `tool_result` entries should you send back, and in how many user messages?

**12.** You are building an agent that should never trigger a sync without user confirmation. What is the most robust way to enforce this?

**13.** What are the two MCP transport types, and which is appropriate for a local development tool?

**14.** Name three signs that a task should NOT be solved with an agent.

**15.** You want to run reconciliation across 200 accounts as fast as possible. Which agentic pattern would you use, and why?

**16.** What is the risk of not setting a `max_iterations` limit in an agent loop?

**17.** In a multi-agent system with a planner and multiple worker agents, how would you choose models for each role to control cost?

**18.** What makes a write tool "safe by default" in the context of agent design?

**19.** Your agent loop receives `stop_reason = "max_tokens"`. What does this mean and what should your code do?

**20.** You are adding an integration platform sync tool to Claude Code via MCP. List the three things you must define: the config entry location, the required fields in that entry, and how Claude Code picks up the change.

---

### Answers

??? note "Reveal Answers"

    **1.** The minimum requirement is an *action-observation loop*: the model must be able to take an external action (call a tool, write to memory, make an API call), receive the result of that action as an observation, and continue reasoning based on it. A chatbot with conversation history does not qualify because it takes no external actions — it only generates text.

    **2.** The `stop_reason` value is `"tool_use"`. This indicates the model has decided to invoke one or more tools and has returned `tool_use` blocks in its `content` array containing the tool name, a unique `id`, and the `input` arguments as a parsed dictionary.

    **3.** First, make the tool idempotent at the server side: before creating a new sync job, check whether a pending job for that account already exists and return the existing job's ID if so. Second, in the agent's system prompt, instruct the model to call `preview_account_sync` before `trigger_account_sync` and to check its prior actions before retrying. The server-side guard is the authoritative fix; the prompt instruction is a second layer of defense.

    **4.** The model decides *when* to call a tool and *what arguments to pass* almost entirely based on the description. The `input_schema` only validates the structure of the arguments after the model has already decided to call the tool. A vague description leads to the model calling the tool at the wrong time, with the wrong intent, or instead of a more appropriate tool. Think of the description as the function's documentation contract — the model reads it to understand purpose, not just signature.

    **5.** After receiving a `tool_use` response, you must: (1) append the full assistant response (including the `tool_use` blocks) to the messages list as a `role: assistant` message, (2) execute all tool calls in that response, (3) append a single `role: user` message whose `content` is a list of `tool_result` objects — one per tool call, each referencing the tool's `id`. The roles must strictly alternate: `user → assistant → user → assistant → ...`. Skipping the assistant message before tool results will cause an API validation error.

    **6.** MCP stands for Model Context Protocol. It solves the N×M connector problem: without a standard, every combination of agent framework (N) and tool/data source (M) requires a custom integration. MCP defines a single protocol (JSON-RPC 2.0 over stdio or SSE) so that a tool written once as an MCP server works with any MCP-compatible host — Claude Code, Claude Desktop, or your own agent — without modification.

    **7.** The handler should catch the exception and return a `TextContent` with a JSON error payload containing the HTTP status code and the response body. You should never let an uncaught exception propagate out of `call_tool` — the MCP host will show a generic error that gives the model no useful information to reason about. Returning a structured error lets the model understand what went wrong and potentially try a different approach or report the error to the user with context.

    **8.** The three MCP capability types are: **Tools** (callable functions that can take actions and return results), **Resources** (read-only data sources the model can read, like documents or API schemas), and **Prompts** (reusable prompt templates that can be parameterized). Most MCP servers expose at least tools; resources and prompts are optional but powerful for giving the model ambient context.

    **9.** ReAct decides the next step only after observing the result of the previous step — it is reactive and sequential. Plan-and-execute first generates a complete plan (a list of steps), then executes them, potentially in parallel. For a reconciliation job covering hundreds of accounts on the integration platform, plan-and-execute is better because: the planner can divide accounts into independent batches, each batch can be executed concurrently with `asyncio.gather`, and if one batch fails, the others are unaffected. ReAct would process accounts sequentially and the failure of one step would interrupt all subsequent ones.

    **10.** The reflection pattern adds a self-critique step after each execution: a second LLM call evaluates whether the output is correct and complete before the agent proceeds or returns the result. Its main cost is that it doubles or triples the number of LLM API calls (and therefore cost and latency) for every task. It is most valuable when correctness is verifiable and errors are expensive — for example, verifying that generated field names exist in your schema before making API calls.

    **11.** You should send back two `tool_result` entries (one per tool call), both in a single user message whose `content` is a list. Do not send them as two separate user messages — the API expects one user message containing all results from a single assistant turn. Each `tool_result` must reference the original `tool_use` block's `id` so the model can match results to calls.

    **12.** The most robust approach is to separate the read-only preview tool from the write execution tool at the tool level — not just in the system prompt. Name the write tool `execute_account_sync` with a description that explicitly says "THIS MAKES REAL CHANGES" and instructs the model to confirm with the user first. Also add a guard in the system prompt. The system prompt can be overridden by adversarial inputs; having two separate tools (one safe, one explicit) creates a structural barrier that is much harder to accidentally or maliciously bypass.

    **13.** The two MCP transport types are **stdio** (the host spawns the server as a subprocess and communicates via stdin/stdout) and **SSE** (Server-Sent Events, where the server is a remote HTTP service). For local development tools — like a crm-middleware client running on your dev machine — `stdio` is appropriate because it requires no network configuration, runs with local credentials, and is automatically managed by the host process lifecycle.

    **14.** Three signs a task should not be solved with an agent: (1) The task can be solved with a single well-prompted LLM call — adding a loop adds latency, cost, and failure modes for no benefit. (2) The steps are fully predetermined and the same every time — this is just a workflow; write it in Python with deterministic logic. (3) The user needs a guarantee of consistency or atomicity that a non-deterministic LLM loop cannot provide — for example, a financial transaction that must be exactly right, not "probably right."

    **15.** Plan-and-execute with parallel batch workers. The planner agent divides 200 accounts into batches of ~20, then `asyncio.gather` dispatches all worker agents concurrently. Each worker independently checks its batch against the EHR system and returns a result. A reviewer agent synthesizes the results. This is far faster than a single ReAct agent processing accounts one at a time, and cheaper because batch workers can use smaller, faster models (Haiku vs. Opus).

    **16.** Without a `max_iterations` limit, an agent in a bad state — for example, one receiving error responses from a tool and retrying indefinitely — will loop forever. Each iteration costs tokens. A production agent without a guard can run up a large API bill before anyone notices. Beyond cost, an unbounded agent can hold a database connection or file handle open indefinitely, causing resource exhaustion. Always set `max_iterations` (typically 10-20) and treat hitting it as a hard error that triggers an alert.

    **17.** Use a cheap, fast model (e.g., claude-haiku-3-5) for worker agents that do repetitive structured tasks — querying, filtering, classifying. Use a mid-tier model (claude-sonnet-4-5) for planning because planning requires some reasoning but not maximum capability. Reserve the most capable model (claude-opus-4-5) for the reviewer or any step where accuracy is critical and errors are expensive. This tiered approach can reduce cost by 5-10× compared to using the most capable model for every agent in the system.

    **18.** A write tool is "safe by default" when its default behavior is the least destructive option available. This means: (1) it requires explicit confirmation arguments (like `confirmed=True`) before executing, (2) it has a corresponding read-only `preview_` variant that shows what *would* happen, (3) its description explicitly warns that it makes real changes, and (4) at the server side it is idempotent — calling it twice with the same arguments does not duplicate state. The goal is that an agent calling the tool by accident or with incomplete reasoning causes no harm.

    **19.** `stop_reason = "max_tokens"` means the model hit the `max_tokens` limit you specified in the API call before it could finish generating a response. The response is truncated mid-generation. Your code should treat this as an error: the tool calls or text in the response may be incomplete and should not be used. The correct fix is to increase `max_tokens` for tasks that require longer responses, or to decompose the task so each step produces a shorter response. Do not silently pass the truncated response as context — a truncated `tool_use` block is malformed JSON and will cause downstream errors.

    **20.** (1) **Config location:** add to `.claude/settings.json` in your project directory (project-scoped) or `~/.claude/settings.json` (user-scoped, available in all projects). (2) **Required fields** in the `mcpServers` entry: `"command"` (the executable to run, e.g., `"python"`) and `"args"` (a list of arguments, e.g., the path to your server script). Optional but common: `"env"` for environment variables. (3) **Pickup:** Claude Code reads MCP server configuration on startup and when you reload the window. After editing settings.json, restart Claude Code (or use `/mcp` to reload) and the new server will appear in the MCP server list and its tools will be available.
