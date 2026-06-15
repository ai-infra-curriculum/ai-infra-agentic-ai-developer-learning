# Resources for mod-101-llm-fundamentals

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as APIs and models evolve.

## Anthropic (Claude)

- **Messages API reference** — request/response shape, parameters, content blocks, errors. <https://docs.anthropic.com/en/api/messages>
- **Streaming the Messages API** — SSE event types (`message_start`, `content_block_delta`, etc.) and code samples. <https://docs.anthropic.com/en/docs/build-with-claude/streaming>
- **Token counting (Count Tokens API)** — the canonical way to count tokens for a Claude model. <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>
- **System prompts** — how the top-level `system` parameter works. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts>
- **Prompt caching** — cache control on stable prefixes, write/read accounting in `usage`. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **Errors and rate limits** — error taxonomy, retry guidance, `429` and `529` behavior, rate-limit headers. <https://docs.anthropic.com/en/api/errors> and <https://docs.anthropic.com/en/api/rate-limits>
- **Models overview** — current model IDs, context windows, capabilities. <https://docs.anthropic.com/en/docs/about-claude/models>
- **Pricing** — per-million-token rates by model. Look up at the time you do exercise-02; do not memorize. <https://www.anthropic.com/pricing>
- **Python SDK** source and changelog — useful when the docs lag behind a release. <https://github.com/anthropics/anthropic-sdk-python>

## OpenAI (GPT family)

- **Chat Completions API reference** — endpoint, parameters, response objects. <https://platform.openai.com/docs/api-reference/chat>
- **Responses API reference** — the newer endpoint with built-in tool calling. <https://platform.openai.com/docs/api-reference/responses>
- **Streaming guide** — `stream=true`, `[DONE]` sentinel, delta chunks. <https://platform.openai.com/docs/guides/streaming-responses>
- **Models overview** — model IDs, context windows, capabilities, deprecation dates. <https://platform.openai.com/docs/models>
- **Tokenizer (`tiktoken`)** — the official encoder for GPT models. <https://github.com/openai/tiktoken>
- **Rate limits** — RPM/TPM, `x-ratelimit-*` headers. <https://platform.openai.com/docs/guides/rate-limits>
- **Error codes** — `400`/`401`/`429`/`5xx` semantics and SDK exception types. <https://platform.openai.com/docs/guides/error-codes>
- **Pricing** — per-million-token rates by model. <https://openai.com/api/pricing/>
- **Python SDK** — <https://github.com/openai/openai-python>

## Standards and infrastructure

- **Server-Sent Events (WHATWG HTML Living Standard, §9.2)** — the wire format every chat-streaming API rides on. <https://html.spec.whatwg.org/multipage/server-sent-events.html>
- **HTTP `Retry-After` header (RFC 9110, §10.2.3)** — authoritative semantics for `Retry-After` on `429` and `503`. <https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after>
- **HTTP status codes (RFC 9110, §15)** — the full list of `4xx` and `5xx` semantics referenced in chapter 8. <https://www.rfc-editor.org/rfc/rfc9110.html#name-status-codes>
- **Idempotency-Key HTTP header (RFC 9457 draft & community deployment)** — pattern background. <https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/>

## Further reading (optional, secondary)

- **Sennrich, Haddow, Birch (2016) — "Neural Machine Translation of Rare Words with Subword Units"** — the original Byte Pair Encoding paper for NMT/tokenization. <https://arxiv.org/abs/1508.07909>
- **Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts"** — the long-context recall finding referenced in chapter 3. <https://arxiv.org/abs/2307.03172>
- **AWS Architecture Blog — "Exponential Backoff and Jitter"** — the standard treatment of full-jitter backoff used in chapter 8. <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>

> Pricing, model IDs, and exact rate-limit numbers change frequently. When an exercise asks for a current value, follow the link, snapshot the URL and the date in your notes, and use that — do not rely on numbers that may be stale in this document.
