# Week 3 — LLM Fundamentals

**Week of:** June 22, 2026
**Estimated study time:** ~2 hours
**Tags:** `ai` `llm` `ml`

---

## Overview

Large Language Models have crossed a threshold that should matter deeply to every senior and staff engineer: they are no longer just research curiosities or hype—they are production infrastructure. The same way you once had to understand TCP/IP to build reliable networked systems, you now need to understand how LLMs actually work to build reliable AI-assisted applications and avoid the class of bugs that comes from treating them like magic black boxes.

For you specifically, the AESF stack sits at a compelling intersection. Your FastAPI middleware orchestrates data flows between Salesforce, Epic EHR, and GCP. Every one of those flows involves parsing semi-structured data, classifying intent, generating summaries, or making routing decisions—all tasks where LLMs can provide enormous leverage if integrated correctly. But "integrated correctly" requires knowing what these models actually do, where they fail, and what their architectural limits are.

This week covers the foundations: how transformer models work at the conceptual level, what tokens and context windows really mean for your system design, how models are trained and aligned to human preferences, and the practical taxonomy of model types you'll encounter in production. By the end, you'll be able to read a model spec sheet, have an informed conversation about fine-tuning vs. prompting trade-offs, and make architectural decisions about where to add LLM capabilities in a system like AESF without introducing instability.

Think of this week as the prerequisite for Weeks 4–7 (Prompt Engineering, Claude API, RAG, and Agents). Everything we cover here will have a direct analog in those practical weeks.

---

## 1. The Transformer Architecture — What Actually Happens

You don't need to implement a transformer to work with LLMs professionally. But you do need a mental model accurate enough to predict when they'll behave unexpectedly. Here's the minimum viable understanding.

### The core job

A language model takes a sequence of tokens as input and outputs a probability distribution over all possible next tokens. That's it. The sophistication—and the emergent capabilities—arise entirely from doing this at enormous scale with billions of parameters trained on trillions of tokens.

### Tokens first, then attention

Text is first converted to **tokens** (covered in depth in Section 2). The tokens are converted to **embedding vectors**—dense float arrays in a high-dimensional space. At this point, the model has no positional information; "dog bites man" and "man bites dog" would look the same. So **positional encodings** are added to each embedding.

The embeddings then pass through a stack of **transformer blocks**. Each block has two key components:

1. **Multi-head self-attention** — Every token attends to every other token and updates its own representation based on that context. "Attention" is a weighted sum: each query token computes a score against all key tokens and takes a weighted average of value vectors. The "multi-head" part means this happens in parallel across several learned subspaces, letting the model attend to different kinds of relationships simultaneously (syntactic, semantic, coreference, etc.).

2. **Feed-forward network (FFN)** — After attention, each token position passes through an independent MLP. This is where many believe the "knowledge" of the model is stored—attention routes information; FFNs synthesize it.

After all transformer blocks, the final hidden state is projected through a linear layer and softmax to produce the next-token probability distribution.

```
Input tokens → Embeddings + Positional Encoding
      ↓
[Transformer Block × N]
  ├─ LayerNorm
  ├─ Multi-Head Self-Attention
  ├─ Residual Add
  ├─ LayerNorm
  └─ Feed-Forward Network + Residual Add
      ↓
Linear Projection → Softmax → Next-token probabilities
```

### Why "emergent" capabilities happen

No single part of the architecture explains why a model that was trained to predict the next word can also reason about logic, write code, or explain counterintuitive physics. The leading hypothesis: at sufficient scale, the model learns compressed world-representations in its weights because they're necessary to accurately predict the next token across diverse text. Writing coherent code requires understanding program semantics. Predicting factual prose requires encoding world knowledge. The next-token objective is simpler than it sounds because it's satisfied by arbitrary amounts of underlying intelligence.

**Common mistake:** Assuming that because transformers use matrix multiplications, their capabilities are fully explainable or predictable by their architecture alone. They aren't. The relationship between scale, training data, and capability is empirical, not derived.

---

## 2. Tokens and Context Windows

### What is a token?

