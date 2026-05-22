---
layout: post
title: "The Parser That Couldn't Parse Itself"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
entry: "21"
excerpt: "The garden backfill wasn't 500 unindexed entries — it was 80 GARDEN.md gaps, one broken YAML parser, and a self-defeating gotcha entry."
---

We started on a test infrastructure nit sweep — three issues filed alongside the index fix. The most interesting was extracting a shared `_make_git_garden()` helper from two test classes with identical setUp code: GitGarden creation, domain scaffolding, `_index/global.md`, commit. Duplicated setup rots quietly — when the garden schema changes, both classes need updating separately. The helper takes a domain list and returns the configured fixture. Both test classes collapsed to a single line.

The two missing tests were straightforward. One verifies CAPTURE works on a local garden with no remote — `integrate_entry.py` doesn't push, so this was always going to pass, but worth having explicit. The other checks the sweep batch commit message format. The test fixtures were also missing `verified: true` and `garden: discovery` fields; they'd been passing because `skip_validate=True` was set, but they weren't realistic garden entries.

Then the backfill.

We ran a script expecting to find ~500 unindexed entries. It found 668 "unindexed" by checking `_summaries/` — but when we staged the results, git showed almost no changes. The garden was already well-indexed. The `_summaries/` files had been committed but were missing from disk; the script regenerated them with identical content. The real gap was different: 80 entries absent from GARDEN.md's By Technology section.

The fix required a surgical approach. `integrate_entry.py` appends to domain INDEX.md files without checking for existing entries — running the full integration function on partially-indexed entries would create duplicates. We extracted just `update_garden_by_technology()` and ran it on the 80 missing entries only. Seventy-six patched cleanly. One failed.

The failing entry was about Kramdown's behaviour when a paragraph is immediately followed by `---` — Jekyll interprets it as a Setext h2 heading rather than a horizontal rule. The entry's title contained the literal string `---`. The YAML parser threw "unexpected end of stream at column 59", pointing to the middle of the title string. Nothing in that error suggests a split operation is the cause.

Claude traced it: `parse_entry` used `content.split('---', 2)`, which fires on any `---` in the string regardless of YAML quoting. The fix is a regex match on line-boundary markers: `re.match(r'^---\n(.*?)\n---\n?(.*)', content, re.DOTALL)`. We submitted the gotcha to the garden immediately — an entry about `---` misinterpretation that broke a parser because it contained `---` in its title. `validate_pr.py` had the same bug. Both fixed, both pushed.

The session finished with the technique→protocol lifecycle documentation. The gap was straightforward: no documented relationship between a universal garden technique and its adoption as a project-specific protocol. We added `garden_ref` to the protocol entry format and updated both skills to describe the bidirectional link. A protocol backed by a garden technique has a rationale. A technique adopted as a protocol has a commitment. Both halves matter.
