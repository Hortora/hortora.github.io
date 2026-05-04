---
layout: post
title: "The Garden Debt and a Java Engine"
date: 2026-05-04
entry: "14"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
excerpt: "Twenty open PRs, 37 stranded entries, 63 ghost entries found in git history — then a native Quarkus service with 171 tests built from scratch under strict TDD."
---

The garden nearly swallowed itself.

Twenty open PRs, 37 entries stranded in branches that would never merge cleanly. The root cause was structural: forage was creating individual PRs for each submission, and once main diverged — from dedup sweeps, mostly — every PR became a conflict because they all touched GARDEN.md. The fix was obvious in retrospect: push directly to main, no PRs. We extracted the 37 stranded entries by reading each branch directly and committing them in a batch.

While we were cleaning up, we found 63 ghost entries — IDs referenced in GARDEN.md with no corresponding files. All recoverable from git history. The validate_garden.py scanner had been looking for `**ID:** GE-XXXX` in entry bodies (the old format) while the newer entries use YAML `id:` frontmatter. One line fix, 63 entries found.

Then we ran the mining pipeline against real code for the first time. Five of Sanne Grinovero's top JVM repos: incus-spawn, quarkus-demo, quarkus-quickstarts, quarkus, hibernate-orm. The extractor does regex-based fingerprinting — interface counts, injection points, SPI service files — and clusters by cosine similarity.

The result was immediately interesting. Quarkus and Hibernate ORM cluster as structurally similar (thousands of interfaces, similar abstraction depth) but diverge on a single signal: injection points per file. Quarkus sits at ~0.4; Hibernate at ~0.002. Two hundred times different. We turned that into a patterns entry — CDI container wiring versus service-loader wiring, with the injection ratio as the discriminator. That's the kind of output I wanted from this pipeline.

The bigger build was garden-engine: a native Quarkus service that ports the Python mining pipeline and adds an AI-driven harvest layer. I chose JLama as the default inference backend — in-process Java, auto-downloads from HuggingFace, no external daemon — backed by Qwen2.5-3B, with Sonnet accessible via `-Dquarkus.profile=sonnet` for quality evaluation runs.

We built it in four phases under strict TDD, 171 tests total. Claude caught four production bugs during the red phase — the period when tests are supposed to fail and don't always fail for the reason you expect:

- SPI detection was keying on filename extension when it should key on path hierarchy (`META-INF/services/`)
- `Files.walk` throws `FileSystemLoopException` on symlinks before a visited-set check can intercept it — fixed with `SimpleFileVisitor` and `toRealPath()`
- `interface` inside string literals was counting as an abstraction declaration
- `git()` was discarding the exit code from `proc.waitFor()`, silently returning garbage on failure

All four would have shipped without the discipline of watching each test fail correctly.

The fourth phase wires `DedupeClassifier` and `EntryMergeService` (both `@RegisterAiService` interfaces, backed by JLama in production) into a `SemanticDeduplicator` that classifies entry pairs as DISTINCT, RELATED, or DUPLICATE, and for duplicates, synthesises a merged entry rather than discarding the lower-scored one. This replaces the Jaccard-based scanner.

One thing that didn't fully land: consuming the langchain4j fork directly. The fork targets Quarkus 3.15.2; garden-engine runs on 3.33.1. API enforcement around `@BuildStep` runtime config tightened between those versions and multiple processors fail the check. The workaround — forcing JOS serialisation via `.mvn/maven.config` — is in and all 171 tests pass. Upgrading the fork's Quarkus target is the next step before the workaround can come out.
