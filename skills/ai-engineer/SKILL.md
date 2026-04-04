---
name: ai-engineer
description: Build production AI applications: LLM integration, function calling, streaming, embedding pipelines, guardrails, cost optimization, and evaluation.
---

# AI Engineer

Build production-grade AI applications that are reliable, cost-effective, and scalable. This skill covers the complete AI engineering stack: from LLM API integration and function calling to streaming architectures, embedding pipelines, guardrails, cost optimization, and systematic evaluation. Every pattern is battle-tested for real-world production use.

---

## Table of Contents

- [LLM Integration Patterns](#llm-integration-patterns)
- [Function Calling and Tool Use](#function-calling-and-tool-use)
- [Streaming Architecture](#streaming-architecture)
- [Prompt Engineering in Production](#prompt-engineering-in-production)
- [Embedding Pipelines](#embedding-pipelines)
- [Guardrails and Safety](#guardrails-and-safety)
- [Cost Optimization](#cost-optimization)
- [Evaluation and Testing](#evaluation-and-testing)
- [Common Architecture Patterns](#common-architecture-patterns)

---

## LLM Integration Patterns

Reliable LLM integration requires provider abstraction, structured outputs, retry logic, and fallback chains. Never hard-code a single provider into your application.

### API Client Setup

**Python -- Unified provider client:**

```python
from openai import OpenAI
from anthropic import Anthropic
from typing import Protocol, Any
import os

class LLMProvider(Protocol):
    def complete(self, messages: list[dict], **kwargs) -> str: ...

class OpenAIProvider:
    def __init__(self, model: str = "gpt-4o"):
        self.client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
        self.model = model

    def complete(self, messages: list[dict], **kwargs) -> str:
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            **kwargs,
        )
        return response.choices[0].message.content

class AnthropicProvider:
    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
        self.model = model

    def complete(self, messages: list[dict], **kwargs) -> str:
        system = next((m["content"] for m in messages if m["role"] == "system"), None)
        user_msgs = [m for m in messages if m["role"] != "system"]
        response = self.client.messages.create(
            model=self.model,
            max_tokens=kwargs.get("max_tokens", 4096),
            system=system or "",
            messages=user_msgs,
        )
        return response.content[0].text

class LocalProvider:
    """For Ollama, vLLM, or any OpenAI-compatible local server."""
    def __init__(self, base_url: str = "http://localhost:11434/v1", model: str = "llama3"):
        self.client = OpenAI(base_url=base_url, api_key="ollama")
        self.model = model

    def complete(self, messages: list[dict], **kwargs) -> str:
        response = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            **kwargs,
        )
        return response.choices[0].message.content
```

**TypeScript -- Provider abstraction:**

```typescript
import OpenAI from "openai";
import Anthropic from "@anthropic-ai/sdk";

interface LLMProvider {
  complete(messages: { role: string; content: string }[]): Promise<string>;
}

class OpenAIProvider implements LLMProvider {
  private client: OpenAI;
  constructor(private model = "gpt-4o") {
    this.client = new OpenAI();
  }
  async complete(messages: { role: string; content: string }[]) {
    const res = await this.client.chat.completions.create({
      model: this.model,
      messages: messages as OpenAI.ChatCompletionMessageParam[],
    });
    return res.choices[0].message.content ?? "";
  }
}

class AnthropicProvider implements LLMProvider {
  private client: Anthropic;
  constructor(private model = "claude-sonnet-4-20250514") {
    this.client = new Anthropic();
  }
  async complete(messages: { role: string; content: string }[]) {
    const system = messages.find((m) => m.role === "system")?.content ?? "";
    const userMsgs = messages.filter((m) => m.role !== "system");
    const res = await this.client.messages.create({
      model: this.model,
      max_tokens: 4096,
      system,
      messages: userMsgs as Anthropic.MessageParam[],
    });
    return res.content[0].type === "text" ? res.content[0].text : "";
  }
}
```

### Structured Outputs

Force LLMs to return valid JSON conforming to a schema:

```python
from pydantic import BaseModel
from openai import OpenAI

class ExtractedEntity(BaseModel):
    name: str
    entity_type: str  # "person" | "org" | "location"
    confidence: float

class ExtractionResult(BaseModel):
    entities: list[ExtractedEntity]
    summary: str

client = OpenAI()

response = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Extract entities from the text."},
        {"role": "user", "content": user_text},
    ],
    response_format=ExtractionResult,
)

result: ExtractionResult = response.choices[0].message.parsed
```

### Retry Logic with Exponential Backoff

```python
import time
import random
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1.0, max_delay=60.0):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except (RateLimitError, APITimeoutError, InternalServerError) as e:
                    if attempt == max_retries:
                        raise
                    delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
                    time.sleep(delay)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3)
def call_llm(provider: LLMProvider, messages: list[dict]) -> str:
    return provider.complete(messages)
```

### Fallback Chains

When primary providers fail, fall through to alternatives:

```python
class FallbackChain:
    def __init__(self, providers: list[LLMProvider]):
        self.providers = providers

    def complete(self, messages: list[dict], **kwargs) -> str:
        errors = []
        for provider in self.providers:
            try:
                return provider.complete(messages, **kwargs)
            except Exception as e:
                errors.append((provider.__class__.__name__, e))
                continue
        raise RuntimeError(f"All providers failed: {errors}")

# Usage: try Claude first, fall back to GPT-4o, then local model
chain = FallbackChain([
    AnthropicProvider(model="claude-sonnet-4-20250514"),
    OpenAIProvider(model="gpt-4o"),
    LocalProvider(model="llama3"),
])
```

### Provider Selection Matrix

| Requirement | Recommended Provider | Reason |
|---|---|---|
| Long context (200k+) | Anthropic Claude | Native 200k context window |
| Structured JSON output | OpenAI GPT-4o | Native `response_format` support |
| On-premise / air-gapped | Ollama + Llama 3 | Fully local, no API calls |
| Vision + text | GPT-4o or Claude | Multi-modal native support |
| Lowest latency | GPT-4o-mini or Haiku | Optimized for speed |
| Code generation | Claude Sonnet or GPT-4o | Top coding benchmarks |

---

## Function Calling and Tool Use

Function calling lets LLMs invoke external tools, query APIs, and take actions. This is the foundation of agentic systems.

### Tool Schema Design

Define tools with precise JSON Schema descriptions. Ambiguous descriptions cause misuse.

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_database",
            "description": "Search the product database by query. Returns top-k results with relevance scores. Use when the user asks about specific products, pricing, or availability.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Natural language search query"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Maximum results to return (1-50)",
                        "default": 10
                    },
                    "category": {
                        "type": "string",
                        "enum": ["electronics", "clothing", "home", "all"],
                        "description": "Product category filter"
                    }
                },
                "required": ["query"]
            }
        }
    }
]
```

### Tool Orchestration Loop

```python
import json

def run_agent(client, messages: list[dict], tools: list[dict], max_turns: int = 10) -> str:
    """Execute an agentic loop with tool use."""
    for turn in range(max_turns):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
        )
        message = response.choices[0].message
        messages.append(message)

        # If no tool calls, we have the final answer
        if not message.tool_calls:
            return message.content

        # Execute each tool call
        for tool_call in message.tool_calls:
            fn_name = tool_call.function.name
            fn_args = json.loads(tool_call.function.arguments)

            # Dispatch to actual function
            result = execute_tool(fn_name, fn_args)

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            })

    return "Max turns reached without final answer."

