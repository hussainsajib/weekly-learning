# Week 5 — Claude API & Building AI Features

**Week of:** July 6, 2026
**Estimated study time:** ~2 hours
**Tags:** `ai` `llm` `claude` `api`

---

## Overview

By 2026, integrating LLM capabilities into backend services is no longer experimental — it is a standard engineering discipline. Senior engineers are expected not just to wire up an API call, but to reason about cost, latency, caching strategies, error budgets, and architectural boundaries. The gap between a junior engineer who can make an API call and a staff engineer who can design an AI-powered service that is reliable, cost-predictable, and production-worthy is wide.

This week focuses on the Anthropic SDK and the Claude API — the most capable LLM API available to engineers as of mid-2026. Your existing stack is a natural fit: FastAPI for AI-powered HTTP endpoints, SQLAlchemy for caching usage metadata, and GKE for hosting inference-adjacent services. Understanding the internals — why prompt caching works as a prefix match, why streaming is mandatory above certain token thresholds, why tool use requires a loop — is what separates an engineer who integrates AI from one who architects around it.

In the AESF context, the most immediately applicable patterns are: using Claude to enrich Salesforce records or classify data from Epic EHR, building a FastAPI middleware layer that proxies Claude calls with proper rate-limit handling and cost tracking, and using prompt caching to reduce costs when the same large document (e.g., a policy schema or EHR context blob) is referenced across many requests. The ETL side is equally relevant — Claude can serve as a document-understanding or data-extraction layer sitting between Pentaho outputs and downstream Salesforce writes.

After studying this week, you should be able to: build a production-grade FastAPI service that wraps the Claude API with streaming, caching, and error handling; reason about token costs and implement prompt caching correctly; use tool use to build agentic workflows; and select the right model tier for a given workload. You will also understand the sharp edges — the silent cache invalidators, the rate limit headers that matter, the token counting mistakes that cause cost surprises.

---

## 1. The Anthropic SDK: Architecture and Client Initialization

### Why It Matters

The Anthropic Python SDK (`anthropic`) is a Stainless-generated client — the same OpenAPI spec generates all 7 official SDKs. This matters because field names on the wire are canonical JSON (snake_case in Python, camelCase in TypeScript), and reasoning about what the SDK does — retries, timeouts, streaming — is far easier once you understand its internals rather than treating it as a black box.

Everything goes through `POST /v1/messages`. Tools, streaming, prompt caching, vision, structured outputs — these are all features of this single endpoint, not separate APIs. The client wraps this with retry logic, type-safe request/response models, and streaming helpers.

### Installation and Setup

```python
pip install anthropic
```

```python
import anthropic

# Production pattern: resolve from environment (ANTHROPIC_API_KEY)
client = anthropic.Anthropic()

# Async client for FastAPI endpoints
async_client = anthropic.AsyncAnthropic()
```

### Client Configuration Internals

```python
import httpx
from anthropic import Anthropic, AsyncAnthropic, DefaultHttpxClient

# Default timeout: 10 minutes (600s) — units are SECONDS in Python
# The SDK refuses non-streaming requests with large max_tokens to prevent timeout issues
client = Anthropic(
    timeout=httpx.Timeout(60.0, read=30.0, write=10.0, connect=2.0),
    max_retries=3,  # default is 2; retries 408/429/5xx with exponential backoff
)

# Per-request override without mutating the client
response = client.with_options(timeout=5.0).messages.create(
    model="claude-haiku-4-5",
    max_tokens=256,
    messages=[{"role": "user", "content": "Classify this text: positive, negative, or neutral?"}]
)
```

**AESF Application:** In `aesf-py-middleware`, the `client = anthropic.Anthropic()` initialization belongs in your `app/core/config.py` or as a FastAPI dependency — not in request handlers. Create it once at startup, reuse it across requests. The SDK handles connection pooling internally via httpx.

### Common Mistake: Recreating the Client Per Request

```python
# WRONG — creates a new connection pool on every request
@app.post("/classify")
async def classify(text: str):
    client = anthropic.Anthropic()  # Don't do this
    return await client.messages.create(...)

# RIGHT — singleton client as a FastAPI dependency
from functools import lru_cache

@lru_cache
def get_anthropic_client() -> anthropic.AsyncAnthropic:
    return anthropic.AsyncAnthropic()

@app.post("/classify")
async def classify(text: str, client: anthropic.AsyncAnthropic = Depends(get_anthropic_client)):
    ...
```

---

## 2. The Messages API: Core Concepts and Model Selection

### The Single Endpoint Mental Model

```
POST /v1/messages
├── model          → which Claude to use
├── max_tokens     → hard output cap (model is NOT aware of this)
├── messages       → conversation history [{"role": "user"|"assistant", "content": ...}]
├── system         → operator-level instructions
├── tools          → tool definitions (user-defined or server-side)
├── output_config  → effort level, structured output format
├── thinking       → adaptive thinking on/off
└── cache_control  → on content blocks, for prompt caching
```

### Current Model Landscape (July 2026)

| Model | ID | Input $/1M | Output $/1M | Context | Best For |
|---|---|---|---|---|---|
| Claude Opus 4.8 | `claude-opus-4-8` | $5.00 | $25.00 | 1M | Default; long-horizon agentic work |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | $3.00 | $15.00 | 1M | High-volume production; balanced |
| Claude Haiku 4.5 | `claude-haiku-4-5` | $1.00 | $5.00 | 200K | Simple classification, fast tasks |
| Claude Fable 5 | `claude-fable-5` | $10.00 | $50.00 | 1M | Most demanding tasks only |

**Key rule: Default to `claude-opus-4-8` unless you have a specific reason to downgrade.** Downgrading for cost is an engineering decision that requires measurement, not assumption.

### Model Selection Decision Tree

```
Is it a simple classification / short extraction?
├── Yes → claude-haiku-4-5 ($1/$5 per MTok)
│
Is it high-volume production with balanced quality needs?
├── Yes → claude-sonnet-4-6 ($3/$15 per MTok)
│
Is it complex reasoning / agentic / long-horizon?
├── Yes → claude-opus-4-8 ($5/$25 per MTok) [DEFAULT]
│
Is it the absolute hardest problem you have?
└── Yes → claude-fable-5 ($10/$50 per MTok) [verify need]
```

