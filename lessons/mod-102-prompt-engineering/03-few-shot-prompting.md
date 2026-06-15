# Few-Shot Prompting

## Why this matters

A few-shot prompt is the trick where you teach a task by example. Instead of describing every edge case in prose, you show the model two or five worked input/output pairs and let it generalize. For narrow, well-defined tasks — classification, extraction, transformation, formatting — a small set of examples often beats a paragraph of instructions, and it is much easier to update.

The technique is older than instruction-tuned chat models (it was Brown et al., 2020, that named it "in-context learning"), and it still works on modern models because the pattern in the prompt is exactly what the model is trained to continue.

## Zero-shot vs. few-shot vs. fine-tuning

A quick mental map:

- **Zero-shot** — instruction only, no examples. Cheapest, lowest signal.
- **Few-shot** — instruction plus 1–8 worked examples in the prompt. More tokens per call, but no training and no infra.
- **Fine-tuning** — bake the pattern into the weights. Lower per-call tokens, requires data prep and training infra, and you re-do it whenever the model changes.

For most application work, you start zero-shot, escalate to few-shot when the failure mode demands it, and only consider fine-tuning when you have a large, stable, valuable dataset and the per-call savings justify the overhead.

## What examples are good at

Few-shot earns its tokens when:

- The output **shape** is fiddly and hard to describe in prose (a specific JSON schema, a domain-specific tagging convention, a particular tone).
- There are **edge cases** whose handling is hard to specify but easy to demonstrate (how to label borderline categories, how to phrase a refusal).
- The model **keeps drifting** from a specific format despite clear instructions ("write in the second person", "always use Australian spelling", "limit to 12 words").

Examples are weaker when:

- The task is **open-ended** ("brainstorm 20 ideas"). Examples bias toward their own pattern.
- The input space is **wide and unstructured** and your few examples are not representative.
- The model already **handles it perfectly zero-shot**. Then examples are just spent tokens.

## Where to put the examples

Two common placements:

**1. Inside the system block, as a labeled exhibit:**

```text
SYSTEM:
You are a sentiment classifier. Reply with one of: positive, neutral, negative.

Examples:
Input: "Loved the new packaging."          -> positive
Input: "It arrived dented but works."      -> neutral
Input: "Worst purchase of the year."       -> negative

USER:
Input: "It's fine, I guess."
```

This is compact and survives across turns. Good for short, stable example sets.

**2. As real assistant turns:**

```json
{
  "model": "claude-sonnet-4-6",
  "messages": [
    { "role": "user",      "content": "Loved the new packaging." },
    { "role": "assistant", "content": "positive" },
    { "role": "user",      "content": "It arrived dented but works." },
    { "role": "assistant", "content": "neutral" },
    { "role": "user",      "content": "It's fine, I guess." }
  ]
}
```

This mirrors the deployed conversation shape exactly. Models tend to pick it up cleanly because there is no ambiguity about what the "output" is — the next assistant turn.

For OpenAI-style chats you can prepend a `system` message with the policy and follow it with the same alternating user/assistant pattern. Both providers accept either shape.

## How to pick examples

A few practical rules:

- **Cover the failure modes.** If you have logged real outputs that went wrong, those are your example candidates. Showing the model the borderline case it kept getting wrong is the highest-leverage example you can add.
- **Cover the class distribution.** For a classifier, include at least one example of every label, especially the rare ones. Otherwise the model will under-predict them.
- **Use the same shape and format as the real input.** If your real inputs are 200-word complaints, do not use 4-word toy examples. The model will assume short inputs are typical.
- **Keep them short enough.** Few-shot examples are paid for on every call. A handful of 200-token examples can easily dominate the input bill on a small task.
- **Watch for label leakage.** If your examples happen to contain a hint of the correct label ("Worst purchase of the year." → negative — the word "worst" is doing all the work), the model may learn the spurious shortcut. Mix in examples where the surface signal is weaker.

## Ordering matters more than people expect

Order effects in few-shot are real and well-documented. Two patterns help:

- **Put the strongest, most representative example first.** It is the one the model is most likely to weight.
- **Put a tricky/edge-case example last.** That is the freshest "instruction" before the real input.

If your examples are randomized per call, fix an order and stop randomizing. Determinism in the prompt setup makes the model's behavior easier to attribute when it drifts.

## A clean transformation example

Task: extract structured data from a freeform shipping address. Few-shot prompt:

```text
SYSTEM:
You convert raw shipping addresses into a normalized JSON object.

Schema:
{ "name": str, "street": str, "city": str, "region": str, "postal": str, "country": str }

If a field is missing, use null. Do not invent.

Examples:

Input:  "Ada Lovelace, 12 Dover St, London W1S 4LL, UK"
Output: {"name":"Ada Lovelace","street":"12 Dover St","city":"London","region":null,"postal":"W1S 4LL","country":"UK"}

Input:  "  Grace Hopper / 1 Naval Way / Arlington, VA 22202 "
Output: {"name":"Grace Hopper","street":"1 Naval Way","city":"Arlington","region":"VA","postal":"22202","country":null}

USER:
Input: "Katherine Johnson, 100 NASA Pkwy, Houston TX 77058"
```

This prompt is doing several jobs the model would otherwise guess at: it shows the model what missing fields look like (`null`, not empty string), it shows two different separator conventions, and it shows that the country can be absent. Those are exactly the kinds of judgment calls that drift across model versions without examples.

## Few-shot's failure modes

A short cheat sheet for when few-shot misbehaves:

- **Pattern lock-in.** The model copies a quirk of your examples (a phrasing, a capitalization, a default value) onto every output. Fix by varying the examples on the dimension you do not want copied.
- **Long-input drift.** With many or long examples, the model can lose track of which block is the actual input. Mark the real input clearly ("Now process the input below.") or use the chat-turn placement so the boundary is structural.
- **Example contamination of the answer.** The model parrots a near-by example's content into a new answer. Especially common with low-`temperature` extraction tasks. Pick examples that look *unlike* the live input.
- **Stale examples.** If your domain changes (new product categories, new error codes), examples written months ago start teaching the model the wrong thing. Examples are code; re-check them when the world changes.

## Few-shot vs. richer instructions

Often the question is: "Should I add another example or another sentence of instruction?" A useful rule of thumb:

- If you can state the rule in one short sentence ("Use ISO-8601 dates."), the sentence is cheaper and clearer than an example.
- If the rule needs a paragraph and three caveats, an example or two will land it faster.
- If you find yourself adding both a paragraph *and* multiple examples for the same rule, you have an ambiguity in the task itself. Resolve the ambiguity first.

## Summary

Few-shot prompting teaches by demonstration. It earns its tokens when the output shape is fiddly, when edge cases matter, or when the model keeps drifting from a specific format. Pick examples that cover real failure modes and the label distribution, place them either in the system block or as real assistant turns, fix the order, and re-check them when the domain changes. When in doubt: try zero-shot first, then add the minimum number of examples that fix the actual problem.
