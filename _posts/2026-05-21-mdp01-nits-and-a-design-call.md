---
layout: post
title: "Nits and a Design Call"
date: 2026-05-21
entry: "36"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
---

A backlog of eight small issues from the convention schema work, most of them
genuinely small. Pull `defaultdict` out of a conditional block. Remove an off-topic
staleness note from a table row. Normalise an inline tuple to a module-level constant.
The kind of cleanup you do between the real sessions.

One of them had a design question buried in it.

The issue described a failure mode in the staleness review workflow: a garden entry's
`invalidation_triggers` says something like "revisit if library X gates feature Y behind
a build flag." The structural fix exists — it was found eventually, during unrelated
work — but there was no mechanism to move from "this trigger describes an open question"
to actually checking whether the answer had arrived.

Claude's initial read was Option A: just display the triggers prominently during the
harvest REVIEW workflow. No schema change, solves the attention problem, done.

I pushed back. Why not a structured field? Why not both?

Thinking it through: Option A alone has no memory. You'd see the same trigger text on
every REVIEW cycle with no signal that you'd already looked at it. Option B alone — an
explicit `invalidation_status: pending` field — is precise but requires someone to set it
the first time, and nothing prompts that. The hybrid solves both: REVIEW always surfaces
triggers, and once you've decided a trigger describes an open structural fix, you set
`pending` and the next REVIEW gives you a distinct prompt instead of the generic
CONFIRM/REVISE/RETIRE choice.

Four files: format check in the entry validator, a row in the submission formats table,
the harvest REVIEW step updated to handle both the trigger display and the pending prompt,
the REVISE workflow updated to describe when to set it. Nothing elaborate — the same
pattern as the `protocol:` linkage field we'd done earlier in the same session.

One gotcha during the `validate_pr.py` work worth noting. We used `replace_all=True` on
an Edit call to rename `Path(garden_root)` to `garden_path` across the function. The
pattern matched the definition line too: `garden_path = Path(garden_root) if garden_root
else None` became `garden_path = garden_path if garden_root else None`. Syntactically
valid Python, no warning from the tool. Tests failed with `UnboundLocalError` pointing
into the function — not at the edit. The lesson: after any `replace_all` that introduces
a new name, check the definition line explicitly.

One issue didn't make it: the GARDEN.md two-level rendering for same-title convention
entries. The filed issue pointed at `integrate_entry.py` as the place to add the
rendering logic, but `integrate_entry.py` doesn't write the By Technology listing at all
— only increments a drift counter. The actual listing is maintained manually.
Implementing the rendering would have required building the update function first, which
is already queued as the larger priority: ~512 garden entries submitted but never indexed
in GARDEN.md. The rendering is a refinement on top of work that hasn't happened yet.

That's next.
