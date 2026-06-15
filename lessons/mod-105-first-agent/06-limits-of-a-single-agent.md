# Limits of a Single Agent

## Why this matters

The four chapters before this one taught you to build a small, sharp tool. This chapter is about the tasks it isn't sharp enough for.

Knowing when to *stop* using a single ReAct agent is just as important as knowing how to build one. The most common production failure mode is not "the agent doesn't work" — it's "the agent half-works, the team keeps adding tools and prompts and steps to make it work better, and six months in nobody can predict what it will do." That trajectory is what the Engineer track (`mod-201`+) is designed to break.

So: what does a single agent do well, where does it break, and what's the next step up?

## What a single agent is good at

The pattern you've built handles a specific shape of task very well:

- **Short trajectories.** 2–8 steps from question to answer.
- **Independent steps.** Step 3 depends on the result of step 2, but the *plan* doesn't change much across steps. Lookup, retrieve, answer.
- **One responsibility.** "Answer support questions over the knowledge base, with access to order operations." The role is single and named.
- **Recoverable failures.** A tool errors; the agent retries; the user sees an apology. Acceptable.
- **Bounded time and cost.** Seconds, not minutes; pennies, not dollars.

You can ship this. Most production "AI assistant" features start here, and many of them never need to leave.

## Where the single agent breaks

Five failure modes you will eventually see, in roughly the order you will see them.

### 1. Loops

The agent calls the same tool with the same arguments two, three, eight times in a row. Sometimes with slight rephrasings; sometimes identical.

**Why it happens.** A tool returns an unhelpful result (empty, ambiguous, weak score), and the model decides to try again. With no way to know the result will keep being unhelpful, it tries forever.

**What you can do in this module.** Detect repeated `(name, input)` calls and break. Surface "I tried searching for X three times and didn't find what I needed." Tighten the retriever or add a more specific tool.

**When you've hit the limit.** The right behavior is "give up and ask the user" or "escalate to a different sub-agent." Both belong outside the loop. Both come up in the Engineer track.

### 2. Drift

The agent starts on task, then wanders. A retrieval returns an interesting-but-tangential chunk; the next Thought is about *that* topic; the next tool call probes it; ten steps in, the agent is answering a question the user didn't ask.

**Why it happens.** The Thought slot has too much freedom. Every Observation expands the surface area of "interesting next steps" and the model picks the most interesting, not the most relevant.

**What you can do in this module.** A tighter system prompt ("stay focused on the customer's original question"), shorter observations, and step caps that stop the trajectory before it strays too far.

**When you've hit the limit.** Once the task itself is open-ended ("research the competitive landscape"), drift is the symptom of a missing planner. A separate "plan" step that commits to a shape before execution begins is the next pattern. That is the planner/worker shape in the Engineer track.

### 3. Planning depth

Some tasks have a right answer that depends on a decision three or four steps ahead. "Book me the cheapest flight that gets me to Tokyo before the conference and uses Star Alliance for status." The agent can't pick a candidate flight without considering whether its connections will be there, whether the carrier qualifies for status, whether there's a cheaper combination it hasn't tried.

**Why it happens.** ReAct decides one step at a time. The model has no obvious place to *plan* the trajectory before executing it.

**What you can do in this module.** Add an explicit `propose_plan` tool the model is encouraged to call first; render the plan in the system context for subsequent steps. This buys you a lot. It is also the seed of the next pattern.

**When you've hit the limit.** Once the planning is genuinely involved — multi-day workflows, codebase refactors, multi-agent coordination — you want a separate *planner* component whose only job is the plan, and an *executor* whose only job is to run it. Plan-and-execute and planner/worker architectures live in the Engineer track.

### 4. Tool-surface bloat

Two tools, the agent picks well. Five tools, it still picks well. Twelve tools, the model starts choosing the wrong one. Twenty tools, you spend more time tuning descriptions than tuning anything else.

**Why it happens.** The tool list is part of the prompt. Each tool's description gets re-evaluated on every turn. Past a certain count, descriptions interfere with each other and the model loses track of which one is the right call.

**What you can do in this module.** Keep the tool count low (five or six is a sweet spot for a single agent). Merge related tools into a single tool with a `kind` parameter. Remove tools that are rarely used.

**When you've hit the limit.** When the surface genuinely requires many tools — "all of GitHub," "all of Salesforce," anything resembling a full API — you want sub-agents specialized to a tool *family*, with a routing layer above them. That is multi-agent territory; supervisor patterns; what some frameworks call "router agents."

### 5. Verification and trust

The agent says it did something. Did it? An action tool returned `{"ok": true}` — does that match what the user wanted? The agent claims it found the answer in chunk X — is the chunk really relevant? On a one-shot demo nobody checks. In production every wrong action is a support ticket.

**Why it happens.** ReAct has no built-in verifier. The model is its own quality gate. When the model is wrong, the loop has no second opinion.

**What you can do in this module.** Add post-hoc validators: citation correctness checks (chapter 4), schema validation on tool inputs (`mod-103` chapter 6), eval sets that grade the agent's outputs on held-out examples. None of these are inside the loop; they're around it.

