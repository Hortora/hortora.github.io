---
layout: post
title: "Cleaning Up, Closing the Loop"
date: 2026-04-21
entry: "12"
type: phase-update
excerpt: "One PR that wouldn't merge, a garden repo with 30 uncommitted commits, and then two phases that closed the pattern discovery loop."
---

The session started with a mess.

PR #78 — a sweep of 52 garden entries — had been sitting open with a `CONFLICTING` status for days. I resolved the merge conflict, force-pushed a clean linear commit, and GitHub continued returning `"mergeable": "CONFLICTING"` via the API for over a minute. `gh pr merge` returned 405. Nothing in the error message hinted at a caching delay. Only pushing another commit triggered re-evaluation. Worth knowing for next time.

Then I looked at the local garden repo. The local `main` was 30 commits ahead of `origin/main`, with uncommitted deletions in the working tree and orphaned rebase metadata from an April 15 session that had never been finalized. It looked like a multi-user conflict. It wasn't — it was five days of entries committed directly to local main instead of through PRs, plus a stale `.git/rebase-merge/` directory that git kept reporting as an active rebase on a different branch.

The fix required some care. Before resetting, we identified 11 entry files that existed only in local main and would be lost in a hard reset — rescued them to a branch first. Then reset local main to `origin/main`, which revealed a bad previous PR merge had silently dropped 10 existing index entries from GARDEN.md. Those were restored in the sweep commit.

While investigating why the files got lost, we added two git hooks to the garden repo: a `pre-commit` that blocks any commit if untracked `GE-*.md` files exist, and a `post-checkout` that warns after any branch switch. Both hooks live in `.githooks/` — committed to the repo — and `garden-setup.sh` now runs `git config core.hooksPath .githooks` after cloning. Any future install gets them automatically.

Then we built.

Phase 3 was the ecosystem mining foundation: `project_registry.py` for CRUD over `registry/projects.yaml`, `feature_extractor.py` to walk source trees and produce structural fingerprints (interface counts, abstraction depth, injection points, SPI patterns — all regex, no compiler), `cluster_pipeline.py` for cosine similarity clustering against known patterns, and `delta_analysis.py` to find new interface and abstract class introductions between git tags with git-blame attribution. 28 new tests; 775 total.

Phase 4 closed the loop. The pipeline produces candidates; now there's something to do with them. `run_pipeline.py` orchestrates registry → extract → cluster → delta → JSON report. `validate_candidates.py` is the human gate: for each cluster candidate it calls a `decide_fn` callback — accept, reject, or skip. Accepted candidates become patterns-garden entry skeletons with a `GP-` ID and `observed_in` provenance. Rejected candidates go into `known_rejections.yaml` and are suppressed in future runs via cosine similarity — close enough to a rejected centroid, automatically skipped.

The `decide_fn` callback pattern is worth noting. The function that drives the interactive review loop never reads stdin directly — it takes a callable. The CLI `__main__` wires it to `input()`; tests inject deterministic lambdas. Every code path — accept, reject, skip, auto-suppression of a pre-rejected candidate — is fully exercisable without mocking. 51 new tests covering unit, correctness, integration, E2E, and a happy path that runs the full pipeline end to end against synthetic git repos and confirms a `GP-*.md` file lands on disk.

826 tests. The pattern discovery loop is now functional: projects go in, structural candidates come out, a human reviews them, confirmed patterns become garden entries.
