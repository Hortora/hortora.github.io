---
layout: post
title: "From prototype to installable"
date: 2026-04-12
entry: "05"
type: phase-update
excerpt: "Four docs pages. One install command. The garden skill is gone. Hortora is now something a stranger could actually use."
---
# From prototype to installable

The CI pipeline has worked since Phase 2. Entries get validated, submitted, integrated. But the only way to get started was to read my session notes — which is not a realistic bar.

This session fixed that.

## Getting Started: from twelve steps to one

Claude and I built four pages on hortora.github.io: Getting Started, How It Works, Skills Reference, and Design Spec. Each has a sidebar nav and a consistent two-column layout. The content existed in scattered README files and specs; putting it in one place with a coherent reading order was the actual work.

The original Getting Started page walked through manual curl, a Python setup script, path configuration. Readable if you were already motivated. I stripped it to the one thing that matters:

```
/install-skills https://github.com/Hortora/soredium
```

One line. That installs forage and harvest into Claude Code. We did the same to the homepage Quick Start: install command, `forage` to capture something, `harvest` to merge it, a small git graphic showing how submissions flow. Three steps.

## Retiring the garden skill

The original `garden` skill handled capture, sweep, merge, and dedupe in one command. We split it into forage (session-time) and harvest (maintenance) in Phase 2, but `garden` lingered — still installed, still referenced in CLAUDE.md files and handover docs.

I wanted it gone. We removed it from cc-praxis, uninstalled it everywhere, and updated all the references.

While sorting that out, Claude caught two bugs in the soredium scripts that had been waiting to surface. In `validate_garden.py`, the GE-ID regex had the shorter legacy pattern `GE-\d{4}` listed before the new-format `GE-YYYYMMDD-xxxxxx` — so the alternation was matching the prefix of every new-format ID and never reaching the full pattern. One reorder, fixed. In `integrate_entry.py`, the script was reading the GE-ID from the submission filename rather than the frontmatter `id:` field — which breaks any submission not following a strict naming convention. Fixed to read frontmatter.

## Garden#16, DEDUPE, drift to zero

Garden#16 — the regex alternation fix — had been open and waiting on CI. CI passed. We merged it.

Drift had reached 20 against a threshold of 10. We ran DEDUPE: 1,002 pairs across java-panama-ffm, quarkus, electron, and tools. Eight related pairs, no duplicates. The six new native-image entries in java-panama-ffm now cross-reference each other — they're companion gotchas from the same GraalVM native build session, and it's worth knowing they're connected when you hit one.

Drift is at zero.
