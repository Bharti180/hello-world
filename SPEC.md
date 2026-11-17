# Spec: Kubernetes Docs Q&A Agent (Capstone Option B)

**Author:** Bharti Negi · **Status:** Draft · **Last updated:** <date>

## 1. Context and problem
The Kubernetes documentation is large (thousands of pages), and finding the right page for a question takes time. Generic chatbots answer confidently but often without sources, or invent details. This project builds an agent that answers questions using only the official docs, cites the page it used, and admits when the docs don't cover a question.

## 2. Goals and non-goals
**Goals**
- Answer Kubernetes questions from the official docs, with a citation (page URL) for every answer.
- Hand off ("the docs don't cover this") instead of guessing when the docs have no answer.
- Measure quality on a fixed set of 30 test questions and improve on that baseline each week.

**Non-goals**
- Not a general Kubernetes expert: no answers from the model's own memory.
- Not running commands against real clusters.
- Not indexing blogs, forums or other non-official sources.

## 3. Data
- **Source:** Kubernetes documentation, `content/en/docs` in https://github.com/kubernetes/website (CC BY 4.0).
- **Scope (v1):** `concepts/` and `tasks/` sections only; add `reference/` later if time allows.
- **Test set:** 30 hand-written questions, each with a gold page URL; about 5 deliberately not covered by the docs (to test handoff). Stored in `eval/questions.jsonl`.

## 4. Design overview
1. **Ingest:** clone the docs, strip Hugo front matter and shortcodes, split pages into chunks by heading, keep metadata (path, title, section, public URL).
2. **Index:** embed chunks and store them in a local vector store.
3. **Classify:** LLM labels each question's topic (`concepts` / `tasks` / `reference`) as validated JSON.
4. **Retrieve and answer:** fetch the top 5 chunks; the LLM answers only from them and cites URLs.
5. **Handoff:** if retrieval scores are low or the context doesn't answer the question, reply that the docs don't cover it and log a doc issue.
6. **Tools (MCP server):** `search_docs`, `get_page`, `open_doc_issue`.
7. **Later:** supervisor with router, retriever and answer-checker agents (LangGraph); queue, tracing and security hardening.

**Stack:** Python 3.12, uv, <LLM provider>, <embedding model>, <vector store, e.g. Chroma>, MCP Python SDK, LangGraph, OpenTelemetry.

## 5. Alternatives considered
- **Fine-tuning a model on the docs:** rejected; costly, hard to update, and can't cite sources.
- **Whole-page chunks:** rejected; too long and noisy. Heading-level chunks give more precise citations.
- **Keyword search only:** kept as a possible hybrid addition, but semantic search handles paraphrased questions better.

## 6. Success metrics
- **Accuracy:** answer judged correct *and* cites the gold page.
- **Retrieval hit rate:** gold page appears in the top 5 results.
- **Handoff accuracy:** uncovered questions are handed off; covered ones are not.
- **Latency and cost:** p95 latency and cost per question, tracked from Week 9.
- Week 6 baseline numbers are recorded in `eval/results.md`; later weeks must not regress.

## 7. Risks and cross-cutting concerns
- **Hallucination:** answer only from retrieved text; answer-checker agent from Week 7.
- **Prompt injection:** instructions hidden in doc text must be ignored (tested in Week 10).
- **Cost:** small subset of docs, caching, token logging.
- **Stale docs:** record the docs commit hash used for indexing.
- **Privacy and auditing:** no user data stored; every lookup logged for audit.

## 8. Milestones
| Week | Gate |
|---|---|
| 3 | Classify question topic as JSON |
| 4 | RAG over doc pages with citations |
| 5 | MCP tools: `search_docs`, `get_page`, `open_doc_issue` |
| 6 | 30 Q&A pairs baseline |
| 7 | Supervisor: router, retriever, answer-checker agents |
| 8 | Rate-limited batch question queue |
| 9 | Trace per question; latency and cost alarms |
| 10 | Injection planted in a doc page; audit lookups |
