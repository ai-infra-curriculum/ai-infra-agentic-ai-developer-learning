# From Tool Loop to Agent

## Why this matters

The previous two modules gave you two tight loops. `mod-103` taught the model to *act* by calling tools. `mod-104` taught the model to *know* by retrieving chunks from your corpus. Both loops were small, predictable, and bounded — exactly the property that made them safe to ship.

This module asks a different question: what happens when the model is allowed to decide, turn by turn, whether to act, retrieve, answer, or keep looking? That decision loop is what we will call an **agent** for the rest of the track. Single-agent, single-process, no orchestration — just one model with tools, retrieval, and a small amount of memory, running a few thinking steps before it answers.

You are not learning a new SDK feature. You are learning a new shape for the same SDK calls you already make.

## Tool loop vs. agent — what actually changes

The mechanics are nearly identical. What changes is who decides when the loop ends and what the next step looks like.

| Property                        | Fixed tool loop (mod-103)            | RAG query (mod-104)                       | Agent (this module)                          |
|---------------------------------|--------------------------------------|-------------------------------------------|----------------------------------------------|
| Who picks the next step         | The model, but only from a fixed menu of tools you wrote for *this* request | You — retrieve on *every* turn, always    | The model, including whether to step at all |
| Loop terminates when            | The model stops asking for tools     | After one retrieval + one answer call     | The model says it's done — or you cap it    |
| Number of LLM calls per request | 1 + (number of tool round trips)     | Exactly 2 (embed + answer)                | Variable — usually 2–8, sometimes more       |
| Failure mode                    | A tool errors                        | Retrieval whiffs                          | The model gets stuck, drifts, or loops      |
| What the user sees              | A final reply                        | A cited answer                            | A reply, optionally with a visible "trace"  |

Two shifts to internalize:

- **The model is now the planner.** In the fixed loop, you decided ahead of time that the assistant would lookup → answer. In the agent, the model gets to decide: maybe lookup → think → lookup again with a sharper query → answer. You no longer write the control flow; you write the *menu* and the *guardrails*.
- **The loop runs longer.** Two LLM calls used to be the norm. Five or six is now normal. Costs and latency add up linearly with steps, and a misbehaving prompt can turn a $0.01 request into a $0.50 one. Step caps stop being polite — they are load-bearing.

## A minimal agent, in one diagram

```
user message
   |
   v
[ model.create(tools=[...], messages=[...]) ]
   |
   +-- text only       -> done; reply to user
   |
   +-- tool_use blocks -> run each tool
   |                      append assistant turn (the tool_use)
   |                      append user turn (tool_results)
   |                      loop
   |
   +-- step_cap hit    -> stop; surface partial state
```

Compare against the call/result loop from `mod-103` chapter 3 — it is the same diagram. The only thing that has changed is the *tools list*: now it includes things like `search_kb` (your retriever from `mod-104`), `lookup_order`, `web_search`, and the loop is expected to call several of them before it answers.

That is the whole technical change. The interesting part is what the model does with the new freedom.

## ReAct: the pattern you are already most of the way to

The framing this module uses comes from the **ReAct** pattern: at each step the agent produces a **Thought** about what to do, picks an **Action** (a tool call with arguments), and reads the resulting **Observation** before the next thought.

```
Thought:    The user is asking about return policy for opened headphones.
            I should look in the knowledge base.
Action:     search_kb(query="return policy opened electronics")
Observation: 3 chunks returned, top score 0.81. Top chunk mentions a 14-day window
             for opened electronics with caveats. Need to confirm the caveats.
Thought:    Top chunk references "clause 3". Search for that clause.
Action:     search_kb(query="opened electronics clause 3 restocking fee")
Observation: 2 chunks returned. Restocking fee is 15% for opened items.
Thought:    I have what I need. Compose the answer with citations.
Action:     final_answer(...)
```

Two things to notice. First, this is the same call/result loop from `mod-103`, just rendered with the model's reasoning surfaced as a "Thought" line. Second, on modern Anthropic and OpenAI models you do **not** prompt the model to literally emit `Thought:` / `Action:` / `Observation:` as text. The model already supports the pattern natively: the *Thought* is the assistant's text content, the *Action* is the `tool_use` block, and the *Observation* is the `tool_result` you send back. Chapter 2 walks through both the original prompted form and the modern native form so you know what is and isn't ceremony.

