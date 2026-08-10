---
layout: post
title: "Three Verbs to Replace One"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [soredium]
tags: [lifecycle, cli-design, work-commands]
---

The work lifecycle had a word problem. "Resume" was doing two jobs — restoring a paused branch from the stack, and picking up where you left off on an active branch. Two different operations, one overloaded verb. On top of that, the menu showed "start" for branches that had already been started, purely because no HANDOFF.md existed yet. Wrong label, correct behaviour.

The fix is three verbs with non-overlapping semantics: `continue` (keep working on this branch), `resume` (restore from the pause stack), and `brief` (orientation summary — what's the state of things?). Each maps to exactly one operation. No guessing which flavour of "resume" you're invoking.

`continue` is where the interesting design sits. It subsumes what HANDOFF.md used to require a separate command for: reading the last session's narrative, loading specs, syncing plan state with GitHub, checking workspace health. The handover document still gets written at session end — its narrative value (what was tried, what reasoning drove decisions) is real. But reading it on resume is now automatic, not a ceremony. Done-detection goes further: if the current issue has been marked complete since last session, `continue` auto-suggests "next" or "end" instead of leaving you staring at finished work.

The wrong-context handling was a small decision that felt worth getting right. `work start` on a feature branch is forgiving — it redirects to `continue` with a note, because the intent is clear. `work resume` on a feature branch is strict — "not paused, use continue." The semantic distinction is the whole point; silently redirecting would undermine it.

`/brief` took shape as a structured data layer rather than a rich CLI skill. `brief.py` composes three existing scripts — `ctx.resolve()`, `work_router.detect_state()`, and `work_health.run_checks()` — into unified KEY=VALUE output. The skill wraps it for the terminal. The real delivery mechanism will be the Trellis frame, showing this information ambiently. Building the data contract now means both consumers read the same structured output without duplicating the composition logic.

Getting `brief.py` to import those scripts cleanly meant refactoring `ctx.py` — previously a 300-line top-level script that printed to stdout as it went. Wrapping the state derivation into `resolve() -> dict` and guarding the print loop with `if __name__ == '__main__'` was straightforward, but `ctx.py` is called by every skill in the system. The golden test (capture CLI output, compare before/after) gave confidence that nothing shifted. `work_health.run_checks()` still prints to stdout — `brief.py` captures it with `redirect_stdout` into a StringIO buffer. Not elegant, but it avoids refactoring a function with fourteen callers for one new consumer.

The lifecycle state machine picked up a self-transition: `work_continue` on the `active` state. No state change, but the event fires — which means worklog observability captures "session continued at this timestamp" without any new infrastructure. The transition table already has the machinery; adding a row costs one line.

What this opens up: the three-verb taxonomy gives the Trellis frame a clean command vocabulary to wire into its UI. `brief.py` is the first script designed from the start to be consumed by both CLI and frame. If the pattern holds, the other lifecycle scripts follow the same refactor — importable functions with a thin CLI wrapper — and the frame gets structured data without shelling out.