### Basic Request Pattern

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system="You are a data analyst specializing in insurance policy data.",
    messages=[
        {"role": "user", "content": "Summarize the key risk factors in this policy renewal."}
    ]
)

# response.content is a list of typed blocks
for block in response.content:
    if block.type == "text":
        print(block.text)

# Always check stop_reason before trusting content
print(response.stop_reason)  # "end_turn" | "max_tokens" | "tool_use" | "refusal"
```

### Common Mistake: Assuming `response.content[0].text` Always Works

```python
# DANGEROUS — will crash on refusal (empty content array) or tool_use blocks
text = response.content[0].text

# SAFE — check stop_reason first, then iterate typed blocks
if response.stop_reason == "refusal":
    handle_refusal()
else:
    text = next((b.text for b in response.content if b.type == "text"), "")
```

---

## 3. Streaming: When and How

### Why Streaming Is Not Optional for Large Outputs

The Anthropic Python SDK will raise a `ValueError` if you request more than ~16,000 tokens in a non-streaming request. The reason is architectural: large non-streaming requests can exceed HTTP read timeouts since the entire response must be buffered before delivery. Above 16K `max_tokens`, always use streaming.

```python
# For interactive endpoints: stream text tokens as they arrive
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a detailed analysis of this Epic EHR data..."}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# When you need the full response after streaming (most API use cases)
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": prompt}]
) as stream:
    final_message = stream.get_final_message()
    # Full Message object with usage, stop_reason, content — same as non-streaming
```

### FastAPI Streaming Endpoint Pattern

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import anthropic

app = FastAPI()
client = anthropic.AsyncAnthropic()

@app.post("/analyze/stream")
async def analyze_stream(prompt: str):
    """Stream a Claude analysis — useful for long-running policy document analysis."""
    async def generate():
        async with client.messages.stream(
            model="claude-opus-4-8",
            max_tokens=16000,
            messages=[{"role": "user", "content": prompt}]
        ) as stream:
            async for text in stream.text_stream:
                yield f"data: {text}\n\n"
            # Send usage stats at end for cost tracking
            final = await stream.get_final_message()
            yield f"data: [DONE] tokens={final.usage.output_tokens}\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Common Mistake: Forgetting to Flush or Ignoring Partial Failures

When a stream is interrupted (network drop, timeout), you may have incomplete content. Always design your stream consumer to handle partial output gracefully — don't treat a mid-stream disconnect as equivalent to a complete response.

---

## 4. Tool Use: Agents and Agentic Loops

### The Mental Model

Tool use is Claude's mechanism for taking actions or requesting data. You define tools (name, description, JSON schema), Claude decides when to call them, and your code executes the call and feeds results back. This creates a loop that runs until Claude stops calling tools.

The critical insight: **Claude does not execute tools**. It emits structured requests (`tool_use` blocks) describing what it wants. Your code is the executor.

### Manual Agentic Loop (Full Control)

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_salesforce_account",
        "description": "Retrieve an Account record from Salesforce by Account ID. Call this when you need current account data before making recommendations.",
        "input_schema": {
            "type": "object",
            "properties": {
                "account_id": {
                    "type": "string",
                    "description": "The Salesforce Account ID (18-char AESF namespace ID)"
                }
            },
            "required": ["account_id"]
        }
    },
    {
        "name": "get_epic_client",
        "description": "Fetch client data from Epic EHR by client code. Use when you need EHR data to cross-reference with Salesforce.",
        "input_schema": {
            "type": "object",
            "properties": {
                "client_code": {"type": "string"},
                "include_policies": {"type": "boolean", "description": "Whether to include policy lines"}
            },
            "required": ["client_code"]
        }
    }
]

def execute_tool(name: str, tool_input: dict) -> str:
    """Your actual tool implementation — calls middleware API, Salesforce, etc."""
    if name == "get_salesforce_account":
        # Call aesf-py-middleware or directly via simple-salesforce
        return json.dumps({"Name": "Acme Corp", "Industry": "Technology"})
    elif name == "get_epic_client":
        return json.dumps({"client_code": tool_input["client_code"], "status": "Active"})
    return "Tool not found"

def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=16000,
            tools=tools,
            messages=messages
        )

        # Done — no more tool calls
        if response.stop_reason == "end_turn":
            return next((b.text for b in response.content if b.type == "text"), "")

        # Handle pause_turn: server-side tool hit iteration limit, re-send to continue
        if response.stop_reason == "pause_turn":
            messages.append({"role": "assistant", "content": response.content})
            continue

        # Process tool calls
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,  # Must match the tool_use block's id
                    "content": result
                })

        # Append assistant response AND tool results before next iteration
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

result = run_agent("What is the current status of Salesforce account 0013x00001ABC123 and does their Epic client data match?")
print(result)
```

### Tool Runner (SDK-Managed Loop)

For simpler cases where you don't need approval gates or custom loop logic:

```python
from anthropic import Anthropic, beta_tool

client = Anthropic()

@beta_tool
def search_policy_database(policy_number: str, include_lines: bool = False) -> str:
    """Search the AESF policy database for a policy record.

    Args:
        policy_number: The AESF__Policy__c policy number from Salesforce.
        include_lines: Whether to include associated AESF__Line__c records.
    """
    # Your actual implementation
    return f"Policy {policy_number}: Active, renewal due 2027-01-15"

# Tool runner handles the loop automatically
runner = client.beta.messages.tool_runner(
    model="claude-opus-4-8",
    max_tokens=16000,
    tools=[search_policy_database],
    messages=[{"role": "user", "content": "Find policy POL-2024-001 and summarize its status."}]
)

for message in runner:
    pass  # runner handles the loop

# Get final text
final = runner.get_final_message()
print(next(b.text for b in final.content if b.type == "text"))
```

### Common Mistake: Forgetting to Append `response.content` Before Tool Results

