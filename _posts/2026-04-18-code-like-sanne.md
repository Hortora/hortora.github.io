---
layout: post
title: "Code Like Sanne"
date: 2026-04-18
entry: "10"
type: phase-update
excerpt: "The taxonomy design went further than expected. Six gardens, developer profiles, 'code like Sanne Grinovero' as a retrieval mode — then we built Phase 1 with 722 tests green."
---

The Area 2 taxonomy design was supposed to take an hour. It took the whole session and went places I didn't expect.

I came in knowing the 5-garden model needed a second look before we built anything against it. We started with my concern about `decisions-garden` — most ADRs are project-specific, so what's the editorial bar for a community canonical garden of decisions? Claude pushed back: the spanning-tree hierarchy solves this. A Quarkus community `decisions-garden` is legitimate and fully auto-capturable from the Quarkus repo. An org's private garden extends it with their specific choices. I'd been thinking of `decisions-garden` as a single community thing when it's valid at every node of the hierarchy.

## Six gardens, not five

The bigger revision was `patterns-garden`. I'd been thinking of it as "known design patterns plus code examples." We split it. `patterns-garden` gets architectural patterns with provenance — observed in real production codebases, with suitability notes and named variants. `examples-garden` gets minimal working code, copy-paste ready. Different embedding models, different editorial bars, different retrieval intent. Merging them would have meant dual embedding models per garden, volume imbalance when the code miner floods the collection, and a confusing editorial bar that's simultaneously high and nonexistent. Six gardens is cleaner than the wrong five.

## The part I didn't see coming

I asked how Claude might use pattern knowledge to better architect a project — how it could notice that serverless-workflow and two other projects make the same structural decision independently. Claude sketched structural clustering: extract structural features from all indexed projects, cluster by similarity, surface candidates that don't match any known pattern. Delta analysis layers on top to give each pattern an origin story — when it was invented, by whom, how it spread.

Developer profiles fall out of the git attribution. "Code like Sanne Grinovero" becomes a retrieval ranking boost — patterns she originated or consistently adopts score higher in your results. Not a filter; a prior. Monoculture prevented.

The trend data design got its own decision: separate `observed_at` (when the pattern appeared in the world) from `indexed_at` (when we processed it). This costs one field at schema time and makes historical backfill free forever. Use only `indexed_at` and every historical record has the wrong timestamp permanently.

## Eight tasks, two reviewers

Then we built Phase 1. Eight tasks, TDD throughout, subagent-driven development with spec compliance and code quality review after each task.

The reviewers earned their keep. The code quality reviewer caught that `GARDEN_TYPES[garden]['valid_types'][0]` was fragile as a string replacement seed — if list order ever changed, `str.replace()` would silently return the original string unchanged, the fixture would go uncorrupted, and the test would pass vacuously. The fix:

```python
original_type = next(
    t for t in GARDEN_TYPES[garden]['valid_types']
    if f'type: {t}' in fixture
)
```

The spec reviewer caught redundant happy-path tests duplicating what `test_all_six_gardens_happy_path` already covered. Both fixed before merge.

722 tests passing on main. `validate_pr.py` validates the `garden` field against all six types, rejects entry types that don't match the garden, and enforces garden-specific required fields. Submission templates and the forage skill updated throughout.

Phase 2 is the patterns-garden extended format — `observed_in`, `suitability`, `variants`, `authors`, `stability`. That's where the new ideas become schema.
