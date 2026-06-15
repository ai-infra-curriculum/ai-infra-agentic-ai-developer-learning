# Retrieval as a Tool

## Why this matters

In `mod-104` you built a RAG loop that retrieved on *every* turn. That was the right baseline — it is cheap, predictable, and most user questions are about your documents anyway. But it has a real cost: you embed and search even when the user asks "thanks!" or "what's two plus two," and you can only search once per request, with the user's literal query, regardless of how good that query is.

In this chapter the agent decides. Retrieval becomes a tool the model calls when it judges it needs documents, and skips when it doesn't. The retriever you wrote in `mod-104` exercise-02 plugs in essentially unchanged. The change is on the protocol side.

This is the most useful tool you will ever expose to an agent. Almost every "AI app over our docs" in production today is this pattern.

## What changes when retrieval is a tool

Compare the two shapes for the same user question:

**`mod-104` (always retrieve):**

```
user question
   |
   v
embed query  -> retrieve top-K  -> stuff into prompt  -> model -> answer
```

Two model-side operations (embed + answer). Always exactly this shape.

**`mod-105` (agentic retrieve):**

```
user question -> model decides:
                    |
                    +-- needs docs -> search_kb(query) -> observation -> ...
                    |
                    +-- already knows -> answer
                    |
                    +-- needs different docs -> search_kb(refined query) -> ...
```

Variable shape per request. The model picks the query, picks K (within reason), can search again with a sharper query if the first result is weak, and can skip search entirely for chit-chat.

You give up some predictability and pay for the extra round trips you get. In return you get four real wins:

- **No search for trivial messages.** "thanks", "ok", "hi" — the agent answers without burning an embedding call and an index lookup.
- **Better queries.** The model can rephrase the user's three-word question into a richer search query the embedding model handles well.
- **Iterative refinement.** First retrieval too broad? The next Thought can write a sharper query. This recovers a lot of the "lost-in-the-middle" failures that plain RAG hits.
- **Multi-source synthesis.** The agent can retrieve, then look up a structured record, then retrieve again with information from the lookup. Pure RAG can't do this; it only does one search and stops.

## The tool definition

The retriever from `mod-104` exercise-02 was a Python function — `search(conn, query_vec, k)`. Wrapping it as a tool is two steps: write a schema the model can read, and write a thin dispatcher that calls your existing code.

```python
SEARCH_KB_TOOL = {
    "name": "search_kb",
    "description": (
        "Search the customer support knowledge base for passages relevant to a "
        "natural-language query. Use this when the user is asking a question "
        "whose answer is likely in our internal documentation (policies, FAQs, "
        "product manuals). Do NOT use it for chit-chat, math, or actions like "
        "placing an order. Returns up to 5 passages with chunk_id, score, "
        "section_title, and a text excerpt."
    ),
    "input_schema": {
        "type": "object",
        "required": ["query"],
        "properties": {
            "query": {
                "type": "string",
                "description": (
                    "A focused, complete natural-language question or phrase. "
                    "Prefer keyword-rich rephrasings of the user's question over "
                    "the literal user text. Example: user asks 'do you ship to Hawaii?' "
                    "-> query 'shipping policy Hawaii non-contiguous US states'."
                ),
            },
            "k": {
                "type": "integer",
                "description": "Number of passages to return. Default 5, max 10.",
                "minimum": 1,
                "maximum": 10,
            },
        },
    },
}


def search_kb(query: str, k: int = 5) -> list[dict]:
    q_vec  = embed_query(query)                       # from mod-104 ch.6
    chunks = search(conn, q_vec, min(k, 10))          # from mod-104 ch.6
    return [
        {
            "chunk_id":      c["chunk_id"],
            "score":         round(c["score"], 3),
            "section_title": c["section_title"],
            "excerpt":       c["text"][:600],         # truncate per chunk
        }
        for c in chunks
    ]
```

Five design choices worth calling out:

