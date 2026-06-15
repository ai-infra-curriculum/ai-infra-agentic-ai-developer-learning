# Prompting as Engineering

## Why this matters

"Prompt engineering" sounds like a tongue-in-cheek phrase, but treat it as engineering or you will pay for it. A prompt is the input to a non-deterministic, version-pinned, billed-by-the-token function. It is code that runs in the model. If you treat it like a sticky note — edited in place, untested, undocumented — your app will degrade silently the next time anyone changes the model, the temperature, or the surrounding context.

This module gives you the loop: how to write prompts that hold up, how to get structured output you can parse, how to validate at the boundary, and how to know whether a change you just made actually helped.

## What a prompt actually controls

Recall from `mod-101`: an LLM is a function over tokens. A prompt is the **only** thing on the input side you fully control. The model weights are fixed once you pin a model ID. The sampler is your knob. The output is what you measure.

So a "prompt" in this module is wider than a single user message. It is every input that conditions a model call:

- The **system** instructions (policy, persona, output format).
- The **user** message (the actual task).
- Any **assistant** prefill (a forced opening of the reply).
- Any **few-shot** turns the model sees as worked examples.
- The **tool / function schemas** you advertise (covered in `mod-103`).
- The **sampling parameters** (`temperature`, `top_p`, `max_tokens`, `stop_sequences`) that decode the output.

Change any of those and you have changed the experiment. Most "the model regressed" reports are really "the request changed and nobody noticed."

## A prompt is code; treat it that way

Three habits separate prompts that survive from prompts that rot:

1. **Version them.** Prompts belong in your repo, not in a Notion page. Diff them in code review. If the prompt is built from templates and retrieved snippets, log the final rendered string for every production call — that is the artifact you will actually want when you debug.
2. **Pin the surroundings.** Record the model ID (the exact dated snapshot when one is available), the sampler settings, and the prompt version that produced a given output. "We used Claude" is not a reproducible experiment. "We used `claude-sonnet-4-6`, `temperature=0`, prompt version `v3`" is.
3. **Test them.** Build a small set of inputs with known expected outputs. Re-run it after any change. This is what chapter 7 calls "evals," and it is the cheapest insurance policy you will buy on an LLM project.

If you ever find yourself thinking "I tweaked the wording and it feels better," stop. Without a golden set, you do not know. You may have just made it worse on the inputs you did not check.

## The engineering loop

A productive prompting workflow looks like this:

```
sketch  ->  test on a few real inputs  ->  inspect outputs
   ^                                                |
   |                                                v
   +-----  edit prompt / sampler / structure  <-----+
```

Some practical implications:

- **Start zero-shot.** Try the simplest prompt that could possibly work — a clear instruction and the input. If that is good enough, you are done; you will not waste tokens shipping few-shot examples nobody needed.
- **Add structure only when needed.** Move from plain instruction → role/system framing → examples → decomposition → tool/schema constraints, only as the failure mode forces you to.
- **Inspect failures, not summaries.** Read the actual model output on the cases that broke. A scoreboard tells you *that* something failed; the output tells you *why*.
- **Change one thing at a time.** If you change the system prompt, the few-shot examples, and `temperature` in the same commit, you cannot attribute the result.

This loop is boring on purpose. The interesting work is the model's; your job is to make the call repeatable and the output measurable.

## Where prompts live in the system

It is worth deciding early *where* your prompts live and how they get rendered. Some viable shapes:

- **Inline strings** for prototypes — fine for one-off scripts and notebooks, painful for anything you will maintain.
- **Template files** (e.g., `prompts/extract_v2.md`) loaded at startup — easy to diff, easy to version, easy to render with a small templating library.
- **Prompt registries** (a folder of YAML/JSON with metadata, or a hosted tool) — useful when non-developers help author prompts, or when you want to A/B different versions in production.

Pick the simplest shape that lets you (a) diff prompts in code review and (b) log the *final* rendered prompt for a given call. The fancy options are not required for the work in this module, but the two requirements are.

## What this module covers

The remaining chapters cover the moves you will reach for, roughly in escalation order:

- Chapter 2: the building blocks of an effective prompt (instruction, role, context, format, examples) and the order they go in.
- Chapter 3: few-shot prompting — when examples beat instructions and how to pick them.
- Chapter 4: decomposition and chain-of-thought — when a hard task should become several easier calls.
- Chapter 5: strategies for reliable structured output (JSON mode, schema-constrained generation, tool calling, prefill).
- Chapter 6: parsing and validating model output at the application boundary.
- Chapter 7: iterating with simple evals — how to know a change actually helped.

## Summary

A prompt is code: version it, pin its surroundings, and test it. Default to the simplest prompt that works, escalate to more structure only when the failure mode demands it, and change one thing at a time. The rest of this module is the toolbox you escalate through.