**When you've hit the limit.** When the task has a verifiable success criterion — "the test passes," "the document type-checks," "the response is consistent with the policy" — a separate *verifier* in the loop closes the gap. The agent acts; the verifier judges; the agent revises. Reflexion, plan-and-verify, and most agentic-coding frameworks are this shape. Engineer track.

## The honest read on "agent" in industry

A search of the AI press in 2025–2026 turns up the word "agent" applied to systems that vary by three or four orders of magnitude in complexity. The framing that survives contact with production work, roughly:

- **Level 0 — function-calling app.** A single model call that may run one tool. The chapter-3 loop from `mod-103`. Not really an "agent" but often shipped under the name. <!-- needs-research: cite Anthropic's "Building Effective Agents" or OpenAI's analogous "patterns for agentic systems" post; both are widely referenced primary sources for the typology that follows -->
- **Level 1 — tool-using loop.** What you built this module. A single LLM, native tool use, ReAct, a few turns of memory. The bulk of "AI app" features today are still here.
- **Level 2 — single agent with planning and verification.** The Level-1 loop with an explicit plan step, often a verifier, sometimes retries with a critique. This is where the Engineer track picks up.
- **Level 3 — multi-agent system.** Several Level-2 agents with distinct roles, coordinated by a supervisor or by a structured handoff protocol. Planner + worker; router + specialists; debate. Engineer track.
- **Level 4 — long-running autonomous agent.** An agent that operates over hours or days, with checkpoints, human-in-the-loop, durable memory, and real consequences in the world. Engineer track and beyond; also where most of the safety conversation lives.

Almost every framework you'll meet — LangGraph, CrewAI, AutoGen, Anthropic's Claude Agent SDK, OpenAI's Agents SDK — is a toolkit for Levels 2 and 3. The shape they help you build is roughly: nodes, edges, state, and a dispatcher. The reason you've now built Level 1 from scratch is so when you read a graph in a framework, you can point at the node that's just a `messages.create(...)` call.

## When to graduate

A pragmatic checklist for "is this still a single-agent problem?":

- **Steps stay under ~10 for the median task.** Yes? Single agent.
- **Tool count stays under ~7.** Yes? Single agent.
- **There's one role, not several.** Yes? Single agent.
- **Failures are recoverable in a single trajectory.** Yes? Single agent.
- **Stakes per action are low enough that "the model said so" is acceptable verification.** Yes? Single agent.

A "no" to any of these isn't a death sentence — you can stretch single-agent further with sharper tools, tighter prompts, and out-of-loop verifiers. But a "no" to two or three is a strong signal that the next architecture earns its complexity.

The pattern in the wild: teams ship Level 1, hit one or two of the limits above, add three or four hacks, and at some point realize the hacks are recreating a planner / verifier / sub-agent badly. Graduating to Level 2 or 3 was the cheaper path. Knowing the limits *before* you spend three months in the hack zone is the goal of this chapter.

## What the Engineer track adds

Without spoiling the next course, the patterns that go beyond a single agent are roughly:

- **Planner + executor.** A planning model produces a structured plan; an execution model runs it. The plan is editable mid-run.
- **Supervisor + workers.** A supervisor routes a task to one of several specialist agents, each with its own tool surface. Common for "all of GitHub" and "all of Salesforce" -shaped tasks.
- **Critique loops.** Reflexion-style: try, evaluate, write a lesson, try again. Powerful on tasks with a programmatic success criterion (tests, type-checkers).
- **Human-in-the-loop checkpoints.** Specific tools or steps require a human approval. Mandatory for high-stakes actions (large refunds, production deploys).
- **Durable memory.** The session ends; the conversation continues next week. Requires actual storage, vector retrieval over user history, and care about what you retain.

You will recognize all of these as compositions of pieces you already understand. None of them remove the ReAct loop; they wrap it.

## What to read next

If this module has made you want to keep going, three pointers worth bookmarking, in order of accessibility:

- **Anthropic — "Building Effective Agents."** Anthropic's primary write-up on the patterns above, with worked examples. The canonical industry post on agent shapes. <https://www.anthropic.com/engineering/building-effective-agents>
- **OpenAI — Cookbook entries on agent patterns and the Agents SDK.** Worked examples of single-agent and multi-agent systems on the OpenAI Responses API. <https://cookbook.openai.com/topic/agents>
- **LangGraph — concepts documentation.** A graph-shaped framework whose docs read like a survey of agent architectures. Useful to skim even if you never adopt it. <https://langchain-ai.github.io/langgraph/concepts/>

`mod-201` (Engineer track) is where you stop being a careful reader of these and start being a careful builder.

## Summary

A single ReAct agent handles a real and broad slice of work: short trajectories, a small tool surface, one role, recoverable failures. It breaks predictably on five fronts — loops, drift, planning depth, tool-surface bloat, and verification — each of which is a signal that a higher-level pattern (planner/executor, supervisor/workers, critique loops, human-in-the-loop, durable memory) earns its complexity. The exercises in this module ship the single-agent ceiling. The next course wraps that ceiling in the patterns that lift it.

You have a working agent now. Be honest about what it can and can't do, and you have already done the hardest job in agent design.