```python
# WRONG — loses tool_use blocks from assistant turn, API will 400
messages.append({"role": "user", "content": tool_results})

# RIGHT — assistant's full content (including tool_use blocks) must be in history
messages.append({"role": "assistant", "content": response.content})  # First
messages.append({"role": "user", "content": tool_results})           # Then results
```

---

## 5. Vision: Image Input to Claude

### Supported Input Methods

Claude supports two ways to provide images: base64 encoding (works anywhere) and URL references (image must be publicly accessible). In the AESF context, base64 is almost always the right choice since you're working with internal documents and EHR data.

```python
import anthropic
import base64

client = anthropic.Anthropic()

# Base64: for internal images, PDFs rendered to image, screenshots
def analyze_document_image(image_path: str, question: str) -> str:
    with open(image_path, "rb") as f:
        image_data = base64.standard_b64encode(f.read()).decode("utf-8")

    # Determine media type
    ext = image_path.rsplit(".", 1)[-1].lower()
    media_type_map = {"jpg": "image/jpeg", "jpeg": "image/jpeg", "png": "image/png", "gif": "image/gif", "webp": "image/webp"}
    media_type = media_type_map.get(ext, "image/png")

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_data
                    }
                },
                {"type": "text", "text": question}
            ]
        }]
    )
    return next(b.text for b in response.content if b.type == "text")

# Example: analyze a scanned insurance document
result = analyze_document_image(
    "policy_scan.png",
    "Extract all coverage amounts, deductibles, and policy dates from this insurance document."
)
```

### Multi-Image Request (e.g., Before/After Comparison)

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "Compare these two Epic EHR screenshots and identify what changed:"},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": before_b64}},
            {"type": "text", "text": "BEFORE"},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": after_b64}},
            {"type": "text", "text": "AFTER"}
        ]
    }]
)
```

### Common Mistake: Using URL Source for Internal/Private Images

```python
# WRONG — Claude cannot access your internal network or auth-gated URLs
{"type": "image", "source": {"type": "url", "url": "https://internal.aesf.com/doc.png"}}

# RIGHT — base64 for anything that isn't a public URL
{"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": b64_data}}
```

---

## 6. Prompt Caching: The Prefix Match Invariant

### The One Rule Everything Follows From

**Prompt caching is a prefix match. Any change anywhere in the prefix invalidates everything after it.**

The render order is: `tools` → `system` → `messages`. A `cache_control` marker on the last system block caches the tools + system prefix together.

Cache pricing:
- Cache write: **1.25× normal input price** (5-minute TTL) or **2×** (1-hour TTL)
- Cache read: **~0.1× normal input price** (90% savings)
- Break-even at 5-min TTL: 2 requests. At 1-hour TTL: 3 requests.

### Correct Caching Pattern

```python
import anthropic

client = anthropic.Anthropic()

# Large stable system prompt — this is what we want to cache
LARGE_SYSTEM_PROMPT = """
You are an expert insurance data analyst working with the AESF (Applied Epic for Salesforce) platform.
You understand the data model including AESF__Policy__c, AESF__Line__c, AESF__Plan__c objects.
Key rules:
- Policy types map to Epic policy_type codes via AESF__Policy_Type__c lookup
- All monetary values are in USD unless noted
- Epic Structure Combinations determine servicing hierarchies
[... 5000+ more tokens of context ...]
"""

def analyze_with_cache(user_question: str) -> dict:
    """Analyze with cached system prompt — ~90% cost reduction after first call."""
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=4096,
        system=[{
            "type": "text",
            "text": LARGE_SYSTEM_PROMPT,
            "cache_control": {"type": "ephemeral"}  # Cache this prefix
        }],
        messages=[{"role": "user", "content": user_question}]
    )

    # Log cache stats to track effectiveness
    usage = response.usage
    return {
        "text": next(b.text for b in response.content if b.type == "text"),
        "cache_created_tokens": usage.cache_creation_input_tokens,
        "cache_read_tokens": usage.cache_read_input_tokens,
        "uncached_tokens": usage.input_tokens,
    }

# First call: pays cache-write cost (1.25×)
r1 = analyze_with_cache("What is the renewal date for policy P-2024-001?")
print(f"Cache created: {r1['cache_created_tokens']} tokens")

# Second call: pays cache-read cost (0.1×)
r2 = analyze_with_cache("List all active lines for account ACC-12345")
print(f"Cache read: {r2['cache_read_tokens']} tokens")  # Should be > 0
```

### Silent Cache Invalidators — The Audit Checklist

If `cache_read_input_tokens` is zero on repeated requests, check for these:

| Root Cause | Example | Fix |
|---|---|---|
| Timestamp in system prompt | `f"Current date: {datetime.now()}"` | Move to messages, not system |
| Non-deterministic serialization | `json.dumps(my_dict)` (unordered) | Use `json.dumps(d, sort_keys=True)` |
| Per-request UUID/ID in prefix | `f"Request ID: {uuid4()}"` | Move IDs to messages or response headers |
| Varying tool set | `tools = get_tools_for_user(user_id)` | Use a stable full tool set; disable via prompting |
| Different model per request | Dynamic model selection in system prompt | Cache is per-model; switching invalidates |

### Caching Conversation History (Multi-Turn)

```python
def multi_turn_with_cache(history: list, new_message: str) -> str:
    """Cache grows incrementally — each turn adds to the cached prefix."""
    # Add cache_control to the last content block in the last turn
    messages_with_cache = list(history)

    # Mark end of history as cacheable
    if messages_with_cache:
        last_msg = messages_with_cache[-1].copy()
        if isinstance(last_msg["content"], list) and last_msg["content"]:
            last_content = dict(last_msg["content"][-1])
            last_content["cache_control"] = {"type": "ephemeral"}
            last_msg["content"] = list(last_msg["content"][:-1]) + [last_content]
            messages_with_cache[-1] = last_msg

    messages_with_cache.append({"role": "user", "content": new_message})

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=4096,
        messages=messages_with_cache
    )
    return next(b.text for b in response.content if b.type == "text")