def execute_tool(name: str, args: dict) -> Any:
    """Route tool calls to implementations."""
    registry = {
        "search_database": search_database,
        "get_weather": get_weather,
        "send_email": send_email,
    }
    fn = registry.get(name)
    if not fn:
        return {"error": f"Unknown tool: {name}"}
    return fn(**args)
```

### Parallel Tool Calls

Both OpenAI and Anthropic support parallel tool calls -- the model can request multiple tools in a single turn:

```typescript
// TypeScript: handling parallel tool calls
async function handleToolCalls(
  message: OpenAI.ChatCompletionMessage,
  toolRegistry: Record<string, (args: any) => Promise<any>>
): Promise<OpenAI.ChatCompletionToolMessageParam[]> {
  if (!message.tool_calls) return [];

  // Execute all tool calls in parallel
  const results = await Promise.all(
    message.tool_calls.map(async (tc) => {
      const fn = toolRegistry[tc.function.name];
      const args = JSON.parse(tc.function.arguments);
      try {
        const result = await fn(args);
        return {
          role: "tool" as const,
          tool_call_id: tc.id,
          content: JSON.stringify(result),
        };
      } catch (error) {
        return {
          role: "tool" as const,
          tool_call_id: tc.id,
          content: JSON.stringify({ error: String(error) }),
        };
      }
    })
  );
  return results;
}
```

### Tool Design Best Practices

| Practice | Why |
|---|---|
| One action per tool | Easier for the model to reason about |
| Return structured data, not prose | Model can interpret structured results better |
| Include error states in responses | Model can recover and retry |
| Set strict parameter validation | Prevent injection or misuse |
| Add usage hints in descriptions | Guide the model on when to use each tool |
| Limit tool count to <20 | Too many tools degrade selection accuracy |

---

## Streaming Architecture

Streaming delivers tokens to the user as they are generated, cutting perceived latency from seconds to milliseconds.

### SSE Endpoint (Server-Sent Events)

**Python -- FastAPI streaming endpoint:**

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from openai import OpenAI
import json

app = FastAPI()

@app.post("/api/v1/chat/stream")
async def stream_chat(request: ChatRequest):
    async def generate():
        client = OpenAI()
        stream = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": request.message}],
            stream=True,
        )
        for chunk in stream:
            delta = chunk.choices[0].delta
            if delta.content:
                data = {"type": "content", "text": delta.content}
                yield f"data: {json.dumps(data)}\n\n"

        yield f"data: {json.dumps({'type': 'done'})}\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )
```

**TypeScript -- Next.js streaming route:**

```typescript
import { OpenAI } from "openai";

export async function POST(req: Request) {
  const { message } = await req.json();
  const client = new OpenAI();

  const stream = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: message }],
    stream: true,
  });

  const encoder = new TextEncoder();
  const readable = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        const text = chunk.choices[0]?.delta?.content ?? "";
        if (text) {
          controller.enqueue(
            encoder.encode(`data: ${JSON.stringify({ text })}\n\n`)
          );
        }
      }
      controller.enqueue(encoder.encode("data: [DONE]\n\n"));
      controller.close();
    },
  });

  return new Response(readable, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

### Client-Side Streaming Parser

```typescript
async function streamChat(
  message: string,
  onToken: (text: string) => void,
  onDone: () => void
) {
  const response = await fetch("/api/v1/chat/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n\n");
    buffer = lines.pop() ?? "";

    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      const payload = line.slice(6);
      if (payload === "[DONE]") {
        onDone();
        return;
      }
      const data = JSON.parse(payload);
      onToken(data.text);
    }
  }
}
```

### Streaming with Tool Use

When streaming with tool calls, the model sends tool call deltas that must be assembled:

```python
def stream_with_tools(client, messages, tools):
    """Stream a response that may include tool calls."""
    stream = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools,
        stream=True,
    )

    tool_calls_buffer = {}
    content_buffer = ""

    for chunk in stream:
        delta = chunk.choices[0].delta

        # Accumulate content tokens
        if delta.content:
            content_buffer += delta.content
            yield {"type": "content", "text": delta.content}

        # Accumulate tool call fragments
        if delta.tool_calls:
            for tc_delta in delta.tool_calls:
                idx = tc_delta.index
                if idx not in tool_calls_buffer:
                    tool_calls_buffer[idx] = {
                        "id": tc_delta.id,
                        "name": tc_delta.function.name or "",
                        "arguments": "",
                    }
                if tc_delta.function.arguments:
                    tool_calls_buffer[idx]["arguments"] += tc_delta.function.arguments

    # After stream ends, execute any tool calls
    if tool_calls_buffer:
        for idx, tc in sorted(tool_calls_buffer.items()):
            args = json.loads(tc["arguments"])
            result = execute_tool(tc["name"], args)
            yield {"type": "tool_result", "name": tc["name"], "result": result}
