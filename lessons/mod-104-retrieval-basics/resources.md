# Resources for mod-104-retrieval-basics

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as APIs, models, and libraries evolve.

## Embeddings providers

- **OpenAI — Embeddings guide** — current embedding models (`text-embedding-3-small`, `text-embedding-3-large`), the `dimensions` parameter, batching, and pricing. <https://platform.openai.com/docs/guides/embeddings>
- **OpenAI — Embeddings API reference** — request/response shape for `client.embeddings.create`. <https://platform.openai.com/docs/api-reference/embeddings>
- **Anthropic — Embeddings overview** — Anthropic does not ship a first-party embedding model; this page documents the recommended path (Voyage AI) for use with Claude. <https://docs.anthropic.com/en/docs/build-with-claude/embeddings>
- **Voyage AI — Embeddings documentation** — `voyage-3` family, `input_type="document"` vs `input_type="query"`, tokenization, and rate limits. <https://docs.voyageai.com/docs/embeddings>
- **Cohere — Embed documentation** — `embed-english-v3.0`, `embed-multilingual-v3.0`, `input_type` semantics, and batch limits. <https://docs.cohere.com/docs/embeddings>
- **Sentence Transformers (BAAI, MTEB)** — the de facto open-source option for local embeddings. <https://www.sbert.net/>
- **MTEB leaderboard** — comparative quality across embedding models on the Massive Text Embedding Benchmark. Useful when choosing between open-source models. <https://huggingface.co/spaces/mteb/leaderboard>

## Vector indexes and search

- **FAISS — official documentation and wiki** — covers `IndexFlatIP`, `IndexHNSWFlat`, `IndexIVFFlat`, quantization, ID maps, and persistence. <https://github.com/facebookresearch/faiss/wiki>
- **FAISS — Python API reference** — generated docs for the Python bindings. <https://faiss.ai/>
- **hnswlib** — focused HNSW implementation with simple Python bindings; good when you just want HNSW. <https://github.com/nmslib/hnswlib>
- **pgvector** — Postgres extension for `vector` columns, `<=>`/`<->`/`<#>` distance operators, and `ivfflat` / `hnsw` indexes. <https://github.com/pgvector/pgvector>
- **PostgreSQL — Indexes** — background on how Postgres composes indexes, which informs how pre-filter + ANN works in pgvector. <https://www.postgresql.org/docs/current/indexes.html>
- **Chroma** — small embedded vector DB; useful for prototypes. <https://docs.trychroma.com/>
- **Qdrant — documentation** — open-source vector DB with first-class filter support. <https://qdrant.tech/documentation/>
- **Weaviate — documentation** — open-source vector DB with hybrid search built in. <https://weaviate.io/developers/weaviate>
- **Pinecone — documentation** — managed vector DB; useful as a reference for the hosted-service shape of the API. <https://docs.pinecone.io/>

## Chunking, tokenization, and parsing

- **OpenAI — Tokenization (`tiktoken`)** — exact tokenization for OpenAI embedding and chat models; required for token-aware chunkers. <https://github.com/openai/tiktoken>
- **Anthropic — Tokenization guidance** — guidance on counting tokens for Claude prompts and chunks. <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>
- **LangChain — Text splitters** — reference implementation of the recursive character splitter described in chapter 4. Useful to read even if you do not import it. <https://python.langchain.com/docs/concepts/text_splitters/>
- **LlamaIndex — Node parsers** — alternative reference implementations of structural and token-aware splitters, including a SentenceWindowNodeParser worth knowing about. <https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/>
- **Unstructured — documentation** — popular library for converting heterogeneous source formats (PDF, HTML, DOCX) into chunkable text with layout-aware metadata. <https://docs.unstructured.io/>
- **pdfplumber** — pure-Python PDF parser used when you want layout and tables out of PDFs. <https://github.com/jsvine/pdfplumber>

## Retrieval-augmented generation patterns

- **Anthropic — Prompt engineering for retrieval / "Long context tips"** — Anthropic's recommended structure for stuffing retrieved context (XML tags, question-last, citation instructions). <https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/long-context-tips>
- **OpenAI — Retrieval-augmented generation guidance** — patterns for retrieving and grounding answers in the Responses API and structured outputs. <https://platform.openai.com/docs/guides/retrieval>
- **OpenAI Cookbook — Question answering using embeddings** — runnable end-to-end example mirroring the exercise-02 shape. <https://cookbook.openai.com/examples/question_answering_using_embeddings>
- **OpenAI Cookbook — Embedding large datasets** — practical guidance on batching, persistence, and retries during ingest. <https://cookbook.openai.com/examples/embedding_long_inputs>

## Evaluation

- **TREC — Information Retrieval Evaluation** — the canonical reference for IR metrics including recall@K, MRR, and nDCG. <https://trec.nist.gov/data/reljudge_eng.html>
- **scikit-learn — `ndcg_score`** — clean Python reference implementation of nDCG you can read to understand the math. <https://scikit-learn.org/stable/modules/generated/sklearn.metrics.ndcg_score.html>
- **Ragas — RAG evaluation framework** — open-source toolkit that operationalizes faithfulness, context precision/recall, and answer relevancy with LLM-as-judge implementations. Worth knowing about even if you do not adopt it. <https://docs.ragas.io/>
- **BEIR benchmark** — a heterogeneous-task retrieval benchmark used in much current embedding literature; useful background when comparing provider models. <https://github.com/beir-cellar/beir>

## Foundational papers (optional, secondary)

- **Lewis et al. (2020) — "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"** — the paper that named RAG and introduced the dense-retrieval + generator combination this module is the production version of. <https://arxiv.org/abs/2005.11401>
- **Karpukhin et al. (2020) — "Dense Passage Retrieval for Open-Domain Question Answering" (DPR)** — the dense-retrieval architecture that made bi-encoder embedding search a credible default. <https://arxiv.org/abs/2004.04906>
- **Reimers & Gurevych (2019) — "Sentence-BERT"** — bi-encoder embedding architecture behind most open-source embedding models in use today. <https://arxiv.org/abs/1908.10084>
- **Malkov & Yashunin (2018) — "Efficient and robust approximate nearest neighbor search using HNSW graphs"** — the HNSW paper. Worth the read once. <https://arxiv.org/abs/1603.09320>
- **Johnson, Douze & Jégou (2017) — "Billion-scale similarity search with GPUs"** — the FAISS paper. <https://arxiv.org/abs/1702.08734>
- **Liu et al. (2023) — "Lost in the Middle: How Language Models Use Long Contexts"** — empirical evidence for the "attention to position" effect referenced in chapter 1. <https://arxiv.org/abs/2307.03172>
- **Es et al. (2023) — "RAGAS: Automated Evaluation of Retrieval Augmented Generation"** — the paper behind the Ragas framework above. <https://arxiv.org/abs/2309.15217>

> Provider docs, library APIs, and model lineups evolve quickly. Where this module shows a specific function signature (e.g., `client.embeddings.create(...)`, `faiss.IndexHNSWFlat(...)`, pgvector's `<=>`), follow the link, confirm against the current docs, and snapshot the URL and date in your code comments. Do not rely on field names or model names in this document if they conflict with what you see in the official reference.