```

### Common Mistake: Putting Volatile Content Early in the Prefix

```python
# WRONG — timestamp at the top of system prompt invalidates EVERYTHING on every request
system_prompt = f"""
Session started: {datetime.now().isoformat()}  # <-- This breaks all caching
You are an expert analyst...
[5000 tokens of context]
"""

# RIGHT — stable content first, volatile content in messages
system_prompt = """
You are an expert analyst...
[5000 tokens of context]
"""
messages = [
    {"role": "user", "content": f"[Session: {datetime.now().isoformat()}] Analyze this..."}
]
```

---

## 7. Token Counting and Cost Management

### Why Token Counting Matters (And Why Not to Use tiktoken)

`tiktoken` is OpenAI's tokenizer. It undercounts Claude tokens by 15-20% on typical text and by much more on code and non-English input. For cost projections, `count_tokens` against the actual model is the only accurate method.

```python
import anthropic

client = anthropic.Anthropic()

# Count tokens BEFORE sending — for cost estimation
def estimate_cost(messages: list, system: str = None, model: str = "claude-opus-4-8") -> dict:
    """Estimate the cost of a request before sending it."""
    kwargs = {
        "model": model,
        "messages": messages,
    }
    if system:
        kwargs["system"] = system

    count = client.messages.count_tokens(**kwargs)

    # Per-model pricing ($/1M tokens)
    pricing = {
        "claude-opus-4-8": {"input": 5.00, "output": 25.00},
        "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
        "claude-haiku-4-5": {"input": 1.00, "output": 5.00},
    }
    p = pricing.get(model, pricing["claude-opus-4-8"])
    input_cost = count.input_tokens * p["input"] / 1_000_000

    return {
        "input_tokens": count.input_tokens,
        "estimated_input_cost_usd": round(input_cost, 6),
        "model": model,
    }

# Example: Check cost before a large document analysis
result = estimate_cost(
    messages=[{"role": "user", "content": open("large_policy_doc.txt").read()}],
    system="You are an insurance data analyst.",
    model="claude-opus-4-8"
)
print(f"Input: {result['input_tokens']:,} tokens, estimated ${result['estimated_input_cost_usd']:.4f}")
```

### Cost Tracking in Production (FastAPI + SQLAlchemy)

```python
from sqlalchemy import Column, Integer, String, Float, DateTime, Text
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime
import anthropic

Base = declarative_base()

class AIUsageLog(Base):
    """Track Claude API usage for cost attribution and anomaly detection."""
    __tablename__ = "ai_usage_logs"

    id = Column(Integer, primary_key=True)
    request_id = Column(String(64), unique=True, index=True)
    model = Column(String(64))
    input_tokens = Column(Integer)
    output_tokens = Column(Integer)
    cache_creation_tokens = Column(Integer, default=0)
    cache_read_tokens = Column(Integer, default=0)
    stop_reason = Column(String(32))
    endpoint = Column(String(128))  # which API endpoint triggered this
    input_cost_usd = Column(Float)
    output_cost_usd = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)

def log_usage(session, response: anthropic.types.Message, endpoint: str):
    """Log API usage after each call for cost tracking."""
    usage = response.usage
    pricing = {
        "claude-opus-4-8": {"input": 5.00, "output": 25.00},
        "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
        "claude-haiku-4-5": {"input": 1.00, "output": 5.00},
    }
    p = pricing.get(response.model, {"input": 5.00, "output": 25.00})

    # Cache reads cost 0.1x, writes cost 1.25x
    effective_input_tokens = (
        usage.input_tokens +
        usage.cache_read_input_tokens * 0.1 +
        usage.cache_creation_input_tokens * 1.25
    )

    record = AIUsageLog(
        request_id=response._request_id,
        model=response.model,
        input_tokens=usage.input_tokens,
        output_tokens=usage.output_tokens,
        cache_creation_tokens=usage.cache_creation_input_tokens,
        cache_read_tokens=usage.cache_read_input_tokens,
        stop_reason=response.stop_reason,
        endpoint=endpoint,
        input_cost_usd=effective_input_tokens * p["input"] / 1_000_000,
        output_cost_usd=usage.output_tokens * p["output"] / 1_000_000,
    )
    session.add(record)
    session.commit()
    return record
```

### Common Mistake: Not Accounting for Cache Costs

```python
# WRONG — undercounts cost when caching is active
total_cost = (usage.input_tokens * input_price + usage.output_tokens * output_price) / 1_000_000

# RIGHT — cache writes cost 1.25x, cache reads cost 0.1x
total_cost = (
    usage.input_tokens * input_price +
    usage.cache_creation_input_tokens * input_price * 1.25 +
    usage.cache_read_input_tokens * input_price * 0.1 +
    usage.output_tokens * output_price
) / 1_000_000
```

---

## 8. Error Handling and Rate Limit Strategies

### Error Taxonomy

The Anthropic SDK raises typed exceptions — always catch specific types, not the base class, so you can distinguish retryable from non-retryable errors.

```python
import anthropic
import time
import random
from typing import Optional

client = anthropic.Anthropic(max_retries=0)  # Disable SDK auto-retry to control it manually

def call_with_backoff(
    max_attempts: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    **create_kwargs
) -> anthropic.types.Message:
    """Call Claude with exponential backoff for rate limits and server errors."""
    last_exc = None

    for attempt in range(max_attempts):
        try:
            return client.messages.create(**create_kwargs)

        except anthropic.AuthenticationError:
            raise  # Never retry auth errors — fix the key

        except anthropic.BadRequestError as e:
            # 400: invalid request — check for common causes
            if "budget_tokens" in str(e):
                raise ValueError("budget_tokens is removed on Opus 4.7+. Use thinking={type: adaptive}") from e
            if "temperature" in str(e):
                raise ValueError("temperature/top_p/top_k removed on Opus 4.7+") from e
            raise  # Other 400s are not retryable

        except anthropic.NotFoundError:
            raise  # Bad model ID — not retryable

        except anthropic.RateLimitError as e:
            last_exc = e
            # Respect the retry-after header if present
            retry_after = int(e.response.headers.get("retry-after", base_delay * (2 ** attempt)))
            wait = min(retry_after + random.uniform(0, 1), max_delay)
            print(f"Rate limited (attempt {attempt+1}/{max_attempts}), waiting {wait:.1f}s")
            time.sleep(wait)

        except anthropic.APIStatusError as e:
            if e.status_code >= 500:  # Server errors are retryable
                last_exc = e
                wait = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
                print(f"Server error {e.status_code} (attempt {attempt+1}/{max_attempts}), waiting {wait:.1f}s")
                time.sleep(wait)
            else:
                raise  # 4xx client errors are not retryable

        except anthropic.APIConnectionError as e:
            last_exc = e
            wait = min(base_delay * (2 ** attempt), max_delay)
            print(f"Connection error (attempt {attempt+1}/{max_attempts}), waiting {wait:.1f}s")
            time.sleep(wait)

    raise last_exc  # All retries exhausted

