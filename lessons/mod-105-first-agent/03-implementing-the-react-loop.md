# Implementing the ReAct Loop

## Why this matters

You now know what ReAct is. This chapter is the loop in code. It is a small file — around 80 lines — but every line bakes in a decision: how to terminate, what to cap, how to format observations, how to surface intermediate text, what to do when the model hits `max_tokens` halfway through a tool call.

Get this loop right and the rest of the module is integration. Get it wrong and every later exercise is debugging an infinite loop or a wedged trace.

## What you are about to write

```
+---------------------+
|                     |
|  user message       |
|                     |
+----------+----------+
           |
           v
+---------------------+        +--------------------+
|                     |   yes  |                    |
|  model.create(...)  +------->+  stop_reason ==    |
|                     |        |  "end_turn"?       |
+----------+----------+        +---------+----------+
           |                             |
           | tool_use                    | return final text
           v                             v
+---------------------+               +-----+
|  for each tool_use: |               |     |
|    execute(tool)    |               | done|
|    build result     |               |     |
+----------+----------+               +-----+
           |
           v
+---------------------+
|  append assistant   |
|  append tool_results|
+----------+----------+
           |
           v
       (loop back)
```

Three guards bracket the loop: a **step cap**, an **input-token guard**, and an **explicit `max_tokens` failure**. Without all three, an agent can keep paying you nothing and burning your account.

## The minimum loop (Anthropic)

```python
import json
from anthropic import Anthropic

client = Anthropic()
MODEL  = "claude-sonnet-4-6"

SYSTEM = """You are a helpful assistant. You may call tools to look up
information or perform actions. Before calling a tool, briefly state in one
sentence what you are looking for and which tool you intend to use."""


def run_agent(user_message: str, tools, dispatch, *, max_steps: int = 8) -> dict:
    messages = [{"role": "user", "content": user_message}]
    trace = []

    for step in range(max_steps):
        response = client.messages.create(
            model=MODEL,
            max_tokens=1024,
            system=SYSTEM,
            tools=tools,
            messages=messages,
        )

        # Always record the trace, even on partial / failed turns.
        trace.append({
            "step":        step,
            "stop_reason": response.stop_reason,
            "text":        "".join(b.text for b in response.content if b.type == "text"),
            "tool_uses":   [
                {"id": b.id, "name": b.name, "input": b.input}
                for b in response.content if b.type == "tool_use"
            ],
        })

        if response.stop_reason == "max_tokens":
            raise RuntimeError(f"hit max_tokens at step {step}; widen the budget")

        if response.stop_reason == "end_turn":
            return {
                "final_text": trace[-1]["text"],
                "messages":   messages,
                "trace":      trace,
                "steps":      step + 1,
            }

        # stop_reason == "tool_use" : carry the assistant turn forward.
        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            try:
                payload = dispatch(block.name, block.input)
                tool_results.append({
                    "type":        "tool_result",
                    "tool_use_id": block.id,
                    "content":     json.dumps(payload),
                })
            except Exception as e:
                tool_results.append({
                    "type":        "tool_result",
                    "tool_use_id": block.id,
                    "content":     json.dumps({"error": str(e)}),
                    "is_error":    True,
                })

        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError(f"exceeded max_steps={max_steps}; partial trace returned")
```

A few things this 60-line block is doing on purpose.

- **Trace before action.** The trace entry is appended at the *top* of each iteration. If something explodes mid-step, you still have a partial trace to read.
- **Errors are observations, not exceptions.** A tool that raises becomes a structured `tool_result` with `is_error: True`. The model sees the error and decides what to do next — usually retry with different arguments or apologize. This is the chapter-6 idea from `mod-103`: errors as feedback.
- **The step cap is non-negotiable.** Eight steps is enough for almost every well-shaped agent task. If you need more, you have a tool-design or prompt problem.
- **The Thought lands automatically.** The `SYSTEM` prompt nudges the model to write one sentence before each tool call. That sentence becomes the visible "Thought" line in your trace.

