---
layout: post
title: "Conventions, Audits, and an Embarrassing Gap"
date: 2026-05-17
entry: "19"
type: phase-update
entry_type: note
subtype: diary
projects: [Hortora]
excerpt: "The convention schema shipped. Then an audit revealed 512 garden entries that exist, are committed, and can't be found by index scan."
---

Session 11 settled the taxonomy — gotcha, technique, undocumented, convention.
Session 12 was the implementation. I brought Claude in to do the work across four
files: validate_pr.py, validate_garden.py, submission-formats.md, and the forage
SKILL.md. The SKILL.md alone had eleven enumeration sites that needed updating —
every place the skill listed three entry types now had to list four. When we
finished, soredium#44 was done.

The schema is straightforward. `validate_pr.py` now detects when a convention
entry shares a title with a sibling in the same domain but carries no `variant:`
field — that's a CRITICAL. `validate_garden.py` adds a whole-garden backup: any
same-title pair where either entry lacks `variant:` is an error. Both checks fire
independently so nothing slips through. The forage skill now has a SWEEP step for
conventions, a proactive trigger for when style choices come up in conversation,
and the editorial bar that distinguishes "deliberate choice between valid
alternatives" from "this is just correct."

Claude caught one gap I'd missed in the design: the CRITICAL message told the
submitter to add `variant:` to the new entry, but said nothing about revising the
existing sibling. Without that instruction, the PR would pass and the garden validator
would immediately fire. One-line fix to the error message, but it would have caused
a confusing failure loop in practice.

We also shipped five entries that had been waiting behind the schema: three-tier
Maven module structure, Maven submodule folder naming, Quartz RAM store, optional
Jandex library pattern, and always-scope-Maven-commands-with-pl. A sixth entry on
the list — blocking/reactive SPI parity — was already in the garden as a technique.
No duplicate needed.

---

## The domain cleanup

With the schema done, I turned to Audit 3: does every entry have a natural home?

The immediate find was `approaches/` — three entries, no coherent identity. The
entries themselves were Java design techniques (a varargs type-capture trick, a
SerializedLambda reflection technique, a routing policy pattern). Nothing about any
of them said `approaches/`. We moved all three to `jvm/`, corrected the `domain:`
frontmatter, and retired the directory.

The `quarkus/` situation is trickier. It has 208 committed entries, all correctly
placed as Quarkus-specific knowledge. But the forage skill now routes new Quarkus
entries to `jvm/` — so old knowledge accumulates in `quarkus/`, new knowledge in
`jvm/`, same technology, two homes. The right long-term fix is a merge of 208
files. The right short-term fix is a note in the skill: `quarkus/` is legacy, don't
write new entries there. We added the note. The merge is a separate operation.

---

## The gap that wasn't on the audit plan

Checking index coverage during the audit, I compared files on disk against entries
indexed in GARDEN.md.

`quarkus/`: 208 files, 67 indexed. `jvm/`: 110 files, 28 indexed. `java/`: 62
files, 34 indexed. Roughly 512 entries committed, real, and findable by `git grep` —
but invisible to forage SEARCH's index scan.

The root cause is that forage CAPTURE's deliver step commits the entry file and
stops. `integrate_entry.py` — the script that updates GARDEN.md, labels, and the
global index — is never called. Every submission since the garden was created has
skipped it. The GARDEN.md index is the fast path for search; git grep is the
fallback. More than half the garden currently lives only in the fallback.

The fix is wiring `integrate_entry.py` into the forage deliver step, but that needs
careful testing before it goes anywhere near a production garden. I'm deferring it
to its own session rather than rushing it. The gap is growing — five to ten new
entries per session — but git grep still finds everything. It's a quality problem,
not a correctness one.

---

## Protocol housekeeping

While the audits were running, we ran the protocol HEALTH check across the casehub
protocol directory. Seven schema violations: `severity: required`, `severity: error`,
`type: convention` — values written before the field vocabulary was standardised. We
normalised all seven, fixed a bug in the HEALTH check itself (`VALID_SCOPES` had
been missing `universal` and `application` since the check was written), and synced
the update.

25 protocols. All pass. All indexed. Unlike the garden.
