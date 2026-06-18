---
layout: post
title: "Phase 2 Unblocked"
date: 2026-06-10
entry: "41"
type: phase-update
entry_type: note
subtype: diary
projects: [Hortora]
tags: [neural-text, rag, splade, reranker]
---

The casehubio/neural-text stack shipped in full — seven chapters across two
journeys (C1–C7). The pieces Hortora needs are `casehub-inference-splade`
(SparseEmbedder) and `casehub-inference-tasks` (CrossEncoderReranker). Both
are published, zero casehub domain dependencies, ArchUnit enforced.

The module boundary held cleanly: `inference-*` is shared with Hortora,
`rag-*` is casehub-specific. Hortora wires its own Qdrant integration using
`SparseEmbedder` for the sparse leg and `CrossEncoderReranker` for precision
mode — no RAG pipeline coupling. That separation was the right call; Hortora's
retrieval stack and casehub's are completely different runtime environments.

Phase 2 in engine is now straightforward: hybrid dense+sparse search
via Qdrant named vector spaces, RRF fusion, optional cross-encoder reranking
for the top-N precision pass. The infrastructure question is answered. What
remains is the design of how engine integrates those primitives into
its retrieval layer.
