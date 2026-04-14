---
layout: post
title: "Five layers"
date: 2026-04-14
entry: "06"
type: phase-update
excerpt: "The staleness problem had six candidate solutions, none quite right alone. We combined them into five. The garden now knows when to doubt itself."
---
# Five layers

The staleness problem has been in IDEAS.md since April 10th. Six candidate solutions, none of them quite right on their own. Every entry in the garden has a `staleness_threshold` field, but nothing ever checks it — a two-year-old entry looks identical to one written yesterday.

I kept thinking: the annotation approach is too passive (warnings become wallpaper), but the systematic review approach is too heavy (nobody runs a dedicated maintenance session every week). Something in between.

## Six solutions, one hybrid

I brought Claude in on the design. We worked through the six documented options and I asked whether a combination would hold better than any single approach. The failure modes of each solution are different, which means stacking them covers the gaps.

We ended up with five layers:

- **S2**: Every SEARCH result now shows the entry's age. Past-threshold entries get a ⚠️ stale warning. This fires always, costs nothing, and is the floor.
- **S6**: At CAPTURE time, library and tool-specific entries prompt for `verified_on` — the version this was verified on. SEARCH uses it to produce a concrete version-gap flag rather than a generic age warning.
- **S3**: Entries scoring ≥12/15 get a soft prompt for "why this fix over the obvious alternative." Preserving reasoning at the moment of capture costs nothing and is the one moment when that context exists.
- **S1**: At SWEEP end, forage checks entries in the domains worked that session against their threshold. Stale entries surface for Confirm/Revise/Retire while the developer still has context.
- **S5**: `validate_garden.py --freshness` reports the full overdue count. `harvest REVIEW` sweeps all domains — the backstop that guarantees nothing escapes indefinitely.

The key: S2 is the always-on baseline. Even if nobody runs SWEEP or REVIEW, stale entries are still flagged at the point of use. The rest of the layers add coverage and discipline on top.

## TDD, and what the code review caught

We implemented `validate_garden.py --freshness` first, TDD — tests before code. Claude caught two things in the code quality pass that are worth keeping: the YAML frontmatter regex `^---\n(.*?)\n---` silently skips files with CRLF line endings (no error, just a miss), and `date.fromisoformat()` raises `ValueError` on invalid calendar dates even after regex format-validation. The regex checks shape, not calendar validity. Both fixed.

The forage and harvest skill updates were the larger surface area — SEARCH annotation, CAPTURE prompts, domain-filtered SWEEP step, and a new `harvest REVIEW` mode for the systematic sweep. Skills synced.

## An audience we hadn't written for

After the implementation, I noticed we had no documentation for platform architects evaluating Hortora for enterprise deployment. The existing docs are either tutorial or reference. Nothing for someone asking whether the system is well-engineered enough to trust.

We built a new Architecture page — six sections, each framed as a property claim:

- *Knowledge never goes stale silently*
- *No entry enters the garden without passing validation*
- *Concurrent sessions cannot corrupt garden state*
- *Relevant entries surface; irrelevant ones don't*
- *The protocol scales beyond a single garden*
- *Duplicate knowledge is eliminated in layers, not all at once*

Each section states the problem, explains the mechanism, then lists concrete guarantees and what degrades gracefully. The "graceful degradation" section is the one that matters most — it separates automatic recovery from things that need operator intervention. That distinction is almost never made explicit in architecture documentation.

The page is live.
