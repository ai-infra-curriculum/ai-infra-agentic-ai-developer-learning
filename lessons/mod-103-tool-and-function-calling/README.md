# mod-103-tool-and-function-calling: Tool & Function Calling

**Estimated effort:** 10 hours

This module turns the model from a text generator into the planner of a small, auditable program. You will design tool definitions the model can read, wire Python functions behind them, run the call/result loop end-to-end, and harden the surface so that malformed arguments and downstream failures become recoverable feedback instead of crashes. By the end you should be able to take a vague task ("answer support questions using our order system") and ship a tool-using app whose behavior you can predict, audit, and defend.

## Learning objectives

- Define tools/functions and let the model choose and call them.
- Wire Python functions as tools and handle the call/result loop.
- Reason about when a tool call is appropriate vs. plain generation.
- Handle tool errors and malformed arguments safely.

## Lecture chapters

1. [From Generation to Action](01-from-generation-to-action.md) — what tool calling is, why it beats "just prompt for it," the mental model the rest of the track is built on.
2. [Anatomy of a Tool Definition](02-anatomy-of-a-tool-definition.md) — name, description, JSON Schema for parameters, and the design choices that make a tool the model uses well.
3. [The Call/Result Loop](03-call-and-result-loop.md) — the multi-turn protocol end to end, with code for Anthropic and OpenAI, parallel calls, stop reasons, and step caps.
4. [Wiring Python Functions as Tools](04-wiring-python-functions.md) — the schema↔function contract, a small registry, dispatch, and testing tools without the model.
5. [When to Call a Tool vs. Generate](05-when-to-call-a-tool.md) — the decision criteria, `tool_choice`, multi-tool selection, side effects, and the latency/cost tradeoffs.
6. [Handling Tool Errors and Malformed Arguments](06-tool-errors-and-malformed-arguments.md) — the four failure modes, validating arguments at the boundary, error-as-feedback, and retry budgets.

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: First function calling](exercises/exercise-01-first-function-calling.md) — define one tool, run the call/result loop, and ship a working single-tool app.
- [exercise-02: Multi-tool app](exercises/exercise-02-multi-tool-app.md) — expose several tools to the model, exercise selection and `tool_choice`, observe parallel calls.
- [exercise-03: Tool error handling](exercises/exercise-03-tool-error-handling.md) — break tools on purpose, see the model recover, and add validation + a retry budget.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise. The chapters intentionally stay tight because the value lands when you run a real loop against a real provider, watch the model do something wrong, and tighten a schema or an error payload to fix it. Bring a paid API key for at least one provider (Anthropic and/or OpenAI), Python 3.10+, and a willingness to throw away your first dispatcher.

`mod-104` (retrieval) and `mod-105` (first agent) build directly on this protocol. The cleaner your loop is at the end of this module, the easier those will be.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
