# mod-105-first-agent: Your First Agent: The Reason-Act Loop

**Estimated effort:** 10 hours

This module turns the call/result loop from `mod-103` and the retriever from `mod-104` into your first agent: a single model that decides, turn by turn, whether to act, retrieve, or answer. You will implement the ReAct (Thought / Action / Observation) loop on top of native tool-use APIs, wire your knowledge-base retriever in as one tool among several, add multi-turn conversation memory with a small key:value scratchpad, and finish by drawing an honest line around what a single agent can and can't do. By the end you should be able to ship a small support-style agent whose every step you can read in a trace, defend in a review, and bound in cost and latency.

## Learning objectives

- Understand and implement a basic ReAct (Thought / Action / Observation) loop.
- Combine tools + retrieval into a single simple agent.
- Add basic memory across turns.
- Recognize the limits of a single agent (handoff to the Engineer track).

## Lecture chapters

1. [From Tool Loop to Agent](01-from-tool-loop-to-agent.md) — what changes when the model decides what the next step is; why "agent" is the same call/result loop with a longer leash.
2. [The ReAct Pattern: Thought, Action, Observation](02-the-react-pattern.md) — the original prompted form vs. the modern native-tool-use form; what the slots map to in current APIs.
3. [Implementing the ReAct Loop](03-implementing-the-react-loop.md) — a ~80-line Python loop with native tool use: stop reasons, step caps, errors-as-observations, parallel calls, surfacing intermediate text.
4. [Retrieval as a Tool](04-retrieval-as-a-tool.md) — wrapping the `mod-104` retriever as `search_kb`; combining retrieval with action tools; iterative refinement on weak hits; citation validation across multiple retrievals.
5. [Conversation Memory](05-conversation-memory.md) — the running transcript, transcript-pressure (drop / window / summarize), and a small key:value scratchpad exposed via `remember` / `recall` tools.
6. [Limits of a Single Agent](06-limits-of-a-single-agent.md) — loops, drift, planning depth, tool-surface bloat, verification; where the Engineer track picks up.

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: ReAct loop intro](exercises/exercise-01-react-loop-intro.md) — build the minimal ReAct loop over two toy tools, surface every Thought, and read your first traces.
- [exercise-02: Tool-using agent](exercises/exercise-02-tool-using-agent.md) — wire your `mod-104` retriever in alongside two action tools; observe routing, iterative retrieval, and parallel calls.
- [exercise-03: Basic conversation memory](exercises/exercise-03-basic-conversation-memory.md) — maintain a multi-turn transcript, add a token-budget policy, and add a key:value scratchpad with `remember` / `recall` tools.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise. The chapters stay tight because the value lands when you watch a real trace go five steps deep, see the model second-guess a weak retrieval, and tighten one tool description to fix it. Bring a paid API key for at least one provider (Anthropic is the path of least resistance for the trace examples in chapter 3; OpenAI works equally well), Python 3.10+, and the retriever you built in `mod-104` exercise-02.

`mod-103` is a hard prerequisite: this module is the same call/result loop with a longer trajectory and more guardrails. If your `mod-103` exercise-02 dispatcher was creaky, fix it before chapter 3 — every problem you defer becomes harder once the loop runs five times per request.

`mod-106` (deployment) turns the agent into a service: a request handler, a step-cap and rate-limit policy, an observability story (per-step traces, token spend, latency), and a way to ship prompt changes without redeploying the world.

`mod-201`+ (Engineer track) is where the limits in chapter 6 stop being limits — planner/executor, supervisor/workers, critique loops, durable memory. Knowing the single-agent ceiling cold is the prerequisite for graduating cleanly.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
