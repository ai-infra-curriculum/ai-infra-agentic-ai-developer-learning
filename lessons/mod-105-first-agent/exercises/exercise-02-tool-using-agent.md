# exercise-02: Tool-Using Agent

**Estimated effort:** 3 hours

## Objective

Wire the `mod-104` retriever in as a `search_kb` tool alongside two action tools, and watch a single agent route between them. You will run an agent that retrieves, looks up, and acts — sometimes in one trajectory — and learn to read the trace when it picks the wrong move.

By the end you should be able to ship a small support agent whose every step you can defend: which tool ran, why, what came back, and whether the final answer was grounded in retrieved context.

## Background

This exercise covers material from:

- [Chapter 3 — Implementing the ReAct Loop](../03-implementing-the-react-loop.md)
- [Chapter 4 — Retrieval as a Tool](../04-retrieval-as-a-tool.md)
- (Touches Chapter 5 of `mod-103` — When to Call a Tool vs. Generate.)

## Prerequisites

- Completed exercise-01 (a working ReAct loop with a step cap and error-as-observation).
- Completed `mod-104` exercise-02 (a working retriever over a small markdown corpus). You will reuse `embed_query(...)` and `search(conn, q_vec, k)` essentially verbatim.
- An API key for one provider with native tool use (Anthropic or OpenAI).
- Python 3.10+, the provider SDK, and access to the SQLite/NumPy index from `mod-104` exercise-02 (or equivalent).
- A small spend cap. Still under a dollar.

## The task

You will extend the ReAct loop from exercise-01 with three tools that together form a credible small support agent:

1. **`search_kb`** — wraps your `mod-104` retriever.
2. **`get_order_status`** — same as `mod-103` exercise-01 (you may reuse the function).
3. **`create_refund`** — idempotent refund creator (same as `mod-103` exercise-02).

Then you will drive the agent through a routing suite and a retrieval-quality suite, and finally validate that any citations the agent emits trace to a chunk it actually retrieved.

### The tools

Schemas to write:

1. **`search_kb(query: str, k: int = 5)`** — return up to `k` chunks (cap at 10) as a list of `{chunk_id, score, section_title, excerpt}`. Truncate `excerpt` to ~600 characters. Description must steer the model away from chit-chat — see chapter 4 for a worked example.

2. **`get_order_status(order_id: str)`** — look up a fake order from a small dict and return `{order_id, status, carrier, sku}` for known orders or `{"error": "not_found"}` for unknown ones (returning the error as data, not raising). Use a small `ORDERS` dict like:

   ```python
   ORDERS = {
       "ORD-1001": {"status": "shipped",     "carrier": "UPS", "sku": "BL-MED-001"},
       "ORD-1002": {"status": "processing",  "carrier": "USPS", "sku": "RD-MED-001"},
       "ORD-1003": {"status": "delivered",   "carrier": "FedEx", "sku": "HS-001"},
   }
   ```

3. **`create_refund(order_id, amount_cents, reason, idempotency_key)`** — write into `REFUNDS = {}` keyed by `idempotency_key`. Reason is one of `damaged | wrong_item | not_received | other`. Return the created (or existing) refund record.

### The corpus

Use the same corpus from `mod-104` exercise-02 *or* the small support corpus below, embedded once into your SQLite/NumPy index:

```text
# returns_policy.md
We accept returns of unused items within 30 days of delivery.
Opened electronics: 14-day return window with a 15% restocking fee (clause 3).
Final-sale items are not returnable.

# shipping_policy.md
Standard shipping to the contiguous US: 2–4 business days.
Shipping to Hawaii and Alaska (non-contiguous US states): 5–7 business days.
International orders ship via UPS Worldwide with tracking included.

# refunds_policy.md
Refunds appear on the original payment method within 5–10 business days.
For damaged goods, we will refund the full purchase price and return shipping.
```

(If you used a different corpus in `mod-104`, that is fine. Just make sure it has at least one returns/refunds-style chunk so the routing prompts below can ground in it.)

## Tasks

### 1. Wire the three tools through a single dispatcher

Refactor exercise-01's `dispatch` to look up by tool name and call into the right implementation. A small dict of name → callable is enough; do not build a generic registry class unless you want to (you wrote one in `mod-103` exercise-02 — feel free to import it).

The loop itself does not change from exercise-01. Confirm with a quick smoke prompt ("What's your return policy?") that the agent calls `search_kb` and answers.