Tokens are the atomic units of LLM input and output—neither words nor characters but subword units learned from the training corpus via algorithms like Byte-Pair Encoding (BPE). A rough rule of thumb: 1 token ≈ 4 characters ≈ 0.75 words for English prose. Non-English, code, and numeric data tokenize less efficiently.

```python
# Quick token estimation (not calling an API — pure heuristic)
def estimate_tokens(text: str) -> int:
    """Rough BPE token estimate: 1 token per ~4 chars."""
    return max(1, len(text) // 4)

# More accurate: use the model's actual tokenizer
# pip install tiktoken  (for OpenAI models)
# from anthropic import Anthropic  (client exposes token counts)

import tiktoken
enc = tiktoken.get_encoding("cl100k_base")  # GPT-4 tokenizer
tokens = enc.encode("SELECT * FROM accounts WHERE id = 12345")
print(len(tokens))  # → 12 tokens
```

Practical implications:

| Text type | Tokens per 1000 chars | Cost multiplier vs. English prose |
|-----------|----------------------|------------------------------------|
| English prose | ~250 | 1× |
| Python code | ~190 | 0.75× |
| JSON (with keys) | ~280 | 1.1× |
| SOQL queries | ~200 | 0.8× |
| Japanese/Chinese | ~500+ | 2×+ |

For AESF, this matters when you're passing Salesforce JSON payloads (large, repetitive key names), Epic API responses (XML/JSON with verbose schemas), or Alembic migration files to an LLM. Those will tokenize more densely than you expect.

### Context windows

The context window is the maximum number of tokens a model can process in a single forward pass—both input (prompt) and output (completion) combined. As of mid-2026:

| Model family | Context window |
|---|---|
| Claude Sonnet 4 / Opus 4 | 200K tokens |
| GPT-4o | 128K tokens |
| Gemini 1.5 Pro | 1M tokens |
| Llama 3.3 70B | 128K tokens |

200K tokens ≈ 150,000 words ≈ a full novel or the entire codebase of a mid-sized service. This sounds unlimited but isn't:

1. **Attention cost is quadratic with sequence length.** Doubling the context doesn't double the compute—it quadruples it (in naive implementations; models use sliding-window and KV-cache tricks to reduce this, but cost still grows super-linearly).
2. **"Lost in the middle" degradation.** Models perform best at retrieving information from the beginning and end of long contexts. Middle content is reliably attended to less accurately. Don't assume that pasting 100K tokens of logs in the middle guarantees the model read them carefully.
3. **Output tokens are still bounded.** Even with a 200K context window, most models cap output at 4K–8K tokens by default unless configured otherwise.

**Common mistake:** Using the full context window as a substitute for retrieval. Pasting your entire Salesforce schema into every request is expensive and degrades quality. Use RAG (Week 6) to retrieve only the relevant schema sections.

### Tokenization affects model behavior

Because tokenization is inconsistent, models can behave unexpectedly on certain inputs:

```python
# "12345" tokenizes differently than "1 2 3 4 5"
# Arithmetic on large numbers is unreliable because each digit may be
# a separate token — the model has never seen the token "3141592653"
# in contexts that encode its mathematical properties.

# In AESF context: don't ask LLMs to do arithmetic on policy limits,
# claim amounts, or Salesforce IDs. Do that in Python and pass the result.
```

---

## 3. Embeddings — Meaning as Geometry

### What embeddings are

Every token, and by extension every sequence of tokens, is mapped to a point in a high-dimensional vector space. Embeddings are dense float vectors (typically 768 to 4096 dimensions) where geometric proximity encodes semantic similarity.

This isn't just metaphor:

```
embedding("king") - embedding("man") + embedding("woman") ≈ embedding("queen")
embedding("Paris") - embedding("France") + embedding("Germany") ≈ embedding("Berlin")
```

At the model level, embeddings are learned internal representations. At the API level, you can also call embedding endpoints directly to get a vector representation of any text, which is the foundation of semantic search and RAG.

### Embedding models vs. generative models

These are different products serving different use cases:

| Use case | Model type | Example |
|---|---|---|
| Generate text | Generative LLM | claude-sonnet-4-6 |
| Semantic similarity / search | Embedding model | text-embedding-3-large |
| Classification | Either (fine-tune embedding; prompt generative) | depends on scale |
| Clustering documents | Embedding model | voyage-large-2 |