The OpenAI version uses `tool_calls` instead of `tool_use`, parses `arguments` as a JSON string, and emits one `role: "tool"` message per call. The shape is identical.

## Termination conditions, in order

There are exactly four ways your loop should stop. Handle them explicitly:

1. **`stop_reason == "end_turn"`.** The model is done. This is the happy path. Return the final text.
2. **`stop_reason == "max_tokens"`.** The model was cut off mid-output. Treat as failure: a `tool_use` may be missing arguments, the final JSON may be truncated. Do not send the partial turn to the next step. Raise and surface.
3. **`step >= max_steps`.** The agent is taking too many steps. Either the prompt is unclear, a tool is returning useless observations, or the model is looping on near-identical calls. Stop and surface the partial trace.
4. **`stop_reason == "stop_sequence"`** (less common). The model emitted a sequence you told it to stop on. Usually treat like `end_turn`.

Two failure modes you may also want to guard:

- **Repeated identical tool call.** If `(name, input)` is identical to the previous step, the agent is stuck. A single retry is normal; three in a row is a loop. Break and surface.
- **Input-token budget.** If `messages` grows past a soft cap (say, 30k tokens), summarize or truncate before the next call. Chapter 5 covers this.

Each of these is a one-liner in the loop. Add them when you see the failure mode, not preemptively.

## Surfacing intermediate text

The assistant's `text` content blocks contain the visible Thought — the "Let me check the knowledge base first" line. You have three rendering choices:

- **Hide everything until the end.** The user sees a final reply, nothing else. Lowest friction, but a slow agent looks broken because there's no signal it's working.
- **Stream the final reply, hide intermediate text.** Same UX as a normal chatbot. Good default for short agents where steps are fast.
- **Show every Thought, plus the tool that's about to run.** "Searching knowledge base for return policy…" → "Found 3 relevant chunks…" → final answer. Highest transparency, slightly higher cognitive load, often the right call for agents that take more than two seconds.

The data is the same in every case. The question is what your UI does with it. Exercise-01 asks you to print every Thought to stdout so you can read along.

## Observation formatting

You serialize tool results with `json.dumps`. Two design knobs:

- **Field order and naming.** The model sees the result as text. Keep names short and meaningful — `status` not `current_order_state`, `chunks` not `retrieved_documents`. The model reads them every turn.
- **Truncation.** A retrieval tool can return a huge blob — five chunks, 800 tokens each. That's 4000 tokens per observation, every turn, forever. Truncate per chunk (e.g., first 600 tokens) and document the truncation in the tool description. This is also where chapter 5's transcript-pressure problem starts.

A small but reliable trick: include the *score* or *confidence* in the observation when the model can use it. `[{chunk_id: ..., score: 0.81, text: "..."}, {..., score: 0.42, ...}]` tells the model the second hit is weak; without scores it treats both as equally relevant. The model uses the signal.

## Parallel tool calls

A single assistant turn can carry several `tool_use` blocks if the model decides multiple calls are independent. The loop above already handles this — `for block in response.content` runs each call — but two things to watch:

- **All results go in a single follow-up user turn.** That is already the case here; just don't accidentally split them across two appends.
- **Order in `tool_results` doesn't matter.** `tool_use_id` matches each result to the right request.

If your tools do I/O (a DB query, a vector search), you can run them concurrently with `asyncio.gather` or a thread pool. Wall-clock latency drops, total cost doesn't change. Worth doing once a single turn takes more than a second.

## Step cap as a forcing function

The hardest part of a step cap is the temptation to raise it when the agent isn't behaving. Resist. A step cap of 8 should solve more than 95% of well-shaped tasks. <!-- needs-research: cite Anthropic's "Building Effective Agents" guidance or a primary source for typical step counts; otherwise mark this as opinion --> When the cap fires:

