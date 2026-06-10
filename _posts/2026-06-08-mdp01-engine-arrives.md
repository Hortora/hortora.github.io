---
layout: post
title: "The retrieval engine takes shape"
date: 2026-06-08
type: phase-update
entry_type: note
subtype: diary
projects: [engine, hortora]
tags: [langchain4j, qdrant, retrieval, rag]
---

The session started with a question: can CaseHub and Hortora share a RAG
library? An hour into the design, the answer was no — because LangChain4J
already is that library.

We went through the full SPI surface that the proposed shared module would need:
document ingestion, chunking, dense embeddings, vector stores, retrieval
pipeline, context assembly, MCP server. LangChain4J with the Quarkus extension
covers every one of them. The proposed library would have been a parallel
implementation of something that already exists and is actively maintained.

What LangChain4J doesn't cover: general ONNX inference for non-embedding tasks.
NLI (hallucination detection), classification (action risk), regression
(confidence estimation), SPLADE sparse embeddings, cross-encoder reranking.
CaseHub needs all of these. That's the genuine gap — and it's narrow enough
to warrant a focused module rather than a framework.

So the design landed at `casehubio/onnx-inference`: a standalone ONNX inference
wrapper, zero RAG coupling, sitting below LangChain4J rather than beside it.
The spec is in Hortora's repo. CaseHub builds it; Hortora takes it as a
dependency when SPLADE validation passes.

With the architecture settled, `Hortora/engine` went from zero to a
working phase 1. Quarkus 3.36.1, Java 25, GraalVM native CI. On startup it
scans `~/.hortora/garden`, embeds every entry via Ollama, upserts into Qdrant.
Exposes `GET /search` and two MCP tools — `garden_search` and `garden_status` —
over HTTP SSE so Claude Code and other AI assistants can query it. Fourteen
tests green, including a mock embedding model and in-memory store so Qdrant
and Ollama aren't needed in CI.

The first feature exposed a correctness problem in the original design. The
search endpoint was filtering by domain after Qdrant returned results — which
silently truncates at scale if the top-N entries happen to be the wrong domain.
The fix was LangChain4J's `IsEqualTo("domain", domain)` filter on
`EmbeddingSearchRequest`, which translates to a Qdrant server-side payload
filter before vector scoring. Writing the test surfaced a separate discovery:
REST Assured's Groovy GPath `$.findAll { }` returns `null` on empty, not `[]`,
so `hasSize(0)` fails even when the filter is working correctly.

Both findings went into the knowledge garden.
