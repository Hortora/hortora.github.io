---
layout: post
title: "Eight Gaps in a Close Pipeline"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [soredium]
tags: [lifecycle, slot-mode, defense-in-depth, bugs]
series: issue-204-work-lifecycle-fixes
---

The slot close pipeline had working code for every step of the landing sequence — rebase, two-hop push, stamp, verify. It just never called it.

`work-end` Step 3.5 described what `merge_slot()` does — per-repo loops, two-hop pushes from slot clone to original to GitHub, landing SHA stamps — then invoked `cmd_land()`, a single-repo function with zero slot awareness. Every claim in the SKILL.md was false for the script it actually called. The result: three slots with ~6,700 lines of engine implementation stamped "branch closed" but never pushed beyond the local clone.

The audit surfaced eight distinct gaps. The interesting part isn't any individual fix — each is small — but how they compound. `cmd_land()` pushes to `origin`, which in a slot clone is a local path, not GitHub. The workspace branch gets stamped but never merged to workspace main. The verification gate that should catch all of this passes old-format stamps without landing SHAs and accepts missing workspace branches as "may be single-repo." Every defense layer had a plausible-sounding reason to pass.

The root fix routes slot mode to `merge_slot()` — the function that already existed, already worked, and was already described in the documentation. The other fixes tighten the layers: workspace merge before stamp, SHA in workspace stamps, verification gates that warn instead of silently passing, scaffold cleanup so stale `.meta` files don't confuse state detection, and two-hop push in work-pause so paused slot branches have a GitHub backup.

Four companion bugs landed alongside: HANDOFF.md detection that checked `main` instead of the working tree (#206), symlink validation that accepted subdirectories of other repos as workspace roots (#207), a missing HANDOFF.md fallback when `what-next` returns nothing (#208), and active branch detection from handover context when returning to main (#205). All in the work lifecycle, all one-line root causes with straightforward fixes.

The pattern across all five issues: the code does what the code says, but the code doesn't do what the documentation says. The gap between description and implementation is where the bugs live.