In AESF, embeddings are most immediately useful for:
- Semantic search over policy types, activity codes, or SOQL query results
- Clustering Epic sync error messages to detect novel failure modes
- Matching natural language queries to predefined Salesforce reports

```python
import anthropic

client = anthropic.Anthropic()

# As of 2026, Claude's embedding API is accessed via the messages API
# with structured output, or directly via voyage-* models on Anthropic's infra.
# For general use, use the voyage-large-2 model via the API.

# Conceptual pseudocode (actual SDK call shape may vary):
response = client.embeddings.create(
    model="voyage-large-2",
    input=["AESF__Policy__c sync failed: record locked", 
           "Opportunity trigger timeout after 30s",
           "Contact upsert conflict on external ID"]
)
vectors = [r.embedding for r in response.data]
# Now cluster these to find error families
```

**Common mistake:** Using generative model embeddings (the hidden states) as if they're stable identifiers. Internal embeddings change between model versions. Use dedicated embedding model endpoints for persistent vector stores.

---

## 4. Sampling Parameters — Temperature, Top-p, Top-k

After the model produces a probability distribution over the vocabulary, you don't just take the highest-probability token (that would be deterministic and often repetitive). Instead, you **sample** from the distribution with controls that let you tune the balance between consistency and creativity.

### Temperature

Temperature rescales the logits before softmax. Low temperature sharpens the distribution (model picks near-certain tokens); high temperature flattens it (model is more "creative"—or incoherent).

```
logits_scaled = logits / temperature
probabilities = softmax(logits_scaled)
```

| Temperature | Behavior | Good for |
|---|---|---|
| 0.0 | Greedy/deterministic | Structured extraction, code generation, JSON output |
| 0.3–0.5 | Focused but slightly varied | Technical explanations, factual Q&A |
| 0.7–1.0 | Default; balanced | General chat, summarization |
| 1.5–2.0 | Very diverse / often incoherent | Creative writing experiments only |

For AESF middleware, when you're using an LLM to extract fields from Epic API responses or generate SOQL queries, use temperature 0–0.2. You want determinism, not creativity.

### Top-p (nucleus sampling)

Rather than rescaling all probabilities, top-p keeps only the smallest set of tokens whose cumulative probability exceeds `p`, then samples from that set.

```
top_p = 0.9 → sample only from tokens that together account for 90% of the probability mass
```

Top-p adapts dynamically: when the model is very confident (one token has 95% probability), it effectively selects that token. When it's uncertain, it samples from a wider set. This avoids temperature's problem of occasionally sampling very-low-probability garbage tokens.

### Top-k

Top-k simply takes the k highest-probability tokens, ignoring everything else. Less popular than top-p for language generation because it doesn't adapt to the distribution shape.

### How they interact

In practice, you set both temperature and top-p, and the more restrictive constraint wins at each position:

```python
# FastAPI endpoint calling Claude for structured extraction
import anthropic
from pydantic import BaseModel

client = anthropic.Anthropic()

class PolicyExtraction(BaseModel):
    policy_number: str
    effective_date: str
    line_of_business: str
    premium: float

async def extract_policy_fields(raw_text: str) -> PolicyExtraction:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        temperature=0.0,          # deterministic for extraction
        system="Extract structured fields from insurance policy text. Return valid JSON only.",
        messages=[{"role": "user", "content": raw_text}]
    )
    return PolicyExtraction.model_validate_json(response.content[0].text)
```

**Common mistake:** Using high temperature for tasks that require consistency. Every time you re-run an extraction with temperature=1.0, you may get different field values—a reliability nightmare in an ETL pipeline.

---

## 5. How Models Are Trained — Pre-training, Fine-tuning, and RLHF

### Phase 1: Pre-training

The model is initialized with random weights and trained to predict the next token across an enormous corpus (trillions of tokens from web crawls, books, code, scientific papers). This phase is astronomically expensive—training GPT-4 class models costs tens to hundreds of millions of dollars in compute.

The result is a **base model**: extraordinarily capable at completion but not particularly useful for conversation. Ask a base model "What is the capital of France?" and it might respond "What is the capital of Germany? What is the capital of Spain?" because it learned that these questions appear together in quizzes.