1. **The description tells the model *when* to call.** Not just what it does. The "do NOT use it for chit-chat" line is load-bearing — without it the model will dutifully search for "hi."
2. **The query parameter is documented as a *rephrasing*, not a forwarding.** The model takes the hint. Recall@K on the first retrieval improves substantially when the model is allowed to refine the query before searching.
3. **K is a parameter, capped at 10.** The model has limited freedom — usually picks 5 — but can ask for fewer on simple lookups or more on complex ones. The cap stops it from asking for fifty.
4. **The result includes the score.** The model uses it. Weak top hit (score 0.42) often triggers a second search with a refined query.
5. **Per-chunk text is truncated.** The agent might call `search_kb` two or three times in one trajectory; full chunks every time blow up the input-token budget fast.

## Combining retrieval with other tools

Retrieval is rarely the only tool. The interesting agent is the one that knows when to retrieve, when to look something up, and when to act.

A small support agent surface:

```python
TOOLS = [
    SEARCH_KB_TOOL,         # docs / FAQs / policies
    GET_ORDER_STATUS_TOOL,  # mod-103 exercise-01
    CREATE_REFUND_TOOL,     # mod-103 exercise-02
    CANCEL_ORDER_TOOL,      # mod-103 exercise-02
]
```

Four tools. The model now has to decide, for each user message:

| User message                                   | Likely path                                |
|------------------------------------------------|--------------------------------------------|
| "Where is ORD-1003?"                           | `get_order_status` → answer                |
| "What's your return policy on opened items?"   | `search_kb` → answer                       |
| "Can I cancel ORD-1003 if it already shipped?" | `get_order_status` → `search_kb` for the cancellation policy → answer |
| "Cancel ORD-1003."                             | `cancel_order` (action, no retrieval needed) |
| "Hi! 👋"                                       | direct answer, no tools                    |
| "I need a refund on ORD-1002, it arrived broken." | `search_kb` for refund policy → `create_refund` → answer |

You will not get every routing decision right on the first prompt. The descriptions are the main lever. Two specific descriptions kept tight will out-perform six vaguely-worded ones every time.

> Iteration on tool descriptions is the smallest unit of work in agent development. Reach for it before you reach for prompt rewrites or extra tools.

## When NOT to use agentic retrieval

A few cases where the always-retrieve baseline beats the agentic version:

- **The corpus is the only thing the agent talks about.** A docs chatbot that has no other tools rarely benefits from "decide whether to retrieve." Just retrieve. You save one round trip per request.
- **Latency is brutally constrained.** Each model call is roughly half a second of wall clock. Agentic retrieval adds one. If you have a 1-second SLA, pre-retrieve.
- **You need an audit trail of *what context was considered* on every answer.** Plain RAG always retrieves; the chunks are always logged. Agentic RAG sometimes doesn't retrieve, and "no chunks logged" is harder to defend in a compliance review.
- **Cost matters more than quality on a small corpus.** A corpus that fits in 8k tokens may be cheaper to *stuff* into the system prompt every turn and not retrieve at all. Yes, that is sometimes the right answer.

Pure RAG (`mod-104`) and agentic RAG (this module) coexist in real systems. They are tools in a kit, not generations of the same thing.

## Forcing retrieval when you must

`tool_choice` lets you make the model retrieve on specific conditions. Most of the time you want `tool_choice="auto"` and let the description do the work. The cases where it doesn't work:

- **Compliance / safety rule.** "For any policy question, you must show the source you're answering from." You can either force a `search_kb` call on every turn (defeats most of the win) or use a small classifier upstream to decide which user messages get an enforced retrieval. The classifier is usually the right answer.
- **Cold start.** A brand-new agent skill that the description hasn't yet steered into. A forced first retrieval gets the model used to using the tool; you can relax to `auto` once it's behaving.
- **Test reproducibility.** Forcing retrieval in evals lets you measure the retriever in isolation from the model's routing.

`tool_choice` is not a substitute for a good description. Treat it as a development aid, not a production knob.

## Iterative retrieval — the model gets a second swing

The agent shape unlocks a specific pattern that pure RAG can't do:

```
[step 0]
  Thought: "Looking up the return policy."
  Action:  search_kb(query="return policy")
  Observation: 5 chunks. Top score 0.42 (weak). Mostly general FAQ.

[step 1]
  Thought: "Top hit is generic; the user asked specifically about opened
            electronics. Searching with that phrase."
  Action:  search_kb(query="return policy opened electronics restocking fee")
  Observation: 5 chunks. Top score 0.78. Clauses 3 and 4 are exactly relevant.

[step 2]
  Thought: "I have what I need. Compose the answer."
  stop_reason: end_turn
```