```

### Backpressure Handling

For high-concurrency scenarios, control memory usage with bounded queues:

```python
import asyncio

async def bounded_stream(client, messages, max_buffer=100):
    """Stream with backpressure to prevent memory overflow."""
    queue = asyncio.Queue(maxsize=max_buffer)

    async def producer():
        stream = client.chat.completions.create(
            model="gpt-4o", messages=messages, stream=True
        )
        for chunk in stream:
            text = chunk.choices[0].delta.content
            if text:
                await queue.put(text)  # Blocks if queue is full
        await queue.put(None)  # Sentinel

    async def consumer():
        while True:
            token = await queue.get()
            if token is None:
                break
            yield token

    asyncio.create_task(producer())
    async for token in consumer():
        yield token
```

---

## Prompt Engineering in Production

Production prompt engineering is about versioning, testing, and managing prompts as first-class artifacts -- not ad-hoc string concatenation.

### Prompt Template System

```python
from string import Template
from dataclasses import dataclass
from typing import Optional
import hashlib

@dataclass
class PromptTemplate:
    name: str
    version: str
    system: str
    user: str
    variables: list[str]

    @property
    def fingerprint(self) -> str:
        content = self.system + self.user
        return hashlib.sha256(content.encode()).hexdigest()[:12]

    def render(self, **kwargs) -> list[dict]:
        for var in self.variables:
            if var not in kwargs:
                raise ValueError(f"Missing variable: {var}")
        return [
            {"role": "system", "content": Template(self.system).safe_substitute(**kwargs)},
            {"role": "user", "content": Template(self.user).safe_substitute(**kwargs)},
        ]

# Define versioned prompts
SUMMARIZER_V2 = PromptTemplate(
    name="summarizer",
    version="2.1.0",
    system="""You are a concise summarizer. Rules:
- Output exactly $num_bullets bullet points
- Each bullet is one sentence, max 20 words
- Focus on actionable insights, not background
- Use present tense""",
    user="Summarize this $doc_type:\n\n$content",
    variables=["num_bullets", "doc_type", "content"],
)

messages = SUMMARIZER_V2.render(
    num_bullets="5",
    doc_type="meeting transcript",
    content=transcript_text,
)
```

### Prompt Version Management

```python
class PromptRegistry:
    """Manage prompt versions with rollback support."""

    def __init__(self):
        self._prompts: dict[str, dict[str, PromptTemplate]] = {}
        self._active: dict[str, str] = {}  # name -> active version

    def register(self, template: PromptTemplate):
        if template.name not in self._prompts:
            self._prompts[template.name] = {}
        self._prompts[template.name][template.version] = template

    def activate(self, name: str, version: str):
        if version not in self._prompts.get(name, {}):
            raise ValueError(f"Unknown prompt: {name}@{version}")
        self._active[name] = version

    def get(self, name: str) -> PromptTemplate:
        version = self._active.get(name)
        if not version:
            raise ValueError(f"No active version for: {name}")
        return self._prompts[name][version]

    def rollback(self, name: str, version: str):
        """Instant rollback to a previous prompt version."""
        self.activate(name, version)

registry = PromptRegistry()
registry.register(SUMMARIZER_V2)
registry.activate("summarizer", "2.1.0")
```

### A/B Testing Prompts

```python
import random
from dataclasses import dataclass

@dataclass
class ABTest:
    name: str
    variants: dict[str, PromptTemplate]  # variant_id -> template
    weights: dict[str, float]            # variant_id -> traffic %

    def assign(self, user_id: str) -> tuple[str, PromptTemplate]:
        """Deterministic assignment based on user_id."""
        hash_val = int(hashlib.md5(f"{self.name}:{user_id}".encode()).hexdigest(), 16)
        normalized = (hash_val % 10000) / 10000.0

        cumulative = 0.0
        for variant_id, weight in self.weights.items():
            cumulative += weight
            if normalized < cumulative:
                return variant_id, self.variants[variant_id]
        # Fallback to last variant
        last = list(self.variants.keys())[-1]
        return last, self.variants[last]

# Track results per variant
def log_ab_result(test_name: str, variant_id: str, user_id: str, metrics: dict):
    """Log to your analytics backend for statistical analysis."""
    analytics.track("prompt_ab_test", {
        "test": test_name,
        "variant": variant_id,
        "user": user_id,
        **metrics,
    })
```

### System Prompt Design Principles

| Principle | Example |
|---|---|
| Be specific about format | "Respond in JSON with keys: answer, confidence, sources" |
| Set constraints explicitly | "Maximum 3 sentences. No bullet points." |
| Define the persona's limits | "If you don't know, say 'I don't have that information.'" |
| Include examples (few-shot) | Provide 2-3 input/output pairs in the system prompt |
| Separate instructions from context | Use XML tags or markdown headers to delineate sections |
| Version your system prompts | Track changes like code -- every edit is a deployment |

---

## Embedding Pipelines

Embedding pipelines convert text into vector representations for semantic search, clustering, and classification.

### Chunking Strategies

```python
from dataclasses import dataclass

@dataclass
class Chunk:
    text: str
    metadata: dict
    index: int