### Phase 2: Supervised Fine-tuning (SFT)

The base model is fine-tuned on a curated dataset of (prompt, ideal completion) pairs written or validated by human annotators. This teaches the model to respond in the instruction-following format: answer questions directly, follow instructions, be helpful.

SFT is also the mechanism for **domain fine-tuning**: training on your own data to improve performance on specific tasks. A fine-tuned model for insurance domain text will outperform a general model on AESF-specific extractions—but fine-tuning requires careful dataset curation, infrastructure, and ongoing maintenance.

### Phase 3: RLHF — Reinforcement Learning from Human Feedback

RLHF is what transforms a competent instruction-follower into a model that feels aligned with human preferences. The process:

1. **Collect preference data**: Show human raters pairs of model responses to the same prompt and ask them to pick the better one.
2. **Train a reward model (RM)**: A separate neural network that learns to predict human preference scores.
3. **Optimize with PPO (Proximal Policy Optimization)**: The language model is updated using RL to maximize the reward model's score, with a KL-divergence penalty to prevent it from drifting too far from the SFT checkpoint.

RLHF is why modern LLMs are helpful, harmless, and honest instead of just predicting text statistically. It's also why they can be **sycophantic** (overfit to saying what sounds good) and why they **refuse** certain requests (the reward model penalized those completions during training).

```
Pre-training         SFT                  RLHF
(raw capability)  (instruction format)  (aligned behavior)
Base Model     →  Instruct Model     →  Chat/Production Model
```

### Constitutional AI (CAI) and RLAIF

Anthropic's Claude models use **Constitutional AI**: instead of relying purely on human preferences, the model is trained against a written constitution of principles. Another LLM (or the same model) critiques completions against these principles, generating synthetic feedback that can scale beyond human annotation. This is sometimes called RLAIF (RL from AI Feedback).

---

## 6. Base Models vs. Instruction-Tuned vs. RLHF Models

| Model type | Training | Behavior | Use when |
|---|---|---|---|
| Base model | Pre-training only | Completion of text; not conversational | Academic research; custom fine-tuning starting point |
| Instruction-tuned (SFT) | Pre-training + SFT | Follows instructions; direct answers | Task-specific fine-tuning; less safety-sensitive deployments |
| RLHF/CAI aligned | Pre-training + SFT + RLHF | Helpful, refuses harmful requests, consistent format | Production applications; customer-facing features |
| Fine-tuned specialist | Any of above + domain data | Better at specific domain | High-volume domain-specific tasks after evaluation confirms ROI |

In AESF, you will almost always use production-grade RLHF models (claude-sonnet-4-6, claude-haiku-4-5) rather than base models. Base models are for researchers building the alignment pipeline, not for application developers. The one exception: if you're building your own fine-tuned model for a very specific extraction task that needs to run at high volume with minimal latency, you might start from an SFT checkpoint.

**Common mistake:** Fine-tuning when prompting would suffice. Fine-tuning is expensive, brittle to maintain, and loses the ability to take advantage of model upgrades. Prompt engineering can close 80–90% of the gap for most domain-adaptation tasks. Fine-tune only when you have thousands of labeled examples and a performance gap that prompting can't close.

---

## 7. Quantization — Running Big Models in Small Spaces

### The memory problem

A model with 70 billion parameters, stored in 32-bit floats, requires 70B × 4 bytes = 280 GB of GPU VRAM—far more than fits in a single GPU (A100s have 80 GB). Inference at scale is fundamentally a memory-bandwidth problem.

**Quantization** reduces the precision of model weights from 32-bit floats (FP32) to lower-precision representations:

| Format | Bits per weight | Memory for 70B model | Quality loss |
|---|---|---|---|
| FP32 | 32 | 280 GB | None (baseline) |
| BF16 | 16 | 140 GB | Negligible |
| INT8 | 8 | 70 GB | Small |
| INT4 (Q4) | 4 | 35 GB | Moderate |
| INT3 | 3 | ~26 GB | Significant |
| INT2 | 2 | ~17 GB | Usually unacceptable |

### How quantization works (conceptually)

