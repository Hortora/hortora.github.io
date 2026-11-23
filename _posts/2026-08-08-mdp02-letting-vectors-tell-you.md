---
layout: post
title: "Letting the Vectors Tell You What Your Content Looks Like"
date: 2026-08-08
entry_type: note
subtype: diary
projects: [grove]
tags: [grove, content-intelligence, clustering, vectors, taxonomy]
---

The garden has 2,633 entries across a dozen domains. I can tell you which ones are stale, which are duplicates, which are outliers — Grove already does that. What I couldn't tell you is whether the domains make sense. Whether the taxonomy I built by hand actually reflects what the content talks about. Whether there are topics hiding inside a domain that should be split, or entries filed in the wrong place that nobody noticed because they're not *wrong* per se, just not where they'd be most useful.

That's the shift this work makes. Grove had a linter — flag problems, show them in a table, let you act. Now it has something closer to an X-ray. DBSCAN clustering across all entries, run per-domain then compared across domains via centroid similarity. The first time we ran it against the live garden: 154 clusters, 649 noise entries, 20 domain overlaps. Every one of those noise entries is a filing question — it doesn't fit any cluster in its current domain, so where does it actually belong?

The trick that makes this work at scale is a two-stage approach. Computing an N×N distance matrix across 2,633 entries is about 3.5 million cosine comparisons — not practical client-side. Instead, we compute one centroid per domain (cheap), build a D×D similarity matrix between those centroids (tiny — maybe 15×15), and only do entry-level comparison across domain pairs that are genuinely similar. Cap it at 20 pairs and the computation stays bounded. Per-domain DBSCAN is fine because individual domains are 20–100 entries.

Tag analysis sits alongside the clustering. Co-occurrence pairs, orphan tags (used exactly once — either too specific to be useful or the curator forgot to apply them elsewhere), and synonym detection. The synonym detection is the interesting part: free-form tags accumulate duplicates that string matching won't catch. "DI" and "injection" cover the same semantic space but look nothing alike. We compare the average embedding vectors of each tag's entry sets — tags whose entries live in similar vector space but rarely appear together on the same entry are synonym candidates. Pre-filter by shared domain overlap and co-occurrence exclusion to keep it tractable.

The health scorer aggregates everything into a single number per entry — staleness, retrieval frequency, outlier distance, version status, duplicate flag — weighted and composited. It runs on-the-fly from cached analysis results, no derivative cache. Right now the scores are thin: only retrieval and version signals have data, and the version registry is empty, so everything scores low. That'll improve as the retrieval history accumulates and the registry gets populated. The architecture is right; the data needs time.

The dashboard itself is functional but static — tables, numbers, expandable sections, action buttons. Looking at it with real data made the next need obvious: filtering, navigation, progressive drill-down. The Sparge curation UI has exactly the right interaction model — filter zone with toggle buttons and counts, list panel on the left, detail panel on the right. That's the next piece of work. The current dashboard stays as a reference while the workbench gets built alongside it.