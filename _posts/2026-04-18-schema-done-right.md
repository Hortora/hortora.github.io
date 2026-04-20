---
layout: post
title: "Schema, Done Right"
date: 2026-04-18
entry: "11"
type: phase-update
excerpt: "Phase 2 was the patterns-garden extended entry format — six optional fields, 747 tests, one design decision worth explaining."
---

Phase 2 was straightforward. The design was already locked in entry 10. The
only real decision was what to do with malformed optional fields.

The six extended patterns-garden fields — `observed_in`, `suitability`,
`variants`, `variant_frequency`, `authors`, `stability` — are all optional.
Someone submitting a pattern doesn't have to fill in the provenance chain or
the developer attribution. But if they do fill it in and get the structure wrong
(`observed_in: not-a-list`, `authors[0].role: inventor`), what should the
validator do?

I made it a WARNING, not a CRITICAL. The reasoning: these fields are
enrichment — they improve retrieval quality and attribution, but the entry
is valid without them. Blocking a contribution because the provenance list is
malformed would be the wrong trade-off. The validator tells you it's wrong;
it doesn't refuse to accept the entry.

The implementation split cleanly into three: `validate_patterns_extended(fm)`
as a pure function returning warning strings, wiring it into `validate()` only
when `garden == 'patterns'`, and verifying that non-patterns gardens are
completely unaffected. The isolation test — inject a malformed `observed_in`
into a discovery entry, confirm no warning fires — is the one that would catch
a future regression if someone forgets the guard.

We built it in seven tasks, subagent-driven, spec plus code quality review
after each. Nothing notable went wrong. The code quality reviewer on Task 1
noted that `VALID_STABILITY` was defined but not yet used — correct, it gets
used in Task 2. The reviewers confirmed each task matched spec before moving on.

747 tests on main. `validate_pr.py` now handles both the 6-garden taxonomy
from Phase 1 and the patterns-garden extended schema from Phase 2.

Phase 3 is the project registry and ecosystem mining pipeline — the first part
that requires new infrastructure rather than just extending the validator.