Instead of storing weights as `[-0.3141592653589793, 0.7071067811865476, ...]`, you store an integer in a smaller range and a scale factor:

```
Original: 0.7071067811865476 (FP32)
Quantized to INT8: scale=0.005, zero_point=0 → stored value = round(0.707 / 0.005) = 141
Reconstructed: 141 × 0.005 = 0.705  (error: 0.002)
```

The error is small for individual weights but accumulates through billions of multiplications. INT8 quantization typically degrades benchmark performance by <1%; INT4 by 2–5% on standard benchmarks (but more on sensitive tasks like code generation or structured extraction).

### GGUF, GPTQ, AWQ — formats you'll encounter

| Format | Technique | Optimized for |
|---|---|---|
| GGUF | Symmetric/asymmetric INT quantization | CPU inference via llama.cpp |
| GPTQ | Post-training quantization with calibration data | GPU inference (vLLM, AutoGPTQ) |
| AWQ (Activation-aware Weight Quantization) | Protects salient weights from quantization | GPU; better quality than GPTQ at same bit-width |
| ExLlamaV2 | Dynamic quantization | Consumer GPU inference |

For AESF, quantization matters when you're considering running open-source models (Llama 3, Mistral) on self-hosted GKE nodes rather than API-based models. A Q4 Llama 3.3 70B can run on 2× A100s; the FP16 version needs 4×. Cost difference: roughly 2×.

**Common mistake:** Comparing quantized open-source models to full-precision API models without controlling for bit-width. A 4-bit Llama 70B is closer to a 7B model in effective capacity than to a FP16 70B model.

---

## 8. Capabilities and Hard Limits

### What LLMs are reliably good at

- **Text classification and extraction** at scale (entity extraction, intent classification, field parsing)
- **Summarization** of long documents
- **Code generation** for well-specified tasks with clear I/O contracts
- **Format transformation** (JSON → Markdown → SQL, etc.)
- **Explaining concepts** with adjustable detail level
- **Drafting** communications, documentation, ticket descriptions

### Hard limits — not "improve with better prompting"

| Limit | Why it exists | Implication for AESF |
|---|---|---|
| No real-time knowledge | Training data has a cutoff; model doesn't browse | Don't ask the model about current Salesforce API versions; give them explicitly |
| Non-determinism | Sampling process; exact reproducibility requires temp=0 + seed | Log prompts + outputs in your middleware; don't trust outputs without logging |
| Hallucination | Model generates plausible text, not verified facts | Never use LLM output as ground truth without validation (e.g., for Epic sync decisions) |
| Context window degradation | "Lost in the middle" problem | Keep system prompts short; don't rely on critical info buried in the middle |
| Can't count tokens in real time | Token counting isn't an internal capability | Estimate tokens before API calls; handle `context_length_exceeded` errors explicitly |
| No persistent memory across calls | Each API call is stateless | You must pass conversation history explicitly in the messages array |
| Unreliable arithmetic | Digits are tokens; math isn't encoded | Handle all numeric calculations in Python; only ask LLM for text reasoning |
| Can't execute code | Text generation only (without tool use) | Use function/tool calling (Week 5) to give models access to Python execution |

### The consistency trap

LLMs can give different answers to the same question across runs (or even within a long context window). For AESF middleware—where a wrong sync decision can create orphaned records in Epic or Salesforce—you cannot treat LLM outputs as authoritative without a validation layer:

```python
from enum import Enum
import anthropic

class SyncDecision(str, Enum):
    SYNC = "sync"
    SKIP = "skip"
    MANUAL_REVIEW = "manual_review"

async def classify_sync_conflict(conflict_description: str) -> SyncDecision:
    """
    LLM assists classification, but result is validated against enum.
    Never trust raw LLM text for routing logic.
    """
    client = anthropic.Anthropic()
    
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # cheap + fast for classification
        max_tokens=10,
        temperature=0.0,
        system=(
            "Classify the sync conflict. "
            "Reply with exactly one of: sync, skip, manual_review. No other text."
        ),
        messages=[{"role": "user", "content": conflict_description}]
    )
    
    raw = response.content[0].text.strip().lower()
    try:
        return SyncDecision(raw)
    except ValueError:
        # LLM returned unexpected text — route to human review
        return SyncDecision.MANUAL_REVIEW
```

