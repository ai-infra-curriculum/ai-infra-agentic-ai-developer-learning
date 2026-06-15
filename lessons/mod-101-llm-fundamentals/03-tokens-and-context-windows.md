# Tokens and Context Windows

## Why this matters

Tokens are the unit the model thinks in, the unit you pay in, and the unit the context window is measured in. Every limit, every bill, and every "the model truncated my output" surprise traces back to tokens. Stop counting characters; start counting tokens.

## What a token actually is

A token is a chunk of text from a fixed vocabulary the model was trained on. Tokenizers vary by family:

- GPT-style models use Byte Pair Encoding (BPE) variants — for OpenAI's recent models that is the `o200k_base` tokenizer.
- Claude uses its own BPE-style tokenizer; the exact split is not publicly published, but the Anthropic SDK exposes a token-counting endpoint.
- Open-weights models (Llama, Mistral) ship their tokenizer with the weights and you can run it locally.

The important property: tokenization is **deterministic and lossless** for a given tokenizer. The same input string always produces the same token sequence, and the tokens reverse back to the exact input bytes.

## Rules of thumb (and why you should not trust them)

For English prose, you will often see "~4 characters per token" or "~0.75 words per token." Those are averages and they are *fine for a back-of-envelope estimate*. They are wrong, often badly, for:

- **Code** — punctuation, indentation, and identifier splits produce more tokens per character.
- **Other languages** — non-ASCII scripts (CJK, Devanagari, Arabic) often produce *more* tokens per character than English in legacy tokenizers; newer multilingual tokenizers narrow the gap.
- **JSON and structured output** — braces, quotes, and field names each take tokens.
- **Numeric IDs and base64** — frequently split into many short tokens.

If a number matters (cost projection, prompt budget), measure it.

## Counting tokens for real

Both major providers ship official tools.

**OpenAI / GPT-style** — use `tiktoken`:

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4.1-mini")
text = "How many tokens is this string?"
tokens = enc.encode(text)
print(len(tokens))  # exact count for that model family
```

**Anthropic / Claude** — call the Messages Count Tokens API, which counts the *same way* the live model will:

```python
from anthropic import Anthropic

client = Anthropic()
count = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system="You are a precise technical assistant.",
    messages=[{"role": "user", "content": "How many tokens is this conversation?"}],
)
print(count.input_tokens)
```

Counting via the API is the canonical answer because the server applies chat-template overhead (role markers, separators) that local heuristics will miss.

## What the context window includes

The "context window" is the model's maximum total tokens for a single request. **It is shared between input and output.** A 200K-token context window with a 64K-token reply leaves 136K for everything you send: system prompt, all prior turns, retrieved documents, tool definitions, tool results, images (which are tokenized too).

A common bug: you set `max_tokens=8192` and a long conversation, then watch responses get truncated. The model ran out of *output* budget because too much of the window was already eaten by *input*. Diagnose by counting input tokens first.

Different models in the same family can have different windows. Always check the model card you are calling — do not assume because Sonnet supports 200K, the smaller sibling does too, or that future versions will keep the same number. Treat the limit as a property of the specific model ID.

## Why context length is not free

Even when the model technically *accepts* 200K tokens, several costs scale with how much you actually send:

- **Latency** — time-to-first-token grows with input length because the server must process the full prompt before generation can start. This is the "prefill" step.
- **Cost** — every input token is billed, even if the model "ignores" most of them. We will quantify this in chapter 7.
- **Recall** — long-context performance is not uniform. Models often attend better to the start and end of the input than the middle (the "lost in the middle" effect). If you stuff a million tokens of context to ask one question, do not be surprised when the answer is wrong.

Bigger context is a tool, not a free upgrade.

## Reasoning about the window: a budget

A useful exercise before any new LLM feature: write a context budget.

```
context_window     = 200_000   # model spec
reserved_output    =  8_192    # max_tokens for the reply
system_prompt      =  ~800     # measured
tool_definitions   =  ~1_200   # measured
running_history    =  ?        # grows with conversation
retrieved_docs     =  ?        # grows with query
fixed_overhead     = ~2_000    # role markers, format tokens
-----------
available_for_data = window - reserved_output - system - tools - overhead
```

Knowing `available_for_data` tells you how many retrieved chunks you can pack, how many tool definitions you can expose, and how aggressively you need to summarize the running history. We will use this budget repeatedly in `mod-104` and `mod-105`.

## Strategies when you run out of room

You will. The standard tools:

- **Trim the system prompt.** Most are too long. Cut examples until quality drops, not before.
- **Summarize the conversation.** Rolling summarization (turns N..N+k become a single "summary" assistant message) is the simplest workable approach.
- **Use a larger-context model** — and pay for it.
- **Retrieve, do not stuff.** Instead of pasting the whole document, retrieve the relevant chunks (covered in `mod-104`).
- **Cache the prefix.** If your system prompt and tools are stable across calls, providers can charge you a discount on those input tokens after the first call. Keep that prefix byte-identical.

## A worked example

```python
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-sonnet-4-6"
WINDOW = 200_000
RESERVED_OUTPUT = 4_096

system = open("system_prompt.md").read()
docs   = open("retrieved.txt").read()
user   = "Summarize the key risks."

used = client.messages.count_tokens(
    model=MODEL,
    system=system,
    messages=[{"role": "user", "content": docs + "\n\n" + user}],
).input_tokens

available_for_reply = WINDOW - used
print(f"Input tokens: {used:,}  Headroom for reply: {available_for_reply:,}")
assert available_for_reply >= RESERVED_OUTPUT, "Trim input before calling."
```

Wire that assertion into any request that builds a variable-size prompt. It will save you in production.

## Summary

Tokens are the model's unit of account. Count them with the official tokenizer; never trust character heuristics for production limits. The context window is a shared budget between input and output — write the budget down before you build, leave headroom for the reply, and reach for retrieval and summarization before you reach for a bigger model.
