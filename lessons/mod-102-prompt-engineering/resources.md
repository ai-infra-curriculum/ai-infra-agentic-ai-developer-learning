# Resources for mod-102-prompt-engineering

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as APIs and models evolve.

## Anthropic (Claude)

- **Prompt engineering overview** — Anthropic's index of techniques: clarity, role, examples, chain-of-thought, prefilling, output structure. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview>
- **Be clear, direct, and detailed** — guidance for instruction-style prompts. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-direct>
- **Use examples (multishot prompting)** — Anthropic's few-shot guidance. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting>
- **System prompts** — how the top-level `system` parameter works on the Messages API. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts>
- **Prefill Claude's response** — controlling the start of the assistant turn (the trick used for structured-output stability in chapter 5). <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prefill-claudes-response>
- **Chain of thought prompting** — when and how to ask Claude to reason. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought>
- **Increase output consistency / JSON mode** — Anthropic's recommended approach for structured output (tool use + `tool_choice`). <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/increase-consistency>
- **Tool use overview** — the protocol that doubles as a portable structured-output mechanism in chapter 5. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **Messages API reference** — request/response shape, content blocks, errors. <https://docs.anthropic.com/en/api/messages>

## OpenAI (GPT family)

- **Prompt engineering guide** — OpenAI's overview of prompt techniques and structure. <https://platform.openai.com/docs/guides/prompt-engineering>
- **Structured Outputs guide** — `response_format` with `json_schema`, strict mode, supported JSON Schema subset, refusals. <https://platform.openai.com/docs/guides/structured-outputs>
- **Function calling guide** — how to define and force a function call; foundational to the tool-calling-for-structured-output pattern in chapter 5. <https://platform.openai.com/docs/guides/function-calling>
- **JSON mode** — the lighter-weight `response_format={"type":"json_object"}` flag (compared to schema-constrained). <https://platform.openai.com/docs/guides/structured-outputs#json-mode>
- **Chat Completions API reference** — request/response objects, parameters. <https://platform.openai.com/docs/api-reference/chat>
- **Responses API reference** — the newer endpoint with built-in structured-output and tool support. <https://platform.openai.com/docs/api-reference/responses>
- **Cookbook — Structured Outputs** — runnable examples covering schemas, refusals, and edge cases. <https://cookbook.openai.com/examples/structured_outputs_intro>

## Validation, schemas, and parsing

- **Pydantic** — Python data validation with JSON Schema generation. Used throughout chapter 6. <https://docs.pydantic.dev/latest/>
- **`pydantic.BaseModel.model_json_schema()`** — produces the schema you can hand to a structured-output API. <https://docs.pydantic.dev/latest/concepts/json_schema/>
- **zod** — TypeScript data validation with `infer` types and `.parse()` / `.safeParse()`. <https://zod.dev/>
- **JSON Schema specification** — the spec your provider's strict mode implements a subset of. Useful when you hit "unsupported keyword" errors. <https://json-schema.org/specification>
- **JSON Schema 2020-12 draft** — the most commonly referenced draft. <https://json-schema.org/draft/2020-12/release-notes>

## Evaluation and prompt iteration

- **OpenAI Evals (open source)** — framework for defining and running evaluations against language models. <https://github.com/openai/evals>
- **OpenAI Cookbook — Getting started with OpenAI Evals** — a worked introduction. <https://cookbook.openai.com/examples/evaluation/getting_started_with_openai_evals>
- **Anthropic — Reduce hallucinations** — practical, eval-aware techniques referenced in chapter 7's discussion of grounding. <https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/reduce-hallucinations>
- **promptfoo** — open-source tool for evaluating prompts and models against test sets. Useful starting point if you outgrow a hand-rolled runner. <https://www.promptfoo.dev/docs/intro/>

## Foundational papers (optional, secondary)

- **Brown et al. (2020) — "Language Models are Few-Shot Learners" (GPT-3)** — the paper that coined "in-context learning" and put few-shot prompting on the map. <https://arxiv.org/abs/2005.14165>
- **Wei et al. (2022) — "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models"** — the original chain-of-thought finding referenced in chapter 4. <https://arxiv.org/abs/2201.11903>
- **Kojima et al. (2022) — "Large Language Models are Zero-Shot Reasoners"** — the "Let's think step by step" zero-shot result. <https://arxiv.org/abs/2205.11916>
- **Zhao et al. (2021) — "Calibrate Before Use: Improving Few-Shot Performance of Language Models"** — example-ordering and label-bias effects referenced in chapter 3. <https://arxiv.org/abs/2102.09690>
- **Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts"** — long-context recall finding referenced in chapter 2's "burying the lede" point. <https://arxiv.org/abs/2307.03172>
- **Zheng et al. (2023) — "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"** — bias and calibration considerations for LLM-as-judge evaluators referenced in chapter 7. <https://arxiv.org/abs/2306.05685>
- **Yao et al. (2023) — "ReAct: Synergizing Reasoning and Acting in Language Models"** — bridge between this module's "decomposition" patterns and the agent loops in `mod-105`. <https://arxiv.org/abs/2210.03629>

> Provider docs evolve quickly. Where this module shows a specific JSON shape (e.g., `response_format`, `tools`, `tool_choice`), follow the link, confirm against the current docs, and snapshot the URL and date in your code comments. Do not rely on field names in this document if they conflict with what you see in the official reference.