### 2. Routing suite

Run each prompt and capture rows in `routing.jsonl` of the form:

```json
{"prompt": "...", "trace": [...], "tools_called": [{"name": "...", "input": {...}}, ...], "final_text": "..."}
```

Suite:

1. "What's your return policy on opened headphones?"  *(should call `search_kb` only)*
2. "Where is order ORD-1001?"  *(should call `get_order_status` only)*
3. "Can I cancel an order that already shipped?"  *(should call `search_kb` for the policy, not touch `get_order_status` since no order ID was given)*
4. "I need a refund on ORD-1002 — it arrived broken."  *(should call `search_kb` for the refund policy, look up the order, and create a refund)*
5. "Hi there!"  *(no tools)*
6. "What's the SKU on ORD-1003 and is that item covered by free returns?"  *(both: look up the order, then search for the returns policy on that SKU's category)*
7. "How long does shipping to Hawaii take, and is my order ORD-1002 going there?"  *(both: search for the shipping policy, look up the order)*

For each, note in `NOTES.md`:

- The tools called, in order.
- Whether the agent skipped a tool it should have used, or used one it shouldn't have.
- The number of steps.

### 3. Iterative retrieval

Pick one prompt whose first retrieval is likely to be weak. Two candidates from the suite: prompt 1 (if your corpus uses "opened" rather than "headphones") or prompt 6 (if "free returns" isn't a phrase in your corpus).

Run it. If the agent's second retrieval refined the query (chapter 4), capture both calls in `iterative.jsonl` with the queries and scores side by side. If the agent answered after a single weak retrieval, **tighten the `search_kb` description** (especially the parameter description) to encourage a second swing on low-score hits, then re-run and capture the change.

Note in `NOTES.md`:

- The two `search_kb` queries, side by side.
- The top score before and after.
- The line(s) you changed in the description.

This is the cheapest, highest-leverage iteration loop in agent development. Practice it.

### 4. Citations and validation

Add a system-prompt instruction asking the model to cite chunks in its final answer using `[chunk_id: <id>]`. Re-run prompts 1, 3, 4, 6, and 7 from the routing suite.

Implement a small validator:

```python
def validate_citations(final_text: str, trace: list[dict]) -> dict:
    cited = set(re.findall(r"\[chunk_id:\s*([\w\-#\.]+)\]", final_text))
    seen = set()
    for step in trace:
        for tu in step.get("tool_uses", []):
            if tu["name"] != "search_kb":
                continue
            for chunk in tu.get("observation", []):
                seen.add(chunk["chunk_id"])
    return {"cited": sorted(cited), "seen": sorted(seen),
            "fabricated": sorted(cited - seen),
            "uncited_relevant": sorted(seen - cited)}
```

(You'll need to record the observation alongside each `tool_use` in your trace — extend the trace shape from exercise-01.)

Capture `citations.jsonl` with one row per re-run prompt:

```json
{"prompt": "...", "final_text": "...", "validation": {"cited": [...], "seen": [...], "fabricated": [...], "uncited_relevant": [...]}}
```

Note in `NOTES.md`:

- The total fabrication count across the runs. Anything other than zero is a finding.
- Whether the model cited every chunk it actually used (vs. citing only the strongest one).

### 5. Parallel tool use

On prompt 7 of the routing suite (two unrelated lookups), check whether the model issued parallel `tool_use` blocks in a single assistant turn or serialized them across two round trips. If serial, re-run with parallel tool use enabled (it's the default on most current models; explicit toggle is provider-specific). If parallel, re-run with parallel tool use disabled and note the step-count difference.

Capture both runs in `parallelism.jsonl` and note in `NOTES.md`:

- Step count, model-call count, and wall-clock latency for each shape.
- Whether the final answer's quality changed.

### 6. Tighten one description

Pick the tool whose behavior in the suite was worst — too eager, missed cases, or got confused with another tool. Edit **only the description** (not the schema, not the system prompt) and re-run the affected prompts. Note the before/after lines in `NOTES.md`.

This is the same iteration you did in `mod-103` exercise-02. It is *still* the highest-leverage knob.

## Starter guidance

A minimal `search_kb` definition matching chapter 4:

```python
SEARCH_KB_TOOL = {
    "name": "search_kb",
    "description": (
        "Search the customer support knowledge base for passages relevant to a "
        "natural-language query. Use this when the user is asking a question "
        "whose answer is likely in our internal documentation (policies, FAQs, "
        "product manuals). Do NOT use it for chit-chat, math, or actions like "
        "placing an order. If the top hit has a weak score (< 0.6) and the "
        "answer is unclear, consider refining the query and searching again. "
        "Returns up to 5 passages with chunk_id, score, section_title, and an excerpt."
    ),
    "input_schema": {
        "type": "object",
        "required": ["query"],
        "properties": {
            "query": {
                "type": "string",
                "description": (
                    "A focused natural-language question or phrase. Prefer "
                    "keyword-rich rephrasings of the user's words over the "
                    "literal user text."
                ),
            },
            "k": {"type": "integer", "minimum": 1, "maximum": 10},
        },
    },
}

def search_kb(query: str, k: int = 5) -> list[dict]:
    q_vec = embed_query(query)
    chunks = search(conn, q_vec, min(k, 10))
    return [
        {"chunk_id":      c["chunk_id"],
         "score":         round(c["score"], 3),
         "section_title": c["section_title"],
         "excerpt":       c["text"][:600]}
        for c in chunks
    ]
```

Reuse exercise-01's loop unchanged except for the `TOOLS` list and the dispatcher. Bump `max_steps` to 10 if a multi-step trajectory needs it; do **not** raise it past 12.

## Things you should not do in this exercise

- Implement query expansion or hybrid search inside `search_kb`. The agent's "second swing" is your iterative-retrieval mechanism for this module. Production-grade hybrid search comes later.
- Add a `final_answer` tool to force structured answers. Cite via inline `[chunk_id: ...]` and validate after the fact.
- Add memory between calls. Each `run_agent(...)` is one-shot. Memory lands in exercise-03.
- Add a fourth or fifth tool to "make routing more obvious." Three tools is the right surface for this exercise.

## Acceptance criteria

You can demonstrate:

- A working agent with three tools wired through a single dispatcher; the only change from exercise-01 is the tools and their implementations.
- `routing.jsonl` covering the seven routing-suite prompts with traces and final answers.
- `iterative.jsonl` showing one prompt's first and second retrieval side by side, with the description change you made.
- `citations.jsonl` with the validator output for at least five re-run prompts. Zero fabricated citations is the bar; if you see any, flag them in `NOTES.md` rather than hiding them.
- `parallelism.jsonl` with one prompt run both shapes and the latency / step-count comparison.
- `NOTES.md` containing:
  - Your before/after for the description you tightened in task 6.
  - One paragraph on what went well, what didn't, and the smallest change that would fix what didn't.
  - Pinned `MODEL`, `TEMPERATURE`, `max_steps`, embedding model, and corpus identifier.

## Reflection

Answer briefly in `NOTES.md`:

1. Across the routing suite, which tool was the model most likely to *over*-call (use when it should have generated) and which to *under*-call? What does that tell you about the descriptions?
2. For the iterative-retrieval prompt: does the model's second query actually look like a better embedding query, or is it just rephrased English? What property of an embedding model's vector space does that reveal?
3. For the citation validation: in a production support setting, what would you do with a "fabricated citation" event? Block the response? Tag it for review? Log and ignore? Defend your choice.
4. Pick one prompt where the agent's behavior felt close to a real production failure. What is the smallest change (description, schema, prompt, tool) that fixes it — and is the fix one a single agent can make, or does it need an external check?

## Stretch goals

- **Hybrid retrieval.** Add a small BM25 / SQLite-FTS scoring pass alongside the vector search inside `search_kb`; merge by reciprocal rank fusion; re-run the routing suite. Note which prompts moved.
- **Tool-choice probes.** For prompt 4 (refund), run three variants: `tool_choice="auto"`, `tool_choice={"name": "create_refund"}` (force), `tool_choice={"type": "none"}` (forbid). Capture the three traces in `tool_choice.jsonl` and note which one the model handled most honestly.
- **A simple confidence floor.** If the top score from `search_kb` is below 0.4, intercept and return `{"error": "low_confidence", "top_score": ...}` instead of the chunks. Watch the model abstain ("I don't see that in the documents"). This is chapter 6 of `mod-104` extended into agent territory.
- **A web-search tool.** Add a fourth tool that calls a real web search API (Brave, Serper, Tavily). Re-run any prompt where the agent should fall back to general knowledge if `search_kb` whiffs. Note what changed in the routing — and what new failure modes appeared.
