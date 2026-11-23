---
layout: post
title: "Grove — Teaching a Garden to See Itself"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [grove]
tags: [vector-analysis, curation, qdrant, clustering]
---

## What I was trying to achieve: a curator's workbench, not just a dashboard

Grove Phase 1 gave us visibility — a domain map, entry detail views, and single-entry curation. You could see the garden's health at a glance and fix one entry at a time. But a garden with 1,253 entries across 24 domains doesn't get curated one entry at a time. Phase 2 needed to make Grove the tool a curator actually reaches for when it's time to clean house.

Three capabilities, each independent: bulk curation for the mechanical work, version lifecycle tracking for the slow decay, and vector quality signals for the patterns no human would spot by scanning titles.

## What we believed going in: the vector work would be hard, the curation work would be easy

I'd expected the vector quality signals — near-duplicate detection, semantic outliers, coverage density clustering — to be the difficult half. Cosine similarity at scale, clustering algorithms, caching strategies. The curation improvements (bulk confirm, retire, re-tag, domain move, trigger reindex) felt like straightforward extensions of existing patterns.

That turned out to be almost exactly right, with one exception that cost more time than all the clustering work combined.

## Eight issues, one Qdrant filter, and a silent zero

The curation improvements landed fast. `CurationService` already knew how to confirm freshness, retire, and edit a single entry via JGit. Extending it to bulk operations meant wrapping the same logic in a loop with a single git commit at the end — `grove: bulk confirm 47 entries` instead of 47 individual commits. Domain move was a `Files.move()` plus a JGit `rm` and `add`. Trigger reindex was the lightest issue in the whole batch: a button that POSTs to the engine's REST API and shows the result.

Version lifecycle was more interesting. The version registry is a simple YAML file mapping stack names to current versions — `quarkus: 3.36.1`, `jdk: 25.0.2`. `VersionScorer` parses the `verified_on` frontmatter field, matches it against the registry, and classifies entries into three tiers: current (within one minor), aging (two or more minors behind), legacy (major version behind). The entry table gained a colour-coded Version column, and the entry detail view shows the tier alongside the verified version. Curators can now see at a glance which entries are drifting.

The vector work is where it got genuinely interesting. Near-duplicate detection fetches dense vectors from Qdrant, computes pairwise cosine similarity within a domain, and flags pairs above 0.92 — excluding pairs already marked as checked in `garden.db`. Results cache in `grove.db` so re-analysis doesn't recompute from scratch.

Semantic outlier detection computes each domain's centroid (element-wise mean of all dense vectors) and ranks entries by distance. The furthest entries are candidates for miscategorisation or overly niche content. Cross-domain similarity takes it further: for each entry, it checks whether any *other* domain's centroid is closer than its own. If so, the entry might be in the wrong domain — and a Move button is right there in the results.

Coverage density clustering was the most algorithmically involved. I needed DBSCAN — unlike k-means, it doesn't require knowing the cluster count in advance, which you don't when you're asking "how diverse is this domain's coverage?" The implementation is about 100 lines of Java on a pre-computed cosine distance matrix: compute the N×N matrix once, then DBSCAN's range queries are just array lookups. No external clustering library needed. The quarkus domain turned out to have 13 clusters across 217 entries with a spread metric of 0.41 — one dominant cluster of 150 entries (the Quarkus-specific gotchas and techniques that all cluster together semantically) and 12 smaller satellite clusters covering niche topics.

Then we tried to verify the UI against live data, and every vector analysis endpoint returned empty. No errors. No warnings. Just `"points": []` from an API call that should have returned hundreds of entries.

The Qdrant scroll API accepts a filter object with a `must` array of conditions. I'd written the filter as `{"match": {"key": "domain", "value": "jvm"}}` — nesting both `key` and `value` inside `match`. The correct format is `{"key": "domain", "match": {"value": "jvm"}}` — `key` at the condition level, `match` containing only the value. The wrong format is valid JSON, returns HTTP 200, and produces a well-formed response with zero points. No schema validation error, no warning header, nothing. It silently matches nothing.

A one-line fix after an hour of "why is the API returning empty when I can see the data in the Qdrant dashboard?" The kind of bug that lands in the garden as a gotcha because the next person will lose the same hour.

## Where this leaves Grove

Phase 2 turns Grove from a read-and-fix-one tool into something that actively surfaces what needs attention. A curator can now:

- Bulk-confirm 47 stale entries in one click instead of 47 individual operations
- See which entries were verified against Quarkus 3.20 and need re-checking against 3.36
- Find near-duplicates that accumulated over months of independent garden submissions
- Identify entries that are semantically closer to another domain than their own
- Understand whether a domain's coverage is concentrated (one tight cluster — potential redundancy) or scattered (thin, diverse coverage)

The open question is compute cost. Pairwise cosine similarity is O(n²) per domain, and DBSCAN's distance matrix has the same scaling. The jvm domain with 1,253 entries produces a 1.5M-element distance matrix. That's fine for on-demand analysis behind a button click, but it won't scale to automatic background analysis across all 24 domains without either sampling or moving the heavy computation to Qdrant's native similarity search. For now, the button-click model is correct — curators trigger analysis when they're ready to act on results, not as a background process that generates noise.

The engine still needs a REST endpoint for the reindex trigger (tracked as engine#79). Until that lands, the reindex button shows a clean "engine returned 404" error rather than failing silently — a lesson the Qdrant filter bug reinforced: silent failures are worse than loud ones.