---
layout: post
title: "Batching the Sweep"
date: 2026-04-18
entry: "09"
type: session
excerpt: "Forage SWEEP was opening one PR per entry — six to eight PRs for a typical session sweep. We fixed it."
---

The forage SWEEP workflow had a performance problem I'd been ignoring: every
confirmed entry produced its own git commit and its own GitHub PR. A typical
session sweep surfaces 6 to 8 entries. That's 6 to 8 separate branches, 6 to
8 push/checkout cycles, 6 to 8 PRs to review.

I asked whether it could be batched. Claude identified the constraint
immediately: user confirmation per entry is inherently sequential — there's no
way around the human-in-the-loop approval step. But the git delivery at the
end is independent of that. Write all the files, validate, then one branch,
one commit, one PR listing everything.

We changed SWEEP Step 4 to do exactly that. Each confirmed entry still goes
through CAPTURE steps 0–6 — ID generation, scoring, duplicate check, draft,
confirm, write — one at a time. Validation and delivery are deferred until all
files are written.

For validation we added a threshold: if 3 or more entries were confirmed, run
`validate_pr.py` in parallel. Below that, sequential — the overhead isn't
worth it for one or two entries.

```bash
for ENTRY_PATH in <list of written entry paths>; do
  python3 validate_pr.py "$ENTRY_PATH" "$GARDEN" &
done
wait
```

The delivery side collapses from N commits and N PRs to one:

```bash
git checkout -b sweep/$(date +%Y%m%d)-batch
git add <all written entry files>
git commit -m "sweep: <N> entries — <slug1>, <slug2>, ..."
gh pr create --title "sweep: <N> entries" ...
```

The change lives in `soredium/forage/SKILL.md`. Pushed as
[soredium#30](https://github.com/Hortora/soredium/issues/30).
