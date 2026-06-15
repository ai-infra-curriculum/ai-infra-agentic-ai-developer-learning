# Strategies for Reliable Structured Output

## Why this matters

Most applications do not actually want prose; they want a Python dict, a TypeScript object, a database row. The gap between "the model is talkative" and "your code needs a parseable result" is where a lot of LLM apps quietly fall apart. Getting structured output *reliably* — not "usually" — is one of the highest-leverage skills in this module.

This chapter is the menu of strategies. The next chapter covers what to do at the boundary when the output still surprises you.

## The four strategies

In rough order of strength:

1. **Prompt-only** — ask for JSON in the prompt and hope.
2. **Vendor JSON mode** — tell the API "respond with valid JSON" so the decoder enforces well-formedness.
3. **Schema-constrained generation** — supply a JSON Schema; the API constrains decoding to outputs that satisfy it.
4. **Tool / function calling** — register a tool whose `parameters` are your schema and ask the model to "call" it. The arguments object is your structured output.

You should know all four and pick deliberately. None of them removes the need to validate at the boundary (chapter 6).

## Strategy 1: prompt-only

The cheapest, oldest move:

```text
SYSTEM:
Reply with a single JSON object and nothing else.

Output format:
{
  "category": "billing" | "shipping" | "returns" | "product" | "other",
  "urgency": "low" | "normal" | "high",
  "summary": "<one sentence>"
}

USER:
<the message>
```

What you get when it works: a parseable JSON object.

What you get when it breaks: prose around the JSON ("Sure! Here's the response: ```json {...} ```"), trailing commentary, code fences, escaped quotes that double-escape on the way back, partially-truncated JSON when you hit `max_tokens`.

A useful tightening trick on providers that support it: **prefill** the assistant turn with `{`. The model continues from there, which prevents the preamble and the code fence in one move:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system="Reply with a single JSON object and nothing else.",
    messages=[
        {"role": "user",      "content": "..."},
        {"role": "assistant", "content": "{"},
    ],
)
# response.content[0].text now starts straight into the object body
raw = "{" + response.content[0].text
```

Prefill is fast, costs almost nothing, and works well for short fixed-shape outputs. It does *not* guarantee schema correctness — only that the reply starts with `{`.

Prompt-only is the right starting point for prototypes and small internal tools. It is not the right place to stop for anything user-facing.

## Strategy 2: vendor JSON mode

Several providers offer a "respond in JSON" flag that pushes the model toward (and in some cases guarantees) syntactically valid JSON. The most common shape is OpenAI's `response_format`:

```python
# OpenAI: "JSON mode" — guarantees valid JSON, does NOT enforce a schema
response = client.chat.completions.create(
    model="gpt-4.1-mini",
    response_format={"type": "json_object"},
    messages=[
        {"role": "system", "content": "Reply with a single JSON object. Schema: {\"category\": str, \"urgency\": str}"},
        {"role": "user", "content": "..."},
    ],
)
data = json.loads(response.choices[0].message.content)
```

What JSON mode buys you: the response will parse as JSON. No more code fences, no more "Sure, here is …" prose.

What JSON mode does *not* buy you: the JSON might be valid but the wrong shape. The model can still produce `{"answer": "billing"}` when you wanted `{"category": "billing"}`. Some providers also require the word "JSON" to appear in your prompt to enable the mode at all.

JSON mode is a strict superset of prompt-only and is essentially free to use. Prefer it whenever your provider offers it.

## Strategy 3: schema-constrained generation

Newer endpoints let you supply an actual JSON Schema and constrain the decoder so the output is *guaranteed* to satisfy the schema. OpenAI calls this "Structured Outputs" via `response_format={"type": "json_schema", ...}`; the relevant Anthropic and Google equivalents are typically reached through tool/function calling, which is Strategy 4.

OpenAI sketch:

```python
schema = {
    "name": "support_classification",
    "schema": {
        "type": "object",
        "additionalProperties": False,
        "required": ["category", "urgency", "summary"],
        "properties": {
            "category": {"type": "string", "enum": ["billing", "shipping", "returns", "product", "other"]},
            "urgency":  {"type": "string", "enum": ["low", "normal", "high"]},
            "summary":  {"type": "string", "maxLength": 200},
        },
    },
    "strict": True,
}

response = client.chat.completions.create(
    model="gpt-4.1-mini",
    response_format={"type": "json_schema", "json_schema": schema},
    messages=[
        {"role": "system", "content": "Classify the message."},
        {"role": "user", "content": "..."},
    ],
)
data = json.loads(response.choices[0].message.content)
```

When this works, it is the strongest guarantee available short of writing your own grammar-constrained decoder: every property is present with the right type, no extra properties, enums are honored. You still validate at the boundary because:

- Not every JSON Schema feature is supported in strict mode (this varies per provider; check the docs).
- The model can still refuse or stop early on `max_tokens`, producing a partial object you must detect.
- Strict schemas can interact poorly with optional fields, recursion, and `oneOf`. Read the provider's restrictions before you build.

Despite those caveats, schema-constrained generation is the *default* you should reach for in new code when your provider supports it.

## Strategy 4: tool / function calling for structured output

Tool calling is the topic of `mod-103`, but it doubles as a portable structured-output mechanism. You define a "tool" whose only purpose is to receive your data, and you tell the model to call it. The arguments the model produces are constrained to your schema by the provider.

Anthropic sketch:

```python
tools = [{
    "name": "record_classification",
    "description": "Record the classification of a customer message.",
    "input_schema": {
        "type": "object",
        "required": ["category", "urgency", "summary"],
        "properties": {
            "category": {"type": "string", "enum": ["billing", "shipping", "returns", "product", "other"]},
            "urgency":  {"type": "string", "enum": ["low", "normal", "high"]},
            "summary":  {"type": "string"},
        },
    },
}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "record_classification"},
    messages=[{"role": "user", "content": "Classify: ..."}],
)