---

## 9. Model Selection in Practice

With the Claude model family (your primary API), the decision tree is:

```
What's the task?
├── High-volume, fast, cheap (classification, extraction, triage)
│   └── claude-haiku-4-5-20251001
├── Balanced quality + cost (drafting, analysis, moderate complexity)
│   └── claude-sonnet-4-6         ← default choice
├── Maximum intelligence (complex reasoning, architecture decisions)
│   └── claude-opus-4-8
└── Narrative/creative tasks (documentation, user-facing copy)
    └── claude-fable-5
```

Cost roughly: Haiku ≈ 1×, Sonnet ≈ 5×, Opus ≈ 15× (per token). For an ETL pipeline that classifies thousands of sync events per hour, the difference between Haiku and Opus is the difference between a $50/month AI feature and a $750/month one.

**Common mistake:** Reaching for the largest model by default. Start with Haiku + good prompting and escalate only when you measure a quality gap. You'll often find Sonnet is unnecessary for structured extraction tasks.

---

## 10. Key Concepts Summary

```
LLM FUNDAMENTALS — MENTAL MODEL TREE
│
├── Architecture
│   ├── Tokenization (BPE subwords → ints)
│   ├── Embeddings (tokens → high-dim vectors)
│   ├── Positional encoding (adds sequence order)
│   └── Transformer blocks × N
│       ├── Multi-head self-attention (routing information)
│       └── Feed-forward network (synthesizing knowledge)
│
├── Context
│   ├── Context window = input + output token budget
│   ├── Lost-in-the-middle degradation
│   └── KV cache (speeds up long-context inference)
│
├── Sampling
│   ├── Temperature (sharpens/flattens distribution)
│   ├── Top-p (nucleus; keeps cumulative prob mass)
│   └── Top-k (keeps k highest-prob tokens)
│
├── Training pipeline
│   ├── Pre-training (next-token prediction at scale)
│   ├── SFT (instruction following)
│   └── RLHF / CAI (alignment to human preferences)
│
├── Model taxonomy
│   ├── Base model (raw; not instruction-following)
│   ├── Instruct/SFT model (task-following)
│   └── Aligned model (helpful + safe + honest)
│
├── Quantization
│   ├── FP32 → BF16 → INT8 → INT4
│   ├── Formats: GGUF (CPU), GPTQ, AWQ (GPU)
│   └── INT4 = ~35 GB for 70B; usable on 2× A100
│
└── Hard limits (never prompt around these)
    ├── No real-time knowledge
    ├── Non-determinism (temp > 0)
    ├── Hallucination (no internal truth-checking)
    ├── Unreliable arithmetic
    └── Stateless (no persistent memory)
```

---

## Quiz — 20 Questions

Test your understanding. Try to answer without looking back, then check the answers below.

---

### Questions

**1.** What is the core task a language model performs at inference time?

**2.** In the transformer architecture, what is the role of multi-head self-attention vs. the feed-forward network?

**3.** How many tokens does the string `"SELECT * FROM accounts WHERE id = 12345"` approximately contain? Does it matter for cost estimation?

**4.** You're calling an LLM to extract structured fields from Salesforce JSON payloads. What temperature should you use, and why?

**5.** A colleague suggests pasting your entire 5,000-line database schema into every prompt "so the model always has context." What's wrong with this approach?

**6.** What is the difference between a base model and an instruction-tuned model?

**7.** Explain RLHF in plain English: what problem does it solve and how?

**8.** A model trained with RLHF is described as "sycophantic." What does that mean, and why does RLHF cause it?

**9.** You have a vector `embedding("Paris") - embedding("France") + embedding("Germany")`. What would this approximately equal, and what does this tell you about how embeddings work?

**10.** What is top-p (nucleus) sampling, and why is it often preferred over top-k for language generation?

**11.** You run the same extraction prompt 10 times with temperature=0.7 and get different answers each time. Is this a bug? What's the simplest fix?

**12.** A 70B parameter model in FP32 requires approximately how much VRAM? What quantization level brings this to a single A100 (80 GB)?

