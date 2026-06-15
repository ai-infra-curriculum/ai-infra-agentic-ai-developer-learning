# exercise-01: ReAct Loop Intro

**Estimated effort:** 3 hours

## Objective

Build the smallest possible ReAct loop on top of native tool use. You will expose two toy tools, run the loop until the model emits a final answer, surface every Thought to stdout so you can read along, and stress the guardrails with prompts that try to make the loop spin or wedge.

By the end you should be able to point at every line of your loop and explain why it's there — including the step cap, the error-as-observation path, and the trace.

## Background

This exercise covers material from:

- [Chapter 1 — From Tool Loop to Agent](../01-from-tool-loop-to-agent.md)
- [Chapter 2 — The ReAct Pattern: Thought, Action, Observation](../02-the-react-pattern.md)
- [Chapter 3 — Implementing the ReAct Loop](../03-implementing-the-react-loop.md)

## Prerequisites

- Completed `mod-103` exercise-01 (a single tool, the call/result loop). You will reuse its loop skeleton and dispatcher pattern.
- An API key for one provider with native tool use (Anthropic or OpenAI).
- Python 3.10+ with the provider SDK installed.
- A small spend cap. This exercise should cost well under a dollar.

## The task

You are going to build a ~100-line script that:

1. Defines two toy tools — one read-only, one mutating.
2. Runs a ReAct loop over a single user message.
3. Prints, per step, the assistant's Thought text and the tool call (name + args) it is about to make.
4. Caps the loop at 8 steps and handles each `stop_reason` explicitly.
5. Turns tool errors into observations rather than crashing the loop.

You will then drive the agent through a small prompt suite designed to exercise multi-step trajectories, parallel calls, and error recovery.

### The tools

Two tools, scoped to keep the surface tiny:

1. **`get_weather(city: str) -> dict`** — return a fake weather record from a small in-memory dict:

   ```python
   WEATHER = {
       "Seattle":   {"tempF": 54, "conditions": "rain"},
       "Phoenix":   {"tempF": 102, "conditions": "sun"},
       "Boston":    {"tempF": 41, "conditions": "snow"},
       "Tokyo":     {"tempF": 68, "conditions": "cloudy"},
   }

   def get_weather(city: str) -> dict:
       if city not in WEATHER:
           raise ValueError(f"unknown city: {city}")
       return {"city": city, **WEATHER[city]}
   ```

   Description: "Return current weather (temperature in F, conditions) for a known city. Use for any weather-related question. Raises if the city is unknown."

2. **`set_thermostat(target_f: int, note: str | None = None) -> dict`** — record into a `THERMOSTAT_LOG: list = []` (an in-memory list) the requested temperature and optional note. Return `{"ok": True, "set_to_f": target_f}`. Validate that `60 <= target_f <= 80` and otherwise raise.

   Description: "Set the thermostat to a target temperature in Fahrenheit. Use when the user explicitly asks to change the temperature. Acceptable range is 60–80°F."

That is the entire surface. Two tools is enough to exercise routing, multi-step calls, and error recovery without drowning in detail.

## Tasks

### 1. Build the loop

Write a single function `run_agent(user_message: str, *, max_steps: int = 8) -> dict` matching the structure in chapter 3:

- Sends the user message + tool definitions to the model.
- For each step, prints `[step N] Thought: ...` followed by `[step N] Action: <tool>(<args>)` for any tool calls.
- Executes each `tool_use` block via a dispatcher that catches exceptions and returns `{"error": str(e)}` as the observation.
- Stops on `end_turn` (return final text) or `max_tokens` (raise) or `max_steps` (raise).
- Returns `{"final_text": ..., "trace": [...], "steps": N}`.

Pin `MODEL` and `TEMPERATURE = 0` at the top of the script.

### 2. System prompt

Set a short system prompt that nudges the model to write a Thought before each tool call. From chapter 2:

```
You are a helpful assistant. You may call tools to look up information or
perform actions. Before calling a tool, briefly state in one sentence what
you are looking for and which tool you intend to use. After receiving a tool
result, briefly state what you learned before deciding the next step.
```

This is what makes the printed Thoughts readable.

### 3. Run the prompt suite

Run each prompt, append the resulting trace to `runs.jsonl` as one row per prompt:

```json
{"prompt": "...", "steps": N, "trace": [...], "final_text": "..."}
```

Suite (all should complete in 8 steps or fewer):

1. "What's the weather in Seattle?"  *(one tool call)*
2. "How does Phoenix compare to Boston right now?"  *(two tool calls; possibly parallel)*
3. "Set the thermostat to 72 because it's cold."  *(one mutating call)*
4. "It's snowing in Boston — turn the heat up a little for me."  *(weather → thermostat chain)*
5. "Hi there."  *(no tools; direct reply)*
6. "What's the weather in Atlantis?"  *(error path; model should apologize gracefully)*
7. "Set the thermostat to 95."  *(validation error; model should report the limit)*

For each prompt note in `NOTES.md`:

- Number of steps (model calls).
- Which tools were called and in what order.
- Whether prompt 2's calls were parallel (single assistant turn with two `tool_use` blocks) or serial (two round trips).
- For prompts 6 and 7: did the model recover gracefully or pass the raw error to the user?

### 4. Break the loop on purpose

Run two adversarial prompts intended to cause failures and watch the guardrails fire:

1. **Step-cap probe.** "Tell me the weather in Seattle, then Phoenix, then Boston, then Tokyo, then Seattle again, then Phoenix again, then write a haiku about all four." Does the model take more than 8 steps? If so, your step cap fires; report what was in the partial trace. (If it batches the calls in parallel, raise `max_steps=4` for this prompt and try again.)
2. **Unknown-tool probe.** Manually inject an assistant turn with a `tool_use` block whose `name` is `summon_dragon`. Confirm your dispatcher returns `{"error": "unknown_tool"}` (or similar) rather than raising — and that the model recovers ("I can't do that, but…").

Capture the partial traces in `adversarial.jsonl`.

### 5. Read your traces

In `NOTES.md`, write a short paragraph (4–8 sentences) for each of:

- The most interesting trajectory in your suite. What did the Thought slot say at the pivot point?
- The error-recovery prompts (6 and 7). What did the model write in the Thought after seeing the error observation?
- Prompt 2 (the comparison). Did the model issue parallel tool calls? If yes, was the trace one assistant turn with two blocks, or two assistant turns? What did your loop output to stdout?

## Starter guidance

A scaffold matching chapter 3, Anthropic flavor:

```python
import json
from anthropic import Anthropic

client = Anthropic()
MODEL  = "claude-sonnet-4-6"

SYSTEM = """You are a helpful assistant. You may call tools to look up
information or perform actions. Before calling a tool, briefly state in one
sentence what you are looking for and which tool you intend to use. After
receiving a tool result, briefly state what you learned before deciding the
next step."""

TOOLS = [GET_WEATHER_TOOL, SET_THERMOSTAT_TOOL]   # define above

def dispatch(name: str, args: dict):
    if name == "get_weather":     return get_weather(**args)
    if name == "set_thermostat":  return set_thermostat(**args)
    raise RuntimeError(f"unknown tool: {name}")

def run_agent(user_message: str, *, max_steps: int = 8) -> dict:
    messages = [{"role": "user", "content": user_message}]
    trace = []
    for step in range(max_steps):
        resp = client.messages.create(
            model=MODEL,
            max_tokens=1024,
            system=SYSTEM,
            tools=TOOLS,
            messages=messages,
        )
        text = "".join(b.text for b in resp.content if b.type == "text")
        tool_uses = [b for b in resp.content if b.type == "tool_use"]

        if text:
            print(f"[step {step}] Thought: {text}")
        for b in tool_uses:
            print(f"[step {step}] Action:  {b.name}({b.input})")

        trace.append({"step": step, "stop_reason": resp.stop_reason,
                      "text": text, "tool_uses": [
                          {"id": b.id, "name": b.name, "input": b.input} for b in tool_uses
                      ]})

        if resp.stop_reason == "max_tokens":
            raise RuntimeError(f"hit max_tokens at step {step}")
        if resp.stop_reason == "end_turn":
            return {"final_text": text, "trace": trace, "steps": step + 1}

        messages.append({"role": "assistant", "content": resp.content})
        tool_results = []
        for b in tool_uses:
            try:
                payload = dispatch(b.name, b.input)
            except Exception as e:
                payload = {"error": str(e)}
            tool_results.append({
                "type": "tool_result", "tool_use_id": b.id,
                "content": json.dumps(payload),
                **({"is_error": True} if "error" in payload else {}),
            })
        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError(f"exceeded max_steps={max_steps}")
```

OpenAI is the same shape with `tool_calls`, JSON-string `arguments`, and a `role: "tool"` follow-up message.

Do **not** add a tool registry class or a planning step yet. Both come in exercise-02 and exercise-03.

## Things you should not do in this exercise

- Add a third tool to "make routing more interesting." Two tools is the point; you are testing the loop, not the surface.
- Wrap the loop in a class or hide it behind a framework (LangChain, Anthropic Agent SDK, OpenAI Agents SDK). The whole point is to see the protocol.
- Raise `max_steps` past 8 to "make the adversarial prompt pass." That defeats the guardrail.
- Add memory between calls. Each `run_agent(...)` call is one-shot. Memory lands in exercise-03.

## Acceptance criteria

You can demonstrate:

- A working script that runs all seven suite prompts and writes `runs.jsonl` rows for each.
- Stdout output that includes a `Thought:` line and an `Action:` line for every tool-using step.
- `MODEL`, `TEMPERATURE`, and `max_steps` pinned and reported in `NOTES.md`.
- `adversarial.jsonl` containing the two adversarial runs and a description of what your guardrails did.
- `NOTES.md` containing:
  - Three short paragraphs from task 5 (most interesting trajectory, error recovery, parallel calls).
  - The pinned parameters.
  - One observation about how readable the trace was — and what you would change about the Thought-prompt to make it more readable.

## Reflection

Answer briefly in `NOTES.md`:

1. For prompt 2 (Phoenix vs. Boston), was the model's choice of parallel vs. serial the right one for your loop's design? Would you force the model into either shape, and why?
2. For prompt 6 (unknown city), suppose `get_weather` had returned `{"error": "unknown_city"}` instead of raising. Would the model's recovery have looked different? What does that tell you about whether tool errors belong as exceptions or as structured returns?
3. Look at your longest trace by step count. What is the smallest change to your tools or system prompt that would have shortened it by at least one step?

## Stretch goals

- **Emit a structured trace.** Save each step as `{"thought": str, "action": {"name": str, "input": {...}}, "observation": ...}` and write a small renderer that prints them as `Thought → Action → Observation` blocks.
- **Add a `final_answer` tool.** Force the model to end the trajectory by calling a `final_answer(text)` tool instead of relying on `end_turn`. Note how this changes the loop's termination condition and the kind of bugs you hit.
- **Re-run the suite on OpenAI.** Same loop shape, different protocol. Note the two or three lines that change — and which behaviors differ between the providers (parallel-calls default, talkativeness, willingness to skip tools on chit-chat).
