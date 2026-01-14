# 00. LLM Fundamentals

[← Назад к списку тем](README.md)

---

## How LLMs Work

### Transformer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Transformer Architecture                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Input: "The cat sat on the"                                        │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Tokenization                              │   │
│  │    "The" → 464, "cat" → 2368, "sat" → 3332, ...             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Token + Position Embeddings                     │   │
│  │    [464] → [0.12, -0.34, ...] + position encoding           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Transformer Blocks (×N layers)                  │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │         Multi-Head Self-Attention                    │    │   │
│  │  │    Q, K, V projections → Attention → Concat → Linear │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │                         ↓                                    │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │              Feed-Forward Network                    │    │   │
│  │  │         Linear → ReLU/GeLU → Linear                  │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │            + Residual connections + LayerNorm               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           ↓                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Output Layer                              │   │
│  │    Linear → Softmax → Probability over vocabulary           │   │
│  │    P("mat") = 0.31, P("floor") = 0.22, P("bed") = 0.15...   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Self-Attention Mechanism

```python
# Simplified attention calculation
def attention(Q, K, V):
    """
    Q, K, V: Query, Key, Value matrices
    d_k: dimension of keys
    """
    d_k = K.shape[-1]

    # Attention scores: how much each token attends to others
    scores = Q @ K.transpose(-2, -1) / math.sqrt(d_k)

    # For decoder (causal): mask future tokens
    # scores = scores.masked_fill(mask == 0, -1e9)

    # Softmax to get attention weights
    attention_weights = F.softmax(scores, dim=-1)

    # Weighted sum of values
    output = attention_weights @ V
    return output

# Multi-head attention: multiple attention patterns
# Each head learns different relationships
class MultiHeadAttention:
    def __init__(self, d_model, n_heads):
        self.n_heads = n_heads
        self.d_k = d_model // n_heads

        self.W_q = Linear(d_model, d_model)
        self.W_k = Linear(d_model, d_model)
        self.W_v = Linear(d_model, d_model)
        self.W_o = Linear(d_model, d_model)

    def forward(self, x):
        # Split into heads, apply attention, concatenate
        batch, seq_len, _ = x.shape
        Q = self.W_q(x).view(batch, seq_len, self.n_heads, self.d_k)
        K = self.W_k(x).view(batch, seq_len, self.n_heads, self.d_k)
        V = self.W_v(x).view(batch, seq_len, self.n_heads, self.d_k)

        attn_output = attention(Q, K, V)
        return self.W_o(attn_output.view(batch, seq_len, -1))
```

### Key Concepts

```
┌─────────────────────────────────────────────────────────────────────┐
│                      LLM Key Concepts                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Autoregressive Generation:                                         │
│  - Predicts next token based on previous tokens                     │
│  - P(token_n | token_1, token_2, ..., token_n-1)                   │
│  - Generates one token at a time                                    │
│                                                                     │
│  Context Window:                                                    │
│  - Maximum sequence length model can process                        │
│  - GPT-4: 128K tokens, Claude: 200K tokens                          │
│  - Attention is O(n²) - longer context = slower                     │
│                                                                     │
│  Parameters:                                                        │
│  - Weights learned during training                                  │
│  - GPT-3: 175B, Llama 2: 7B/13B/70B, GPT-4: ~1.7T (rumored)        │
│  - More params ≈ more knowledge, but diminishing returns            │
│                                                                     │
│  Training:                                                          │
│  - Pre-training: next token prediction on massive corpus            │
│  - Fine-tuning: supervised learning on specific tasks               │
│  - RLHF: reinforcement learning from human feedback                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tokenization

### How Tokenization Works

```python
# BPE (Byte Pair Encoding) - most common
from tiktoken import encoding_for_model

enc = encoding_for_model("gpt-4")

text = "Hello, world!"
tokens = enc.encode(text)  # [9906, 11, 1917, 0]
decoded = enc.decode(tokens)  # "Hello, world!"

# Token count matters for:
# - API pricing (per token)
# - Context window limits
# - Response length

