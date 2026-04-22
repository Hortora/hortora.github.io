---
layout: post
title: "Letting the Garden Tend Itself"
date: 2026-04-22
entry: "13"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
excerpt: "Manual harvest sessions were consuming context window and blocking R&D. The fix was a background Claude agent with its own isolated settings, firing on every commit."
---

Manually running harvest sessions to keep the garden deduplicated was starting to feel like housekeeping — necessary but never the reason I sat down to work.

By the time we started today, the drift counter had hit 69 entries since the last sweep. That meant working through 50 unchecked pairs: reading both entries, classifying them, adding cross-references, discarding duplicates. We did it — four duplicates removed, cross-references wired across ten clusters — but it consumed two hours and a significant chunk of the context window.

I didn't want to repeat that pattern.

The obvious move was to automate it. The less obvious question was *how*, without the automation itself consuming the context window I was trying to protect.

The answer turned out to be `claude --print` in the garden's own directory, with its own `.claude/settings.json` pre-approving the commands it needs. When that fires from a git post-commit hook in the background, it's completely isolated from whatever I'm doing in the main session. Different context, different permissions, different log file. The main session never knows it happened.

We designed this as a pure post-commit hook: the garden agent fires whenever a commit adds new GE-*.md files, processes all unchecked pairs, and commits the results. Fully autonomous — score and date tiebreakers handle duplicates without asking.

One design decision I kept second-guessing was the on-demand interactive mode that appeared early in the design. Eventually I cut it. The hook covers the primary case; if something goes wrong, running `./garden-agent.sh` manually from a terminal is the safety valve. A formal interactive mode would have been complexity without a use case.

The installer is a single bash script — idempotent, no tokens consumed. Running it twice gives you "already present" for everything. The post-commit hook uses a sentinel comment to prevent double-installation when appended to an existing hook file:

```bash
# garden-agent: auto-installed
_new_entries=$(git diff --name-only HEAD~1 HEAD 2>/dev/null \
  | grep -E "^[^/]+/GE-[0-9]{8}-[0-9a-f]{6}\.md$")
if [[ -n "$_new_entries" ]]; then
    nohup "$_GARDEN_ROOT/garden-agent.sh" --hook >> "$_LOG" 2>&1 &
fi
```

The TDD produced 12 tests covering creation, idempotency, append-to-existing, and the `HORTORA_GARDEN` environment variable. That last one was a gap Claude flagged before approving the implementation — nothing was verifying that the installer respects `HORTORA_GARDEN` when set to a directory other than the current working directory. A reasonable thing to miss; a bad thing to ship without.

Claude also caught that the installed `garden-agent.sh` was using `$(pwd)` as its fallback garden path while the spec said `$HOME/.hortora/garden`. Both produce the same result when the hook fires (git hooks run with cwd at the repo root), but the spec version is deterministic regardless of how the script gets invoked. We aligned it.

New machine setup: clone the repos, run `garden-agent-install.sh` from inside the garden directory. The agent wires itself in and starts processing on the next commit.
