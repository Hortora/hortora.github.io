---
layout: post
title: "Thirteen State Locations, One Predicate"
date: 2026-08-06
type: phase-update
entry_type: note
subtype: diary
projects: [soredium]
tags: [lifecycle, state-management, work-health]
---

The soredium work lifecycle system had thirteen places tracking two facts: "is this issue done?" and "is this branch closed?" Five locations for each, plus three cross-cutting validators that re-derived the same answers. Each maintained independently. When something changed externally — another session closing an issue, another context merging a branch — the caches went stale silently.

The WI 490 case was representative: a HANDOFF.md item described a casehub branch as "not merged, not stamped" for days after the branch was already merged. No validation path existed to catch it.

I traced all thirteen locations to their roots and found the same pattern everywhere: the system caches state that has authoritative sources elsewhere (GitHub for issues, git for branches, `.plan` for queue position), then adds reactive validation to detect cache staleness. The fix: stop caching, start deriving.

## The Architecture

Four components, twelve implementation phases.

`is_closed()` is a single predicate in `lifecycle.py` that answers "is this branch done?" definitively. Five git operations, all local, sub-second. Returns one of five states: CLOSED, MERGED_UNSTAMPED, STAMPED_UNMERGED, OPEN, DELETED. Everything that previously implemented its own closure check — `verify_slot_close.py`, `audit_slot_merges.py`, the handover epic hygiene checks, `EPIC-CLOSED.md` marker files — now calls this one function.

`work_health.py` replaces scattered validation with a single check registry. Entry-scope runs at session start in under three seconds. Wrap-scope runs at session end. Close-scope is a hard gate for work-end. Each check reports its own status; auto-fix is limited to safe operations like marking closed GitHub issues as `[x]` in the `.plan` queue.

The `.plan` file becomes the single orchestration layer. On main it's the global curated queue — a prioritised subset of open GitHub issues. On feature branches it's the active work queue. `work_health.py` validates every `.plan` issue against GitHub on session start with a single batch API call, marking closed items automatically.

HANDOFF.md shrinks to session context only: what was done, what's next, any cross-module blockers. Under 200 tokens. No work tracking, no backlog, no queue progress — all of that moved to `.plan`.

## The Subtle Bug in Priority Ordering

`is_closed()` takes both project and workspace repos and returns the worst state across them. The priority ordering is project-centric: CLOSED < MERGED_UNSTAMPED < OPEN < STAMPED_UNMERGED < DELETED. This works for the common case — checking whether a branch is safe to close.

But artifact recovery checks workspace branches for unpromoted blogs and specs. When the project doesn't have a branch (DELETED, priority 4) and the workspace does (STAMPED_UNMERGED, priority 3), the combined state returns DELETED. The workspace's closure status is masked entirely.

The fix was simple — check the workspace directly for artifact recovery instead of going through the combined predicate. But it's the kind of thing that only surfaces when you migrate actual consumers. The design spec was correct about the predicate's contract ("local branch lifecycle, not distributed state"); the integration revealed which callers need the raw per-repo state rather than the combined answer.

## Self-Cleaning Intent Files

Crash recovery for pause and resume uses intent files — `.pausing` and `.resuming` in the workspace's design directory. Each tracks per-step completion: `wip_project: pending|done`, `stack_push: pending|done`, etc. Written atomically (temp file + rename), never git-tracked, removed on success.

The interesting part is the pre-condition validation. When `work_health.py` detects a `.pausing` file, it doesn't immediately offer recovery. It first checks: is the branch still checked out AND is the stack entry missing? If the branch is not checked out and the stack entry exists, the pause actually completed — the intent file is stale. Remove it silently.

This makes intent files self-cleaning. A crash during cleanup (after the operation succeeded but before the intent file was removed) doesn't produce a spurious recovery prompt next session. The pre-condition check catches the gap that intent files alone can't cover — the operation succeeded but the cleanup didn't.

## The Spec-to-Implementation Gap

The design spec was produced in the prior session: first-principles state analysis, adversarial design review across four dimensions (54 issues, all resolved), twelve phased implementation plan. This session implemented Phases 6–12 — seven phases in one pass.

The gap between spec and implementation was surprisingly small. One bug (the priority ordering issue above). No design changes. The adversarial review had already caught the cases that would have surfaced during implementation — including the single-repo `.plan` overwrite problem, which the spec solved with save/restore before it could bite.

Detailed specs earn their cost when the implementation crosses enough boundaries that holding the full picture in one context would be difficult. Thirteen state locations across five scripts, three skills, and two repos is exactly that case. The spec was the compression that made one-session implementation possible.