def chunk_by_tokens(text: str, max_tokens: int = 512, overlap: int = 50) -> list[Chunk]:
    """Token-based chunking with overlap for context continuity."""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o")
    tokens = enc.encode(text)
    chunks = []

    start = 0
    idx = 0
    while start < len(tokens):
        end = min(start + max_tokens, len(tokens))
        chunk_tokens = tokens[start:end]
        chunk_text = enc.decode(chunk_tokens)
        chunks.append(Chunk(text=chunk_text, metadata={"token_count": len(chunk_tokens)}, index=idx))
        start += max_tokens - overlap
        idx += 1

    return chunks

def chunk_by_semantic_boundary(text: str, max_tokens: int = 512) -> list[Chunk]:
    """Split on paragraph/section boundaries, respecting token limits."""
    paragraphs = text.split("\n\n")
    chunks = []
    current = ""
    idx = 0

    for para in paragraphs:
        if estimate_tokens(current + para) > max_tokens and current:
            chunks.append(Chunk(text=current.strip(), metadata={}, index=idx))
            idx += 1
            current = ""
        current += para + "\n\n"

    if current.strip():
        chunks.append(Chunk(text=current.strip(), metadata={}, index=idx))

    return chunks
```

### Embedding Model Selection

| Model | Dimensions | Speed | Quality | Cost |
|---|---|---|---|---|
| text-embedding-3-small (OpenAI) | 1536 | Fast | Good | $0.02/1M tokens |
| text-embedding-3-large (OpenAI) | 3072 | Medium | Excellent | $0.13/1M tokens |
| voyage-3 (Voyage AI) | 1024 | Fast | Excellent | $0.06/1M tokens |
| all-MiniLM-L6-v2 (local) | 384 | Very Fast | Good | Free |
| nomic-embed-text (local) | 768 | Fast | Very Good | Free |
| BGE-large-en-v1.5 (local) | 1024 | Medium | Excellent | Free |

### Batch Embedding Pipeline

```python
from openai import OpenAI
from typing import Iterator
import numpy as np

def batch_embed(
    texts: list[str],
    model: str = "text-embedding-3-small",
    batch_size: int = 100,
) -> np.ndarray:
    """Embed texts in batches to avoid API limits."""
    client = OpenAI()
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i : i + batch_size]
        response = client.embeddings.create(model=model, input=batch)
        embeddings = [item.embedding for item in response.data]
        all_embeddings.extend(embeddings)

    return np.array(all_embeddings)
```

### Incremental Updates

```python
class EmbeddingIndex:
    """Maintain an embedding index with incremental updates."""

    def __init__(self, vector_store, embed_fn):
        self.store = vector_store
        self.embed_fn = embed_fn
        self.seen_hashes: set[str] = set()

    def upsert_documents(self, documents: list[dict]):
        """Only embed and store documents that have changed."""
        new_docs = []
        for doc in documents:
            doc_hash = hashlib.sha256(doc["content"].encode()).hexdigest()
            if doc_hash not in self.seen_hashes:
                new_docs.append(doc)
                self.seen_hashes.add(doc_hash)

        if not new_docs:
            return 0

        texts = [d["content"] for d in new_docs]
        embeddings = self.embed_fn(texts)

        vectors = [
            {
                "id": doc["id"],
                "values": emb.tolist(),
                "metadata": doc.get("metadata", {}),
            }
            for doc, emb in zip(new_docs, embeddings)
        ]
        self.store.upsert(vectors)
        return len(new_docs)
```

### Hybrid Search (Dense + Sparse)

Combine embedding similarity with keyword matching for better recall:

```python
def hybrid_search(query: str, top_k: int = 10, alpha: float = 0.7) -> list[dict]:
    """
    Hybrid search combining dense (embedding) and sparse (BM25) retrieval.
    alpha: weight for dense score (1.0 = pure dense, 0.0 = pure sparse)
    """
    # Dense retrieval
    query_embedding = embed_fn(query)
    dense_results = vector_store.query(query_embedding, top_k=top_k * 2)

    # Sparse retrieval (BM25)
    sparse_results = bm25_index.search(query, top_k=top_k * 2)

    # Reciprocal Rank Fusion
    scores: dict[str, float] = {}
    k = 60  # RRF constant

    for rank, result in enumerate(dense_results):
        scores[result["id"]] = scores.get(result["id"], 0) + alpha / (k + rank + 1)

    for rank, result in enumerate(sparse_results):
        scores[result["id"]] = scores.get(result["id"], 0) + (1 - alpha) / (k + rank + 1)

    # Sort by combined score, return top_k
    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_k]
    return [{"id": doc_id, "score": score} for doc_id, score in ranked]
```

---

## Guardrails and Safety

Production AI systems need multiple layers of defense: input validation, output filtering, PII protection, and rate limiting.

### Input Validation

```python
import re
from dataclasses import dataclass

@dataclass
class ValidationResult:
    is_valid: bool
    reason: str | None = None

def validate_input(user_input: str) -> ValidationResult:
    """Multi-layer input validation."""
    # Length check
    if len(user_input) > 10_000:
        return ValidationResult(False, "Input exceeds maximum length")

    if len(user_input.strip()) == 0:
        return ValidationResult(False, "Empty input")

    # Prompt injection detection (basic heuristic layer)
    injection_patterns = [
        r"ignore\s+(all\s+)?previous\s+instructions",
        r"you\s+are\s+now\s+(?:a|an)\s+",
        r"system\s*:\s*",
        r"<\|im_start\|>",
        r"\[INST\]",
        r"IMPORTANT:\s*override",
    ]
    for pattern in injection_patterns:
        if re.search(pattern, user_input, re.IGNORECASE):
            return ValidationResult(False, "Potential prompt injection detected")

    return ValidationResult(True)