# Simpler option: rely on SDK's built-in retry (default max_retries=2)
client_with_retry = anthropic.Anthropic(max_retries=3)
```

### Rate Limit Headers You Should Monitor

```python
def get_rate_limit_info(response: anthropic.types.Message) -> dict:
    """Extract rate limit headers from raw response for monitoring."""
    # Access the raw response via _request_id and headers
    # In production, use a middleware or response hook to log these
    return {
        "request_id": response._request_id,
        # Headers available via .with_raw_response.create():
        # x-ratelimit-limit-requests
        # x-ratelimit-limit-tokens
        # x-ratelimit-remaining-requests
        # x-ratelimit-remaining-tokens
        # x-ratelimit-reset-requests
        # x-ratelimit-reset-tokens
    }

# To access headers:
raw = client.messages.with_raw_response.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
remaining_requests = raw.headers.get("x-ratelimit-remaining-requests")
message = raw.parse()
```

### Common Mistake: Treating 400 Errors as Retryable

```python
# WRONG — 400 errors (invalid request) will never succeed on retry
except anthropic.APIStatusError:
    time.sleep(1)
    retry()  # Will fail again with the same 400

# RIGHT — only retry network/server errors, not client errors
except anthropic.RateLimitError:
    time.sleep(retry_after)
    retry()  # This makes sense
except anthropic.BadRequestError:
    log_and_fix_the_request()  # Never retry — fix the payload
```

---

## 9. Building AI-Powered APIs with FastAPI

### Production FastAPI + Claude Integration Pattern

```python
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from pydantic import BaseModel, Field
from functools import lru_cache
import anthropic
import asyncio
from datetime import datetime

app = FastAPI(title="AESF AI Service")


# --- Models ---

class ClassifyRequest(BaseModel):
    text: str = Field(..., min_length=1, max_length=50000)
    categories: list[str] = Field(default=["policy", "account", "contact", "opportunity"])

class ClassifyResponse(BaseModel):
    category: str
    confidence: str
    reasoning: str
    input_tokens: int
    output_tokens: int
    cache_hit: bool


# --- Dependencies ---

@lru_cache
def get_client() -> anthropic.AsyncAnthropic:
    return anthropic.AsyncAnthropic()

CLASSIFICATION_SYSTEM = """
You are a Salesforce data classifier for the AESF (Applied Epic for Salesforce) platform.
Classify user input into one of the provided categories.
Respond with valid JSON only: {"category": "...", "confidence": "high|medium|low", "reasoning": "..."}
"""


# --- Endpoints ---

@app.post("/classify", response_model=ClassifyResponse)
async def classify_text(
    request: ClassifyRequest,
    background_tasks: BackgroundTasks,
    client: anthropic.AsyncAnthropic = Depends(get_client),
):
    """Classify text into AESF data categories using Claude."""
    categories_str = ", ".join(request.categories)

    try:
        response = await client.messages.create(
            model="claude-haiku-4-5",  # Fast + cheap for classification
            max_tokens=256,
            system=[{
                "type": "text",
                "text": CLASSIFICATION_SYSTEM,
                "cache_control": {"type": "ephemeral"}  # Cache system prompt
            }],
            messages=[{
                "role": "user",
                "content": f"Categories: {categories_str}\n\nText to classify:\n{request.text}"
            }],
            output_config={"format": {
                "type": "json_schema",
                "schema": {
                    "type": "object",
                    "properties": {
                        "category": {"type": "string"},
                        "confidence": {"type": "string", "enum": ["high", "medium", "low"]},
                        "reasoning": {"type": "string"}
                    },
                    "required": ["category", "confidence", "reasoning"],
                    "additionalProperties": False
                }
            }}
        )

        if response.stop_reason == "refusal":
            raise HTTPException(status_code=422, detail="Content policy violation")

        import json
        result = json.loads(next(b.text for b in response.content if b.type == "text"))

        # Log usage asynchronously (don't block the response)
        background_tasks.add_task(
            log_api_usage,
            request_id=response._request_id,
            model=response.model,
            usage=response.usage
        )

        return ClassifyResponse(
            category=result["category"],
            confidence=result["confidence"],
            reasoning=result["reasoning"],
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens,
            cache_hit=response.usage.cache_read_input_tokens > 0,
        )

    except anthropic.RateLimitError:
        raise HTTPException(status_code=429, detail="AI service rate limit reached. Retry after backoff.")
    except anthropic.APIStatusError as e:
        raise HTTPException(status_code=503, detail=f"AI service error: {e.status_code}")


async def log_api_usage(request_id: str, model: str, usage):
    """Background task: log usage to DB for cost attribution."""
    # Write to your ai_usage_logs table
    pass


# Health check that verifies the AI service is reachable
@app.get("/health/ai")
async def ai_health(client: anthropic.AsyncAnthropic = Depends(get_client)):
    try:
        count = await client.messages.count_tokens(
            model="claude-haiku-4-5",
            messages=[{"role": "user", "content": "ping"}]
        )
        return {"status": "healthy", "test_tokens": count.input_tokens}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

### Common Mistake: Blocking the Event Loop in Async FastAPI

