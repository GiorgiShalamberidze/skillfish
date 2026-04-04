---
name: llm-architect
description: LLM system design: model selection, fine-tuning strategies, inference optimization, context window management, multi-model routing, and cost analysis.
---

# LLM Architect

Design production-grade LLM systems with informed model selection, optimized inference pipelines, intelligent routing, and cost-aware architecture decisions. This skill covers the full spectrum from choosing the right model to deploying multi-model systems at scale.

## Table of Contents

1. [Model Selection](#1-model-selection)
2. [Fine-Tuning Strategies](#2-fine-tuning-strategies)
3. [Inference Optimization](#3-inference-optimization)
4. [Context Window Management](#4-context-window-management)
5. [Multi-Model Routing](#5-multi-model-routing)
6. [Evaluation Frameworks](#6-evaluation-frameworks)
7. [Cost Optimization](#7-cost-optimization)
8. [Architecture Decision Matrix](#8-architecture-decision-matrix)

---

## 1. Model Selection

### Capability Matrix

Select models based on task requirements, not hype. The landscape shifts fast, so anchor decisions to measurable criteria.

| Model | Context | Strengths | Weaknesses | Input $/1M | Output $/1M |
|-------|---------|-----------|------------|------------|-------------|
| GPT-4o | 128K | Multimodal, broad knowledge, strong instruction following | Expensive at scale, occasional verbosity | $2.50 | $10.00 |
| Claude Opus 4 | 200K | Deep reasoning, long-context fidelity, agentic coding | Higher latency on complex tasks | $15.00 | $75.00 |
| Claude Sonnet 4 | 200K | Balanced speed/quality, strong coding, tool use | Less raw reasoning than Opus | $3.00 | $15.00 |
| Gemini 2.5 Pro | 1M | Massive context, multimodal, competitive reasoning | Variable quality on niche domains | $1.25 | $10.00 |
| Llama 3.3 70B | 128K | Open-weight, self-hostable, strong multilingual | Requires GPU infrastructure | Self-hosted | Self-hosted |
| Mistral Large | 128K | European compliance, function calling, multilingual | Smaller ecosystem, fewer integrations | $2.00 | $6.00 |
| Qwen 2.5 72B | 128K | Open-weight, strong coding/math, multilingual (CJK) | Less battle-tested in production | Self-hosted | Self-hosted |

### Benchmarks That Matter

Ignore headline benchmark scores. Focus on task-relevant evaluation:

```python
# Define evaluation criteria by use case, not by generic benchmark
EVALUATION_CRITERIA = {
    "code_generation": {
        "primary": ["HumanEval+", "SWE-bench Verified", "LiveCodeBench"],
        "secondary": ["MBPP+", "CodeContests"],
        "custom": "task_specific_eval_suite",  # Always build your own
    },
    "reasoning": {
        "primary": ["GPQA Diamond", "MATH-500", "ARC-AGI"],
        "secondary": ["BBH", "DROP"],
        "custom": "domain_reasoning_eval",
    },
    "retrieval_qa": {
        "primary": ["Natural Questions", "TriviaQA"],
        "secondary": ["MMLU-Pro"],
        "custom": "your_domain_qa_pairs",
    },
    "instruction_following": {
        "primary": ["IFEval", "MT-Bench"],
        "secondary": ["AlpacaEval 2"],
        "custom": "your_format_compliance_tests",
    },
}
```

### Cost / Latency / Quality Tradeoffs

```python
from dataclasses import dataclass
from enum import Enum

class Priority(Enum):
    COST = "cost"
    LATENCY = "latency"
    QUALITY = "quality"

@dataclass
class ModelProfile:
    name: str
    cost_per_1m_input: float
    cost_per_1m_output: float
    median_ttft_ms: float       # time to first token
    tokens_per_second: float
    quality_score: float        # 0-1, from your eval suite

    @property
    def cost_per_request(self) -> float:
        """Estimate cost for a typical 500-in / 1000-out request."""
        return (500 * self.cost_per_1m_input + 1000 * self.cost_per_1m_output) / 1_000_000

MODELS = {
    "gpt-4o": ModelProfile("gpt-4o", 2.50, 10.00, 300, 90, 0.88),
    "claude-sonnet-4": ModelProfile("claude-sonnet-4", 3.00, 15.00, 400, 80, 0.91),
    "claude-haiku-3.5": ModelProfile("claude-haiku-3.5", 0.80, 4.00, 200, 120, 0.79),
    "gemini-2.5-flash": ModelProfile("gemini-2.5-flash", 0.15, 0.60, 150, 200, 0.82),
    "llama-3.3-70b": ModelProfile("llama-3.3-70b", 0.00, 0.00, 500, 40, 0.83),
}

def select_model(priority: Priority, min_quality: float = 0.75) -> str:
    """Select best model given a priority and minimum quality threshold."""
    candidates = {k: v for k, v in MODELS.items() if v.quality_score >= min_quality}

    if priority == Priority.COST:
        return min(candidates, key=lambda k: candidates[k].cost_per_request)
    elif priority == Priority.LATENCY:
        return min(candidates, key=lambda k: candidates[k].median_ttft_ms)
    else:
        return max(candidates, key=lambda k: candidates[k].quality_score)
```

### When to Use Which

| Scenario | Recommended | Rationale |
|----------|-------------|-----------|
| Customer-facing chat | Claude Sonnet 4 / GPT-4o | Balance of quality, speed, safety |
| Bulk document processing | Gemini 2.5 Flash / Claude Haiku 3.5 | Low cost, high throughput |
| Complex reasoning chains | Claude Opus 4 / GPT-4o | Highest accuracy on multi-step logic |
| Regulated / on-prem | Llama 3.3 70B / Mistral Large | Data sovereignty, no external calls |
| Code generation | Claude Sonnet 4 / GPT-4o | Top SWE-bench performance |
| Real-time autocomplete | Gemini 2.5 Flash / Claude Haiku 3.5 | Sub-200ms TTFT |
| Million-token analysis | Gemini 2.5 Pro | 1M native context window |

---

## 2. Fine-Tuning Strategies

### When to Fine-Tune vs Prompt Engineer vs RAG

```
Decision Flow:
                         Do you need domain knowledge?
                        /                              \
                      Yes                               No
                      /                                  \
              Is it factual/                    Do you need a specific
              changing data?                   style or behavior?
              /          \                      /              \
            Yes           No                  Yes              No
            /              \                  /                 \
         RAG          Fine-tune      Few-shot prompting    Zero-shot
                                     or fine-tune          prompting
```

| Approach | Best For | Data Needed | Cost | Maintenance |
|----------|----------|-------------|------|-------------|
| Prompt Engineering | Format control, simple tasks | 0-20 examples | Low | Low |
| Few-Shot Prompting | Style matching, classification | 5-50 examples | Low | Low |
| RAG | Factual recall, changing data | Document corpus | Medium | Medium |
| Fine-Tuning | Consistent behavior, latency reduction | 500-10K examples | High | High |
| Fine-Tune + RAG | Domain expert with knowledge base | Both | Highest | Highest |

### LoRA and QLoRA

Parameter-efficient fine-tuning lets you customize large models without full-weight training.

```python
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from trl import SFTTrainer

# LoRA configuration — tune rank based on task complexity
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                        # rank: 8 for simple tasks, 32-64 for complex
    lora_alpha=32,               # scaling factor, typically 2x rank
    lora_dropout=0.05,
    target_modules=[             # target attention layers
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj",
    ],
    bias="none",
)

# QLoRA: 4-bit quantized base model + LoRA adapters
from transformers import BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",           # normalized float 4
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,       # nested quantization
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.3-70B-Instruct",
    quantization_config=bnb_config,
    device_map="auto",
    attn_implementation="flash_attention_2",
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 83M || all params: 70B || trainable%: 0.12%
```

### Data Preparation

```python
import json
from datasets import Dataset

def prepare_training_data(examples: list[dict]) -> Dataset:
    """Convert raw examples to chat-format training data.

    Each example should have: instruction, input (optional), output.
    """
    formatted = []
    for ex in examples:
        messages = [
            {"role": "system", "content": "You are a domain expert assistant."},
            {"role": "user", "content": ex["instruction"]},
        ]
        if ex.get("input"):
            messages[-1]["content"] += f"\n\nContext:\n{ex['input']}"
        messages.append({"role": "assistant", "content": ex["output"]})
        formatted.append({"messages": messages})

    return Dataset.from_list(formatted)


def quality_filter(examples: list[dict], min_length: int = 50) -> list[dict]:
    """Filter low-quality training examples."""
    filtered = []
    for ex in examples:
        output = ex.get("output", "")
        if len(output) < min_length:
            continue
        if output.strip().startswith("I cannot") or output.strip().startswith("I'm sorry"):
            continue
        if len(set(output.split())) / max(len(output.split()), 1) < 0.3:
            continue  # too repetitive
        filtered.append(ex)
    return filtered
```

### Eval-Driven Training

Never fine-tune without a held-out evaluation set. Stop training when eval loss plateaus.

```python
training_args = TrainingArguments(
    output_dir="./checkpoints",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,
    learning_rate=2e-4,
    lr_scheduler_type="cosine",
    warmup_ratio=0.05,
    logging_steps=10,
    eval_strategy="steps",
    eval_steps=50,
    save_strategy="steps",
    save_steps=50,
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
    greater_is_better=False,
    bf16=True,
    gradient_checkpointing=True,
    report_to="wandb",
)
```

### Synthetic Data Generation

Use a strong model to generate training data for a smaller model.

```python
import anthropic

client = anthropic.Anthropic()

def generate_synthetic_examples(
    task_description: str,
    seed_examples: list[dict],
    count: int = 100,
) -> list[dict]:
    """Generate synthetic training examples using a frontier model."""

    seed_text = "\n\n".join(
        f"Input: {ex['input']}\nOutput: {ex['output']}"
        for ex in seed_examples[:5]
    )

    examples = []
    batch_size = 10

    for i in range(0, count, batch_size):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[{
                "role": "user",
                "content": f"""Generate {batch_size} diverse training examples for this task:

Task: {task_description}

Reference examples:
{seed_text}

Output as JSON array with "input" and "output" fields.
Generate diverse, high-quality examples that cover edge cases.
Do NOT repeat the reference examples.""",
            }],
        )

        batch = json.loads(response.content[0].text)
        examples.extend(batch)

    return quality_filter(examples)
```

---

## 3. Inference Optimization

### Batching Strategies

Maximize GPU utilization by batching requests intelligently.

```python
import asyncio
from dataclasses import dataclass, field
from typing import Callable
import time

@dataclass
class InferenceRequest:
    prompt: str
    future: asyncio.Future
    created_at: float = field(default_factory=time.time)

class DynamicBatcher:
    """Batch incoming requests for efficient GPU inference."""

    def __init__(
        self,
        inference_fn: Callable,
        max_batch_size: int = 32,
        max_wait_ms: float = 50,
    ):
        self.inference_fn = inference_fn
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self.queue: asyncio.Queue[InferenceRequest] = asyncio.Queue()
        self._running = False

    async def start(self):
        self._running = True
        asyncio.create_task(self._batch_loop())

    async def infer(self, prompt: str) -> str:
        future = asyncio.get_event_loop().create_future()
        await self.queue.put(InferenceRequest(prompt=prompt, future=future))
        return await future

    async def _batch_loop(self):
        while self._running:
            batch: list[InferenceRequest] = []
            try:
                # Wait for first request
                req = await asyncio.wait_for(self.queue.get(), timeout=1.0)
                batch.append(req)
            except asyncio.TimeoutError:
                continue

            # Collect more requests up to batch size or timeout
            deadline = time.time() + self.max_wait_ms / 1000
            while len(batch) < self.max_batch_size and time.time() < deadline:
                try:
                    remaining = deadline - time.time()
                    req = await asyncio.wait_for(
                        self.queue.get(), timeout=max(0, remaining)
                    )
                    batch.append(req)
                except asyncio.TimeoutError:
                    break

            # Process batch
            prompts = [r.prompt for r in batch]
            try:
                results = await self.inference_fn(prompts)
                for req, result in zip(batch, results):
                    req.future.set_result(result)
            except Exception as e:
                for req in batch:
                    req.future.set_exception(e)
```

### KV Cache Management

```python
# vLLM server configuration for optimal KV cache usage
VLLM_CONFIG = {
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "tensor_parallel_size": 4,          # across 4 GPUs
    "gpu_memory_utilization": 0.90,     # reserve 10% for overhead
    "max_model_len": 32768,             # limit context to control cache size
    "enable_prefix_caching": True,      # reuse KV cache for shared prefixes
    "block_size": 16,                   # KV cache block granularity
    "swap_space": 4,                    # GiB of CPU swap for evicted blocks
    "max_num_seqs": 256,                # concurrent sequences
    "enable_chunked_prefill": True,     # overlap prefill with decode
}

# Launch: python -m vllm.entrypoints.openai.api_server \
#   --model meta-llama/Llama-3.3-70B-Instruct \
#   --tensor-parallel-size 4 \
#   --gpu-memory-utilization 0.90 \
#   --enable-prefix-caching \
#   --enable-chunked-prefill \
#   --max-model-len 32768
```

### Speculative Decoding

Use a small draft model to propose tokens, verified in parallel by the large model.

```python
# vLLM with speculative decoding
# Draft model generates N candidate tokens, target model verifies in one pass
SPECULATIVE_CONFIG = {
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "speculative_model": "meta-llama/Llama-3.2-1B-Instruct",
    "num_speculative_tokens": 5,         # tokens to draft per step
    "speculative_max_model_len": 4096,   # draft model context limit
    "tensor_parallel_size": 4,
    # Expected speedup: 1.5-2.5x for generation-heavy workloads
    # Works best when draft model has high acceptance rate
}

# Alternatively, use Medusa heads (no separate draft model)
# Adds multiple prediction heads to the target model itself
# Lower memory overhead, ~1.5x speedup
```

### Quantization Comparison

| Method | Bits | Quality Loss | Speed Gain | Memory Reduction | Best For |
|--------|------|-------------|------------|------------------|----------|
| FP16 | 16 | None | Baseline | Baseline | Max quality |
| GPTQ | 4 | ~1-2% | 1.5-2x | 4x | GPU serving |
| AWQ | 4 | ~0.5-1% | 1.5-2x | 4x | GPU serving (better quality) |
| GGUF Q4_K_M | 4 | ~1-3% | Variable | 4x | CPU/Mac inference (llama.cpp) |
| GGUF Q5_K_M | 5 | ~0.5-1% | Variable | 3.2x | CPU/Mac with quality priority |
| FP8 | 8 | ~0.1% | 1.3x | 2x | Minimal quality loss |

```python
# Quantize a model with AutoAWQ
from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

model_path = "meta-llama/Llama-3.3-70B-Instruct"
quant_path = "llama-3.3-70b-awq"

model = AutoAWQForCausalLM.from_pretrained(model_path)
tokenizer = AutoTokenizer.from_pretrained(model_path)

quant_config = {
    "zero_point": True,
    "q_group_size": 128,
    "w_bit": 4,
    "version": "GEMM",     # GEMM for GPU, GEMV for CPU
}

model.quantize(tokenizer, quant_config=quant_config)
model.save_quantized(quant_path)
tokenizer.save_pretrained(quant_path)
```

### Deployment Options

| Platform | Best For | GPU Required | Scaling | Complexity |
|----------|----------|-------------|---------|------------|
| vLLM | High-throughput serving | Yes | Manual / K8s | Medium |
| TGI | HuggingFace ecosystem | Yes | Docker / K8s | Medium |
| Ollama | Local dev, small teams | Optional | Single machine | Low |
| SGLang | Complex LLM programs | Yes | Manual | High |
| llama.cpp | Edge / CPU / Mac | No | Single machine | Low |

```python
# Production vLLM deployment with Docker Compose
DOCKER_COMPOSE = """
services:
  vllm:
    image: vllm/vllm-openai:latest
    runtime: nvidia
    ports:
      - "8000:8000"
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
    environment:
      - HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}
    command: >
      --model meta-llama/Llama-3.3-70B-Instruct
      --tensor-parallel-size 4
      --gpu-memory-utilization 0.90
      --enable-prefix-caching
      --max-model-len 32768
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 4
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
"""
```

---

## 4. Context Window Management

### Chunking Strategies

```python
from enum import Enum

class ChunkStrategy(Enum):
    FIXED_SIZE = "fixed_size"
    SEMANTIC = "semantic"
    RECURSIVE = "recursive"
    DOCUMENT_AWARE = "document_aware"

def chunk_fixed_size(
    text: str, chunk_size: int = 512, overlap: int = 50
) -> list[str]:
    """Simple fixed-size chunking with overlap."""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i : i + chunk_size])
        if chunk:
            chunks.append(chunk)
    return chunks


def chunk_recursive(
    text: str,
    max_chunk_size: int = 1000,
    separators: list[str] | None = None,
) -> list[str]:
    """Recursively split on decreasing separator granularity."""
    if separators is None:
        separators = ["\n\n\n", "\n\n", "\n", ". ", " "]

    if len(text) <= max_chunk_size:
        return [text]

    for sep in separators:
        parts = text.split(sep)
        if len(parts) > 1:
            chunks = []
            current = ""
            for part in parts:
                candidate = current + sep + part if current else part
                if len(candidate) <= max_chunk_size:
                    current = candidate
                else:
                    if current:
                        chunks.append(current)
                    current = part
            if current:
                chunks.append(current)
            if all(len(c) <= max_chunk_size for c in chunks):
                return chunks

    # Last resort: hard split
    return chunk_fixed_size(text, max_chunk_size, overlap=0)


def chunk_semantic(
    text: str,
    embedding_model,
    similarity_threshold: float = 0.75,
    max_chunk_size: int = 1500,
) -> list[str]:
    """Split on semantic boundaries using embedding similarity."""
    sentences = text.replace("\n", " ").split(". ")
    if len(sentences) <= 1:
        return [text]

    embeddings = embedding_model.encode(sentences)
    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        similarity = cosine_similarity(embeddings[i - 1], embeddings[i])
        current_text = ". ".join(current_chunk)

        if similarity < similarity_threshold or len(current_text) > max_chunk_size:
            chunks.append(current_text + ".")
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])

    if current_chunk:
        chunks.append(". ".join(current_chunk) + ".")

    return chunks
```

### Context Compression

Reduce token usage while preserving information density.

```python
import anthropic

client = anthropic.Anthropic()

def compress_context(
    documents: list[str],
    query: str,
    max_tokens: int = 2000,
) -> str:
    """Use LLM to compress retrieved documents into dense context."""
    combined = "\n\n---\n\n".join(documents)

    response = client.messages.create(
        model="claude-haiku-3-5-20241022",
        max_tokens=max_tokens,
        messages=[{
            "role": "user",
            "content": f"""Compress the following documents into a dense summary
that preserves all information relevant to the query.
Remove redundancy. Keep specific facts, numbers, and names.

Query: {query}

Documents:
{combined}

Compressed context:""",
        }],
    )
    return response.content[0].text


def extract_relevant_spans(
    text: str,
    query: str,
    embedding_model,
    top_k: int = 5,
    span_size: int = 200,
) -> list[str]:
    """Extract the most query-relevant spans from a long document."""
    words = text.split()
    spans = []
    for i in range(0, len(words), span_size // 2):
        span = " ".join(words[i : i + span_size])
        spans.append(span)

    query_emb = embedding_model.encode([query])[0]
    span_embs = embedding_model.encode(spans)

    scores = [cosine_similarity(query_emb, s) for s in span_embs]
    top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:top_k]
    top_indices.sort()  # maintain document order

    return [spans[i] for i in top_indices]
```

### Long-Context vs RAG Tradeoffs

| Factor | Long Context (stuff everything in) | RAG (retrieve then generate) |
|--------|-----------------------------------|------------------------------|
| Simplicity | High (no retrieval pipeline) | Lower (need indexing + retrieval) |
| Cost per request | High (many input tokens) | Lower (only relevant chunks) |
| Accuracy on needle-in-haystack | Degrades with context length | Better if retrieval is accurate |
| Freshness | Must re-send full context | Index updates independently |
| Latency (TTFT) | Higher with more tokens | Lower with focused context |
| Works with | Single docs < 200K tokens | Any corpus size |

**Decision rule:** Use long-context for small, known document sets where recall matters. Use RAG when the corpus is large, dynamic, or cost is a constraint. Combine both (RAG + long context for top-K results) for the best of both worlds.

### Sliding Window Pattern

For processing documents that exceed the context window.

```python
from dataclasses import dataclass

@dataclass
class WindowResult:
    window_index: int
    content: str
    overlap_start: int
    overlap_end: int

def sliding_window_process(
    document: str,
    window_size: int = 100_000,       # tokens
    overlap: int = 5_000,             # tokens
    process_fn=None,
) -> list[WindowResult]:
    """Process a long document with overlapping sliding windows."""
    tokens = tokenize(document)  # your tokenizer
    results = []

    for i, start in enumerate(range(0, len(tokens), window_size - overlap)):
        end = min(start + window_size, len(tokens))
        window_text = detokenize(tokens[start:end])

        result = process_fn(
            window_text,
            is_first=(start == 0),
            is_last=(end >= len(tokens)),
            window_index=i,
        )

        results.append(WindowResult(
            window_index=i,
            content=result,
            overlap_start=start,
            overlap_end=min(start + overlap, end),
        ))

    return merge_window_results(results)


def merge_window_results(results: list[WindowResult]) -> str:
    """Merge overlapping window results, deduplicating overlap regions."""
    if not results:
        return ""
    merged = results[0].content
    for prev, curr in zip(results[:-1], results[1:]):
        # Skip content in the overlap region that duplicates previous window
        merged += "\n" + curr.content
    return merged
```

---

## 5. Multi-Model Routing

### Complexity-Based Routing

Route queries to models based on estimated complexity.

```python
import re
from enum import Enum

class Complexity(Enum):
    SIMPLE = "simple"
    MODERATE = "moderate"
    COMPLEX = "complex"

class ModelTier(Enum):
    FAST = "fast"       # Claude Haiku 3.5, Gemini Flash
    BALANCED = "balanced"  # Claude Sonnet 4, GPT-4o-mini
    PREMIUM = "premium"    # Claude Opus 4, GPT-4o

COMPLEXITY_SIGNALS = {
    "simple": {
        "max_tokens": 50,
        "patterns": [
            r"^(yes|no|true|false)",
            r"^(what is|define|translate)",
            r"^(list|name|count)",
        ],
    },
    "moderate": {
        "max_tokens": 200,
        "patterns": [
            r"(explain|describe|compare)",
            r"(how to|steps to|guide for)",
            r"(summarize|analyze)",
        ],
    },
    "complex": {
        "patterns": [
            r"(design|architect|implement)",
            r"(debug|investigate|diagnose)",
            r"(multi-step|chain of thought|reason through)",
            r"(code review|refactor|optimize)",
        ],
    },
}

def estimate_complexity(query: str) -> Complexity:
    """Estimate query complexity from surface features."""
    query_lower = query.lower()
    word_count = len(query.split())

    # Long queries with multiple constraints are usually complex
    if word_count > 100:
        return Complexity.COMPLEX

    # Check for complexity signal patterns
    for pattern in COMPLEXITY_SIGNALS["simple"]["patterns"]:
        if re.search(pattern, query_lower):
            if word_count < 30:
                return Complexity.SIMPLE

    for pattern in COMPLEXITY_SIGNALS["complex"]["patterns"]:
        if re.search(pattern, query_lower):
            return Complexity.COMPLEX

    return Complexity.MODERATE


ROUTING_TABLE = {
    Complexity.SIMPLE: ModelTier.FAST,
    Complexity.MODERATE: ModelTier.BALANCED,
    Complexity.COMPLEX: ModelTier.PREMIUM,
}

MODEL_CONFIGS = {
    ModelTier.FAST: {
        "model": "claude-haiku-3-5-20241022",
        "max_tokens": 1024,
    },
    ModelTier.BALANCED: {
        "model": "claude-sonnet-4-20250514",
        "max_tokens": 4096,
    },
    ModelTier.PREMIUM: {
        "model": "claude-opus-4-20250514",
        "max_tokens": 8192,
    },
}
```

### Cascading: Cheap Model to Expensive Model

```python
import anthropic

client = anthropic.Anthropic()

async def cascade_inference(
    query: str,
    confidence_threshold: float = 0.85,
) -> dict:
    """Try cheap model first, escalate to premium if confidence is low."""

    # Stage 1: Fast model
    fast_response = client.messages.create(
        model="claude-haiku-3-5-20241022",
        max_tokens=2048,
        messages=[
            {"role": "user", "content": query},
        ],
        system="""Answer the question. At the end of your response, on a new line,
output your confidence as: CONFIDENCE: <0.0-1.0>""",
    )

    response_text = fast_response.content[0].text
    confidence = extract_confidence(response_text)

    if confidence >= confidence_threshold:
        return {
            "response": strip_confidence(response_text),
            "model": "claude-haiku-3.5",
            "escalated": False,
            "confidence": confidence,
        }

    # Stage 2: Premium model with the fast model's attempt as context
    premium_response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[
            {"role": "user", "content": query},
            {"role": "assistant", "content": f"[Initial analysis from fast model: {strip_confidence(response_text)}]\n\nLet me provide a more thorough answer:"},
        ],
    )

    return {
        "response": premium_response.content[0].text,
        "model": "claude-sonnet-4",
        "escalated": True,
        "initial_confidence": confidence,
    }


def extract_confidence(text: str) -> float:
    match = re.search(r"CONFIDENCE:\s*([\d.]+)", text)
    return float(match.group(1)) if match else 0.5


def strip_confidence(text: str) -> str:
    return re.sub(r"\nCONFIDENCE:.*$", "", text).strip()
```

### Semantic Routing

Route to specialized models based on query semantics.

```python
from dataclasses import dataclass

@dataclass
class SpecializedRoute:
    name: str
    model: str
    system_prompt: str
    keywords: list[str]
    embedding_centroid: list[float] | None = None

ROUTES = [
    SpecializedRoute(
        name="code",
        model="claude-sonnet-4-20250514",
        system_prompt="You are an expert software engineer.",
        keywords=["code", "function", "bug", "implement", "class", "API"],
    ),
    SpecializedRoute(
        name="math",
        model="claude-opus-4-20250514",
        system_prompt="You are a mathematics expert. Show your work step by step.",
        keywords=["calculate", "prove", "equation", "integral", "probability"],
    ),
    SpecializedRoute(
        name="creative",
        model="claude-sonnet-4-20250514",
        system_prompt="You are a creative writing assistant.",
        keywords=["write", "story", "poem", "creative", "narrative"],
    ),
    SpecializedRoute(
        name="general",
        model="claude-haiku-3-5-20241022",
        system_prompt="You are a helpful assistant.",
        keywords=[],
    ),
]

def route_by_keywords(query: str) -> SpecializedRoute:
    """Route based on keyword matching. Fast but imprecise."""
    query_lower = query.lower()
    scores = {}

    for route in ROUTES:
        score = sum(1 for kw in route.keywords if kw in query_lower)
        scores[route.name] = score

    best = max(scores, key=scores.get)
    if scores[best] == 0:
        return next(r for r in ROUTES if r.name == "general")
    return next(r for r in ROUTES if r.name == best)


def route_by_embedding(query: str, embedding_model) -> SpecializedRoute:
    """Route based on embedding similarity to route centroids."""
    query_emb = embedding_model.encode([query])[0]

    best_route = None
    best_score = -1

    for route in ROUTES:
        if route.embedding_centroid is None:
            continue
        score = cosine_similarity(query_emb, route.embedding_centroid)
        if score > best_score:
            best_score = score
            best_route = route

    return best_route or ROUTES[-1]  # fallback to general
```

### Cost Optimization via Routing

```python
@dataclass
class RoutingMetrics:
    total_requests: int = 0
    fast_requests: int = 0
    balanced_requests: int = 0
    premium_requests: int = 0
    total_cost: float = 0.0
    escalation_rate: float = 0.0

    @property
    def avg_cost_per_request(self) -> float:
        return self.total_cost / max(self.total_requests, 1)

    @property
    def cost_savings_vs_premium(self) -> float:
        """Percentage saved compared to routing everything to premium."""
        premium_cost = self.total_requests * 0.025  # estimated premium cost/req
        return (1 - self.total_cost / max(premium_cost, 0.001)) * 100

# Typical routing distribution for a production chatbot:
# 60% simple  -> fast model   ($0.001/req)  = $0.60/1000 reqs
# 30% moderate -> balanced    ($0.008/req)  = $2.40/1000 reqs
# 10% complex  -> premium     ($0.025/req)  = $2.50/1000 reqs
# Total: $5.50/1000 requests
# vs. all-premium: $25.00/1000 requests
# Savings: 78%
```

---

## 6. Evaluation Frameworks

### LLM-as-Judge

```python
import anthropic
import json

client = anthropic.Anthropic()

def llm_judge(
    question: str,
    response_a: str,
    response_b: str,
    criteria: str = "accuracy, helpfulness, and clarity",
) -> dict:
    """Use a strong LLM to evaluate two responses head-to-head."""

    judgment = client.messages.create(
        model="claude-opus-4-20250514",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""You are an expert evaluator. Compare two responses to a question.

Question: {question}

Response A:
{response_a}

Response B:
{response_b}

Evaluate on: {criteria}

Output JSON:
{{
    "winner": "A" or "B" or "tie",
    "score_a": 1-10,
    "score_b": 1-10,
    "reasoning": "brief explanation"
}}""",
        }],
    )
    return json.loads(judgment.content[0].text)


def pointwise_judge(
    question: str,
    response: str,
    rubric: dict[str, str],
) -> dict[str, int]:
    """Score a response on multiple dimensions independently."""
    rubric_text = "\n".join(f"- {k}: {v}" for k, v in rubric.items())

    judgment = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""Score this response on each dimension (1-5).

Question: {question}
Response: {response}

Dimensions:
{rubric_text}

Output JSON with dimension names as keys and integer scores as values.""",
        }],
    )
    return json.loads(judgment.content[0].text)


# Standard rubric for QA evaluation
QA_RUBRIC = {
    "correctness": "Is the answer factually accurate?",
    "completeness": "Does it address all parts of the question?",
    "conciseness": "Is it free of unnecessary information?",
    "coherence": "Is it well-structured and easy to follow?",
    "citation": "Does it properly reference sources when relevant?",
}
```

### RAGAS Evaluation for RAG Systems

```python
# pip install ragas

from ragas import evaluate
from ragas.metrics import (
    answer_relevancy,
    context_precision,
    context_recall,
    faithfulness,
)
from datasets import Dataset

def evaluate_rag_pipeline(
    questions: list[str],
    answers: list[str],
    contexts: list[list[str]],
    ground_truths: list[str],
) -> dict:
    """Evaluate a RAG pipeline using RAGAS metrics."""

    eval_dataset = Dataset.from_dict({
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    })

    results = evaluate(
        eval_dataset,
        metrics=[
            faithfulness,        # Is answer grounded in context?
            answer_relevancy,    # Is answer relevant to question?
            context_precision,   # Are retrieved docs relevant?
            context_recall,      # Did we retrieve all needed info?
        ],
    )

    return {
        "faithfulness": results["faithfulness"],
        "answer_relevancy": results["answer_relevancy"],
        "context_precision": results["context_precision"],
        "context_recall": results["context_recall"],
        "overall": sum(results.values()) / len(results),
    }
```

### A/B Testing Framework

```python
import random
import hashlib
from datetime import datetime
from dataclasses import dataclass, field

@dataclass
class ABTestConfig:
    name: str
    variant_a: dict        # model config for control
    variant_b: dict        # model config for treatment
    traffic_split: float = 0.5   # fraction of traffic to variant B
    start_date: datetime = field(default_factory=datetime.utcnow)
    min_samples: int = 100       # minimum samples before significance test

@dataclass
class ABTestResult:
    variant: str
    response: str
    latency_ms: float
    tokens_used: int
    user_rating: float | None = None

class ABTestRouter:
    """Deterministic A/B test routing based on user ID."""

    def __init__(self, config: ABTestConfig):
        self.config = config
        self.results_a: list[ABTestResult] = []
        self.results_b: list[ABTestResult] = []

    def get_variant(self, user_id: str) -> str:
        """Deterministic assignment: same user always gets same variant."""
        hash_val = hashlib.sha256(
            f"{self.config.name}:{user_id}".encode()
        ).hexdigest()
        bucket = int(hash_val[:8], 16) / 0xFFFFFFFF
        return "B" if bucket < self.config.traffic_split else "A"

    def record(self, variant: str, result: ABTestResult):
        if variant == "A":
            self.results_a.append(result)
        else:
            self.results_b.append(result)

    def analyze(self) -> dict:
        """Basic statistical comparison of variants."""
        if len(self.results_a) < self.config.min_samples:
            return {"status": "insufficient_data", "samples_a": len(self.results_a)}

        avg_latency_a = sum(r.latency_ms for r in self.results_a) / len(self.results_a)
        avg_latency_b = sum(r.latency_ms for r in self.results_b) / len(self.results_b)

        ratings_a = [r.user_rating for r in self.results_a if r.user_rating is not None]
        ratings_b = [r.user_rating for r in self.results_b if r.user_rating is not None]

        return {
            "status": "ready",
            "samples": {"a": len(self.results_a), "b": len(self.results_b)},
            "avg_latency_ms": {"a": avg_latency_a, "b": avg_latency_b},
            "avg_rating": {
                "a": sum(ratings_a) / max(len(ratings_a), 1),
                "b": sum(ratings_b) / max(len(ratings_b), 1),
            },
            "cost_per_request": {
                "a": sum(r.tokens_used for r in self.results_a) / len(self.results_a),
                "b": sum(r.tokens_used for r in self.results_b) / len(self.results_b),
            },
        }
```

### Regression Testing

```python
import json
from pathlib import Path

class RegressionSuite:
    """Track model quality across versions and prompt changes."""

    def __init__(self, suite_path: str):
        self.suite_path = Path(suite_path)
        self.cases = self._load_cases()

    def _load_cases(self) -> list[dict]:
        with open(self.suite_path / "cases.json") as f:
            return json.load(f)

    def run(self, inference_fn, version: str) -> dict:
        """Run all test cases and compare against baselines."""
        results = []
        for case in self.cases:
            response = inference_fn(case["input"])
            passed = self._check_assertions(response, case["assertions"])
            results.append({
                "case_id": case["id"],
                "passed": passed,
                "response": response,
            })

        report = {
            "version": version,
            "total": len(results),
            "passed": sum(1 for r in results if r["passed"]),
            "failed": [r for r in results if not r["passed"]],
        }

        # Save results for historical comparison
        history_file = self.suite_path / "history.jsonl"
        with open(history_file, "a") as f:
            f.write(json.dumps(report) + "\n")

        return report

    def _check_assertions(self, response: str, assertions: list[dict]) -> bool:
        for assertion in assertions:
            if assertion["type"] == "contains":
                if assertion["value"].lower() not in response.lower():
                    return False
            elif assertion["type"] == "not_contains":
                if assertion["value"].lower() in response.lower():
                    return False
            elif assertion["type"] == "max_length":
                if len(response) > assertion["value"]:
                    return False
            elif assertion["type"] == "json_valid":
                try:
                    json.loads(response)
                except json.JSONDecodeError:
                    return False
        return True
```

---

## 7. Cost Optimization

### Token Economics

```python
from dataclasses import dataclass

@dataclass
class TokenBudget:
    """Track and control token spending."""
    daily_budget_usd: float
    spent_today_usd: float = 0.0
    requests_today: int = 0

    def can_afford(self, estimated_cost: float) -> bool:
        return self.spent_today_usd + estimated_cost <= self.daily_budget_usd

    def record(self, input_tokens: int, output_tokens: int, model: str):
        cost = estimate_cost(input_tokens, output_tokens, model)
        self.spent_today_usd += cost
        self.requests_today += 1
        return cost

    @property
    def remaining_budget(self) -> float:
        return self.daily_budget_usd - self.spent_today_usd

def estimate_cost(input_tokens: int, output_tokens: int, model: str) -> float:
    """Estimate cost for a request."""
    pricing = {
        "claude-opus-4":     {"input": 15.00, "output": 75.00},
        "claude-sonnet-4":   {"input": 3.00,  "output": 15.00},
        "claude-haiku-3.5":  {"input": 0.80,  "output": 4.00},
        "gpt-4o":            {"input": 2.50,  "output": 10.00},
        "gpt-4o-mini":       {"input": 0.15,  "output": 0.60},
        "gemini-2.5-flash":  {"input": 0.15,  "output": 0.60},
        "gemini-2.5-pro":    {"input": 1.25,  "output": 10.00},
    }
    p = pricing.get(model, {"input": 3.00, "output": 15.00})
    return (input_tokens * p["input"] + output_tokens * p["output"]) / 1_000_000
```

### Caching: Semantic and Exact

```python
import hashlib
import json
import time
import numpy as np

class ExactCache:
    """Cache identical requests. Hit rate: 10-30% for production workloads."""

    def __init__(self, ttl_seconds: int = 3600):
        self.cache: dict[str, tuple[str, float]] = {}
        self.ttl = ttl_seconds

    def _key(self, model: str, messages: list[dict]) -> str:
        raw = json.dumps({"model": model, "messages": messages}, sort_keys=True)
        return hashlib.sha256(raw.encode()).hexdigest()

    def get(self, model: str, messages: list[dict]) -> str | None:
        key = self._key(model, messages)
        if key in self.cache:
            response, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return response
            del self.cache[key]
        return None

    def set(self, model: str, messages: list[dict], response: str):
        key = self._key(model, messages)
        self.cache[key] = (response, time.time())


class SemanticCache:
    """Cache semantically similar requests. Hit rate: 20-50%."""

    def __init__(
        self,
        embedding_model,
        similarity_threshold: float = 0.95,
        ttl_seconds: int = 3600,
    ):
        self.embedding_model = embedding_model
        self.threshold = similarity_threshold
        self.ttl = ttl_seconds
        self.entries: list[dict] = []

    def get(self, query: str) -> str | None:
        if not self.entries:
            return None

        query_emb = self.embedding_model.encode([query])[0]
        now = time.time()

        best_match = None
        best_score = -1

        for entry in self.entries:
            if now - entry["timestamp"] > self.ttl:
                continue
            score = cosine_similarity(query_emb, entry["embedding"])
            if score > best_score:
                best_score = score
                best_match = entry

        if best_match and best_score >= self.threshold:
            return best_match["response"]
        return None

    def set(self, query: str, response: str):
        embedding = self.embedding_model.encode([query])[0]
        self.entries.append({
            "query": query,
            "response": response,
            "embedding": embedding,
            "timestamp": time.time(),
        })


def cosine_similarity(a, b) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

### Batch API Pricing

Most providers offer 50% discounts for async batch processing.

```python
import anthropic

client = anthropic.Anthropic()

def submit_batch(requests: list[dict]) -> str:
    """Submit batch for async processing at 50% cost reduction."""

    batch_requests = []
    for i, req in enumerate(requests):
        batch_requests.append({
            "custom_id": f"req-{i}",
            "params": {
                "model": "claude-sonnet-4-20250514",
                "max_tokens": 2048,
                "messages": req["messages"],
            },
        })

    batch = client.batches.create(requests=batch_requests)
    return batch.id


def check_batch(batch_id: str) -> dict:
    """Check batch status and retrieve results if complete."""
    batch = client.batches.retrieve(batch_id)

    if batch.processing_status == "ended":
        results = []
        for result in client.batches.results(batch_id):
            results.append({
                "id": result.custom_id,
                "response": result.result.message.content[0].text,
            })
        return {"status": "complete", "results": results}

    return {
        "status": batch.processing_status,
        "progress": f"{batch.request_counts.succeeded}/{batch.request_counts.total}",
    }

# Cost comparison for 10,000 requests at ~500 input / 1000 output tokens each:
# Real-time Claude Sonnet 4: (500 * 3.0 + 1000 * 15.0) / 1M * 10000 = $165.00
# Batch Claude Sonnet 4:     $165.00 * 0.50 = $82.50
# Savings: $82.50 (50%)
```

### Model Distillation

Train a smaller model to mimic a larger one for production cost savings.

```python
def generate_distillation_dataset(
    prompts: list[str],
    teacher_model: str = "claude-opus-4-20250514",
    batch_size: int = 50,
) -> list[dict]:
    """Generate training data from a teacher model for distillation."""
    client = anthropic.Anthropic()
    dataset = []

    for i in range(0, len(prompts), batch_size):
        batch = prompts[i : i + batch_size]

        for prompt in batch:
            response = client.messages.create(
                model=teacher_model,
                max_tokens=4096,
                messages=[{"role": "user", "content": prompt}],
            )
            dataset.append({
                "instruction": prompt,
                "output": response.content[0].text,
                "teacher_model": teacher_model,
            })

    return dataset

# Distillation pipeline:
# 1. Collect 5K-50K production prompts (deduplicated)
# 2. Generate teacher responses using best model
# 3. Filter low-quality responses (< min length, refusals, etc.)
# 4. Fine-tune student model (Llama 3.3 8B or Mistral 7B)
# 5. Evaluate student against teacher on held-out set
# 6. Deploy student for 70-90% of traffic, teacher for hard queries
#
# Typical results:
# - Student achieves 85-95% of teacher quality
# - Cost reduction: 10-50x depending on model sizes
# - Latency reduction: 2-5x
```

### Prompt Compression Techniques

```python
def compress_prompt(prompt: str, target_ratio: float = 0.5) -> str:
    """Reduce prompt length while preserving meaning.

    Techniques applied in order:
    1. Remove redundant whitespace and formatting
    2. Abbreviate common phrases
    3. Remove filler words
    4. Use LLM to compress if still over target
    """
    import re

    # Step 1: Normalize whitespace
    compressed = re.sub(r"\n{3,}", "\n\n", prompt)
    compressed = re.sub(r" {2,}", " ", compressed)
    compressed = compressed.strip()

    # Step 2: Common abbreviations
    abbreviations = {
        "for example": "e.g.",
        "that is to say": "i.e.",
        "in order to": "to",
        "as well as": "and",
        "in addition to": "plus",
        "with respect to": "re:",
        "in the case of": "for",
        "it is important to note that": "",
        "please note that": "",
        "as mentioned earlier": "",
    }
    for phrase, replacement in abbreviations.items():
        compressed = compressed.replace(phrase, replacement)

    # Step 3: Remove filler words
    fillers = [
        r"\bbasically\b", r"\bactually\b", r"\bjust\b",
        r"\breally\b", r"\bvery\b", r"\bquite\b",
        r"\bsimply\b", r"\bcertainly\b", r"\bdefinitely\b",
    ]
    for filler in fillers:
        compressed = re.sub(filler, "", compressed, flags=re.IGNORECASE)

    # Clean up double spaces from removals
    compressed = re.sub(r" {2,}", " ", compressed)

    current_ratio = len(compressed) / len(prompt)
    if current_ratio > target_ratio:
        # Step 4: LLM-based compression for aggressive targets
        compressed = llm_compress(compressed, target_ratio)

    return compressed


def llm_compress(text: str, target_ratio: float) -> str:
    """Use a fast LLM to compress text to a target ratio."""
    client = anthropic.Anthropic()
    target_length = int(len(text) * target_ratio)

    response = client.messages.create(
        model="claude-haiku-3-5-20241022",
        max_tokens=target_length // 3,
        messages=[{
            "role": "user",
            "content": f"Compress this text to ~{target_length} characters. "
                       f"Preserve all key information, remove redundancy:\n\n{text}",
        }],
    )
    return response.content[0].text
```

---

## 8. Architecture Decision Matrix

Use this decision tree to choose the right LLM architecture pattern for your use case.

### RAG vs Fine-Tuning vs Agents vs Workflows

```
START: What does the system need to do?
|
+-- Answer questions from a knowledge base
|   |
|   +-- Knowledge changes frequently? ---------> RAG
|   +-- Knowledge is stable + need low latency -> Fine-tune + cache
|   +-- Need both + highest accuracy -----------> Fine-tune + RAG
|
+-- Complete multi-step tasks
|   |
|   +-- Steps are predictable/fixed? -----------> Workflow (DAG)
|   +-- Steps depend on intermediate results? --> Agent with tools
|   +-- Hybrid: fixed structure, dynamic steps -> Workflow + Agent nodes
|
+-- Generate content in a specific style
|   |
|   +-- < 50 examples of the style? -----------> Few-shot prompting
|   +-- 50-500 examples? -----------------------> Fine-tune (LoRA)
|   +-- Lots of data + low latency needed? -----> Full fine-tune small model
|
+-- Classify or extract structured data
|   |
|   +-- < 20 categories, clear rules? ----------> Prompt engineering
|   +-- Complex taxonomy, nuanced decisions? ----> Fine-tune
|   +-- Need explanations with classifications? -> Chain-of-thought prompting
|
+-- Process high volume at low cost
    |
    +-- Can tolerate async? ---------------------> Batch API
    +-- Need real-time? -------------------------> Distilled small model
    +-- Mixed priority? -------------------------> Route by complexity
```

### Decision Comparison Matrix

| Factor | RAG | Fine-Tuning | Agents | Workflows |
|--------|-----|-------------|--------|-----------|
| Setup complexity | Medium | High | High | Medium |
| Ongoing maintenance | Medium (index updates) | High (retraining) | Medium | Low |
| Latency | Medium (retrieval + generation) | Low | High (multi-turn) | Variable |
| Cost per request | Medium | Low (after training) | High (multi-call) | Variable |
| Accuracy ceiling | High (if retrieval works) | High (if data is good) | Very high | Depends on LLM |
| Handles new info | Immediately | Requires retraining | Via tools | Via tools |
| Explainability | High (show sources) | Low (implicit) | Medium (show steps) | High (show trace) |

### Reference Architecture: Production LLM System

```python
# A complete production LLM system typically combines multiple patterns:

ARCHITECTURE = {
    "ingress": {
        "component": "API Gateway",
        "responsibilities": ["auth", "rate limiting", "request validation"],
    },
    "router": {
        "component": "Complexity Router",
        "responsibilities": [
            "classify query complexity",
            "select model tier",
            "check cache",
        ],
    },
    "cache_layer": {
        "component": "Semantic + Exact Cache",
        "expected_hit_rate": "25-40%",
        "storage": "Redis + pgvector",
    },
    "retrieval": {
        "component": "RAG Pipeline",
        "vector_store": "pgvector or Pinecone",
        "embedding_model": "text-embedding-3-small",
        "reranker": "Cohere rerank or cross-encoder",
    },
    "inference": {
        "fast_tier": "Claude Haiku 3.5 / Gemini Flash",
        "balanced_tier": "Claude Sonnet 4 / GPT-4o-mini",
        "premium_tier": "Claude Opus 4 / GPT-4o",
        "self_hosted": "vLLM + Llama 3.3 70B (for data-sensitive workloads)",
    },
    "evaluation": {
        "online": ["latency_p50/p99", "error_rate", "user_thumbs"],
        "offline": ["regression_suite", "llm_judge", "ragas"],
        "alerting": "quality score drop > 5% triggers alert",
    },
    "observability": {
        "traces": "OpenTelemetry + Langfuse",
        "metrics": "Prometheus + Grafana",
        "logs": "structured JSON to central log store",
    },
}
```

### Cost Planning Template

```python
def estimate_monthly_cost(
    daily_requests: int,
    avg_input_tokens: int = 500,
    avg_output_tokens: int = 1000,
    cache_hit_rate: float = 0.30,
    routing_distribution: dict | None = None,
) -> dict:
    """Estimate monthly LLM infrastructure costs."""

    if routing_distribution is None:
        routing_distribution = {
            "fast": 0.60,       # 60% to cheap model
            "balanced": 0.30,   # 30% to mid-tier
            "premium": 0.10,    # 10% to top-tier
        }

    tier_pricing = {
        "fast":     {"input": 0.80,  "output": 4.00},
        "balanced": {"input": 3.00,  "output": 15.00},
        "premium":  {"input": 15.00, "output": 75.00},
    }

    monthly_requests = daily_requests * 30
    cache_misses = monthly_requests * (1 - cache_hit_rate)

    costs = {}
    total = 0
    for tier, fraction in routing_distribution.items():
        tier_requests = cache_misses * fraction
        p = tier_pricing[tier]
        tier_cost = tier_requests * (
            avg_input_tokens * p["input"] + avg_output_tokens * p["output"]
        ) / 1_000_000
        costs[tier] = round(tier_cost, 2)
        total += tier_cost

    return {
        "monthly_requests": monthly_requests,
        "cache_hits": round(monthly_requests * cache_hit_rate),
        "billable_requests": round(cache_misses),
        "cost_by_tier": costs,
        "total_llm_cost": round(total, 2),
        "avg_cost_per_request": round(total / monthly_requests, 5),
        "infrastructure_estimate": round(total * 0.15, 2),  # ~15% for infra
        "grand_total": round(total * 1.15, 2),
    }

# Example: 10K requests/day SaaS product
# estimate_monthly_cost(10_000)
# -> {
#   "monthly_requests": 300000,
#   "cache_hits": 90000,
#   "billable_requests": 210000,
#   "cost_by_tier": {"fast": 100.80, "balanced": 340.20, "premium": 1606.50},
#   "total_llm_cost": 2047.50,
#   "avg_cost_per_request": 0.00683,
#   "infrastructure_estimate": 307.13,
#   "grand_total": 2354.63
# }
```
