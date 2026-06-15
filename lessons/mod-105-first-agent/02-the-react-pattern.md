# The ReAct Pattern: Thought, Action, Observation

## Why this matters

You can build a working tool-using agent without ever hearing the word "ReAct." But the pattern is what almost every modern agent framework — LangGraph, CrewAI, Anthropic's Claude Agent SDK, OpenAI's Responses API agents, the small one you build in chapter 3 — is implementing under the hood. Knowing the pattern by name lets you read other people's code, debug a bad trace, and make deliberate choices about how much of the model's reasoning to surface.

This chapter does two things. First, it covers the **original prompted form** of ReAct so the vocabulary makes sense. Second, it shows the **modern native-tool-use form** that you will actually use in chapter 3, because today the model already produces Thoughts and Actions in the right slots without being told to.

## The pattern, in one frame

ReAct interleaves *reasoning* and *acting* across a multi-step trajectory:

```
Thought_1  -> Action_1 (tool call)
Observation_1
Thought_2  -> Action_2 (tool call)
Observation_2
...
Thought_N  -> Final answer
```

Three slots, repeating:

- **Thought.** Free-form natural-language reasoning the model writes for itself: "I need to know X. The best next step is Y."
- **Action.** A tool invocation, structured: name + arguments. The action is what the *application* will execute.
- **Observation.** What the tool returned. Always grounded in the real world — a query result, an API response, a retrieved chunk. This becomes context the model can think about in the next Thought.

The terminating step is a Thought that decides the model has enough and emits a final answer instead of another action.

The original framing comes from Yao et al. (2022), *"ReAct: Synergizing Reasoning and Acting in Language Models."* Their argument was that pure chain-of-thought ("just think harder") hallucinates, pure act-only loops have no chance to plan, and the alternating shape was strictly better than either on knowledge-intensive tasks. <!-- needs-research: confirm ReAct paper publication date and primary venue; cite arxiv id 2210.03629 -->

## The original (prompted) form

In the original paper the model is a plain text-completion LLM. It does not have a structured tool-call API. So the *protocol* is enforced by the prompt:

```
You are an agent that can think and act. Use this format:

Thought: <one sentence about what to do next>
Action: <one of: search[query] | lookup[key] | finish[answer]>
Observation: <I will fill this in for you>

Begin.

Question: When was the author of "The Old Man and the Sea" born?
Thought:
```

The application's job is:

1. Sample text from the model until it emits `Action: ...`.
2. Parse the action line. If it's `finish[...]`, you're done.
3. Otherwise execute the action (call a search engine, look up a database row).
4. Append `Observation: <result>\nThought:` and sample again.

That is the entire trick. The model does the planning in the `Thought:` lines and the executing in the `Action:` lines; the application does the I/O and stitches Observations into the prompt.

Two practical observations on this form:

- **It works on any text model.** Even small open models can follow the protocol if you give a few-shot example or two.
- **It is brittle.** Any time the model gets clever — adds a stray Observation line, forgets the brackets, freelances a new action name — the parser breaks. Every "ReAct from scratch" tutorial spends half its time on the parser.

You will not use this form in chapter 3. You should still recognize it on sight: it shows up in legacy tutorials, in open-source frameworks targeting models that don't support tool use, and in research papers.

## The modern (native tool-use) form

Today, both Anthropic and OpenAI ship a *structured* version of this pattern as native API features. The model emits the Action as a typed `tool_use` block (Anthropic) or `tool_calls` entry (OpenAI) — not as a string you have to parse. Your application reads that structured field, executes the tool, and sends the result back as another structured block.

The mapping from ReAct vocabulary to the protocol you already know from `mod-103`:

| ReAct slot     | Where it lives in the protocol                                    |
|----------------|-------------------------------------------------------------------|
| **Thought**    | The assistant turn's `text` content blocks (the model's prose).   |
| **Action**     | The assistant turn's `tool_use` block (Anthropic) or `tool_calls` entry (OpenAI). |
| **Observation**| The next user turn's `tool_result` block (Anthropic) or `role: "tool"` message (OpenAI). |
| **Final answer** | The assistant turn that has `text` but no `tool_use` and `stop_reason == "end_turn"`. |

A trace from a modern API call looks like:

```
[user]      "Where is my order ORD-1003 and is it in stock for re-order?"

[assistant] text:     "Let me check on that order first."
            tool_use: get_order_status(order_id="ORD-1003")
                                                                    <-- Thought + Action
[user]      tool_result: {"order_id": "ORD-1003", "status": "shipped",
                          "carrier": "UPS", "sku": "BL-MED-001"}
                                                                    <-- Observation
[assistant] text:     "Order ORD-1003 shipped. Checking stock on that SKU."
            tool_use: search_products(query="BL-MED-001")
                                                                    <-- Thought + Action
[user]      tool_result: [{"sku": "BL-MED-001", "in_stock": true, ...}]
                                                                    <-- Observation
[assistant] text:     "ORD-1003 shipped via UPS and BL-MED-001 is still in stock,
                       so you can reorder."
                                                                    <-- Final answer
```

Three things to notice:

- **Nothing in the prompt says "Thought" or "Action."** The pattern is now a property of the *protocol*, not the prompt. You expose tools, the model picks one (with a Thought along for the ride), the protocol carries the result back. You did not invent the ceremony; you inherited it.
- **The Thought is optional.** Sometimes the assistant turn is *just* a `tool_use` block with no leading text. That is still ReAct — there was a Thought, the model just didn't write it down.
- **The loop terminator is identical to `mod-103`.** "No more tool_use blocks; `stop_reason == end_turn`" is the same condition you wired in the call/result loop.