```

### Output Filtering

```python
class OutputFilter:
    """Filter LLM outputs before returning to users."""

    def __init__(self):
        self.blocked_patterns = [
            r"\b\d{3}-\d{2}-\d{4}\b",    # SSN
            r"\b\d{16}\b",                 # Credit card
            r"(?i)password\s*[:=]\s*\S+",  # Leaked passwords
        ]

    def filter(self, output: str) -> tuple[str, list[str]]:
        """Returns (filtered_output, list_of_redactions)."""
        redactions = []
        filtered = output

        for pattern in self.blocked_patterns:
            matches = re.findall(pattern, filtered)
            if matches:
                redactions.extend(matches)
                filtered = re.sub(pattern, "[REDACTED]", filtered)

        return filtered, redactions

    def check_refusal(self, output: str) -> bool:
        """Detect if the model refused to answer (may need follow-up)."""
        refusal_signals = [
            "I cannot", "I'm unable to", "I can't help with",
            "not appropriate", "against my guidelines"
        ]
        return any(signal.lower() in output.lower() for signal in refusal_signals)
```

### PII Detection

```python
class PIIDetector:
    """Detect and redact PII from text."""

    PII_PATTERNS = {
        "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
        "phone_us": r"\b(?:\+1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b",
        "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
        "credit_card": r"\b(?:\d{4}[-\s]?){3}\d{4}\b",
        "ip_address": r"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b",
    }

    def detect(self, text: str) -> list[dict]:
        findings = []
        for pii_type, pattern in self.PII_PATTERNS.items():
            for match in re.finditer(pattern, text):
                findings.append({
                    "type": pii_type,
                    "value": match.group(),
                    "start": match.start(),
                    "end": match.end(),
                })
        return findings

    def redact(self, text: str) -> str:
        for pii_type, pattern in self.PII_PATTERNS.items():
            text = re.sub(pattern, f"[{pii_type.upper()}_REDACTED]", text)
        return text
```

### Content Moderation Pipeline

```python
class ModerationPipeline:
    """Multi-stage content moderation."""

    def __init__(self, client: OpenAI):
        self.client = client

    async def moderate(self, text: str) -> dict:
        # Stage 1: OpenAI moderation API (fast, free)
        moderation = self.client.moderations.create(input=text)
        result = moderation.results[0]

        if result.flagged:
            return {
                "allowed": False,
                "reason": "content_policy",
                "categories": {k: v for k, v in result.categories.__dict__.items() if v},
            }

        # Stage 2: Custom business rules
        if self._contains_competitor_mentions(text):
            return {"allowed": True, "warning": "competitor_mention"}

        return {"allowed": True}

    def _contains_competitor_mentions(self, text: str) -> bool:
        competitors = ["competitor_a", "competitor_b"]
        return any(c in text.lower() for c in competitors)
```

### Rate Limiting per User

```python
import time
from collections import defaultdict

class TokenBucketLimiter:
    """Per-user rate limiting with token bucket algorithm."""

    def __init__(self, tokens_per_minute: int = 10_000, requests_per_minute: int = 20):
        self.token_limit = tokens_per_minute
        self.request_limit = requests_per_minute
        self.buckets: dict[str, dict] = defaultdict(lambda: {
            "tokens": tokens_per_minute,
            "requests": requests_per_minute,
            "last_refill": time.time(),
        })

    def check(self, user_id: str, estimated_tokens: int) -> tuple[bool, str]:
        bucket = self.buckets[user_id]
        self._refill(bucket)

        if bucket["requests"] <= 0:
            return False, "Request rate limit exceeded"

        if bucket["tokens"] < estimated_tokens:
            return False, "Token rate limit exceeded"

        bucket["requests"] -= 1
        bucket["tokens"] -= estimated_tokens
        return True, "OK"

    def _refill(self, bucket: dict):
        now = time.time()
        elapsed = now - bucket["last_refill"]
        if elapsed >= 60:
            bucket["tokens"] = self.token_limit
            bucket["requests"] = self.request_limit
            bucket["last_refill"] = now
```

### Hallucination Detection

```python
def detect_hallucination(response: str, sources: list[str], client) -> dict:
    """Use an LLM to check if the response is grounded in the provided sources."""
    check_prompt = f"""You are a fact-checking assistant. Determine if the RESPONSE is fully supported by the SOURCES.

SOURCES:
{chr(10).join(f'[{i+1}] {s}' for i, s in enumerate(sources))}

RESPONSE:
{response}

For each claim in the response, check if it is:
- SUPPORTED: directly stated or logically implied by sources
- UNSUPPORTED: not found in any source
- CONTRADICTED: conflicts with source information

Return JSON: {{"claims": [{{"claim": "...", "verdict": "...", "source_id": N|null}}], "overall": "grounded"|"partially_grounded"|"hallucinated"}}"""

    result = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": check_prompt}],
        response_format={"type": "json_object"},
    )
    return json.loads(result.choices[0].message.content)
```

---

## Cost Optimization

AI costs can explode without careful management. Token counting, caching, model routing, and prompt compression are your tools.

### Token Counting

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4o") -> int:
    """Count tokens for accurate cost estimation."""
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

def estimate_cost(
    input_tokens: int,
    output_tokens: int,
    model: str = "gpt-4o",
) -> float:
    """Estimate cost in USD."""
    pricing = {
        # per 1M tokens: (input, output)
        "gpt-4o":           (2.50, 10.00),
        "gpt-4o-mini":      (0.15, 0.60),
        "claude-sonnet-4-20250514": (3.00, 15.00),
        "claude-haiku-3.5": (0.80, 4.00),
    }
    input_rate, output_rate = pricing.get(model, (5.0, 15.0))
    return (input_tokens * input_rate + output_tokens * output_rate) / 1_000_000

# Usage tracking
class CostTracker:
    def __init__(self):
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.total_cost = 0.0
        self.calls = 0

    def record(self, input_tokens: int, output_tokens: int, model: str):
        self.total_input_tokens += input_tokens
        self.total_output_tokens += output_tokens
        self.total_cost += estimate_cost(input_tokens, output_tokens, model)
        self.calls += 1

    def report(self) -> dict:
        return {
            "total_calls": self.calls,
            "total_input_tokens": self.total_input_tokens,
            "total_output_tokens": self.total_output_tokens,
            "total_cost_usd": round(self.total_cost, 4),
            "avg_cost_per_call": round(self.total_cost / max(self.calls, 1), 4),
        }
```

