# AI Infrastructure Agentic AI Developer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). Live cohorts &amp; team programs: **[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**.
<!-- /aicg:site-banner -->

> **Status:** ✅ Curriculum authored — 6 modules with lecture chapters and hands-on exercises, plus 2 capstone projects. Quizzes and labs are scaffolded and fill in on subsequent content cycles. AI-assisted content is under ongoing human review.

---

## 🎯 Overview

This is the **entry rung of the Agentic track** (level 20) and the on-ramp to the
[Agentic AI Engineer](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning)
track (level 30). It teaches the LLM-application fundamentals that every higher
agentic track assumes you already have, and maps directly to the high-volume
**"AI Engineer / LLM application developer"** hiring title.

You start from a single LLM API call and finish by shipping a containerized,
tool-using agent behind a validated API. Each module builds on the last: API
mechanics → reliable prompts and structured output → tools that take action →
retrieval over your own documents → a reason-act agent that ties it together →
deploying the whole thing as a service.

### What You'll Master

- **Call LLM APIs** with a working mental model of tokens, context windows, temperature, streaming, cost, and robust error/retry handling
- **Engineer prompts** that hold up, produce schema-constrained JSON, and are validated and iterated against simple evals
- **Wire tools and functions** so the model can plan and act, with the call/result loop and malformed-argument handling under control
- **Build retrieval** — embeddings, similarity search, chunking, indexing, and a simple RAG query loop you can defend with a recall number
- **Implement your first agent** with the ReAct (Thought/Action/Observation) loop, retrieval-as-a-tool, and conversation memory
- **Ship an LLM app** as a FastAPI service with secrets management, structured logging, a cost/usage guardrail, and a Dockerfile

---

## 📊 Modules

Six modules, 10 hours each (60 hours of module content). Every module ships
lecture chapters, three hands-on exercises, and a `resources.md` reading list.
Quizzes and labs are scaffolded and authored on subsequent content cycles.

| Module | Topic | Hours | Exercises | Quiz |
|--------|-------|-------|-----------|------|
| [mod-101](./lessons/mod-101-llm-fundamentals/README.md) | LLM Fundamentals for Application Developers | 10h | 3 | Planned |
| [mod-102](./lessons/mod-102-prompt-engineering/README.md) | Prompt Engineering & Structured Output | 10h | 3 | Planned |
| [mod-103](./lessons/mod-103-tool-and-function-calling/README.md) | Tool & Function Calling | 10h | 3 | Planned |
| [mod-104](./lessons/mod-104-retrieval-basics/README.md) | Retrieval Basics (Embeddings & Simple RAG) | 10h | 3 | Planned |
| [mod-105](./lessons/mod-105-first-agent/README.md) | Your First Agent: The Reason-Act Loop | 10h | 3 | Planned |
| [mod-106](./lessons/mod-106-llm-app-deployment/README.md) | Shipping an LLM Application | 10h | 3 | Planned |

---

## 🛠️ Projects

Two capstones (30 hours combined) that compose the module skills end-to-end.

| Project | Focus | Hours | Builds On |
|---------|-------|-------|-----------|
| [project-101](./projects/project-101-llm-powered-app/README.md) | Ship an LLM-powered application — prompting, structured output, tool calling, and simple RAG behind a validated API with secrets management and a usage guardrail | 18h | mod-101 → mod-104, mod-106 |
| [project-102](./projects/project-102-tool-using-agent/README.md) | A simple tool-using agent — a single ReAct loop with 2–3 tools, retrieval, and conversation memory; document where it breaks down (the bridge into the Engineer track) | 12h | mod-103 → mod-105 |

---

## 🎓 Prerequisites

This is an entry-level track, but it assumes you can already write and run
Python comfortably. Before starting you should have:

- **Python** — functions, classes, virtual environments, `pip`, reading/writing files, and running scripts from a terminal
- **HTTP & JSON basics** — what a request/response is, status codes, and parsing JSON
- **Command line** — navigating directories, environment variables, running commands
- **Git basics** — clone, branch, commit
- **An API key** — access to an LLM provider (e.g., Anthropic or OpenAI) for the exercises

No prior machine-learning, deep-learning, or agent experience is required — those
concepts are introduced from first principles. See
[PREREQUISITES.md](./PREREQUISITES.md) for the full assumed-skills list.