# Common tokenizers:
# - tiktoken (OpenAI): GPT models
# - SentencePiece: Llama, many open models
# - tokenizers (HuggingFace): BERT, RoBERTa
```

### Token Counting

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4") -> int:
    """Count tokens for a given model."""
    enc = tiktoken.encoding_for_model(model)
    return len(enc.encode(text))

# Rough estimates:
# - 1 token ≈ 4 characters in English
# - 1 token ≈ 0.75 words
# - 100 tokens ≈ 75 words

# For messages (chat format):
def count_message_tokens(messages: list[dict], model: str = "gpt-4") -> int:
    """Count tokens in chat messages including overhead."""
    enc = tiktoken.encoding_for_model(model)

    tokens = 0
    for message in messages:
        tokens += 4  # Every message has overhead
        tokens += len(enc.encode(message["role"]))
        tokens += len(enc.encode(message["content"]))
    tokens += 2  # Priming for assistant response

    return tokens
```

### Tokenization Gotchas

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Tokenization Pitfalls                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Inconsistent tokenization:                                      │
│     "hello" → 1 token, " hello" → 2 tokens                          │
│     Spaces matter!                                                  │
│                                                                     │
│  2. Numbers:                                                        │
│     "123" → might be 1-3 tokens depending on tokenizer              │
│     "1234567890" → multiple tokens                                  │
│                                                                     │
│  3. Code:                                                           │
│     Indentation, symbols → many tokens                              │
│     Minified code uses fewer tokens                                 │
│                                                                     │
│  4. Non-English:                                                    │
│     Chinese, Japanese → more tokens per character                   │
│     Rare languages → very inefficient                               │
│                                                                     │
│  5. Special tokens:                                                 │
│     <|endoftext|>, [CLS], [SEP], <s>, </s>                         │
│     Model-specific, don't appear in output                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Inference Parameters

### Temperature

```python
# Temperature controls randomness in output distribution
# Before softmax: logits / temperature

def sample_with_temperature(logits, temperature=1.0):
    """
    temperature = 0: always pick highest probability (greedy)
    temperature = 1: sample from original distribution
    temperature > 1: more random/creative
    temperature < 1: more focused/deterministic
    """
    if temperature == 0:
        return torch.argmax(logits)

    scaled_logits = logits / temperature
    probs = F.softmax(scaled_logits, dim=-1)
    return torch.multinomial(probs, num_samples=1)

# Use cases:
# - Code generation: temperature = 0-0.2 (deterministic)
# - Creative writing: temperature = 0.7-1.0 (varied)
# - Brainstorming: temperature = 1.0-1.5 (creative)
```

### Top-p (Nucleus Sampling)

```python
def sample_top_p(logits, top_p=0.9):
    """
    Only sample from tokens whose cumulative probability <= top_p
    Dynamically adjusts vocabulary size based on confidence
    """
    sorted_logits, sorted_indices = torch.sort(logits, descending=True)
    cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)

    # Remove tokens with cumulative probability above threshold
    sorted_indices_to_remove = cumulative_probs > top_p
    # Keep at least one token
    sorted_indices_to_remove[0] = False

    indices_to_remove = sorted_indices[sorted_indices_to_remove]
    logits[indices_to_remove] = float('-inf')

    return torch.multinomial(F.softmax(logits, dim=-1), num_samples=1)

# top_p = 0.9: consider tokens comprising 90% probability mass
# top_p = 1.0: consider all tokens
# Often used with temperature for fine control
```

### Other Parameters

```python
# API call with common parameters
response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Write a poem"}],

    # Sampling parameters
    temperature=0.7,      # Randomness (0-2)
    top_p=0.9,           # Nucleus sampling (0-1)

    # Output control
    max_tokens=500,      # Max response length
    stop=["\n\n", "END"], # Stop sequences

    # Penalties (reduce repetition)
    frequency_penalty=0.5,   # Penalize frequent tokens
    presence_penalty=0.5,    # Penalize already-used tokens

    # Reproducibility
    seed=42,             # For deterministic output

    # Multiple completions
    n=1,                 # Number of completions
)
```

---

