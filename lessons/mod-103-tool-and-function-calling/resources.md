# Resources for mod-103-tool-and-function-calling

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as APIs and models evolve.

## Anthropic (Claude)

- **Tool use overview** — the canonical reference for how Claude advertises tools, emits `tool_use` blocks, and consumes `tool_result` blocks. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **How to implement tool use** — step-by-step walk-through of the request/response loop with code samples. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/implement-tool-use>
- **Controlling tool use with `tool_choice`** — `auto`, `any`, `tool`, `none`, and `disable_parallel_tool_use`. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/implement-tool-use#controlling-claudes-output>
- **Token-efficient tool use** — guidance on schema-token cost and how to keep tool definitions lean. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/token-efficient-tool-use>
- **Messages API reference** — request/response shape, content blocks, `stop_reason`, `tool_use` / `tool_result` blocks. <https://docs.anthropic.com/en/api/messages>
- **Stop reasons** — full enumeration including `tool_use`, `end_turn`, `max_tokens`, `stop_sequence`. <https://docs.anthropic.com/en/api/handling-stop-reasons>
- **Errors and rate limits** — taxonomy and retry guidance you will reuse in the safe dispatcher. <https://docs.anthropic.com/en/api/errors>
- **Python SDK source and changelog** — useful when documented field names lag a release. <https://github.com/anthropics/anthropic-sdk-python>

## OpenAI (GPT family)

- **Function calling guide** — how to define `tools` of `type: "function"`, force calls with `tool_choice`, and consume `tool_calls` on the assistant message. <https://platform.openai.com/docs/guides/function-calling>
- **Function calling — parallel function calling** — when the model can emit multiple `tool_calls` per turn and how to disable it. <https://platform.openai.com/docs/guides/function-calling#parallel-function-calling>
- **Structured Outputs guide** — `response_format` with `json_schema`, strict mode, supported JSON Schema subset; useful when reading how OpenAI constrains tool argument shapes. <https://platform.openai.com/docs/guides/structured-outputs>
- **Chat Completions API reference** — `messages`, `tools`, `tool_calls`, `tool_choice`, response shapes. <https://platform.openai.com/docs/api-reference/chat>
- **Responses API reference** — the newer endpoint with first-class tool support. <https://platform.openai.com/docs/api-reference/responses>
- **OpenAI Cookbook — Function calling with the OpenAI API** — runnable end-to-end examples. <https://cookbook.openai.com/examples/how_to_call_functions_with_chat_models>
- **OpenAI Cookbook — Structured Outputs intro** — covers the schema-constrained alternative referenced in chapter 5 of `mod-102`. <https://cookbook.openai.com/examples/structured_outputs_intro>
- **Python SDK** — <https://github.com/openai/openai-python>

## Validation, schemas, and parsing

- **Pydantic — Models and validation** — the validator chapter 4 and chapter 6 rely on. <https://docs.pydantic.dev/latest/concepts/models/>
- **Pydantic — JSON Schema generation** (`model_json_schema`) — produces the schema you can hand to a structured-output API or a tool definition. <https://docs.pydantic.dev/latest/concepts/json_schema/>
- **JSON Schema specification** — the spec your provider's strict mode implements a subset of. Useful when you hit "unsupported keyword" errors. <https://json-schema.org/specification>
- **JSON Schema 2020-12 draft** — the most commonly referenced draft. <https://json-schema.org/draft/2020-12/release-notes>

## Adjacent protocols and ecosystem

- **Model Context Protocol (MCP) specification** — the open protocol several SDKs use to wire tools and resources into model clients; useful background when you see "MCP server" referenced in vendor docs. <https://modelcontextprotocol.io/specification>
- **OpenAPI Specification (3.1.0)** — JSON Schema-aligned spec used by many tool-routing libraries when generating tool definitions from existing HTTP APIs. <https://spec.openapis.org/oas/v3.1.0>
- **HTTP `Idempotency-Key` header (IETF draft)** — background for the idempotency-key pattern in chapter 4 and in the `create_refund` tool. <https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/>

## Foundational papers (optional, secondary)

- **Schick et al. (2023) — "Toolformer: Language Models Can Teach Themselves to Use Tools"** — the early result showing models can learn tool invocation. Useful background for how today's tool-calling APIs evolved. <https://arxiv.org/abs/2302.04761>
- **Yao et al. (2023) — "ReAct: Synergizing Reasoning and Acting in Language Models"** — the reasoning-then-action loop that the chapter-3 protocol is the production version of; bridges directly into `mod-105`. <https://arxiv.org/abs/2210.03629>
- **Patil et al. (2023) — "Gorilla: Large Language Model Connected with Massive APIs"** — points to the failure modes (hallucinated arguments, wrong endpoints) that chapter 6's validation layer addresses. <https://arxiv.org/abs/2305.15334>

> Provider docs evolve quickly. Where this module shows a specific JSON shape (e.g., `tools`, `tool_choice`, `input_schema`, `tool_calls`), follow the link, confirm against the current docs, and snapshot the URL and date in your code comments. Do not rely on field names in this document if they conflict with what you see in the official reference.
