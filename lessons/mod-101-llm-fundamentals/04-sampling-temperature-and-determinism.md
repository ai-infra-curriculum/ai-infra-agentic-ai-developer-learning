# Sampling, Temperature, and Determinism

## Why this matters

A model produces a probability distribution over the next token. A *sampler* turns that distribution into a single chosen token. The sampler is configurable, and the configuration is one of the few knobs that meaningfully changes model behavior without touching the prompt. If you do not understand the knobs, you will tune them by superstition.

## Greedy vs. random sampling

Two extremes:

- **Greedy decoding** — always pick the highest-probability token. Maximally repetitive; can get stuck in loops; useful when you need a confident, conservative answer.
- **Random sampling** — sample from the full distribution. Maximum variety; maximum chance of going off the rails.

Real samplers sit between, shaping the distribution before drawing from it.

## Temperature

Temperature `T` rescales the model's logits before softmax:

```
softmax(logits / T)
```

The effect:

- `T → 0`: the distribution collapses onto the top token. Behavior approaches greedy.
- `T = 1`: the distribution is the model's "raw" output.
- `T > 1`: flattens the distribution. Low-probability tokens become more likely. Output gets weirder.

Notes on the parameter range:

- **Anthropic**: `temperature` is in `[0.0, 1.0]`; default `1.0`.
- **OpenAI**: `temperature` is in `[0.0, 2.0]`; default `1.0`.

That mismatch matters: `temperature=1.5` is invalid for Anthropic and merely "high" for OpenAI. Encode the legal range per provider when you build adapters.

A simple heuristic: use `temperature=0` (or close to it) for tasks where there is *one right answer* (classification, structured extraction, code transforms), and higher values for tasks where variety is the point (brainstorming, creative writing, paraphrasing).

## top_p and top_k

`top_p` (nucleus sampling) restricts the draw to the smallest set of tokens whose cumulative probability is ≥ `p`. `top_p=0.9` means "consider just the head of the distribution; ignore the long tail."

`top_k` keeps only the top `k` tokens. Some APIs expose it; others do not.

Conventional advice: **tune one of `temperature` or `top_p`, not both.** Composing the two is hard to reason about. Pick a default and leave the other alone.

## "Deterministic" sampling

A common misconception: `temperature=0` guarantees identical output for identical input. It does not. Reasons:

- Token-tie breaking is implementation-defined and may not be reproducible across servers.
- Floating-point math on GPUs is non-associative; minor batching changes can shift logits enough to swap the top token.
- Some providers route requests across heterogeneous hardware that produces tiny numeric differences.
- Server-side updates (new tokenizer build, new model snapshot) silently change outputs.

OpenAI exposes a `seed` parameter and a `system_fingerprint` in the response to give you *more* reproducibility — same seed + same fingerprint + same request → likely the same output — but they explicitly call it best-effort, not a guarantee.

For tests and evals, pin to the lowest temperature your provider allows, set a seed if available, and **also** assert on structure or semantics, not exact strings.

## stop_sequences and max_tokens

Two more knobs that shape the *length* of the output:

- **`max_tokens`** — a hard cap on the generated reply. The model stops there regardless of whether the answer is complete. Always set it. "Whatever the model decides" is not a budget; it is a footgun.
- **`stop_sequences`** — strings that, when emitted, end generation. Useful when you are streaming structured output and want to cut off after a known marker (e.g., `\n###`).

When generation stops, the response includes a **stop reason**:

- `end_turn` / `stop` — the model produced its end-of-turn token. Healthy.
- `max_tokens` — you ran out of budget. The reply is *truncated*. Either raise `max_tokens` or ask for a shorter answer.
- `stop_sequence` — a stop string matched.
- `tool_use` — the model called a tool (Anthropic). Covered in `mod-103`.

Always inspect the stop reason. Treating a truncated reply as complete is a common, ugly bug.

## A side-by-side

```python
# Anthropic
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    temperature=0,         # range [0.0, 1.0]
    stop_sequences=["\n###"],
    messages=[...],
)

# OpenAI
client.chat.completions.create(
    model="gpt-4.1-mini",
    max_tokens=1024,
    temperature=0,         # range [0.0, 2.0]
    top_p=1,
    stop=["\n###"],
    seed=42,               # best-effort determinism
    messages=[...],
)
```

## A simple decision table

| Task                                  | Suggested starting point |
| ------------------------------------- | ------------------------ |
| Structured extraction / classification | `temperature=0`          |
| Code generation                        | `temperature=0` – `0.2`  |
| Q&A over retrieved docs                | `temperature=0` – `0.3`  |
| Summarization                          | `temperature=0.2` – `0.5`|
| Brainstorming, ideation                | `temperature=0.7` – `1.0`|
| Creative writing                       | `temperature=0.7` – `1.0`|

These are *starting points*, not gospel. Move the dial and look at outputs.

## Summary

A sampler turns the model's next-token distribution into a chosen token. `temperature` rescales how peaky that distribution is; `top_p` clips its tail; `max_tokens` and `stop_sequences` bound the length. "Deterministic" sampling is a best-effort property, not a guarantee — pin everything you can and assert on meaning, not bytes. Inspect the stop reason on every response.
