# exercise-02: Tokens, Context, and Cost

**Estimated effort:** 2 hours

## Objective

Build the habits that keep an LLM project's bill predictable: count tokens with official tools, write a defensible context budget for a real request, and project cost from measured `usage` data — not from hand-waving.

## Background

This exercise covers material from:

- [Chapter 3 — Tokens and Context Windows](../03-tokens-and-context-windows.md)
- [Chapter 5 — Choosing a Model](../05-choosing-a-model.md)
- [Chapter 7 — Estimating Cost and Latency](../07-cost-and-latency.md)

## Prerequisites

- Completed [exercise-01](exercise-01-first-llm-api-calls.md).
- For Anthropic: access to the Messages Count Tokens endpoint (any standard API key).
- For OpenAI: `pip install tiktoken`.
- A short corpus of text (~10,000 words) to experiment with. A few public-domain Project Gutenberg chapters work well. Save it as `corpus.txt` next to your code.

## Tasks

### 1. Count tokens accurately

Write a function `count_tokens(text, model) -> int` that returns the exact input-token count for a given text *for the specified model*. Implement it for **at least one** of:

- Anthropic: call `client.messages.count_tokens(model=..., messages=[{"role":"user","content":text}])`.
- OpenAI / GPT family: use `tiktoken.encoding_for_model(model)` and `.encode(text)`.

Then:

- Measure `count_tokens("Hello, world.", model)`.
- Measure your `corpus.txt`.
- Measure a chunk of code (paste in any open-source source file).
- Measure a JSON blob (about 1 KB).
- Compare against the rule-of-thumb estimate `len(text) / 4`. Report where the rule of thumb is best and worst.

### 2. Token-budget calculator

Write a script that takes:

- A `system_prompt.txt` file.
- A `tools.json` file (a list of tool definitions; if your provider doesn't use tools yet, simulate this with a JSON-string of pretend definitions).
- A target model and its published context window size.
- A target `max_tokens` for the reply.

It should print:

```
Model:          claude-sonnet-4-6
Window:         200000
System:         812
Tools:          1204
Fixed overhead: 2000   (assumed)
Reserved reply: 4096
-------------------------------------
Available for messages + retrieval: 191888 tokens
```

Bonus: have the script accept an arbitrary number of additional files representing retrieved documents, and report how many of them fit under the budget.

### 3. Measure real `usage` data

Modify your exercise-01 chat program so that on every response it prints:

- Model ID.
- `input_tokens` and `output_tokens` from the response usage.
- The ratio `output_tokens / max_tokens` (how close you came to the cap).
- The `stop_reason`.

Run at least 20 real chat turns and collect the numbers (a CSV is fine). Save them as `usage_log.csv`.

### 4. Project cost from usage

Using your `usage_log.csv`:

- Look up the **current** input and output prices for the model you used. Pull them from the provider's official pricing page and save the snapshot URL + date in a comment.
- Compute the cost of each row, the total cost of all 20 turns, and the projected daily cost at 100K, 1M, and 10M calls per day at this average size.
- Identify the single biggest token contributor across the run (system prompt? user input? assistant output?). Quantify it.

### 5. Try one optimization

Pick one and demonstrate the improvement with numbers:

- **Trim the system prompt** by half. Re-run a sample of the conversation. Did quality change? Did `input_tokens` change as expected?
- **Cap `max_tokens` lower.** Re-run. Did your responses get truncated (check `stop_reason`)?
- **Enable prompt caching** for a stable system prompt (Anthropic Messages with cache_control, or OpenAI prompt caching where supported). Make 5 successive calls and observe the cached vs. uncached input token counts in the response usage.

## Starter guidance

For the token counter, the canonical Anthropic call is:

```python
from anthropic import Anthropic
client = Anthropic()
count = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system="You are a helpful assistant.",
    messages=[{"role": "user", "content": text}],
)
print(count.input_tokens)
```

For OpenAI / GPT models:

```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4.1-mini")
print(len(enc.encode(text)))
```

For the budget calculator, structure it as a simple class so you can extend it later:

```python
class ContextBudget:
    def __init__(self, *, window, reserved_reply, overhead):
        ...
    def reserve(self, label, tokens):
        ...
    def remaining(self) -> int:
        ...
    def fit(self, items: list[tuple[str, int]]) -> list[str]:
        ...
```

For the cost projection, do *not* hardcode the prices in this document — look them up at the time you do the exercise and cite the source in your code.

## Acceptance criteria

You can demonstrate:

- A working `count_tokens()` that matches the official tool's count exactly (not approximated).
- A token-budget script that reports remaining headroom given system, tools, overhead, and reserved reply, and refuses to fit more than the available budget allows.
- A `usage_log.csv` from at least 20 real API calls with input/output token counts, model ID, and stop reasons.
- A cost projection table at 100K, 1M, and 10M calls/day using *current* published prices (cited).
- A measured before/after comparison of one optimization with both token-count and quality observations.

## Reflection

Answer briefly:

1. For the four text types you measured (English prose, code, JSON, your corpus), where did the "4 chars per token" rule break down?
2. Where in your code is `max_tokens` set? Is it set to a value that fits your typical reply, or is it artificially high?
3. If your system prompt is byte-identical across calls, which provider's caching feature would help, and how would you verify it from the `usage` field?

## Stretch goals

- Plot a histogram of input vs. output tokens from your log.
- Write a "context overrun" simulator: feed in growing conversation histories until your assertion fires, then implement a rolling summarization fallback.
- Compare token counts for the *same* text across two models in different families. Explain the differences.
- Estimate the per-image token cost for a vision model by sending the same image at 3 different resolutions.