```python
# WRONG — using sync client in async FastAPI endpoint blocks the event loop
@app.post("/analyze")
async def analyze(text: str):
    client = anthropic.Anthropic()  # Sync client
    response = client.messages.create(...)  # Blocks the event loop!
    return response

# RIGHT — use AsyncAnthropic in async endpoints
@app.post("/analyze")
async def analyze(text: str, client: anthropic.AsyncAnthropic = Depends(get_client)):
    response = await client.messages.create(...)  # Non-blocking
    return response
```

---

## 10. Extended Thinking and Effort Control

### Adaptive Thinking (Opus 4.6+ and Sonnet 4.6)

On Claude Opus 4.6, 4.7, 4.8, and Sonnet 4.6, use `thinking: {type: "adaptive"}`. The model decides when and how much to think. Do **not** use `budget_tokens` for new code — it is deprecated on 4.6 and removed (returns 400) on 4.7/4.8.

```python
# Adaptive thinking with effort control
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # Show thinking summaries
    output_config={"effort": "high"},  # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "Design a data reconciliation algorithm for syncing Salesforce Opportunities with Epic EHR opportunities, handling conflict resolution when both sides update simultaneously."}]
)

for block in response.content:
    if block.type == "thinking":
        print(f"[Thinking]: {block.thinking[:200]}...")
    elif block.type == "text":
        print(f"[Response]: {block.text}")
```

### Effort Level Selection Guide

| Effort | Use When | Notes |
|---|---|---|
| `low` | Simple lookups, latency-sensitive, not intelligence-sensitive | Fast and cheap |
| `medium` | Most production tasks; balanced quality/cost | Good default for batch processing |
| `high` | Complex reasoning, agentic tasks | Default for most API use cases |
| `xhigh` | Coding, agentic work, long-horizon tasks | Best for Claude Code-style work |
| `max` | Hardest problems where correctness > cost | Diminishing returns on many tasks |

### Common Mistake: Using `budget_tokens` on Opus 4.7+

```python
# WRONG — returns 400 Bad Request on Opus 4.7, 4.8, and Fable 5
response = client.messages.create(
    model="claude-opus-4-8",
    thinking={"type": "enabled", "budget_tokens": 10000},  # 400 error!
    ...
)

# RIGHT — use adaptive thinking
response = client.messages.create(
    model="claude-opus-4-8",
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    ...
)
```

---

## N. Key Concepts Summary

```
Claude API Mental Model
│
├── CLIENT
│   ├── anthropic.Anthropic()          — sync, for scripts/workers
│   ├── anthropic.AsyncAnthropic()     — async, for FastAPI
│   └── Config: timeout (seconds), max_retries, base_url
│
├── MESSAGES ENDPOINT (POST /v1/messages)
│   ├── model          → claude-opus-4-8 (default), sonnet-4-6, haiku-4-5
│   ├── max_tokens     → hard cap (model unaware); stream if >16K
│   ├── messages       → [{role, content}] — stateless, send full history
│   ├── system         → stable instructions → cache this
│   ├── tools          → user-defined or server-side tools
│   ├── thinking       → {type: "adaptive"} on Opus 4.6+
│   ├── output_config  → {effort: "high", format: {json_schema}}
│   └── cache_control  → on content blocks; prefix match invariant
│
├── TOOL USE (agentic loop)
│   ├── Claude emits tool_use blocks → your code executes
│   ├── Send tool_result blocks back as user message
│   ├── Loop until stop_reason == "end_turn"
│   └── Tool Runner: auto-manages loop via @beta_tool decorator
│
├── STREAMING
│   ├── Required for max_tokens > ~16K
│   ├── client.messages.stream() context manager
│   └── stream.get_final_message() → full Message with usage
│
├── PROMPT CACHING
│   ├── Prefix match: any change invalidates downstream
│   ├── Render order: tools → system → messages
│   ├── cache_control on content blocks: {"type": "ephemeral"}
│   ├── Write cost: 1.25× (5min TTL) or 2× (1hr TTL)
│   ├── Read cost: 0.1× (90% savings)
│   └── Silent invalidators: timestamps, random IDs, unsorted JSON
│
├── TOKEN COUNTING
│   ├── client.messages.count_tokens() — model-specific, accurate
│   ├── NEVER use tiktoken — wrong tokenizer, undercounts by 15-20%
│   └── Track: input_tokens + cache_creation_tokens + cache_read_tokens
│
├── ERROR HANDLING
│   ├── AuthenticationError    → fix key, never retry
│   ├── BadRequestError (400)  → fix payload, never retry
│   ├── RateLimitError (429)   → retry with backoff, respect retry-after
│   ├── APIStatusError (5xx)   → retry with backoff
│   └── APIConnectionError     → retry with backoff
│
└── COST = (input × price_in) + (cache_write × price_in × 1.25)
          + (cache_read × price_in × 0.1) + (output × price_out)
```

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** You are building a FastAPI endpoint that calls Claude. Should you use `anthropic.Anthropic()` or `anthropic.AsyncAnthropic()`? What is the key mistake to avoid with client initialization?

**2.** What is the default `max_retries` on the Anthropic SDK client, and which HTTP status codes trigger automatic retries?

**3.** You add `thinking={"type": "enabled", "budget_tokens": 8000}` to a request to `claude-opus-4-8`. What will happen, and what is the correct replacement?

**4.** A colleague reports that their `cache_read_input_tokens` is always 0 despite sending identical requests. You find their system prompt contains `f"Today is {datetime.now().strftime('%Y-%m-%d')}"`. Why does this break caching, and how would you fix it?

**5.** What is the render order for prompt content when computing cache keys, and why does this matter for placing `cache_control` breakpoints?

**6.** Explain the difference between `max_tokens` and `output_config: {task_budget}`. Which one is the model aware of?

**7.** You have a FastAPI endpoint that classifies short text (< 200 tokens) into 5 categories, running 10,000 times/day. Which Claude model would you choose and why? Include a rough cost calculation.

**8.** Your agentic loop is hitting an infinite loop — Claude keeps calling tools without reaching `end_turn`. Name three likely causes and one defensive measure to add to the loop.

**9.** In the manual agentic loop, you collect `tool_results` and need to send them back. What is the correct structure of the messages array at the point you make the next API call?

