# exercise-03: Basic Conversation Memory

**Estimated effort:** 2 hours

## Objective

Take the single-shot agent from exercise-02 across turns. You will maintain a running transcript so the user can ask follow-up questions, add a token-budget policy that trims and summarizes when the conversation grows, and expose a small `remember` / `recall` scratchpad so durable facts (order IDs, preferences, confirmations) survive both trimming and summarization.

By the end you should be able to hold a 15+ turn conversation with the agent, watch the transcript stay under a soft token budget, and demonstrate that a fact saved in the scratchpad is still recoverable after the rest of the transcript has been summarized away.

## Background

This exercise covers material from:

- [Chapter 4 — Retrieval as a Tool](../04-retrieval-as-a-tool.md) (you reuse `search_kb`)
- [Chapter 5 — Conversation Memory](../05-conversation-memory.md)

## Prerequisites

- Completed exercise-02 (a working three-tool agent: `search_kb`, `get_order_status`, `create_refund`).
- An API key for one provider with native tool use (Anthropic or OpenAI).
- Python 3.10+, the provider SDK, and `tiktoken` (for token estimation; or use Anthropic's `client.beta.messages.count_tokens` endpoint).
- A small spend cap; this exercise stays well under a dollar.

## The task

You will turn exercise-02's stateless `run_agent(...)` into a stateful `AgentSession.chat(...)` and add three things, layered in this order:

1. **A running transcript** that persists across `chat()` calls.
2. **A token-budget policy** that, when the transcript estimates over a soft cap, prunes stale tool round trips and (if still over) rolls older turns into a summary.
3. **A `remember` / `recall` scratchpad** exposed as two new tools, rendered into the system prompt each turn.

Then you will drive the session through a multi-turn conversation that exercises follow-ups, transcript pressure, and scratchpad durability.

## Tasks

### 1. The session class

Build `AgentSession` matching the shape in chapter 5:

```python
class AgentSession:
    def __init__(self, tools, dispatch, *, soft_token_cap: int = 6000):
        self.tools          = tools
        self.dispatch       = dispatch
        self.messages       = []
        self.scratchpad     = {}
        self.soft_token_cap = soft_token_cap

    def chat(self, user_message: str, *, max_steps: int = 8) -> dict:
        ...
```

The internal loop is exercise-02's `run_agent` body. The differences:

- `messages` is `self.messages`, not local.
- Before the first `messages.create(...)` call in each `chat()`, run a transcript-maintenance pass (task 3 below).
- Build the system prompt fresh on every turn to render the current scratchpad (task 4 below).
- Return `{"reply": final_text, "steps": N, "transcript_tokens_after": ...}`.

### 2. The follow-up suite

Reuse the three tools from exercise-02 (`search_kb`, `get_order_status`, `create_refund`). Run a single `AgentSession` through this multi-turn script and write each `(user, reply)` to `session.jsonl`:

1. user: "Where is order ORD-1001?"
2. user: "What carrier is that?"   *(must resolve "that" from turn 1)*
3. user: "Can I cancel it now?"   *(must remember the order ID *and* hit the policy in the KB)*
4. user: "Actually I want a refund instead — it arrived broken."   *(should create a refund without re-asking the order ID)*
5. user: "What's your return window on opened electronics?"   *(a fresh policy question; no order context needed)*
6. user: "Got it. So for that headset I ordered — am I inside the window?"   *(must combine ORD-1001's SKU/category with the policy from turn 5)*

For each turn, capture in `NOTES.md`:

- Whether the agent resolved the pronoun ("that", "it", "the headset I ordered") correctly.
- The token count of `self.messages` after the turn (you'll need a small helper).
- The tools called.

If the agent fails to resolve a follow-up correctly, that is a finding — *do not* fix it by rewording the prompts; explain in `NOTES.md` why the failure happened (transcript trimmed too early, scratchpad not rendered, etc.).

### 3. Token-budget policy

Implement the maintenance pass from chapter 5:

```python
def maintain_transcript(self):
    if self._token_count() <= self.soft_token_cap:
        return
    self._drop_stale_tool_payloads(older_than_turns=5)
    if self._token_count() <= self.soft_token_cap:
        return
    self._summarize_older(keep_verbatim=4)
```

Where:

- `_token_count()` estimates the cost of `self.messages` (use `tiktoken` for OpenAI tokenization, Anthropic's `count_tokens` for Claude, or `len(json.dumps(self.messages)) // 4` as a rough ballpark).
- `_drop_stale_tool_payloads(older_than_turns=N)` walks `self.messages` and replaces old `tool_use`/`tool_result` blocks with a single `text` block like `"[tool {name} called, result elided]"`. Keep the recent N turns intact.
- `_summarize_older(keep_verbatim=4)` calls the model with a dedicated summarization system prompt (see chapter 5) over all but the last 4 turns and replaces them with one synthetic user turn `"[summary of earlier conversation] ..."`.

Set `soft_token_cap = 6000` to force the policy to trigger during the session (a realistic cap for this corpus and prompt suite).

After the maintenance pass runs, log to `transcript_events.jsonl`:

```json
{"turn": N, "action": "drop_stale" | "summarize" | "noop",
 "tokens_before": ..., "tokens_after": ..., "messages_kept": M}
```

### 4. The scratchpad

Add two tools:

- **`remember(key, value)`** — sets `self.scratchpad[key] = value` and returns `{"ok": True, "key": key}`.
- **`recall(key)`** — returns `{"key": key, "value": ...}` or `{"error": "not_found"}`.

Use the descriptions from chapter 5 verbatim (or write your own — but make sure the description tells the model *when* to save a fact, with examples).

Render the scratchpad into the system prompt every turn:

```python
def _build_system(self):
    if self.scratchpad:
        lines = [f"- {k}: {v}" for k, v in self.scratchpad.items()]
        memory_block = "Known facts about this session:\n" + "\n".join(lines)
    else:
        memory_block = "No saved session facts yet."
    return BASE_SYSTEM + "\n\n" + memory_block
```

Update `BASE_SYSTEM` to include a one-sentence nudge: "When the user states a fact you expect to need later in the conversation (order IDs, preferences, confirmations), save it with the `remember` tool."

### 5. Scratchpad durability

After running the follow-up suite, run two additional turns:

7. user: "By the way, I prefer FedEx for any future shipments."   *(should trigger `remember` to save `preferred_carrier: FedEx`)*
8. (Force a transcript collapse: optionally inject 4–6 chit-chat turns to push `messages` past the cap and trigger summarization.)
9. user: "Remind me — what carrier did I say I preferred?"   *(after summarization, the scratchpad is still in the system prompt; the model should answer without calling `recall`, or call `recall` if it forgot)*

Capture the trace in `scratchpad.jsonl` showing:

- The `remember` call at turn 7.
- The summarization event at turn 8 (from `transcript_events.jsonl`).
- The answer to turn 9 and whether it required a `recall` call.

This is the test that the scratchpad outlives transcript pressure. If it doesn't, find out why (scratchpad not in the system prompt? rendering bug? summarizer overwrote it?) and fix it.

### 6. Read the session

In `NOTES.md`, write a short paragraph (5–8 sentences) for each:

- The follow-up handling (turns 1–6). Where did the agent shine? Where did it fail to chain context?
- The maintenance pass. Did `drop_stale` solve it, or did you need the summarizer too? What did the summary look like — did it preserve the critical fact (the order ID, the refund) or did it lose it?
- The scratchpad. Was it durable across summarization? If not, what does that say about *which* memory layer should hold which kind of fact?

## Starter guidance

A skeleton matching chapter 5:

```python
class AgentSession:
    def __init__(self, tools, dispatch, *, soft_token_cap: int = 6000):
        self.tools          = tools
        self.dispatch       = dispatch
        self.messages       = []
        self.scratchpad     = {}
        self.soft_token_cap = soft_token_cap
        self.events         = []   # transcript_events.jsonl rows

    def chat(self, user_message: str, *, max_steps: int = 8) -> dict:
        self.maintain_transcript()
        self.messages.append({"role": "user", "content": user_message})
        system = self._build_system()

        for step in range(max_steps):
            resp = client.messages.create(
                model=MODEL, max_tokens=1024,
                system=system, tools=self.tools,
                messages=self.messages,
            )

            if resp.stop_reason == "max_tokens":
                raise RuntimeError(f"max_tokens at step {step}")
            if resp.stop_reason == "end_turn":
                final = "".join(b.text for b in resp.content if b.type == "text")
                self.messages.append({"role": "assistant", "content": resp.content})
                return {"reply": final, "steps": step + 1,
                        "transcript_tokens_after": self._token_count()}

            self.messages.append({"role": "assistant", "content": resp.content})
            tool_results = []
            for b in resp.content:
                if b.type != "tool_use":
                    continue
                try:
                    payload = self.dispatch(self, b.name, b.input)
                except Exception as e:
                    payload = {"error": str(e)}
                tool_results.append({
                    "type": "tool_result", "tool_use_id": b.id,
                    "content": json.dumps(payload),
                })
            self.messages.append({"role": "user", "content": tool_results})

        raise RuntimeError(f"exceeded max_steps={max_steps}")
```

Note the dispatcher now takes `self` so `remember` and `recall` can mutate `self.scratchpad`. Adjust the other tools accordingly (or accept `self` and ignore it).

Do **not** add per-user persistence (a DB row keyed by user_id). The session is in-memory only for this exercise; multi-tenant memory is `mod-106` territory.

## Things you should not do in this exercise

- Use LangChain's `ConversationSummaryMemory` or any framework's memory module. Build your own; the point is to see the seams.
- Embed the scratchpad. Two tools and a dict are the right shape. Vector memory is its own course.
- Raise `soft_token_cap` to "make the maintenance pass not fire." Forcing the policy is the point.
- Add a fourth or fifth user-visible feature beyond memory. Keep the surface focused.

## Acceptance criteria

You can demonstrate:

- A working `AgentSession` class running the 6+ turn follow-up suite, with `session.jsonl` capturing each `(user, reply)` pair plus the post-turn token count.
- A `transcript_events.jsonl` showing at least one `drop_stale` event and at least one `summarize` event during the session (the cap is tuned to force this).
- A `scratchpad.jsonl` proving that a fact saved at turn 7 is still recoverable at turn 9 (after a summarization event).
- `NOTES.md` containing:
  - The three short paragraphs from task 6.
  - The pinned `MODEL`, `soft_token_cap`, summarizer settings, and `max_steps`.
  - One observation about a follow-up the agent got wrong — if any — and your read on whether the cause was prompt, transcript management, or the scratchpad.

## Reflection

Answer briefly in `NOTES.md`:

1. The summarizer is just another LLM call. What's the worst thing it could do to your session that you would not notice — and how would you catch it in production?
2. The scratchpad stores facts as strings. When does that break? Sketch one example fact that is awkward to store as a single string and one that would be fine.
3. You now have three memory layers: transcript, scratchpad, and (implicitly) the model itself. For each of these facts, which layer would you put it in?
   - The user's order ID for this session.
   - The user's preferred shipping carrier across all sessions.
   - The fact that "opened electronics have a 14-day return window."
4. If the session ran for an hour and accumulated 200 turns, which of your three layers is the bottleneck — and is the right fix to engineer harder against the bottleneck, or to graduate to a different architecture?

## Stretch goals

- **Persist the session to disk.** On `__del__` / a `save()` method, write `messages` and `scratchpad` to a JSON file keyed by a session ID. On `AgentSession.load(session_id)`, hydrate. Now the agent survives a process restart.
- **Per-user scratchpads.** Add a `user_id` parameter to `AgentSession`; store scratchpads in a small SQLite table keyed by `(user_id, key)`. The transcript stays per-session.
- **Vector memory.** When `drop_stale` runs, instead of replacing dropped tool payloads with a placeholder, embed the text content (a one-sentence summary of the assistant turn that follows the tool call) and store it in a small vector index. Add a `recall_episode(query)` tool that retrieves the closest such summary. This is the seed of "long-term memory" — try one user turn that benefits from it ("did we ever talk about my last order?") and note whether it helped.
- **Eval the summarizer.** Pick three sessions, summarize each with two different summarizer system prompts, and judge (by hand) which preserves more action-relevant facts. This is the cheapest way to see why summarizer tuning is a real ongoing task in production systems.