> ReAct as a *prompting technique* (the original 2022 paper) and ReAct as a *protocol* (what native tool-use APIs implement today) are not the same thing. Chapter 2 disentangles them.

## What this module covers

Six chapters, each one short on theory and long on the code shape that makes the theory real.

1. **From tool loop to agent.** (You are here.) The agent is the tool loop where the model picks the next step.
2. **The ReAct pattern: Thought, Action, Observation.** The original prompted form, the modern native form, and what's load-bearing about each.
3. **Implementing the ReAct loop.** A 60-line Python loop with native tool use: termination, step caps, observation formatting, surfacing intermediate reasoning.
4. **Retrieval as a tool.** Wrapping your `mod-104` retriever as `search_kb`. When the agent should retrieve, when it shouldn't, and how to combine retrieval with other tools.
5. **Conversation memory.** Short-term (running transcript), context-window pressure (summarization), and a small key:value scratchpad for things you want to outlive a single turn.
6. **Limits of a single agent.** The failure modes you will see — loops, drift, planning depth — and where the handoff to the Engineer track (multi-agent, planners, supervisors) actually starts.

The exercises run in parallel:

- **exercise-01.** Build the minimal ReAct loop over two toy tools and watch it think.
- **exercise-02.** Wire your `mod-104` retriever in alongside two action tools — a real "tools + retrieval in one agent" build.
- **exercise-03.** Add multi-turn conversation memory and a summarization fallback when the transcript grows.

## What this module does *not* cover

It is useful to name what is out of scope so you know where to stop:

- **Multiple cooperating agents.** Planner/worker, supervisor/researcher, debate. Those are the Engineer track.
- **External orchestration frameworks.** LangGraph, CrewAI, AutoGen, Agno, Anthropic's Claude Agent SDK. You will recognize their shapes after this module, and chapter 6 points at where you cross over.
- **Long-term memory stores.** Vector-backed user memory, journal files, MCP memory servers. Chapter 5 introduces the concept; production memory is its own course.
- **Agent evaluation.** We will measure steps, latency, and cost in the exercises. Quality evals for multi-step agents (faithfulness, trajectory scoring) belong to a later module.

The goal is one well-behaved agent you understand top to bottom, not a stack of frameworks you only half-trust.

## How this module connects to the rest of the track

- **`mod-103` (tools).** This module re-uses your call/result loop verbatim. If yours wasn't clean, fix it before you start chapter 3 — every problem you defer here gets harder under an agent loop.
- **`mod-104` (retrieval).** Your retriever from exercise-02 becomes a tool the agent decides when to call. Exercise-02 of this module wires it in.
- **`mod-106` (deployment).** The agent becomes a service: a request handler, a step-cap policy, an observability story (per-step traces, token spend, latency), and a way to ship prompt changes without redeploying the world.
- **Engineer track.** When the single-agent shape stops being enough — when you need a planner, a verifier, a tool-using sub-agent, or human-in-the-loop checkpoints — that is where the next course picks up. Chapter 6 is the bridge.

## A note on production agents

The word "agent" is doing heavy lifting in the industry right now. It means at least three different things depending on who is talking:

1. **A tool-using LLM call in a loop** (this module).
2. **A multi-step planner + executor** with a structured plan it edits as it goes.
3. **A long-running autonomous system** that operates a computer, takes actions over hours or days, and reports back.

Almost everything you read about "agents" online is talking about (2) or (3). What you build here is (1). It is the substrate the others are made of — every planner is still calling `messages.create(tools=...)` somewhere, every autonomous system still has a step cap somewhere — and once you understand (1) top to bottom, (2) and (3) read as patterns rather than magic.

## Summary

An agent, in this module, is the same call/result loop you already wrote in `mod-103`, with the model promoted from "picks tools" to "decides what the next step is." That promotion is small in code and large in implication: the loop runs longer, costs scale with steps, and a misbehaving prompt can spin. The rest of the module is the framing the model uses to make those decisions (ReAct), the shape of the loop on modern native-tool-use APIs, the integration of retrieval as one tool among many, multi-turn memory, and an honest read on where a single agent stops being enough.