**10.** What is the `stop_reason` value when Claude's safety classifiers decline a request on `claude-fable-5`? What is the cost to you if the refusal happens before any output? What if it happens mid-stream?

**11.** You want to cache a large 10,000-token system prompt. The minimum cacheable prefix for `claude-opus-4-8` is what value, and how do you verify the cache is actually being used?

**12.** A junior engineer on your team uses `tiktoken` to estimate Claude token counts for cost projections. What is wrong with this approach and what should they use instead?

**13.** You are running `claude-opus-4-8` with `max_tokens=128000`. What must you do to avoid an SDK error, and why?

**14.** Describe two distinct approaches to structured output with Claude (beyond just prompting for JSON). What are the trade-offs of each?

**15.** In the AESF middleware context, you want Claude to analyze incoming Salesforce webhook payloads and enrich them with Epic EHR data before writing to the queue tables. Sketch the tool definitions you would create and explain the loop flow.

**16.** What is `stream.get_final_message()` and why is it preferred over collecting streaming chunks manually in most API use cases?

**17.** Explain the difference between `cache_creation_input_tokens`, `cache_read_input_tokens`, and `input_tokens` in the usage object. Write the correct formula for total effective cost.

**18.** You receive a `RateLimitError`. The response header `retry-after` says 45 seconds. Your exponential backoff formula gives a 3-second wait. Which should you use and why?

**19.** When should you use `tool_choice: {"type": "any"}` versus `{"type": "auto"}`? Give a concrete AESF example for each.

**20.** You are designing a batch processing job that uses Claude to extract structured data from 50,000 Epic EHR documents overnight. What API surface would you use instead of individual `messages.create()` calls, and what is the cost benefit?

---

### Answers