Two retrievals, one answer, three round trips. The first retrieval was weak; the model adapted. With `mod-104`'s always-retrieve loop, that first weak result would have been the only context the model ever saw.

The cost is one extra round trip on questions where the first hit was bad. The benefit is recovering quality on exactly the questions where retrieval was failing.

The model decides to re-search based on the observation it got. If you don't surface the score, it can't tell the difference between a strong and weak hit, and you lose the trigger. Including the score is the smallest change that enables this behavior.

## Citations on agentic retrieval

In `mod-104` chapter 6 you returned `[chunk_id]` citations the model wrote into the final answer. The same pattern works here, with one wrinkle: now the model might have called `search_kb` two or three times before answering. The set of `chunk_id`s the model is allowed to cite is the *union* across all retrieval observations in this trajectory.

A small validator at the end:

```python
def cited_chunk_ids(final_text: str) -> set[str]:
    return set(re.findall(r"\[chunk_id:\s*([\w\-#]+)\]", final_text))

def all_seen_chunk_ids(trace: list[dict]) -> set[str]:
    seen = set()
    for step in trace:
        for tu in step["tool_uses"]:
            if tu["name"] != "search_kb":
                continue
            # we logged the observation alongside or downstream of this trace entry
            for chunk in tu.get("observation", []):
                seen.add(chunk["chunk_id"])
    return seen

invalid = cited_chunk_ids(final) - all_seen_chunk_ids(trace)
assert not invalid, f"fabricated citations: {invalid}"
```

If `invalid` is non-empty, the model invented a chunk_id. That is a useful signal, both for debugging and for production telemetry.

## Cost and latency shape

A naive agent that calls `search_kb` twice and answers once is roughly:

| Step                          | Cost                  | Latency       |
|-------------------------------|-----------------------|---------------|
| Model call 1 (Thought + Action)| input + small output  | 0.5–1.5 s     |
| `search_kb` (embed + lookup)  | one embedding call    | 50–200 ms     |
| Model call 2 (Thought + Action)| growing input + small | 0.5–1.5 s     |
| `search_kb` (embed + lookup)  | one embedding call    | 50–200 ms     |
| Model call 3 (final text)     | growing input + reply | 0.5–2 s       |

Total: roughly 2–5 seconds, 3 model calls, 2 embedding calls. The model calls dominate; the retrievals are negligible. The most effective optimizations are:

- **Cache the system prompt** if your provider supports prompt caching. The system prompt is the same across requests; caching it can drop the input-token cost meaningfully on long sessions.
- **Use the smaller / faster model for routing** if you have a multi-tier provider. Many production agents use a smaller model to decide the next step and a larger model only for the final reply. Out of scope here; worth knowing about.
- **Run independent tool calls in parallel.** Two `search_kb` calls in one assistant turn run concurrently if your dispatcher allows it.

## Common mistakes

- **A vague `search_kb` description.** The agent calls it for "hi" or skips it for "what's your return policy." Fix the description before doing anything else.
- **Returning whole chunks every time.** Two retrieval rounds × five chunks × 800 tokens = 8000 tokens of observation. Truncate per-chunk to a few hundred tokens.
- **Forgetting to surface the score.** The model can't decide to re-search if it can't tell the first hit was weak.
- **Letting the agent search with the literal user text.** Sometimes fine; sometimes terrible. Hint in the parameter description that the query should be a focused rephrasing.
- **No citation validation.** Hallucinated chunk_ids look correct until you check them. Add the validator.
- **Trying to do everything agentically.** A docs-only chatbot with no other tools is usually better off as plain RAG. Pick the shape that fits.

## Summary

Retrieval becomes a tool the agent decides to call. The retriever code from `mod-104` is unchanged; you wrap it in a `search_kb` definition whose description tells the model *when* to call it and whose result includes the score so the model can re-search on weak hits. Combine retrieval with action tools (status lookup, refund, cancel) and the agent now routes per request — sometimes retrieving, sometimes looking up, sometimes answering directly. The shape buys you query rewriting and iterative refinement that pure RAG can't do; it costs a few hundred milliseconds and a model call. The next chapter takes the agent across turns: memory.