### Semantic Caching

Cache responses for semantically similar queries to avoid redundant API calls:

```python
import numpy as np

class SemanticCache:
    """Cache LLM responses by semantic similarity of the input."""

    def __init__(self, embed_fn, similarity_threshold: float = 0.95):
        self.embed_fn = embed_fn
        self.threshold = similarity_threshold
        self.cache: list[dict] = []  # {embedding, query, response}

    def get(self, query: str) -> str | None:
        if not self.cache:
            return None

        query_emb = self.embed_fn(query)
        for entry in self.cache:
            similarity = np.dot(query_emb, entry["embedding"]) / (
                np.linalg.norm(query_emb) * np.linalg.norm(entry["embedding"])
            )
            if similarity >= self.threshold:
                return entry["response"]
        return None

    def set(self, query: str, response: str):
        embedding = self.embed_fn(query)
        self.cache.append({
            "embedding": embedding,
            "query": query,
            "response": response,
        })
```

### Model Routing by Complexity

Route simple queries to cheap models, complex ones to expensive models:

```python
class ModelRouter:
    """Route requests to appropriate models based on complexity."""

    def __init__(self):
        self.simple_model = "gpt-4o-mini"    # $0.15/1M input
        self.complex_model = "gpt-4o"         # $2.50/1M input

    def classify_complexity(self, messages: list[dict]) -> str:
        """Heuristic complexity classification."""
        user_msg = messages[-1]["content"]
        token_count = count_tokens(user_msg)

        # Simple heuristics -- replace with a trained classifier for production
        if token_count < 100 and not any(
            kw in user_msg.lower()
            for kw in ["analyze", "compare", "explain why", "design", "architect"]
        ):
            return "simple"

        if any(kw in user_msg.lower() for kw in ["code review", "debug", "refactor", "design system"]):
            return "complex"

        return "complex"  # Default to complex for safety

    def route(self, messages: list[dict]) -> str:
        complexity = self.classify_complexity(messages)
        return self.simple_model if complexity == "simple" else self.complex_model
```

### Prompt Compression

Reduce token usage by compressing context before sending to the LLM:

```python
def compress_context(documents: list[str], query: str, max_tokens: int = 2000) -> str:
    """Select and compress relevant context to fit token budget."""
    # Score documents by relevance
    scored = []
    for doc in documents:
        # Simple TF-IDF-like relevance scoring
        query_terms = set(query.lower().split())
        doc_terms = set(doc.lower().split())
        overlap = len(query_terms & doc_terms) / max(len(query_terms), 1)
        scored.append((overlap, doc))

    # Sort by relevance, take top documents that fit
    scored.sort(reverse=True)
    compressed = []
    current_tokens = 0

    for score, doc in scored:
        doc_tokens = count_tokens(doc)
        if current_tokens + doc_tokens > max_tokens:
            # Truncate last document to fit
            remaining = max_tokens - current_tokens
            if remaining > 50:
                truncated = doc[:remaining * 4]  # Rough char-to-token ratio
                compressed.append(truncated + "...")
            break
        compressed.append(doc)
        current_tokens += doc_tokens

    return "\n---\n".join(compressed)
```

### Cost Optimization Decision Matrix

| Strategy | Savings | Implementation Effort | Risk |
|---|---|---|---|
| Model routing (mini vs full) | 90%+ on simple queries | Low | Misrouting complex queries |
| Semantic caching | 30-60% on repeated patterns | Medium | Stale responses |
| Prompt compression | 20-40% token reduction | Low | Lost context |
| Batch API (non-real-time) | 50% cost reduction | Low | Higher latency (24h) |
| Fine-tuning small model | 80%+ on narrow tasks | High | Training data required |
| Local models (Ollama) | 100% API cost elimination | Medium | Lower quality |

---

## Evaluation and Testing

You cannot improve what you do not measure. Systematic evaluation separates production AI from prototypes.

### Eval Framework Architecture

```
+------------------+     +-----------+     +-------------+     +----------+
| Golden Dataset   | --> | Runner    | --> | Judges      | --> | Reporter |
| (inputs/outputs) |     | (call LLM)|     | (score each)|     | (metrics)|
+------------------+     +-----------+     +-------------+     +----------+
```

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class EvalCase:
    input: str
    expected_output: str | None
    metadata: dict | None = None

@dataclass
class EvalResult:
    case: EvalCase
    actual_output: str
    scores: dict[str, float]  # metric_name -> score (0-1)
    latency_ms: float

class EvalRunner:
    def __init__(self, provider: LLMProvider, judges: list["Judge"]):
        self.provider = provider
        self.judges = judges

    def run(self, dataset: list[EvalCase]) -> list[EvalResult]:
        results = []
        for case in dataset:
            start = time.time()
            output = self.provider.complete([
                {"role": "user", "content": case.input}
            ])
            latency = (time.time() - start) * 1000

            scores = {}
            for judge in self.judges:
                scores[judge.name] = judge.score(case, output)

            results.append(EvalResult(
                case=case,
                actual_output=output,
                scores=scores,
                latency_ms=latency,
            ))
        return results

    def summary(self, results: list[EvalResult]) -> dict:
        metric_names = results[0].scores.keys() if results else []
        return {
            metric: {
                "mean": sum(r.scores[metric] for r in results) / len(results),
                "min": min(r.scores[metric] for r in results),
                "max": max(r.scores[metric] for r in results),
            }
            for metric in metric_names
        }