??? note "Reveal Answers"

    **1.** Use `anthropic.AsyncAnthropic()` in an async FastAPI application. The sync `Anthropic()` client will block the event loop when called from an async endpoint, preventing other requests from being served concurrently. The key initialization mistake is creating a new client instance inside each request handler — this creates a new connection pool on every request, which wastes resources and is slow. Instead, create the client once as a module-level singleton or FastAPI dependency using `@lru_cache` or `lifespan`, and inject it via `Depends()`.

    **2.** The default `max_retries` is 2. The SDK automatically retries on: 408 (Request Timeout), 409 (Conflict), 429 (Rate Limit), and any 5xx server error (500, 502, 503, 529). It uses exponential backoff with jitter. Connection errors (`APIConnectionError`) are also retried. Client errors 400, 401, 403, and 404 are **not** retried because they indicate a problem with the request itself that retrying would not fix.

    **3.** The request will return a `400 Bad Request` error. The `thinking: {type: "enabled", budget_tokens: N}` parameter was deprecated on Opus 4.6 and fully removed on Opus 4.7 and later. On Opus 4.8, sending this parameter is an invalid request. The correct replacement is `thinking={"type": "adaptive"}` combined with `output_config={"effort": "high"}` (or another effort level). Adaptive thinking lets the model decide when and how much to think, and works significantly better than a fixed token budget in practice.

    **4.** Prompt caching is a prefix match — any change to the prefix bytes invalidates the entire cache downstream from that point. `datetime.now()` produces a different string on every request, making the system prompt a unique string every time. The cache key changes on every call, so no cache entry is ever reused. The fix is to move volatile content like timestamps out of the system prompt entirely. Place the current date in the `messages` array as part of the user turn, or inject it only in the first message of a session — never in the `system` field that is meant to be stable and cached.

    **5.** The render order for cache key computation is: `tools` → `system` → `messages`. This order is fixed and cannot be changed. It matters for breakpoint placement because a `cache_control` marker on the last system block will cache everything that rendered before it — that means all tool definitions plus the entire system prompt. If you put a breakpoint on a messages block, only what appears before that point in the messages is cached. Understanding this order lets you place breakpoints at true stability boundaries: tools (rarely change), then system (changes per feature/mode), then messages (changes every turn).

    **6.** `max_tokens` is a hard ceiling enforced by the API — the model is **not** aware of it. If the model hits it mid-generation, the response is cut off and `stop_reason` becomes `"max_tokens"`. `output_config: {task_budget}` (beta, Opus 4.7+) is a budget the model **is aware of** — it receives a running countdown during generation and self-moderates its output length and approach to finish gracefully within budget. They serve different purposes: `max_tokens` prevents runaway costs via truncation; `task_budget` improves quality by letting the model plan around constraints.

    **7.** Use `claude-haiku-4-5` at $1.00/$5.00 per million tokens. Short classification tasks (< 200 token input) do not require Opus-class reasoning. With a stable system prompt of ~500 tokens cached, and an average 200-token input + 50-token output per request: input at ~0.1× cache cost (after first call) + output at full price. Approximate daily cost: 10,000 requests × (200 tokens × $1 × 0.1 + 50 tokens × $5) / 1,000,000 ≈ $0.20 input + $2.50 output = ~$2.70/day vs. ~$7.50/day on Sonnet or ~$13.75/day on Opus. Haiku is appropriate here; save Opus for complex analysis tasks.

    **8.** Three causes of infinite agentic loops: (1) Tool returns an error format that confuses Claude into retrying the same call, (2) The tool's description or result keeps triggering the same follow-up call (tool is poorly scoped), (3) `stop_reason == "pause_turn"` is not handled and the loop re-sends without the assistant message (creating a loop). The defensive measure is a max iterations counter: track `iteration = 0; max_iterations = 10`, increment on each loop, and raise an exception or return a partial result if exceeded. Also always check `stop_reason == "end_turn"` first and break immediately.

    **9.** The correct messages array at the next API call should be:
    ```
    [
      {"role": "user", "content": <original user message>},
      {"role": "assistant", "content": <response.content>},  # full content list with tool_use blocks
      {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "...", "content": "..."},
        ... # all tool results in ONE user message
      ]}
    ]
    ```
    The critical requirement is that the assistant's `response.content` (which includes the `tool_use` blocks) must appear before the tool results. Sending tool results without the preceding assistant tool_use blocks will cause a 400 error. All parallel tool results must go in a single user message, not multiple separate messages.

    **10.** The `stop_reason` is `"refusal"`. If the refusal happens before any output (the safety classifier fires on the input), the response has an empty `content` array and you are **not billed at all** — no input or output tokens are charged. If it happens mid-stream after some output has been generated, you are billed for the output tokens already streamed, and you should discard the partial output rather than treating it as complete. Always check `stop_reason` before accessing `response.content[0]` to avoid index errors on empty content arrays.

    **11.** The minimum cacheable prefix for `claude-opus-4-8` is **4096 tokens**. A 10,000-token system prompt easily meets this threshold. To verify the cache is working: after the first call, check `response.usage.cache_creation_input_tokens` — it should be non-zero (showing the cache was written). On the second identical call, `response.usage.cache_read_input_tokens` should be non-zero (showing the cache was read). If `cache_read_input_tokens` is 0 on repeated calls, a silent invalidator is changing the prefix bytes between requests.

    **12.** `tiktoken` is OpenAI's tokenizer. It undercounts Claude tokens by approximately 15-20% on typical English text, and by much more on code, non-English text, and structured data formats. This makes cost projections inaccurate and can cause you to underestimate how close you are to context limits. The correct approach is `client.messages.count_tokens(model="claude-opus-4-8", messages=[...])` — this calls the actual Anthropic API tokenizer for the specific model, giving a correct count. It's a lightweight API call (no inference, no output tokens billed).

    **13.** You must use streaming. The Anthropic SDK raises a `ValueError` if you request more than approximately 16,000 output tokens in a non-streaming request. The reason is that large non-streaming requests can exceed the SDK's HTTP read timeout (10 minutes default) since the full response must be buffered before delivery. Use `client.messages.stream(...)` as a context manager. If you still need a single `Message` object, call `stream.get_final_message()` inside the context — this gives you the complete response with usage statistics and stop reason, identically structured to a non-streaming response.

    **14.** Two approaches: (1) **Structured outputs via `output_config.format`** — set `output_config={"format": {"type": "json_schema", "schema": {...}}}`. Claude's output is guaranteed to validate against your schema. No need for a prefill or parsing errors. Best for extracting well-defined data. Tradeoff: first-request latency as the schema is compiled (24-hour cache after that). (2) **Strict tool use via `strict: true`** — add `"strict": True` to a tool definition alongside `additionalProperties: False` and `required` on all fields. Claude's `tool_use.input` will always be valid per schema. Best when you want structured *actions* rather than structured *text output*. Tradeoff: only applies to tool call inputs, not to the final text response.

    **15.** Two tool definitions: `get_salesforce_account(account_id: str)` — fetches Account data from AESF Salesforce via the middleware API; and `get_epic_client(client_code: str, include_policies: bool)` — fetches client data from Epic EHR via the middleware. The loop flow: (1) Claude receives the webhook payload as the user message, (2) Claude calls `get_salesforce_account` to get current Salesforce data, (3) Your code executes the middleware call and returns the result, (4) Claude calls `get_epic_client` to get EHR context, (5) Your code executes and returns, (6) Claude generates an enriched payload recommendation and reaches `end_turn`, (7) You write the result to the queue table. Tool descriptions should specify *when* to call them, not just what they do.

    **16.** `stream.get_final_message()` is a helper method available inside the `messages.stream()` context manager that waits for the stream to complete and returns a fully-assembled `Message` object — identical in structure to what `messages.create()` returns (with `usage`, `stop_reason`, `content` with all blocks). It is preferred over manual chunk collection because: (1) it handles reconnection and partial assembly internally, (2) it guarantees you have the complete usage statistics including `cache_read_input_tokens`, and (3) it simplifies downstream code that only needs the final result and not the streaming events. Use raw event iteration only when you need per-token delivery to a user interface.

    **17.** The three usage fields mean: `input_tokens` = tokens processed at full input price (not cached), `cache_creation_input_tokens` = tokens written to the cache this request (1.25× cost for 5-min TTL, 2× for 1-hour TTL), `cache_read_input_tokens` = tokens served from cache this request (0.1× cost). The correct cost formula is: `total_cost = (input_tokens × price_in + cache_creation_tokens × price_in × 1.25 + cache_read_tokens × price_in × 0.1 + output_tokens × price_out) / 1_000_000`. Total prompt size = `input_tokens + cache_creation_tokens + cache_read_tokens` — not just `input_tokens` alone.

    **18.** Use the `retry-after` header value (45 seconds), not your computed backoff (3 seconds). The `retry-after` header is set by the API based on when the rate limit will actually reset. Retrying before this time will result in another rate limit error. Your exponential backoff formula is designed as a fallback heuristic for when no server guidance is available — when the server tells you exactly when to retry, that value takes precedence. In code: `retry_after = int(e.response.headers.get("retry-after", your_computed_backoff))` and then `time.sleep(retry_after + random.uniform(0, 1))` (small jitter to prevent thundering herd).

    **19.** Use `tool_choice: {"type": "any"}` when you require Claude to use a tool on every turn — for example, when building a structured extraction pipeline where every response must be a `extract_policy_data` tool call, not free text. You want to guarantee a structured output format. Use `tool_choice: {"type": "auto"}` (the default) when Claude should decide whether a tool is needed — for example, an AESF assistant that can answer simple questions from context but should call `get_account_data` only when it actually needs fresh Salesforce data. `auto` produces more natural behavior in conversational agents; `any` is better for guaranteed structured pipelines.

    **20.** Use the **Message Batches API** (`client.messages.batches.create()`). It processes up to 100,000 requests asynchronously and costs 50% of standard pricing — half price for input and output tokens. For 50,000 documents at Opus pricing, this is a significant saving. The trade-off is latency: batches complete within 1 hour (sometimes much faster), not in real-time. This is ideal for overnight ETL jobs. Implementation: create a batch with all 50,000 requests, poll `client.messages.batches.retrieve(batch_id).processing_status` until `"ended"`, then stream results with `client.messages.batches.results(batch_id)`. Match results to inputs via `custom_id` — results arrive in unpredictable order, so never assume position corresponds to submission order.
