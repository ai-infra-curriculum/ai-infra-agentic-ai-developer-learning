# Chunking Documents

## Why this matters

Embeddings work on text. Your corpus arrives as files — markdown handbooks, PDFs of policy docs, HTML pages from a help center, code, support transcripts. None of them are at a useful unit of granularity for retrieval. A 50-page handbook embedded as one vector is "the handbook is about returns and shipping and a few other things"; it cannot be top-K relevant to anything specific. A handbook split into individual sentences is precise but each sentence loses its context. The right unit is somewhere in between — a *chunk* — and choosing it well is the highest-leverage decision in the whole pipeline.

Chunking is also where bad systems quietly fail. A retrieval that "doesn't find the answer" usually means the answer lives in a chunk that got split awkwardly, or in a chunk that got bundled with five unrelated topics. Before you tune embeddings or K or anything else, get chunking right.

## What a "chunk" is

A chunk is a triple:

```
(chunk_id, text, metadata)
```

- **`chunk_id`** — stable identifier you can cite back to. Often `f"{doc_id}#{ordinal}"` or a content hash.
- **`text`** — the unit of text you will embed and (later) stuff into the prompt.
- **`metadata`** — `{document_id, source_path, section_title, page, created_at, embed_model, ...}`. The model never sees the chunk_id; it sees the text and optionally the source title or URL for citation.

Everything in this chapter is about how to pick the boundaries of `text` so chunks are useful at retrieval time.

## What a good chunk looks like

A chunk is useful when, *in isolation*, it answers a plausible question. That gives you the working definition: split your documents so that each chunk is locally coherent and self-contained.

Three concrete properties to aim for:

- **Single-topic.** One section, one idea, one Q&A pair. Mixing topics dilutes the embedding's signal.
- **Self-contained enough to read.** "Section 3 also applies" is useless when the model has only the chunk in context; either expand the chunk or carry forward enough preceding context.
- **Sized to fit comfortably in the model's input.** Embedding models have a max token input; generation models have a context window. Aim well under both. Chunk too big and you waste prompt tokens at query time; chunk too small and you fragment the answer across many chunks.

The contradictory goals — small enough to be precise, big enough to be self-contained — are what makes chunking interesting.

## Three chunking strategies, in order of how often you should reach for them

### 1. Structural (preferred when documents have structure)

If your source has headings, sections, paragraphs, slides, or message turns, *use them*. The author already drew the boundaries; you almost never beat the author.

For markdown:

```python
import re

HEADING = re.compile(r"^(#{1,6})\s+(.*)$", re.MULTILINE)

def split_markdown_by_heading(text: str, doc_id: str) -> list[dict]:
    """Yield (heading-anchored) sections of a markdown document."""
    sections = []
    cursor = 0
    current_title = "preamble"
    for m in HEADING.finditer(text):
        start = m.start()
        if start > cursor:
            sections.append({
                "doc_id": doc_id,
                "title": current_title,
                "text": text[cursor:start].strip(),
            })
        current_title = m.group(2).strip()
        cursor = m.end() + 1
    sections.append({
        "doc_id": doc_id,
        "title": current_title,
        "text": text[cursor:].strip(),
    })
    return [s for s in sections if s["text"]]
```

For HTML, parse with BeautifulSoup and walk semantic tags (`<article>`, `<section>`, `<h2>`). For PDFs, use a layout-aware parser (e.g. `pdfplumber`, `unstructured`, or a hosted parser like AWS Textract) and treat each detected section as a candidate chunk. <!-- needs-research: confirm current "best default" PDF chunker; library landscape changes -->

Then a *secondary* pass splits any section that is still too large with one of the strategies below. Structural chunking gets you 80% of the way for almost any corpus.

### 2. Recursive character splitter (when structure is weak or inconsistent)

Walk a list of preferred separators from "biggest" to "smallest" and split the document at the first one that produces small-enough chunks. The classic order:

```
["\n\n", "\n", ". ", " "]
```

A minimal implementation, modelled on the pattern that LangChain popularized:

```python
def recursive_split(text: str, max_chars: int = 1500, overlap: int = 150,
                    separators=("\n\n", "\n", ". ", " ")) -> list[str]:
    if len(text) <= max_chars:
        return [text]
    for sep in separators:
        if sep in text:
            parts = text.split(sep)
            chunks, buf = [], ""
            for p in parts:
                if len(buf) + len(p) + len(sep) <= max_chars:
                    buf = (buf + sep + p) if buf else p
                else:
                    if buf:
                        chunks.append(buf)
                    if len(p) > max_chars:
                        chunks.extend(recursive_split(p, max_chars, overlap, separators[1:]))
                    else:
                        buf = p
            if buf:
                chunks.append(buf)
            # Add overlap by carrying the tail of each chunk into the next:
            with_overlap = []
            for i, c in enumerate(chunks):
                if i == 0:
                    with_overlap.append(c)
                else:
                    with_overlap.append(chunks[i-1][-overlap:] + c)
            return with_overlap
    return [text]  # no separator found; return as-is
```

Two design choices baked in:

- **Prefer big separators.** Paragraph breaks are better split points than sentence breaks, which are better than word breaks. You only fall down to the next level when the higher level fails.
- **Overlap.** Carry a small tail of each chunk into the next so that an answer straddling a boundary still appears whole in *some* chunk. Typical values are 10–20% of `max_chars`. Overlap costs storage (you embed the overlap region twice) but recovers a lot of edge-case answers.

This implementation chunks by *characters*. For most English text, roughly 1 token ≈ 4 characters; a `max_chars=1500` chunk is around 350–400 tokens. If you need token-exact bounds, use the model's tokenizer.

### 3. Token-aware splitting (when you need precision)

For tight chunk-size guarantees — usually because you are near an embedding model's max input or you want chunks that sum cleanly into a target prompt budget — split on tokens directly.

```python
import tiktoken  # OpenAI's tokenizer
enc = tiktoken.encoding_for_model("text-embedding-3-small")

def split_by_tokens(text: str, max_tokens: int = 400, overlap: int = 40) -> list[str]:
    ids = enc.encode(text)
    chunks = []
    i = 0
    while i < len(ids):
        window = ids[i : i + max_tokens]
        chunks.append(enc.decode(window))
        if i + max_tokens >= len(ids):
            break
        i += max_tokens - overlap
    return chunks
```

Token splitting cuts mid-sentence routinely. That is fine when used as a *secondary* pass after structural splitting (each section gets sub-divided on tokens). Used as the *primary* splitter on prose, it produces chunks that look unnatural to the reader. <!-- needs-research: tiktoken supports OpenAI models; for non-OpenAI embedders use the provider's tokenizer (e.g. `transformers` AutoTokenizer with the matching model name) -->

## Chunk size: orders of magnitude, then tune

There is no universal "best" chunk size. There are sensible ranges and a method:

- **200–500 tokens** — high-precision retrieval; works well for FAQ-style content, support tickets, code snippets. Each chunk is a focused unit. You usually need a higher K (5–10) to assemble a complete answer.
- **500–1000 tokens** — the common default for prose handbooks, policy docs, technical articles. Balances precision and self-containment. K = 3–5 is typical.
- **1000–2000+ tokens** — fewer, bigger chunks. Better for narrative documents where context matters; worse for needle-in-a-haystack queries. K is small (2–3).

The method is the one chapter 7 sets up: build a small eval set, sweep `chunk_size ∈ {300, 600, 1200}` and `overlap ∈ {0, 100, 200}`, measure recall@K and answer groundedness, pick the row that wins. Three runs, an hour of work, and you stop guessing.

## Overlap