```

### Golden Datasets

```python
# Store eval datasets as versioned JSONL files
import json

def load_golden_dataset(path: str) -> list[EvalCase]:
    cases = []
    with open(path) as f:
        for line in f:
            data = json.loads(line)
            cases.append(EvalCase(
                input=data["input"],
                expected_output=data.get("expected_output"),
                metadata=data.get("metadata"),
            ))
    return cases

# Example golden dataset entry (JSONL):
# {"input": "What is the return policy?", "expected_output": "30-day full refund...", "metadata": {"category": "policy", "difficulty": "easy"}}
# {"input": "Compare plan A vs plan B for enterprise", "expected_output": null, "metadata": {"category": "comparison", "difficulty": "hard"}}
```

### LLM-as-Judge

Use a strong LLM to evaluate the output of the system under test:

```python
class LLMJudge:
    """Use an LLM to score outputs on specific criteria."""

    def __init__(self, client, criteria: str, name: str):
        self.client = client
        self.criteria = criteria
        self.name = name

    def score(self, case: EvalCase, output: str) -> float:
        prompt = f"""You are an evaluation judge. Score the RESPONSE on the following criteria:

CRITERIA: {self.criteria}

USER QUERY: {case.input}
{f'EXPECTED ANSWER: {case.expected_output}' if case.expected_output else ''}

RESPONSE: {output}

Score from 0.0 to 1.0 where 1.0 is perfect. Return ONLY a JSON object:
{{"score": 0.X, "reasoning": "brief explanation"}}"""

        result = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
        )
        parsed = json.loads(result.choices[0].message.content)
        return parsed["score"]

# Standard judges
faithfulness_judge = LLMJudge(
    client, name="faithfulness",
    criteria="Does the response only contain information supported by the provided context? Penalize any fabricated facts."
)
relevance_judge = LLMJudge(
    client, name="relevance",
    criteria="Does the response directly address the user's question? Penalize tangential or off-topic content."
)
coherence_judge = LLMJudge(
    client, name="coherence",
    criteria="Is the response well-structured, logical, and easy to follow? Penalize disjointed or contradictory statements."
)
```

### Regression Testing

```python
class RegressionSuite:
    """Run evals on every prompt change to catch regressions."""

    def __init__(self, runner: EvalRunner, baseline_path: str):
        self.runner = runner
        self.baseline = self._load_baseline(baseline_path)

    def _load_baseline(self, path: str) -> dict[str, float]:
        with open(path) as f:
            return json.load(f)  # {"faithfulness": 0.92, "relevance": 0.88, ...}

    def run_and_compare(self, dataset: list[EvalCase], threshold: float = 0.05) -> dict:
        """Run eval suite and flag regressions beyond threshold."""
        results = self.runner.run(dataset)
        current = self.runner.summary(results)

        regressions = {}
        for metric, stats in current.items():
            baseline_score = self.baseline.get(metric, 0)
            delta = stats["mean"] - baseline_score
            if delta < -threshold:
                regressions[metric] = {
                    "baseline": baseline_score,
                    "current": stats["mean"],
                    "delta": round(delta, 4),
                }

        return {
            "passed": len(regressions) == 0,
            "current_scores": {m: s["mean"] for m, s in current.items()},
            "regressions": regressions,
        }
```

### Key Evaluation Metrics

| Metric | What It Measures | When to Use |
|---|---|---|
| Faithfulness | Is output grounded in provided context? | RAG systems, Q&A |
| Relevance | Does output address the actual question? | All chat/Q&A systems |
| Coherence | Is output well-structured and logical? | Long-form generation |
| Completeness | Does output cover all required aspects? | Summarization, reports |
| Toxicity | Does output contain harmful content? | User-facing applications |
| Latency (p50/p95) | Response time distribution | All production systems |
| Cost per query | Average spend per request | Budget-constrained systems |

---

## Common Architecture Patterns

Choosing the right architecture is the highest-leverage decision in AI engineering. Here is a decision matrix and implementation guidance for the most common patterns.

### Architecture Decision Matrix

```
Is the task...
|
+-- Answering questions from documents? --> RAG
|
+-- Taking multi-step actions? --> Agent
|
+-- Having a conversation? --> Chatbot
|
+-- Condensing long text? --> Summarizer
|
+-- Categorizing input? --> Classifier
|
+-- Generating structured data? --> Structured Extraction
```

| Pattern | Latency | Cost | Complexity | Best For |
|---|---|---|---|---|
| RAG | Medium | Medium | Medium | Knowledge Q&A, support bots |
| Agent | High | High | High | Multi-step tasks, tool use |
| Chatbot | Low | Low-Med | Low | Conversational interfaces |
| Summarizer | Medium | Low | Low | Report generation, digests |
| Classifier | Low | Very Low | Low | Routing, tagging, triage |

### RAG (Retrieval-Augmented Generation)

```
User Query --> Embed --> Vector Search --> Top-K docs --> LLM --> Answer
                                     \--> Reranker --/
```

```python
class RAGPipeline:
    def __init__(self, vector_store, embed_fn, llm_provider, reranker=None):
        self.vector_store = vector_store
        self.embed_fn = embed_fn
        self.llm = llm_provider
        self.reranker = reranker

    def query(self, question: str, top_k: int = 5) -> dict:
        # Retrieve
        query_embedding = self.embed_fn(question)
        candidates = self.vector_store.query(query_embedding, top_k=top_k * 3)

        # Rerank (optional but recommended)
        if self.reranker:
            candidates = self.reranker.rank(question, candidates)[:top_k]
        else:
            candidates = candidates[:top_k]

        # Generate
        context = "\n\n".join(
            f"[Source {i+1}]: {doc['text']}" for i, doc in enumerate(candidates)
        )
        messages = [
            {"role": "system", "content": f"Answer based on the provided sources. Cite sources as [Source N].\n\nContext:\n{context}"},
            {"role": "user", "content": question},
        ]
        answer = self.llm.complete(messages)

        return {
            "answer": answer,
            "sources": [{"id": d["id"], "score": d["score"]} for d in candidates],
        }
