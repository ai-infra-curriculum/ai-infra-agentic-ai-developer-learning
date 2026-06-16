# Estimating Cost and Latency

## Why this matters

LLM calls are a *metered, network-bound* operation. Treating them as "free" or "fast" leads directly to runaway invoices and laggy UIs in production. Two skills you need before shipping anything serious: estimate cost per call before you write the call, and estimate latency before users feel it.

## How pricing actually works

For chat-style APIs from the major providers (Anthropic, OpenAI, Google, and most others), the bill is built from three meters:

- **Input tokens** — everything you send: system prompt, tools, messages, images, files. Priced per million tokens.
- **Output tokens** — what the model generates. Priced per million tokens, typically 3–5× the input rate.
- **Optional discounts and surcharges** —
  - **Prompt caching** discounts repeated input prefixes after the first call (often 90% off cached tokens, with a smaller surcharge on the *first* write to the cache).
  - **Batch APIs** discount asynchronous jobs (often ~50% off) at the cost of higher latency (minutes-to-hours).
  - **Image / vision tokens** are billed per image at a model-specific rate.

Specific dollar figures change frequently across providers and tiers; this chapter avoids hard-coded prices and teaches you to read the pricing page and the `usage` field on every response.

## The cost formula you should keep in your head

```
cost_per_call ≈ (input_tokens × price_in) + (output_tokens × price_out)
```

Per million tokens, with `Ti` and `To` as input/output token counts and prices in dollars per million:

```
$ per call ≈ (Ti × $in + To × $out) / 1_000_000
```

To turn that into project economics, multiply by traffic:

```
$ per day = calls_per_day × $ per call
```

If your call has a 4,000-token system prompt and you make a million calls a day, you are paying for 4 billion input tokens *just for the system prompt*. That is exactly the situation prompt caching exists to defuse.

## Where token counts come from

Every chat response carries usage metadata. Read it.

**Anthropic `Message.usage`** includes:

- `input_tokens`
- `output_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`

**OpenAI `response.usage`** includes:

- `prompt_tokens`
- `completion_tokens`
- `total_tokens`
- `prompt_tokens_details.cached_tokens` (when caching applies)

```python
response = client.messages.create(...)
print(response.usage.input_tokens, response.usage.output_tokens)
```

Persist these numbers in your logs, tagged by feature/endpoint. Within a week you will know exactly which features dominate your bill, and the cost discussion will go from anxiety to spreadsheet.

## Cost estimation before you build

Before you write the call, do a fingertip estimate:

1. Write the system prompt and any fixed tool definitions. Count their tokens.
2. Estimate a typical user message length.
3. Estimate a typical reply length (this is what `max_tokens` should reflect).
4. Multiply by your expected daily volume.

A worked example. Suppose:

- System prompt: 1,200 tokens.
- Average user input: 300 tokens.
- Average reply: 500 tokens (your `max_tokens` is 1,024 but you usually don't hit it).
- Volume: 200,000 calls/day.
- Workhorse-tier model with input price `$X_in` and output price `$X_out` per million tokens.

```
daily_input  = 200_000 × (1_200 + 300) = 300M tokens
daily_output = 200_000 × 500           = 100M tokens
daily_cost   = (300 × X_in) + (100 × X_out)     # in dollars, prices per 1M
```

Now: is the system prompt stable across calls? If yes, prompt caching can take the recurring 1,200 tokens × 200,000 calls down by ~90% (after warm-up). That is where most "I cut my bill in half" stories come from.

## Levers to lower cost

In rough order of payoff:

- **Cache stable prefixes.** System prompts, tool schemas, retrieved-but-unchanging documents. The first call writes to the cache; subsequent calls within the TTL pay a fraction.
- **Use a smaller model for routing or easy cases.** (See chapter 5.)
- **Cap `max_tokens` realistically.** A 4,096-token cap that the model never hits is fine for safety. A 4,096-token cap that the model *does* hit silently triples your cost.
- **Trim the system prompt.** It is almost always longer than it needs to be.
- **Batch where you can.** Offline jobs go through the batch API at a meaningful discount.
- **Retrieve, don't stuff.** Sending the whole document costs more and answers worse than sending a few good chunks.
- **Compress conversation history.** Rolling summaries cap the per-turn input cost growth.

## Latency: the three components

End-to-end latency for a chat call is roughly:

```
total_latency ≈ network_RTT + prefill + (output_tokens × time_per_token) + tail_overhead
```

Each piece behaves differently:

- **Network RTT** — your service ↔ the provider's regional endpoint. Tens to low-hundreds of ms.
- **Prefill** — the server processes your *entire* input before generation can start. Scales with input length. Long contexts cost real prefill time. Small models prefill faster.
- **Per-token generation time** — once generation starts, the model emits ~N tokens/sec depending on tier. Output length × this rate is the "talking" portion.
- **Tail overhead** — TLS, queueing, the final response/stop event.

The metric to watch in a UI is **time-to-first-token (TTFT)**, which is dominated by network RTT + prefill. The metric to watch in batch is **total wall time**, dominated by output_tokens × per-token time.

## Practical latency tactics

- **Stream.** Doesn't lower total latency, lowers perceived latency. Default on for chat UIs.
- **Shorten the input.** Less prefill, less TTFT. This is the single biggest TTFT lever.
- **Pick a smaller tier** if quality is adequate. Smaller models prefill and generate faster.
- **Set realistic `max_tokens`.** A high cap doesn't slow you down on its own, but if the model fills it, you pay all of `output_tokens × per-token` time.
- **Co-locate**, where possible — pick the provider region closest to your service to cut RTT.
- **Pre-warm caches.** First call after a cache miss is slower; in low-volume features, periodic keepalives keep the cache hot.
- **Parallelize independent calls.** If a user request needs three independent LLM calls, fire them concurrently.

## Measure, do not guess

Instrument every LLM call with at least:

- Model ID and version.
- `input_tokens`, `output_tokens`, `cache_read_input_tokens` (if applicable).
- TTFT and total wall time.
- `stop_reason`.
- Request ID from the response headers (vital for support tickets).

Once you have those four weeks of data, every cost or latency conversation becomes a query, not an argument.

## Summary

Cost is a linear function of input and output tokens at known per-model rates. Latency is the sum of RTT, prefill, generation, and tail. Both are measurable per call (`usage` field), both are controllable (caching, shorter prompts, smaller tiers, streaming), and both are addressable *before* you ship — write the budget, measure the call, and stop guessing.