- **Re-read the trace.** Was every Thought reasonable? If yes, the prompt is fine but the tools are weak. If Thoughts drift, the prompt or model needs work.
- **Look for near-duplicates.** Three calls to `search_kb` with rephrasings of the same query is the model trying to recover from a bad retriever. Fix retrieval.
- **Look for missing tools.** "I would need to know X, but I have no tool for that" — the model wrote the answer to your missing-tool question for you.

Raising the cap from 8 to 16 to 32 instead of fixing the root cause is the most expensive way to debug an agent.

## Error responses as observations

A tool that fails is a normal event in agent-land. Compare two reactions:

- **Crash.** `dispatch(...)` raises; you bubble it up; the user sees an error. The model never sees the failure; it doesn't learn from it; rerunning the same prompt fails the same way.
- **Observation.** The error becomes a `tool_result` with `is_error: True`; the model reads it; usually it apologizes or retries with a corrected argument. Sometimes it gives up cleanly: "I tried to look up that order but the system reported it doesn't exist."

The minimum loop above does the second. This is the single most important behavior for a usable agent. Without it, every flaky downstream service crashes your agent; with it, your agent gracefully degrades.

Two refinements worth knowing about:

- **Validate at the boundary.** If a tool expects `order_id: str` and the model passes `order_id: 123`, validate and return `{"error": "order_id must be a string"}` instead of crashing on the `**args` unpack. The model recovers; the production logs stay quiet.
- **Cap retries per tool.** If a single tool errors three times on the same arguments, the model is going to keep trying. Inject `{"error": "retry budget exceeded"}` and let the model give up.

Both come from `mod-103` chapter 6. Apply them here.

## A complete trace, end-to-end

Putting the pieces together for a query that exercises two tools:

```
[step 0]
  prompt: "Where is order ORD-1003, and is the item still in stock?"
  assistant text: "I'll check the order status first."
  tool_use: get_order_status(order_id="ORD-1003")

[step 0 observation]
  tool_result: {"order_id": "ORD-1003", "status": "shipped",
                "carrier": "UPS", "sku": "BL-MED-001"}

[step 1]
  assistant text: "Order shipped. Checking stock on BL-MED-001."
  tool_use: search_products(query="BL-MED-001")

[step 1 observation]
  tool_result: [{"sku": "BL-MED-001", "in_stock": true, ...}]

[step 2]
  assistant text: "ORD-1003 shipped via UPS, and BL-MED-001 is still in stock."
  stop_reason: end_turn
  -> return
```

Three model calls, two tool invocations, one final reply. Roughly two seconds of wall clock, a few hundred tokens of input, a hundred or so tokens of output. That is the size of agent you are building this module.

## Common mistakes

- **No step cap.** Free-tier-burning behavior. Always cap; eight is fine to start.
- **Wrapping tool errors as exceptions.** The model can't see them, can't recover, and the agent dies silently. Return them as observations.
- **Forgetting to append the assistant turn before the tool results.** The provider rejects the request; you spend an hour debugging an API error.
- **Sending non-string tool_result content.** `json.dumps` everything; even an empty success should be a JSON object, not an empty string.
- **Treating every retry as progress.** Two near-identical calls is fine; five is a loop. Detect and break.
- **Not surfacing the assistant text.** Your trace shows tool calls only; you cannot tell *why* the model picked them. Always log the text content too.
- **Raising `max_tokens` when the agent runs long.** That cap is per-call; the loop runs many calls. Raising it doesn't help and may mask a real problem (the model thinks it needs to write a huge final reply when actually it should keep using tools).

## Summary

The ReAct loop is a small, bounded Python function that calls the model with tools, executes any `tool_use` blocks, sends the results back as `tool_result`s, and stops when the model emits a final answer or you hit a step cap. The non-trivial work is in the guards: explicit handling for each stop reason, error-as-observation for tool failures, a step cap that is load-bearing, an input-token guard for long sessions, and a trace you can read after the fact. The next chapter wires retrieval in as one of those tools — the most useful and the most common one.
