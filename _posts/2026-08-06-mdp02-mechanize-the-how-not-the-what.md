---
layout: post
title: "Mechanize the How, Not the What"
date: 2026-08-06
entry_type: note
subtype: diary
projects: [soredium]
tags: [mechanization, garden, slots, plan-queue]
---

Three inline git sequences in forage had been on the audit list since April. Each one followed the same pattern: the LLM decides what to capture, writes the entry, then executes four or five git commands — add, commit, pull --rebase, push — as prose instructions. If the rebase conflicts, there's no recovery. If the push fails, the garden repo is in a half-rebased state.

The principle that finally moved them off the list: **mechanize the HOW, not the WHAT.** The LLM adds value in deciding what's garden-worthy and writing the content. Once the file is on disk, `git add && commit && pull --rebase && push` is plumbing. A Python script handles it atomically with conflict recovery — abort the rebase, report the error, let the caller decide. The LLM orchestrates; the script executes.

`garden_commit.py` has two subcommands: `commit` (stage files, index directories, garden.db, commit) and `push` (pull --rebase, push, abort on conflict). The forage skill now calls these instead of emitting raw git commands. Same for the fork sync in work-start — three topology cases (upstream remote, fork remote, single remote) that were inline prose are now `branch_create.py sync-main`. The script detects which model applies and runs the right sequence. Network failures warn but don't block — stale main is suboptimal, not fatal.

The more interesting piece came from a slot problem. `git clone --shared` doesn't carry gitignored directories. When a project uses yarn portal resolutions pointing to a gitignored `.casehub-packages/` directory — twenty built packages from another repo — the clone looks complete but the build chain is broken. The error messages point to import resolution, not to a missing directory. You'd investigate package.json and node_modules before suspecting the real cause.

The fix: after cloning, scan the source repo for top-level directories that are gitignored but present. Skip regenerable ones (node_modules, build, dist, target, IDE artifacts). Symlink the rest into the clone. Detection is clean — `git check-ignore -q` per directory, no `.gitignore` parsing needed.

The second piece of work was about what happens when implementation surfaces more work than the plan anticipated. A `.plan` queue has agreed items — the issues you set out to close. But during implementation, follow-up work appears: gaps discovered, refactoring opportunities, things that should be done but weren't in scope. Until now, those either got filed as ad-hoc issues or disappeared when the session ended.

`.plan` now has a `## Deferred` section. Each item carries scale, complexity, and target repo scope. When the agreed queue completes, `advance()` signals `has_deferred=True` instead of `queue_complete`. The work skill filters deferred items by available repos — if you're in a slot, only items targeting repos cloned into that slot get offered. The prompt biases toward continuing: promote matching items and keep working, rather than stopping to file issues and create new slots.

The repo-scoping is the key constraint. Deferred items that need repos not available in the current context can't be done here regardless of intent. They get filed as GitHub issues for a future slot. But everything that *can* be done in the current context stays in play.