# Find the tool_use block; its .input is your structured object.
for block in response.content:
    if block.type == "tool_use" and block.name == "record_classification":
        data = block.input
        break
```

The `tool_choice` parameter forces the model to call that specific tool, which is the trick that turns "tool use" into "structured output." On Anthropic, this is currently the most reliable way to get a strict JSON object out.

OpenAI exposes the same pattern through `tools=[...]` and `tool_choice={"type": "function", "function": {"name": "..."}}`; the arguments come back on `response.choices[0].message.tool_calls[0].function.arguments` as a JSON string you parse.

You are not actually executing a tool here; you are using the tool protocol as a transport for typed data.

## Choosing between strategies

A reasonable default ladder:

1. **Prototype or one-off script** → prompt-only, ideally with prefill.
2. **Anything that ships** → JSON mode if supported, schema-constrained if supported, tool calling otherwise.
3. **Multiple structured outputs in one call** (e.g., "classify *and* extract") → schema-constrained or tool calling, where the schema carries the full shape. Avoid asking for "JSON containing JSON" with prompt-only; it parses badly.

When to deliberately *not* force structure:

- The output is genuinely prose for a human (a summary, an email draft, a chat reply).
- The shape varies by input and a single schema would be a lie.
- You need streaming. Some structured-output modes are not stream-friendly (or only emit at the end). Streaming a partial schema-constrained JSON object is doable on some providers but adds complexity; if you do not need the streaming UX, skip the headache.

## A note on schema design

The model has to *read* the schema you send. A clean schema produces cleaner output:

- **Name fields like English nouns**, not abbreviations. `customer_email_address` beats `cust_em`.
- **Use enums when the set is closed.** "Pick one of these five" is much stronger than "category (string)".
- **Add `description` strings** on fields whose meaning is not obvious from the name. The model uses them as guidance.
- **Require what is required.** Optional fields that should always be present produce inconsistency.
- **Forbid extras.** `additionalProperties: false` (in strict modes that support it) prevents the model from inventing fields.
- **Keep nesting shallow.** Deeply nested structures are harder to fill correctly. If you find yourself with five levels, consider a flatter shape.

The schema is part of the prompt. Spend the same care on it that you spent on the system block.

## A small comparison

| Strategy              | Valid JSON? | Schema-correct? | Streamable? | Where it shines |
|-----------------------|-------------|------------------|-------------|-----------------|
| Prompt-only + prefill | usually     | no               | yes         | prototypes, very simple shapes |
| JSON mode             | yes         | no               | yes         | quick wins, any provider that offers it |
| Schema-constrained    | yes         | yes (per provider rules) | partial | most production extraction / classification |
| Tool calling          | yes         | yes              | partial     | portable structured output; foundational for `mod-103` |

"Per provider rules" because each provider has its own list of supported JSON Schema features in strict mode. Read the docs.

## What every strategy still needs

Whichever strategy you pick, the next chapter applies: **parse and validate at the boundary, with a typed library, and have a deliberate plan for what to do when parsing fails.** Even schema-constrained generation can return a partial response, a refusal, or a tool call you did not expect. Structured-output strategies make the happy path much more reliable; they do not let you skip the unhappy path.

## Summary

You have four levers for structured output: prompt-only, JSON mode, schema-constrained generation, and tool calling. Start with the weakest that works for your stage; in production, default to schema-constrained or tool calling. Design the schema deliberately — names, enums, descriptions, required fields. Whatever strategy you choose, validation at the boundary is still mandatory.