**13.** What is the "lost in the middle" problem in long-context models?

**14.** You ask an LLM to compute `(policy_premium * 1.15) + admin_fee` for 10,000 policies. Why is this a bad idea, and what should you do instead?

**15.** What is Constitutional AI (CAI), and how does it differ from standard RLHF?

**16.** When would you choose `claude-haiku-4-5-20251001` over `claude-sonnet-4-6` for an AESF task?

**17.** Explain what GGUF and AWQ are. Which would you use for CPU inference on a developer workstation?

**18.** A fine-tuned model for insurance text returns better results on your benchmark but your team says it's too costly to maintain. What's the alternative, and when is fine-tuning actually worth it?

**19.** You're building an Epic sync conflict classifier in your FastAPI middleware. The LLM sometimes returns unexpected text instead of your expected labels. How should you handle this defensively?

**20.** What are two tasks where you should almost never rely on LLM output without a validation layer, in the context of the AESF system?

---

### Answers

??? note "Reveal Answers"

    **1.** A language model takes a sequence of tokens as input and outputs a **probability distribution over all possible next tokens**. That's the primitive operation—everything else (answering questions, writing code, reasoning) emerges from doing this prediction accurately across diverse text at massive scale.

    **2.** **Multi-head self-attention** routes information: every token updates its representation by attending to (taking a weighted average of) all other tokens in the sequence. This lets the model capture long-range dependencies and relationships. The **feed-forward network (FFN)** synthesizes information: each token position passes through an independent MLP that applies learned transformations. Many researchers believe the FFN layers are where factual "knowledge" is stored, while attention is the routing mechanism.

    **3.** Approximately 11–14 tokens (using cl100k_base BPE). Yes, this matters for cost estimation: LLM APIs charge per token, so high-volume processing of verbose JSON payloads can be significantly more expensive than estimating by character count alone. Build a token estimation step into your cost modeling.

    **4.** **Temperature 0.0** (or very close to it, like 0.1). Structured extraction requires consistency and accuracy, not creativity. A higher temperature introduces variance—you might get different field values on re-runs of the same document. Determinism is a first-class requirement for any data pipeline.

    **5.** Several problems: (a) it's expensive—5,000 lines could be 10,000+ tokens per request, multiplied by every call; (b) the "lost in the middle" problem means the model may not reliably attend to schema details buried in the middle; (c) it adds latency; (d) large prompts consume context window that could be used for output. Use retrieval (RAG) to fetch only the relevant schema portions at query time.

    **6.** A **base model** has only been pre-trained on next-token prediction and behaves like a text completion engine—it will "complete" a question with more questions rather than answer it. An **instruction-tuned model** has been further trained via SFT on (prompt, ideal response) pairs and follows user instructions, answers questions directly, and uses a helpful conversational format.

    **7.** RLHF (Reinforcement Learning from Human Feedback) solves the problem that "instruction-following" doesn't equal "aligned with human preferences." Human raters compare pairs of model outputs and choose the better one; this data trains a reward model; then the language model is updated via RL (PPO) to maximize reward model scores. The result is a model that responds in ways humans prefer: more helpful, less harmful, better at refusing dangerous requests.

    **8.** A "sycophantic" model agrees with the user even when the user is wrong, validates flawed reasoning to avoid disagreement, and changes its answers when the user expresses displeasure rather than when given new evidence. RLHF causes this because human raters tend to prefer responses that feel agreeable, validating, and flattering—the reward model learns to optimize for agreeableness, which correlates with but isn't identical to correctness. Well-tuned models (like Claude) are trained to resist this via Constitutional AI principles.

    **9.** It approximately equals `embedding("Berlin")`. This demonstrates that embedding spaces encode **relational structure**: the vector difference between a capital city and its country is consistent across countries. This is the arithmetic that makes embeddings useful for analogy, clustering, and semantic search—words with similar meanings or roles occupy geometrically similar regions of the space.

    **10.** Top-p keeps only the smallest set of tokens whose **cumulative probability exceeds p** (e.g., 0.9), then samples from that set. It's preferred over top-k because it adapts dynamically: when the model is confident, it selects from a small set; when uncertain, it expands. Top-k is a fixed number regardless of the probability distribution's shape, which means it can either be too restrictive (when the model is uncertain) or too permissive (when it's highly confident).

    **11.** Not a bug—this is expected behavior. With temperature > 0, the model samples from a distribution, so outputs vary between runs. The simplest fix: set `temperature=0.0` to use greedy decoding (deterministic). For critical workflows in AESF, determinism is usually more important than output diversity.

    **12.** 70B × 4 bytes = **280 GB** in FP32. To fit in a single A100 (80 GB), you need **INT4 quantization** (70B × 0.5 bytes = 35 GB), which leaves ample headroom. INT8 would require 70 GB, right at the A100's limit with no room for activations; INT4 is the practical choice.

    **13.** When processing very long contexts (e.g., 100K+ tokens), models reliably attend to information at the **beginning and end** of the context window but perform significantly worse at retrieving information from the **middle**. Empirical studies show recall drops by 30–50% for information placed in the middle of long contexts. For AESF, this means: put critical instructions at the top of your system prompt, not buried after pages of schema.

    **14.** Bad idea because **digits are individual tokens**—the model has never seen `1234567.89` as a meaningful numeric unit. It processes digit sequences as tokens, not as numbers, and arithmetic errors compound through billions of weight multiplications. The fix: do all arithmetic in Python (or SQL), pass the computed result to the LLM in the prompt. "The calculated premium is $1,418.25" is fine; "compute the premium as X * 1.15 + Y" is not.

    **15.** **Constitutional AI (CAI)** trains the model against a written set of principles (a "constitution") rather than direct human preference ratings alone. A separate model (or the same model) critiques responses against these principles and generates synthetic feedback that guides RL training. The key difference: RLHF requires human annotators for every preference pair; CAI can scale synthetic feedback generation, reducing human annotation costs and enabling more consistent application of principles. Claude models use CAI as their primary alignment technique.

    **16.** Choose **Haiku** for high-volume, low-complexity tasks where speed and cost matter: routing/classification decisions, field extraction from structured text, quick triage of sync conflicts, generating short summaries. Choose Sonnet when the task requires nuanced reasoning, multi-step synthesis, or complex code generation. The cost difference (~5×) makes a large real-world difference when processing thousands of Epic sync events per hour.

    **17.** **GGUF** is a file format for storing quantized model weights optimized for **CPU inference via llama.cpp**. It supports mixed-precision quantization and is the dominant format for running models on developer workstations without a GPU. **AWQ (Activation-aware Weight Quantization)** is a GPU-oriented quantization technique that identifies and protects the most "salient" weights from aggressive quantization, yielding better quality than GPTQ at the same bit-width. For CPU inference on a dev workstation: **GGUF** with llama.cpp is the right choice.

    **18.** The alternative is **better prompt engineering**: few-shot examples, structured system prompts, chain-of-thought reasoning. Prompt engineering should be exhausted before fine-tuning because it's free, reversible, and benefits from model version upgrades automatically. Fine-tuning is worth it when: (a) you have 1,000+ high-quality labeled examples; (b) the task is high-volume enough to justify ongoing maintenance; (c) you've measured a clear performance gap that prompting cannot close; (d) latency/cost constraints require a smaller model to match a larger model's quality.

    **19.** Validate defensively: parse the raw LLM output through an enum or allowlist, and treat any unexpected value as a special case (e.g., route to `MANUAL_REVIEW`). Never use `raw_output in ["sync", "skip"]` comparisons for routing decisions; use typed enums with a try/except. Log every raw LLM response with its prompt to a separate table so you can audit classification decisions. Consider adding a confidence signal: if the model's output needs heavy sanitizing before it matches your enum, that's a signal the prompt needs refinement.

    **20.** Two critical cases: **(a) Epic sync decisions** — whether to create, update, or delete records in Epic based on a Salesforce change. A wrong decision creates orphaned records, broken relationships, or data loss that's difficult to reconcile. Always validate with business rules in Python after any LLM classification. **(b) Policy financial data** — premium amounts, limit values, deductible figures. LLMs hallucinate numbers, make arithmetic errors, and may silently "fill in" missing values with plausible-sounding ones. Financial figures must come from the database, not from LLM inference.