## Model Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Model Comparison (2024)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  OpenAI:                                                            │
│  ├── GPT-4o          Best multimodal, fast, 128K context            │
│  ├── GPT-4 Turbo     Strongest reasoning, 128K context              │
│  ├── GPT-3.5 Turbo   Fast, cheap, good for simple tasks             │
│  └── o1              Reasoning model, chain-of-thought built-in     │
│                                                                     │
│  Anthropic:                                                         │
│  ├── Claude 3 Opus   Strongest, best for complex analysis           │
│  ├── Claude 3.5 Sonnet  Best balance speed/quality, 200K context   │
│  └── Claude 3 Haiku  Fastest, cheapest, good for simple tasks       │
│                                                                     │
│  Open Source:                                                       │
│  ├── Llama 3.1       405B/70B/8B, strong open model                 │
│  ├── Mistral         7B/8x7B MoE, efficient                         │
│  ├── Mixtral 8x22B   Best open MoE                                  │
│  └── Qwen 2.5        Strong multilingual                            │
│                                                                     │
│  Specialized:                                                       │
│  ├── Gemini          Google, multimodal, 1M context                 │
│  ├── Cohere Command  Enterprise, RAG-optimized                      │
│  └── Code Llama      Code generation specialist                     │
│                                                                     │
│  Selection criteria:                                                │
│  - Task complexity → bigger model                                   │
│  - Latency requirements → smaller/faster model                      │
│  - Cost sensitivity → open source or smaller models                 │
│  - Privacy → self-hosted open source                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Pricing Comparison

```python
# Cost per 1M tokens (approximate, 2024)
PRICING = {
    # Model: (input_cost, output_cost)
    "gpt-4-turbo": (10, 30),
    "gpt-4o": (5, 15),
    "gpt-3.5-turbo": (0.5, 1.5),
    "claude-3-opus": (15, 75),
    "claude-3.5-sonnet": (3, 15),
    "claude-3-haiku": (0.25, 1.25),
    "llama-3.1-70b": (0, 0),  # Self-hosted: infra costs only
}

def estimate_cost(
    input_tokens: int,
    output_tokens: int,
    model: str
) -> float:
    """Estimate API cost in USD."""
    input_cost, output_cost = PRICING[model]
    return (input_tokens * input_cost + output_tokens * output_cost) / 1_000_000
```

---

## Context Window Management

### Strategies for Long Context

```python
# 1. Truncation - simple but loses info
def truncate_to_fit(messages: list, max_tokens: int) -> list:
    """Keep most recent messages that fit."""
    result = []
    total = 0
    for msg in reversed(messages):
        tokens = count_tokens(msg["content"])
        if total + tokens > max_tokens:
            break
        result.insert(0, msg)
        total += tokens
    return result

# 2. Summarization - compress old context
def summarize_context(messages: list, max_tokens: int) -> list:
    """Summarize older messages, keep recent ones."""
    recent = messages[-5:]  # Keep last 5
    older = messages[:-5]

    if older:
        summary = llm.summarize(older)  # Summarize older context
        return [{"role": "system", "content": f"Previous context: {summary}"}] + recent
    return recent

# 3. RAG - retrieve relevant context
def rag_context(query: str, messages: list, max_tokens: int) -> list:
    """Retrieve only relevant past messages."""
    # Embed query
    query_embedding = embed(query)

    # Find relevant messages
    relevant = vector_search(query_embedding, messages, top_k=10)

    return relevant + messages[-3:]  # Relevant + recent

# 4. Sliding window with overlap
def sliding_window(text: str, window_size: int, overlap: int) -> list:
    """Process long text in overlapping chunks."""
    tokens = tokenize(text)
    chunks = []
    for i in range(0, len(tokens), window_size - overlap):
        chunk = tokens[i:i + window_size]
        chunks.append(detokenize(chunk))
    return chunks
```

---

## См. также

- [Prompt Engineering](./01-prompt-engineering.md) — техники создания эффективных промптов для LLM
- [AI-системы в архитектуре](../03-system-design/15-ai-systems.md) — проектирование систем с использованием LLM

---

## На интервью

### Типичные вопросы

1. **How does attention work?**
   - Q, K, V projections from input
   - Attention scores = softmax(QK^T / sqrt(d_k))
   - Output = weighted sum of values
   - Multi-head: parallel attention patterns

2. **What is temperature?**
   - Scales logits before softmax
   - Higher = more random, lower = more deterministic
   - 0 = greedy (always highest prob)

3. **Token vs word?**
   - Token is subword unit (BPE, SentencePiece)
   - "unhappiness" → ["un", "happiness"] or ["un", "hap", "pi", "ness"]
   - ~4 chars or 0.75 words per token in English

4. **Context window limits?**
   - Max tokens model can process at once
   - Attention is O(n²) - quadratic scaling
   - Strategies: truncate, summarize, RAG

5. **Why RLHF?**
   - Align model with human preferences
   - Reduces harmful outputs
   - Improves instruction following

6. **Open vs closed models trade-offs?**
   - Closed: better quality, no infra, privacy concerns
   - Open: full control, privacy, customization, infra costs

---

[← Назад к списку тем](README.md)