```

### Agent Architecture

```
User Input --> Planner --> Tool Selection --> Execute --> Observe --> Loop or Respond
                  ^                                        |
                  +----------------------------------------+
```

```typescript
interface AgentState {
  messages: Message[];
  turnCount: number;
  maxTurns: number;
}

async function runAgent(
  client: OpenAI,
  initialPrompt: string,
  tools: OpenAI.ChatCompletionTool[],
  toolHandlers: Record<string, (args: any) => Promise<any>>,
  maxTurns = 10
): Promise<string> {
  const state: AgentState = {
    messages: [
      { role: "system", content: AGENT_SYSTEM_PROMPT },
      { role: "user", content: initialPrompt },
    ],
    turnCount: 0,
    maxTurns,
  };

  while (state.turnCount < state.maxTurns) {
    const response = await client.chat.completions.create({
      model: "gpt-4o",
      messages: state.messages,
      tools,
    });

    const message = response.choices[0].message;
    state.messages.push(message);
    state.turnCount++;

    // Final answer -- no tool calls
    if (!message.tool_calls?.length) {
      return message.content ?? "";
    }

    // Execute tools in parallel
    const toolResults = await Promise.all(
      message.tool_calls.map(async (tc) => {
        const handler = toolHandlers[tc.function.name];
        const args = JSON.parse(tc.function.arguments);
        const result = await handler(args);
        return {
          role: "tool" as const,
          tool_call_id: tc.id,
          content: JSON.stringify(result),
        };
      })
    );

    state.messages.push(...toolResults);
  }

  return "Agent reached maximum turns without completing the task.";
}
```

### Chatbot with Memory

```python
class ChatBot:
    """Stateful chatbot with sliding window memory."""

    def __init__(self, provider: LLMProvider, system_prompt: str, max_history: int = 20):
        self.provider = provider
        self.system_prompt = system_prompt
        self.max_history = max_history
        self.history: list[dict] = []

    def chat(self, user_message: str) -> str:
        self.history.append({"role": "user", "content": user_message})

        # Sliding window: keep system + last N messages
        messages = [{"role": "system", "content": self.system_prompt}]
        messages.extend(self.history[-self.max_history:])

        response = self.provider.complete(messages)
        self.history.append({"role": "assistant", "content": response})

        return response

    def summarize_and_compact(self):
        """When history gets long, summarize older messages."""
        if len(self.history) <= self.max_history:
            return

        old_messages = self.history[:-10]
        recent_messages = self.history[-10:]

        summary = self.provider.complete([
            {"role": "system", "content": "Summarize this conversation in 3 bullet points."},
            *old_messages,
        ])

        self.history = [
            {"role": "system", "content": f"Previous conversation summary: {summary}"},
            *recent_messages,
        ]
```

### Summarizer

```python
def summarize_long_document(
    text: str,
    provider: LLMProvider,
    chunk_size: int = 3000,
    style: str = "executive brief"
) -> str:
    """Map-reduce summarization for documents exceeding context limits."""
    chunks = chunk_by_tokens(text, max_tokens=chunk_size)

    # Map: summarize each chunk
    chunk_summaries = []
    for chunk in chunks:
        summary = provider.complete([
            {"role": "system", "content": f"Summarize the following text as a {style}. Be concise and preserve key facts."},
            {"role": "user", "content": chunk.text},
        ])
        chunk_summaries.append(summary)

    # Reduce: combine summaries into final
    combined = "\n\n".join(f"Section {i+1}:\n{s}" for i, s in enumerate(chunk_summaries))
    final = provider.complete([
        {"role": "system", "content": f"Combine these section summaries into one coherent {style}. Remove redundancy."},
        {"role": "user", "content": combined},
    ])

    return final
```

### Classifier

```python
def classify_with_llm(
    text: str,
    categories: list[str],
    provider: LLMProvider,
) -> dict:
    """Zero-shot classification using an LLM. Fast and cheap with mini models."""
    categories_str = ", ".join(categories)
    response = provider.complete([
        {"role": "system", "content": f"""Classify the input into exactly one category.
Categories: {categories_str}
Return JSON: {{"category": "...", "confidence": 0.X, "reasoning": "brief"}}"""},
        {"role": "user", "content": text},
    ])
    return json.loads(response)

# Example: route support tickets
result = classify_with_llm(
    text="My payment failed and I was still charged twice",
    categories=["billing", "technical", "account", "feature_request", "general"],
    provider=OpenAIProvider(model="gpt-4o-mini"),  # Use cheap model for classification
)
# {"category": "billing", "confidence": 0.95, "reasoning": "Payment and charge issues are billing-related"}
```

### When to Use What

| Signal | Pattern | Why |
|---|---|---|
| "Answer from our docs" | RAG | Grounds responses in your data |
| "Book a flight, then hotel" | Agent | Multi-step, requires tool use |
| "Help me brainstorm" | Chatbot | Conversational, stateful |
| "TLDR this report" | Summarizer | Condensation task |
| "Is this spam or not?" | Classifier | Categorization with fixed labels |
| "Extract name/email/date" | Structured Extraction | Schema-conforming output |
| "Write me a blog post" | Generation Pipeline | Long-form content creation |
| "Translate to Spanish" | Direct LLM Call | Single-turn, no retrieval needed |