If you forget every other word in this chapter, remember this: on modern providers, ReAct is the *native* shape of multi-step tool use. You don't need a special framework to do it. You need the same call/result loop you already have, with the model deciding it has enough steps to do interesting things.

## Why the Thought matters even if the model already writes it

If the Thought is implicit in the protocol, why care about it as a concept?

- **Debugging.** When an agent does the wrong thing, the first question is "what was it thinking before that action?" Reading the assistant's `text` block answers that — sometimes you find it had the right idea and chose the wrong tool, sometimes you find it had the wrong idea from the start. Without explicit Thoughts in the trace, an agent looks like a black box that called a tool for inscrutable reasons.
- **Steering.** A system prompt that says "Briefly explain what you are about to do before calling a tool" reliably produces Thought text. This is cheap, modestly improves quality on multi-step tasks, and makes traces much more readable. The cost is a few extra output tokens per step.
- **User-facing transparency.** "Searching the knowledge base for return policy…" rendered in the chat UI is the surfaced Thought. The protocol already has the data; you choose whether to show it.
- **Surfacing dead ends.** Without a Thought slot the model has no place to write "this approach isn't working; let me try X" before it tries X. With one, you can see the pivot.

A common system-prompt nudge:

```
Before calling a tool, briefly state in one sentence what you are looking
for and which tool you intend to use. After receiving a tool result, briefly
state what you learned before deciding the next step. Keep these statements
short — one sentence each.
```

That is enough to turn a silent agent into a self-narrating one. Chapter 3's exercise asks you to render those statements to the user.

## When ReAct shines, and when it doesn't

ReAct is a strong default. It is not universally optimal.

**Plays well with ReAct:**

- *Information-seeking with branches.* Search → read → re-search-with-sharper-query → answer. The pivot in the middle is the whole point.
- *Multi-tool routing.* "Cancel this order, refund the difference, and email a confirmation." Three tools, three Thoughts, three Actions, one final reply.
- *Looking up something the model doesn't know.* Anything that would otherwise hallucinate — a customer's actual order status, a fact in your private docs, a current API result.

**Where ReAct is the wrong shape:**

- *Single deterministic transform.* "Summarize this paragraph." Zero tool calls; you don't need an agent for this. A plain `messages.create` is faster and cheaper.
- *Tasks with a known sequential plan.* "For each row in this CSV, look up X and write Y." A `for` loop over rows with two model calls each is more reliable and *much* cheaper than asking the model to drive the iteration. This is the "is this actually an agent?" question — usually it isn't.
- *Tasks needing genuine planning depth.* Booking flights, refactoring a codebase, multi-day workflows. ReAct's "decide one step at a time" shape struggles when the right step three turns from now constrains the step you make now. That is the planner/worker shape from the Engineer track (`mod-201`+).
- *Tasks needing a verifier.* "Write code that passes these tests." ReAct can take a swing, but you usually want a separate execute-tests-and-judge step. That is also Engineer track.

A useful gut check before reaching for an agent: *would a 20-line script with two LLM calls do this?* If yes, write the script. Agents are for when the next step legitimately depends on what just happened.

## ReAct adjacents you should recognize by name

- **Chain-of-thought (CoT).** Reasoning without acting. A subset of the Thought slot. The original paper called out that CoT *alone* tends to hallucinate facts. ReAct's contribution was the act-in-between.
- **Plan-and-execute / Plan-and-act.** The agent first emits a full plan ("step 1: search; step 2: call X; step 3: call Y"), then executes it. Less common as a pure form; more common as the model first calling a `plan` tool then executing actions referencing the plan.
- **Reflexion.** ReAct plus a self-critique step: after a failure, the agent writes a "lesson" and tries again. Good for tasks with verifiable success criteria (e.g., a passing test). Out of scope here.
- **Toolformer / function-calling-only.** The model picks a tool but does not surface the reasoning. Faster, cheaper, less interpretable. This is what `mod-103`'s fixed loop already was, before you let the model decide how many turns to take.

You do not need to implement any of these. You should be able to look at a framework's agent and say, "oh, that's a plan-and-execute on top of a verifier."

## The contract the model is following

When you expose tools and let the model loop, the model is following an implicit contract you wrote. Modern training and RLHF have baked it in; you don't have to teach it. The contract reads, roughly:

> Use a tool when you need information you don't have or an action you can't do yourself. Stop when you can produce a final answer. Don't fabricate observations. Don't call tools whose arguments you don't know.

When an agent misbehaves, it is almost always misreading the contract because *you* did one of:

- Wrote a tool description that lied about when to call.
- Exposed a tool whose schema looks easy to call but whose result is unhelpful.
- Wrote a system prompt that pressures the model into producing an answer even when the right move is to refuse.
- Forgot to send a tool result and the model is now guessing.

Every chapter from here on assumes that contract is in force. The work is making sure your half of it (descriptions, schemas, prompts, observations) matches what you want the model to do.

## Summary

ReAct is the alternating Thought / Action / Observation pattern that produces almost every multi-step LLM trajectory you'll work with. The original form prompted the model to emit the pattern as text and parsed it; the modern form turns each slot into a typed field on the call/result loop you already wrote in `mod-103`. The Thought slot is what you read when an agent does the wrong thing; the Action slot is the only thing the application actually executes; the Observation slot is your only chance to ground the model in reality. Chapter 3 builds the loop.
