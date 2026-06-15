# Choosing a Model

## Why this matters

"Which model should I use?" is the most common question on every new LLM project, and the most common cause of wasted spend. The right answer is rarely "the biggest one." It is "the smallest one that demonstrably passes the task at acceptable cost and latency." This chapter is a decision framework, not a tier list.

## The five axes to compare

Hold these in your head when evaluating a model for a specific job:

1. **Capability** — does it actually solve this task? Reasoning depth, instruction following, code quality, vision, tool use, long-context recall.
2. **Cost** — input and output price per million tokens. They are *not* the same number; output is usually 3–5× input.
3. **Latency** — time-to-first-token (TTFT) and output throughput (tokens/sec). Different from cost.
4. **Context window** — total tokens per request. Big window does not mean uniform recall (see chapter 3).
5. **Constraints** — provider availability in your region, data residency, retention policy, fine-tuning support, rate-limit headroom, deprecation horizon.

A "good" model is one that passes capability for your task while sitting at the lowest acceptable point on cost and latency.

## Tiers, not specific names

Every major provider organizes their lineup into roughly three tiers:

- **Flagship / "Opus" / "GPT" class** — strongest reasoning, slowest, most expensive. Right for hard problems and offline batch.
- **Workhorse / "Sonnet" / "GPT mini" class** — strong, fast enough for interactive use, mid-priced. The pragmatic default for most production features.
- **Cheap & fast / "Haiku" / "nano" / "small" class** — best-in-class latency and cost, weaker on multi-step reasoning. Right for classification, extraction, routing, and high-volume work.

Specific model IDs change every few months. The tier structure has been stable across releases. Match your task to a *tier* first, then pick the current model ID in that tier.

## A practical sequence

When you cannot decide, try this order:

1. **Start with the workhorse tier**, default settings (`temperature=0` for deterministic tasks).
2. **Evaluate on a fixed test set** of 10–50 realistic examples. Score them — even by hand, even just "good/ok/bad."
3. **If it passes, try the cheap tier.** Score again. If it still passes, ship it.
4. **If it fails, try the flagship tier.** If *that* passes, you have a real problem and a real cost; design around it (caching, batching, distillation, fewer calls per request).
5. **If even flagship fails**, the problem is your prompt, your context, or your task definition. No model upgrade will fix it.

Most teams skip the "try cheaper" step and overspend by 5–10×. Do not skip it.

## A worked routing example

A common architecture in agent and chatbot systems is *model routing*: a cheap model classifies the request, then either answers it directly or hands off to a heavier model.

```python
from anthropic import Anthropic

client = Anthropic()

def classify(user_text: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=16,
        temperature=0,
        system='Classify the request as one of: "simple", "complex". Respond with one word.',
        messages=[{"role": "user", "content": user_text}],
    )
    return response.content[0].text.strip().lower()

def answer(user_text: str) -> str:
    tier = classify(user_text)
    model = "claude-haiku-4-5" if tier == "simple" else "claude-sonnet-4-6"
    response = client.messages.create(
        model=model,
        max_tokens=1024,
        messages=[{"role": "user", "content": user_text}],
    )
    return response.content[0].text
```

This pattern works because most production traffic is easy ("What time is your office open?") and only a minority needs the bigger model. The classifier itself costs a few tokens.

## Cost vs. latency are different problems

A common mistake: assume the cheap model is also fast. It usually is, but not always. Latency depends on:

- **Tier and architecture** of the model.
- **Where you are calling from** — the round-trip to your provider's region.
- **Whether streaming is on** — affects perceived latency dramatically (chapter 6).
- **Prefill vs. generation** — long inputs increase TTFT regardless of model tier.

If you have a soft latency target (e.g., "first token in under 800 ms"), you need to measure it under real traffic shapes, not assume the cheaper model meets it.

## Capability gotchas

A few non-obvious failure modes worth knowing before you commit:

- **Long-context recall is not uniform.** A 200K window does not mean the model uses every token equally well. Test with realistic placement of the answer-bearing chunk.
- **Vision and tool use are model-dependent.** Some tiers have them, some do not. Read the model card.
- **Reasoning models trade latency for quality.** Some models (e.g., "thinking" or "reasoning" modes) take noticeably longer per request because they generate intermediate reasoning tokens internally. Their cost reflects those tokens.
- **Knowledge cutoffs.** Each model has a training cutoff. If your task depends on recent facts, you need retrieval, not a smarter model.

## Pinning models in production

Use **exact model IDs**, not aliases like "latest". An alias that silently re-points to a new model will change your output and your bill. Read your provider's deprecation policy; switch model IDs deliberately and re-run your evals when you do.

## Summary

Pick a tier first (cheap, workhorse, flagship) and a model ID inside it second. Default to the workhorse, try the cheap tier on a real test set, and only pay for flagship when you have demonstrated need. Treat capability, cost, latency, context, and constraints as *separate* axes — they trade against each other, and you cannot optimize them all at once.