---

## 🚀 Getting Started

### Clone

```bash
git clone https://github.com/ai-engineering-curriculum/agentic-ai-developer-learning.git
cd ai-infra-agentic-ai-developer-learning
```

### Repository Structure

```text
ai-infra-agentic-ai-developer-learning/
├── lessons/mod-XXX-*/        modules: lecture chapters, exercises/, labs/, quizzes/, resources.md
├── projects/project-XXX-*/   multi-module capstones
├── CURRICULUM.md             role-level coverage map
├── PREREQUISITES.md          assumed entry skills
├── VERSIONS.md               release history
└── README.md                 this file
```

### How to Work Through It

```bash
# 1. Create and activate a virtual environment
python3.11 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# 2. Set your LLM provider key
export ANTHROPIC_API_KEY="sk-..." # or OPENAI_API_KEY, per the exercise

# 3. Start with Module 101
cat lessons/mod-101-llm-fundamentals/README.md
```

1. **Read the lecture chapters** in order (the numbered `NN-*.md` files in each module).
2. **Do the three exercises** in `exercises/` — each is a self-contained build with objectives and validation.
3. **Skim `resources.md`** for deeper external reading.
4. **Move to the next module** only once you can complete its exercises unassisted.
5. **Build the capstones** in `projects/` once you've covered the modules they depend on.

Work the modules in numeric order — each one explicitly builds on the protocols
and pipelines introduced in the previous module.

---

## 📖 Curriculum Overview

### mod-101: LLM Fundamentals for Application Developers — 10h

A working application-developer's mental model of LLM APIs: what a call is, what
each knob does (temperature, sampling, model choice), what it costs, and how to
handle failures. Covers chat/messages format, roles, streaming, token and
context windows, cost/latency estimation, and error/rate-limit/retry handling.

### mod-102: Prompt Engineering & Structured Output — 10h

Turns LLM API skills into a repeatable engineering practice: prompts that hold
up, schema-constrained JSON you can parse, validation at the application
boundary, and iteration against simple evals. Covers few-shot prompting,
decomposition and reasoning, structured-output strategies, and output parsing.

### mod-103: Tool & Function Calling — 10h

Turns the model from a text generator into the planner of a small, auditable
program. Covers tool-definition anatomy, the call/result loop, wiring Python
functions as tools, deciding when a tool call is appropriate, and handling tool
errors and malformed arguments safely.

### mod-104: Retrieval Basics (Embeddings & Simple RAG) — 10h

Puts *your documents* in front of the model at query time. Builds both pipelines
every retrieval system needs — an offline ingest that chunks, embeds, and
indexes a corpus, and an online query loop that retrieves and assembles a
grounded, cited answer — and evaluates retrieval quality with recall@K.

### mod-105: Your First Agent: The Reason-Act Loop — 10h

Combines the tool loop and the retriever into a first agent: a single model that
decides, turn by turn, whether to act, retrieve, or answer. Implements the ReAct
(Thought/Action/Observation) pattern on native tool-use APIs, wires retrieval in
as a tool, adds conversation memory, and draws an honest line around the limits
of a single agent.

### mod-106: Shipping an LLM Application — 10h

Turns a laptop script into a service: a FastAPI process with Pydantic-validated
requests, environment-managed secrets, structured logs that include token usage,
a cost/usage guardrail, and a Dockerfile so the app runs the same way on another
machine. Finishes with running the whole thing end-to-end and locally.

---

## 🔗 Paired Solutions Repo

Reference implementations for every exercise and project live in the paired
solutions repository:

[`ai-infra-agentic-ai-developer-solutions`](https://github.com/ai-engineering-curriculum/agentic-ai-developer-solutions)

Try each exercise yourself first; consult the solutions repo to compare
approaches once you have a working attempt.

---

## ➡️ Where This Leads

This track is the on-ramp to the rest of the Agentic ladder:

1. **Agentic AI Developer** (level 20) — you are here
2. **[Agentic AI Engineer](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning)** (level 30) — multi-agent systems, orchestration, evaluation
3. **Senior AI Engineer** (level 40) and **Systems Architect** (level 48) — production agentic platforms at scale

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
