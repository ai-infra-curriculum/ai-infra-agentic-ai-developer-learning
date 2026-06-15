# Resources for mod-105-first-agent

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as APIs, models, and the agent ecosystem evolve.

## Provider tool-use and agent docs

- **Anthropic — Tool use overview** — the call/result protocol this module's loop is built on: `tool_use` blocks, `tool_result` blocks, `stop_reason` values, parallel tool use, `tool_choice`. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use>
- **Anthropic — Building with tool use end to end** — a worked example of multi-step tool calling close to the shape in chapter 3. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview>
- **Anthropic — "Building Effective Agents"** — Anthropic's primary write-up on agent patterns, the typology referenced in chapter 6, and design guidance for keeping single-agent systems honest. <https://www.anthropic.com/engineering/building-effective-agents>
- **Anthropic — Claude Agent SDK** — Anthropic's first-party SDK for shipping agents on top of Claude; useful to read for vocabulary even if you build from raw `messages.create(...)`. <https://docs.anthropic.com/en/docs/claude-agent-sdk>
- **Anthropic — Token counting** — the API for measuring input-token budgets you'll need in chapter 5's transcript policy. <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>
- **OpenAI — Function calling guide** — equivalent protocol for OpenAI: `tool_calls`, JSON-string `arguments`, `role: "tool"` follow-ups, `tool_choice`. <https://platform.openai.com/docs/guides/function-calling>
- **OpenAI — Responses API and Agents SDK** — the current OpenAI shape for multi-step agent applications, including the Agents SDK patterns. <https://platform.openai.com/docs/guides/agents>
- **OpenAI Cookbook — Agents topic** — runnable examples of single-agent and multi-agent patterns on the OpenAI Responses API. <https://cookbook.openai.com/topic/agents>
- **OpenAI — Tokenization (`tiktoken`)** — the tokenizer for OpenAI models you'll use in chapter 5 token-budgeting. <https://github.com/openai/tiktoken>

## ReAct, the original pattern

- **Yao et al. (2022) — "ReAct: Synergizing Reasoning and Acting in Language Models"** — the paper that introduced the Thought / Action / Observation interleaving this module is named after. Worth reading once for context; the modern native-tool-use APIs implement the same pattern without the brittle string parsing. <https://arxiv.org/abs/2210.03629>
- **ReAct project page (Princeton)** — additional materials, prompts, and trajectories from the original paper. <https://react-lm.github.io/>

## Adjacent agent patterns (chapter 6 background)

- **Wei et al. (2022) — "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models"** — the reasoning-only pattern ReAct extends. Useful for understanding the "Thought" slot in isolation. <https://arxiv.org/abs/2201.11903>
- **Shinn et al. (2023) — "Reflexion: Language Agents with Verbal Reinforcement Learning"** — adds a critique-and-retry layer on top of ReAct. The primary reference for the "critique loops" pattern in chapter 6. <https://arxiv.org/abs/2303.11366>
- **Schick et al. (2023) — "Toolformer: Language Models Can Teach Themselves to Use Tools"** — early work on tool-using LLMs without the explicit ReAct framing; useful background. <https://arxiv.org/abs/2302.04761>
- **Wang et al. (2023) — "Voyager: An Open-Ended Embodied Agent with Large Language Models"** — long-horizon autonomous-agent reference; relevant to the "Level 4" category in chapter 6. <https://arxiv.org/abs/2305.16291>

## Agent frameworks (recognize-on-sight)

You do not need to use these to complete the exercises. Read enough of each to map their vocabulary onto the loop you wrote:

- **LangGraph — concepts** — a graph-shaped framework whose concept docs read like a survey of agent architectures (single, supervisor, planner/executor). <https://langchain-ai.github.io/langgraph/concepts/>
- **LangChain — Agents documentation** — older `AgentExecutor` patterns and the newer LangGraph-based agents. <https://python.langchain.com/docs/concepts/agents/>
- **LlamaIndex — Agents documentation** — `ReActAgent`, `FunctionAgent`, and `AgentWorkflow` patterns; explicit ReAct framing in code. <https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/>
- **CrewAI — documentation** — role-and-task framing for multi-agent systems; useful as a contrast to the single-agent shape this module ships. <https://docs.crewai.com/>
- **AutoGen (Microsoft) — documentation** — multi-agent conversation framework; another contrast point. <https://microsoft.github.io/autogen/>

## Memory and conversation management

- **Anthropic — Prompt caching** — the production technique for amortizing a large stable system prompt (including a rendered scratchpad) across many turns. <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- **OpenAI — Prompt caching** — the equivalent for OpenAI. <https://platform.openai.com/docs/guides/prompt-caching>
- **Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts"** — empirical evidence for the long-context attention degradation that motivates chapter 5's transcript-pressure policies. <https://arxiv.org/abs/2307.03172>
- **Model Context Protocol (MCP)** — open protocol for connecting LLMs to external context sources (tools, resources, memory servers). Worth knowing the name; an increasingly common substrate for memory and tool layers. <https://modelcontextprotocol.io/>
- **Packer et al. (2023) — "MemGPT: Towards LLMs as Operating Systems"** — a research take on agents managing their own memory hierarchy; relevant to chapter 5's "scratchpad vs. transcript vs. external store" mental model. <https://arxiv.org/abs/2310.08560>

## Evaluation and observability

- **LangSmith** — tracing, evaluation, and prompt-iteration platform widely used with LangChain / LangGraph agents. Useful as a reference for what production agent observability looks like even if you don't adopt it. <https://docs.smith.langchain.com/>
- **OpenTelemetry — GenAI semantic conventions** — the emerging standard for tracing LLM and agent calls in OpenTelemetry. Worth knowing about before you wire your first tracer. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **Ragas — RAG evaluation framework** — relevant when the agent's retrieval step needs to be evaluated in isolation. <https://docs.ragas.io/>

## Primary references for the chapter examples

- **Anthropic — `tool_choice`** — semantics for `auto`, `any`, `tool`, and `none`, plus parallel-call toggles. <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/tool-choice>
- **OpenAI — `tool_choice`** — equivalent control surface on the OpenAI side. <https://platform.openai.com/docs/api-reference/chat/create#chat-create-tool_choice>
- **Anthropic — Streaming with tool use** — surfaces `input_json_delta` deltas for the agent UI patterns mentioned in chapter 3. <https://docs.anthropic.com/en/docs/build-with-claude/streaming>
- **OpenAI — Streaming** — streaming responses including tool-call deltas. <https://platform.openai.com/docs/guides/streaming-responses>

> Provider docs, framework APIs, and the "agent" vocabulary itself evolve quickly. Where this module shows a specific field name (`tool_use`, `tool_calls`, `stop_reason`, `tool_choice`) or a specific model id, follow the link, confirm against the current docs, and snapshot the URL and date in your code comments. Do not rely on names in this document if they conflict with what you see in the official reference.
