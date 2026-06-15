# mod-102-prompt-engineering: Prompt Engineering & Structured Output

**Estimated effort:** 10 hours

This module turns the LLM API skills from `mod-101` into a repeatable engineering practice: write prompts that hold up, get structured output you can parse, validate at the application boundary, and iterate against simple evals. By the end you should be able to start with a vague task and end with a versioned prompt, a typed output, and a small test suite that catches regressions.

## Learning objectives

- Apply core prompting techniques: instructions, role, few-shot examples, decomposition, chain-of-thought.
- Get reliable structured output (JSON / schema-constrained) from models using the right strategy per situation.
- Parse and validate model output at the application boundary so the rest of your code can treat the LLM as a typed function.
- Iterate on prompts with a small golden set and simple evaluators — turn "I improved the prompt" into a number.

## Lecture chapters

1. [Prompting as Engineering](01-prompting-as-engineering.md) — prompts as code, the iteration loop, what a "prompt" actually controls.
2. [Anatomy of an Effective Prompt](02-anatomy-of-an-effective-prompt.md) — role, instruction, context, constraints, format, examples; where each block goes.
3. [Few-Shot Prompting](03-few-shot-prompting.md) — when examples beat instructions, how to pick and order them.
4. [Decomposition, Chain-of-Thought, and Multi-Step Prompts](04-decomposition-and-reasoning.md) — splitting hard tasks across calls vs. asking the model to reason in one.
5. [Strategies for Reliable Structured Output](05-structured-output-strategies.md) — prompt-only, JSON mode, schema-constrained, and tool-calling, side by side.
6. [Parsing and Validating Output at the Boundary](06-parsing-and-validating-output.md) — the untrusted-input mental model, typed validators, repair loops.
7. [Iterating on Prompts with Simple Evals](07-iterating-with-evals.md) — golden sets, cheap evaluators, LLM-as-judge, regression testing in CI.

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: Prompt patterns](exercises/exercise-01-prompt-patterns.md) — apply role, instruction, few-shot, and decomposition to a small extraction/classification task.
- [exercise-02: Structured JSON output](exercises/exercise-02-structured-json-output.md) — compare prompt-only, JSON mode, schema-constrained, and tool-calling approaches on the same task.
- [exercise-03: Output validation](exercises/exercise-03-output-validation.md) — build a validation boundary, a small repair loop, and a golden-set eval that gates change.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise. The chapters intentionally stay short because the value lands when you run a real prompt against a real provider, watch your output break in a real way, and iterate against a small eval set you can actually trust. Bring a paid API key for at least one provider (Anthropic and/or OpenAI), Python 3.10+ (or Node 20+), and a willingness to delete prompts that did not work.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
