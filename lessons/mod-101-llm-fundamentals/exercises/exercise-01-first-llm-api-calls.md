# exercise-01: First LLM API Calls

**Estimated effort:** 2 hours

## Objective

Make your first LLM API calls end-to-end: read an API key from the environment, issue single-turn and multi-turn requests, parse the response, and tune the basic sampling and length parameters. By the end you should be able to write a small chatbot loop from scratch without copying boilerplate.

## Background

This exercise covers material from:

- [Chapter 1 — What an LLM API Actually Is](../01-what-is-an-llm-api.md)
- [Chapter 2 — The Chat/Messages Format and Roles](../02-chat-messages-and-roles.md)
- [Chapter 4 — Sampling, Temperature, and Determinism](../04-sampling-temperature-and-determinism.md)
- [Chapter 5 — Choosing a Model](../05-choosing-a-model.md)

Pick **one** provider for this exercise. Anthropic (`anthropic` SDK) and OpenAI (`openai` SDK) are both fine. The README links the docs you will need.

## Prerequisites

- Python 3.10+ (or Node 20+ if you prefer JS/TS).
- A funded API key for your chosen provider. Use a small spend cap; this exercise should cost cents.
- `pip install anthropic` (or `pip install openai`).
- `ANTHROPIC_API_KEY` (or `OPENAI_API_KEY`) exported in your shell. Never commit keys.

## Tasks

Build a small CLI program (one file is fine) that demonstrates the following. Keep each task in its own function so you can run them independently.

### 1. Single-turn call

- Take a string prompt from the command line.
- Send it to a workhorse-tier model (e.g., `claude-sonnet-4-6` or `gpt-4.1-mini`) with a sensible `max_tokens`.
- Print the response text and the `stop_reason` (Anthropic) or `finish_reason` (OpenAI).

### 2. System prompt

- Add a system prompt that establishes a clear persona and an output rule (e.g., "Reply in fewer than 30 words. Always end with a period.").
- Show that the rule is honored by sending two different user prompts that would otherwise produce long answers.

### 3. Multi-turn conversation

- Build a REPL: read a line from stdin, append it as a `user` message, send the full message history, print the reply, and append the reply as an `assistant` message before looping.
- Exit cleanly on Ctrl-D / EOF.
- Verify the model can answer follow-up questions that depend on earlier turns (e.g., "What's the capital of Japan?" → "And of South Korea?").

### 4. Sampling knobs

- Add command-line flags for `--temperature`, `--max-tokens`, and `--model`.
- Send the same prompt at `temperature=0` three times and at `temperature=0.9` three times. Note what you observe about variability.

### 5. Stop reasons

- Force a `max_tokens` truncation by setting `--max-tokens 32` on a prompt that asks for a long answer. Confirm the stop reason indicates truncation.
- Force a clean stop by raising `--max-tokens` to a comfortable value.
- Print the `stop_reason` after every call so you build the habit.

## Starter guidance

A reasonable skeleton:

```python
import os
import sys
import argparse
from anthropic import Anthropic

def build_client() -> Anthropic:
    return Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

def single_turn(client, model, system, prompt, *, temperature, max_tokens):
    raise NotImplementedError

def multi_turn(client, model, system, *, temperature, max_tokens):
    raise NotImplementedError

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--model", default="claude-sonnet-4-6")
    parser.add_argument("--temperature", type=float, default=0.0)
    parser.add_argument("--max-tokens", type=int, default=1024)
    parser.add_argument("--mode", choices=["single", "chat"], default="single")
    parser.add_argument("prompt", nargs="?")
    args = parser.parse_args()
    # dispatch
    ...

if __name__ == "__main__":
    main()
```

Things you do *not* need to do in this exercise:

- Handle rate limits or implement retries (covered in exercise-03).
- Stream the response (covered in exercise-03).
- Count tokens (covered in exercise-02).

## Acceptance criteria

You can demonstrate that:

- Your program reads the API key from an environment variable; the key never appears in source.
- A single-turn call returns a parsed response *and* prints the model's stop reason.
- A system prompt visibly changes behavior across multiple user prompts.
- The multi-turn REPL maintains context across turns (the model correctly answers a follow-up that depends on the first turn).
- `--temperature 0` repeated runs produce highly similar (often identical) outputs; `--temperature 0.9` repeated runs produce visibly varied outputs.
- A constrained `--max-tokens` produces a truncated answer whose stop reason indicates truncation.

## Reflection

In a short comment at the top of your file (or a separate `NOTES.md`), answer:

1. Where in your code is the chat history reset? Why?
2. What happens if you forget to append the assistant message to the history before the next turn? Try it.
3. What was the smallest model you tried that still gave acceptable answers for your prompts?

## Stretch goals

- Add a `--system` flag that loads the system prompt from a file.
- Add a `/reset` command in chat mode that clears the history without exiting.
- Add a `/save` command that writes the conversation as JSON to disk so you can resume it later.
- Re-implement *one* of your calls without the SDK — make the HTTPS POST directly using `httpx` or `requests`. Inspect the headers and JSON body until you have built one by hand.