Overlap is the chunker's hedge against splitting an answer in half. A user asks "what is the SLA for refunds?" and the SLA sentence happens to span a chunk boundary; with no overlap, neither chunk is a great hit. With 100–200 characters of overlap, one chunk now contains the full sentence.

Rules of thumb:

- Default to 10–20% overlap on prose.
- Zero overlap when chunks are already short (under ~150 tokens) and self-contained (FAQ entries, code functions). Overlap there just duplicates content.
- Higher overlap (30%+) only when you have measured a recall problem on boundary-straddling answers.

Overlap is cheap in retrieval correctness and not free in storage. A 20% overlap means you store and embed ~1.2× the corpus.

## Metadata: the bit everyone forgets

Every chunk needs metadata. At minimum:

```json
{
  "chunk_id":      "handbook-returns#04",
  "document_id":   "handbook-returns",
  "source_path":   "/policies/handbook/returns.md",
  "source_url":    "https://docs.example.com/policies/returns#section-4",
  "section_title": "Returning opened electronics",
  "ordinal":       4,
  "char_start":    12450,
  "char_end":      13870,
  "created_at":    "2026-03-12",
  "embed_model":   "text-embedding-3-small",
  "embed_version": "2024-01",
  "chunker":       "markdown-recursive-v1"
}
```

What you get from this:

- **Citations.** The query loop in chapter 6 will render answers like "Yes — see *Returning opened electronics*." That title comes from metadata.
- **Filters.** "Only chunks from this tenant," "only chunks created in the last 90 days," "only chunks from the API reference." Chapter 3's filter discussion is what consumes these fields.
- **Re-ingest decisions.** When you change the chunker or the embedding model, you need to know which chunks are stale. `chunker` and `embed_version` are how.
- **Debugging.** When retrieval surfaces a wrong chunk, you want to open the source file and look at it. `source_path` and `char_start/char_end` make that one step.

Decide your metadata schema *before* you embed a million chunks. Adding a field later is cheap; back-filling it across an existing index is not.

## Chunking code and structured content

Prose advice does not transfer cleanly:

- **Source code.** Split at function/class boundaries, not at character counts. Each chunk should be one symbol with its docstring. Treesitter, tree-sitter-languages, or `ast` for Python gives you the structure. Carry the function signature into the chunk text so the embedding sees the name.
- **Tables (CSV, Excel).** Embed at the row level only if rows are self-describing; otherwise serialize a small group of rows + header into a single chunk.
- **Conversation logs / tickets.** Treat each ticket as a doc and each message as a candidate chunk; carry the ticket subject as preamble in every chunk so the embedding has context.
- **Slides.** One slide ≈ one chunk; include speaker notes if you have them.
- **Q&A pairs.** Each Q+A is one chunk. Do not separate them.

## Common mistakes

- **Embedding whole documents because "embeddings handle long inputs now."** They handle them in the sense of not erroring; retrieval quality on long inputs is worse, not better.
- **Splitting on `\n` only.** Single-newline splits collide with formatted text (lists, code blocks). Prefer the recursive order.
- **No overlap on prose corpora.** You will lose answers that straddle boundaries and never know why.
- **Chunks that include only the heading and the first sentence.** A common bug in heading-based splitters where the section body is mis-attached to the *next* heading. Test with a corpus you know.
- **Metadata as an afterthought.** Reindexing 50k chunks because you forgot `section_title` is the kind of pain you take exactly once.
- **One chunker for every content type.** A markdown chunker on a PDF produces garbage. Identify the source format first.

## Summary

Chunking is dominant in determining how well retrieval works. Prefer structural splits driven by the document's own format; fall back to a recursive character splitter with a couple-hundred-character overlap when structure is weak; use a token-aware splitter when you need exact bounds. Aim for chunks that are single-topic and self-contained, typically in the 300–1000 token band, and attach enough metadata to cite, filter, and re-ingest. Pick sizes by sweeping a small eval set (chapter 7), not by argument.
