# Conversation Memory

## Why this matters

Everything you've built so far is *stateless* across user turns. Each `run_agent(user_message, ...)` call starts with an empty `messages` list. The agent remembers nothing from the previous question. Ask "where is ORD-1003?" and then "and when is it arriving?" — the second question fails because "it" has no referent.

This chapter is the smallest step from a one-shot agent to a usable conversational one. We will define what *memory* actually means for an LLM (it isn't magic — the model has no state of its own), how to maintain a transcript across turns, how to keep that transcript from blowing the context window, and how to write a small key:value scratchpad for facts that should outlive a single turn.

This is the floor of production memory. Real systems layer vector stores, user profiles, episodic logs, and graph-shaped relations on top. You will recognize all of those as extensions of the floor we lay here.

## What "memory" actually is for an LLM

The model itself has no memory between API calls. Every `messages.create(...)` is independent. What we call "the agent's memory" is *whatever you put back into the messages list on the next call*. There are three places memory can live, by how persistent it is:

| Memory layer       | Lives in                              | Survives          | Costs                         |
|--------------------|---------------------------------------|-------------------|-------------------------------|
| **Short-term**     | `messages` list, in the same process  | Until you drop it | Input tokens, every turn      |
| **Working / scratchpad** | Application state (a `dict`, a row in a DB) | Across turns in this session, optionally across sessions | Application state to maintain |
| **Long-term**      | External store (DB, vector index, user profile) | Across sessions, users, weeks | Storage + a retrieve-and-inject step per call |

Short-term memory is what makes "and when is it arriving?" work — the previous turn is still in the messages list. Working memory is what makes "the user already told me their order number is ORD-1003, no need to re-ask" work. Long-term memory is what makes "you said last week you prefer carriers other than UPS" work.

This module covers the first two. The third is out of scope here; chapter 6 names what to read next.

## Short-term: the running transcript

The simplest possible memory: keep the same `messages` list across turns.

```python
class AgentSession:
    def __init__(self, tools, dispatch):
        self.tools    = tools
        self.dispatch = dispatch
        self.messages = []                          # persisted across turns

    def chat(self, user_message: str, *, max_steps: int = 8) -> dict:
        self.messages.append({"role": "user", "content": user_message})

        for step in range(max_steps):
            response = client.messages.create(
                model=MODEL,
                max_tokens=1024,
                system=SYSTEM,
                tools=self.tools,
                messages=self.messages,
            )

            if response.stop_reason == "max_tokens":
                raise RuntimeError("hit max_tokens")

            if response.stop_reason == "end_turn":
                final_text = "".join(b.text for b in response.content if b.type == "text")
                self.messages.append({"role": "assistant", "content": response.content})
                return {"reply": final_text, "steps": step + 1}

            # tool_use path
            self.messages.append({"role": "assistant", "content": response.content})
            tool_results = []
            for block in response.content:
                if block.type != "tool_use":
                    continue
                payload = self.dispatch(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(payload),
                })
            self.messages.append({"role": "user", "content": tool_results})

        raise RuntimeError(f"exceeded max_steps={max_steps}")
```

That's it. The `messages` list now persists across calls to `chat()`, including the tool calls and their results from previous turns. The agent can resolve "it" because the previous user/assistant turns are still in front of it.

Two things to verify:

- **The final assistant turn must be appended.** A common bug is to return the reply but forget to push the `response.content` into `messages`. The next user turn then comes in with no record of the last reply, and the model gets confused.
- **Tool round trips from prior turns are kept.** Don't strip them out to save tokens — the model frequently refers back to a previous lookup ("when I checked your order, it was shipped via UPS..."), and removing the prior `tool_result` breaks that.

## The transcript will grow

A conversation with 20 user messages, each triggering 2–3 model calls and a tool round trip or two, produces a `messages` list of 80+ entries. Past some point — usually somewhere between 10k and 30k tokens depending on the model and your patience — three bad things happen:

1. **Cost climbs linearly.** You re-send the full transcript every turn. By turn 50, each new message is paying for 49 turns of history.
2. **Latency climbs.** Long input takes longer to prefill.
3. **Quality degrades.** Models attend less reliably to information buried in the middle of a long context. <!-- needs-research: cite "Lost in the Middle" (Liu et al., 2023) and current provider guidance on long-context attention -->

You have four levers, in order of how brutal they are.

### Lever 1: Drop tool round trips after they stop being relevant

A `tool_result` from 30 turns ago — say, an old order lookup — is rarely useful now. You can keep the assistant's *text* describing what was found ("I checked ORD-1003 and it shipped via UPS") and drop the `tool_use` / `tool_result` pair. The model now has a summary in conversational form and no raw tool payload.

This is the cheapest, safest reduction. It loses very little.

### Lever 2: Sliding window

Keep the system prompt, the last K turns, and drop the rest. A typical K is 10 or 20.

```python
def trim_window(messages, keep_last_turns=20):
    if len(messages) <= keep_last_turns * 2:
        return messages
    return messages[-keep_last_turns * 2:]
```

Crude but effective. The agent forgets the start of a long conversation. Often fine for support chat and search assistants; bad for tasks where the original request continues to matter ("I told you at the start I want vegetarian options").

### Lever 3: Rolling summarization

When the transcript hits a threshold, ask the model to summarize the older portion into a few sentences and replace those messages with the summary. The recent N turns stay verbatim; everything before that is compressed.

```python
def summarize_older(session, *, keep_verbatim=10):
    if len(session.messages) <= keep_verbatim * 2:
        return

    older = session.messages[:-keep_verbatim * 2]
    recent = session.messages[-keep_verbatim * 2:]

    summary_resp = client.messages.create(
        model=MODEL,
        max_tokens=400,
        system=(
            "Summarize this customer-support conversation for an assistant that "
            "needs to continue helping the customer. Capture: the customer's "
            "request, any order or refund IDs mentioned, any decisions made, "
            "and any preferences they expressed. Use short, factual sentences."
        ),
        messages=[{"role": "user", "content": _render_for_summary(older)}],
    )
    summary_text = summary_resp.content[0].text

    session.messages = [
        {"role": "user", "content": f"[summary of earlier conversation]\n{summary_text}"},
    ] + recent
```

The recent verbatim turns let the model see exactly how the conversation was going. The summary preserves the facts that matter — IDs, decisions, preferences — at a fraction of the token cost.

Two pitfalls:

- **Summaries lose nuance.** If the customer's tone matters ("they're already upset"), say so in the system prompt for the summarizer.
- **The summary itself can be wrong.** The summarizer is the same kind of LLM, with the same hallucination risk. Don't put high-stakes facts into a summary without a second pass.

This is the most common production pattern. LangChain, LlamaIndex, the Anthropic Claude Agent SDK, and most "memory" examples in tutorials boil down to "summarize and replace."

### Lever 4: Token-budget-driven hybrid

Production systems combine the three: drop irrelevant tool round trips greedily, slide a window over the recent past, and summarize anything older when you cross a token threshold. The result is a transcript that stays under a soft budget regardless of how long the user keeps chatting.

```python
def maintain_transcript(session, *, soft_token_cap=8000):
    if estimated_tokens(session.messages) <= soft_token_cap:
        return

    drop_stale_tool_payloads(session.messages, older_than_turns=5)
    if estimated_tokens(session.messages) <= soft_token_cap:
        return

    summarize_older(session, keep_verbatim=10)
```

`estimated_tokens` can be a literal call to `tiktoken` (OpenAI) or Anthropic's token-counting endpoint, or a rough `len(json.dumps(messages)) // 4` for ballpark.

The exact tuning is corpus-dependent. Set a budget, watch what gets dropped, adjust.

## Working memory: a key:value scratchpad

Some facts deserve a more durable home than the running transcript. The user's stated email address, the order they're talking about this session, a confirmation that they accepted a quote — these should not silently vanish when the transcript slides or summarizes.

The simplest pattern is a small dict and two tools:

```python
class AgentSession:
    def __init__(self, tools, dispatch):
        ...
        self.scratchpad: dict[str, str] = {}

REMEMBER_TOOL = {
    "name": "remember",
    "description": (
        "Save a single fact about the current conversation to a key:value "
        "scratchpad that survives across turns. Use for user-stated facts you "
        "expect to need later: order IDs, email addresses, preferences, "
        "confirmations. Keys are short slugs (e.g. 'current_order_id'). "
        "Overwrites existing values for the same key."
    ),
    "input_schema": {
        "type": "object",
        "required": ["key", "value"],
        "properties": {
            "key":   {"type": "string", "description": "A short slug like 'current_order_id' or 'preferred_carrier'."},
            "value": {"type": "string", "description": "The fact to remember, as a short string."},
        },
    },
}

RECALL_TOOL = {
    "name": "recall",
    "description": (
        "Read back a fact previously saved with the `remember` tool. Returns "
        "the saved value or {\"error\": \"not_found\"} if no such key exists."
    ),
    "input_schema": {
        "type": "object",
        "required": ["key"],
        "properties": {"key": {"type": "string"}},
    },
}

def remember(session: AgentSession, key: str, value: str) -> dict:
    session.scratchpad[key] = value
    return {"ok": True, "key": key}

def recall(session: AgentSession, key: str) -> dict:
    if key in session.scratchpad:
        return {"key": key, "value": session.scratchpad[key]}
    return {"error": "not_found"}
```

You inject the current scratchpad into the system prompt at the start of every call so the model knows what's in there without having to call `recall` every turn:

```python
def build_system():
    if session.scratchpad:
        lines = [f"- {k}: {v}" for k, v in session.scratchpad.items()]
        memory_block = "Known facts about this session:\n" + "\n".join(lines)
    else:
        memory_block = "No saved session facts yet."
    return BASE_SYSTEM + "\n\n" + memory_block
```

Now the agent has a tiny, deliberate memory that survives transcript pruning. The model writes to it when something is worth keeping; you render it back on every turn at near-zero cost.

Two notes:

- **Scratchpads are not vector memory.** You're not embedding anything; you're not doing similarity search. This is the dumbest, smallest, most reliable memory primitive. Reach for vectors only when the scratchpad runs out of room or you need to retrieve from many sessions at once.
- **You can ship the scratchpad to a DB.** Same pattern, different backing store. The persistence layer is invisible to the agent — it only sees `remember` and `recall`.

## Memory and the multi-tenant case

Once your agent is online, scratchpads belong to *users*, not to processes. The minimum changes:

- **Key the scratchpad by user_id.** A `Postgres` row, a Redis hash, whatever — `(user_id, key)` is unique.
- **Load on session start; persist on session end** (or write-through on every change).
- **Scope keys carefully.** A key like `preferred_carrier` is per-user; a key like `current_order_id` is per-session-within-user.

The same model holds for transcripts: store them with a session_id, hydrate on resume, summarize on close. This is the line between a memory-aware *agent* and a memory-aware *service*; the service work is `mod-106`.

## Anti-patterns that look like memory but aren't

A few things people reach for that don't help:

- **Re-asking the model to "remember this."** "Please remember that my order is ORD-1003" doesn't write anything — there's nothing under the hood to write to. The model just notes it for *this turn's* context. Next turn it's gone unless your transcript persists.
- **Telling the model "you have memory."** It doesn't, unless you give it tools that act like memory. A model that *claims* to have memory is hallucinating.
- **Using a vector store for everything.** Vector memory is great for "find the relevant past fact." For "this is the order_id I'm working on right now," it's a thousand times more complicated than a `dict`. Pick the cheapest tool that works.

## Common mistakes

- **Not appending the final assistant turn back into messages.** The model has no record of what it just said; the next user turn lands with broken context.
- **Stripping tool round trips too aggressively.** "It's old, drop it" is fine for trivia. For the order you just looked up, you usually want to keep at least the assistant's summary text.
- **No transcript cap at all.** Conversation runs long; bill triples; latency triples; quality drops. Pick a budget and trim.
- **Summarizing with the same system prompt as the agent.** The summarizer is a different task. Give it its own instructions.
- **A scratchpad that nobody reads.** The model saves a fact and never references it because you forgot to render the scratchpad into the system prompt. Render it.
- **Trusting summaries with high-stakes facts.** A summary that misses "the customer agreed to a partial refund of $40" is worse than no summary at all. Keep critical facts in the scratchpad.

## Summary

Memory for a single agent is a layered thing: the running transcript (the messages list) carries short-term context; a token-budget policy keeps it from blowing up via tool-payload pruning, sliding windows, and rolling summarization; and a small key:value scratchpad — exposed as `remember` and `recall` tools — gives the agent a durable place to write facts that should outlive trimming. None of this is magic and none of it is new. It is mechanical state management around an LLM that has none of its own. The next chapter takes an honest look at where this one-agent shape stops being enough.
