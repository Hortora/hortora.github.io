---
layout: post
title: "Beyond Claude"
date: 2026-04-15
entry: "08"
type: phase-update
excerpt: "The MCP server is the moment the garden stops being a Claude Code feature and becomes something any AI tool can use. Then we looked at the foundations and didn't like what we saw."
---
# Beyond Claude

I'd been thinking of the garden as a Claude Code thing. The skills, the forage workflow, the harvest sessions — all of it assumes Claude. That felt fine until Phase 8, when we built the MCP server.

An MCP server changes what the garden is. Once `garden_search` and `garden_capture` are exposed as tools, the garden doesn't belong to Claude anymore. Cursor can query it. Copilot can submit to it. Any AI with MCP support can read and write. The federation model we built in Phase 6 — canonical gardens, child extensions, peer networks — stops being a design document and starts being infrastructure anyone can attach to.

That shift happened quietly. We spent most of this session building scaffolding: the tooling for initializing canonical gardens, schema validation, domain routing, the `_augment/` layer for child gardens, the upstream deduplication check. Each piece was necessary. None of it felt like a milestone. Then we wrote the MCP server and I realized: the garden is now a protocol, not a plugin.

## What surprised me about Phase 7

The web app was supposed to be the "human interfaces" milestone. We built it — a garden browser that fetches entries from GitHub's raw content API, renders them with Fuse.js search, shows staleness indicators, slides in a detail panel on click. It works. I'd wanted it to feel like a field guide, and it does: parchment and forest green, entry cards with type badges, staleness warnings in amber.

What surprised me was the Obsidian story. I'd expected to write a plugin. Instead we wrote documentation and a set of Dataview queries, and it turned out to be enough. The garden's YAML frontmatter is already Obsidian-native — `tags:` maps directly to Obsidian tags, `score:` and `staleness_threshold:` are sortable properties. Five Dataview queries give you staleness views, high-value entries, domain summaries, and recent captures. No plugin needed. The entry format doing double duty as a human knowledge tool and an AI retrieval format felt like something going right.

## The flat file reckoning

The scaling conversation happened organically. I noticed CHECKED.md was already 4,925 lines at 240 entries. At 1,000 entries with meaningful DEDUPE coverage it would hold hundreds of thousands of rows. Every lookup is a full scan. Every write is a full file rewrite. Concurrent CI runs produce merge conflicts.

The obvious answer was to shard by domain — `java/CHECKED.md`, `tools/CHECKED.md`. That delays the problem by roughly the number of domains. The less obvious answer was git refs: each checked pair as a ref like `refs/garden/checked/<pair-hash>`. Git's ref store is a B-tree, O(log n) lookup, native sync via push/pull. Zero new dependencies.

I liked that answer for about twenty minutes. Then I worked through the math. At 1,000 entries with 10% pair check coverage: ~50,000 refs. Fine. At 5,000 entries: millions of refs. Git operations — status, fetch, gc, clone — degrade linearly with ref count. The same ceiling as CHECKED.md, just higher. Git refs were designed for branches and tags, not for storing millions of application records.

SQLite is the right answer. Not because it's clever, but because it's the correct tool: O(log n) primary key lookup, ACID writes, scales to 100 million rows practically, Python stdlib, single file. The concern about losing git diff readability turned out to be solvable with a textconv driver — `.gitattributes: garden.db diff=sqlite3` — which makes `git diff` show SQL row changes instead of binary noise. We wrote the ADR, then built the migration.

## The two wrong answers I almost chose

The 8-character SHA-256 hash as a primary key came up during implementation. It seemed obviously fine — SHA-256 is collision-resistant. What I missed: collision resistance is a property of the full 256-bit output. An 8-hex-char (32-bit) truncation has a birthday collision probability of about 11% at 1,000 rows. `INSERT OR IGNORE` silently discards the collision — no error, no log. The data loss is undetectable except by auditing row counts.

The fix was anticlimactic: use the canonical pair string itself as the primary key. It's 40 characters. That's fine. The hash was solving a problem — a short surrogate key — that didn't exist.

## Where things stand

The garden is now a platform. It has a schema, a federation protocol, an MCP server, a web browser, and a SQLite backend. The canonical gardens haven't been created yet — `jvm-garden` and `tools-garden` exist as concepts but not repos. That's the practical next step: run `init_garden.py`, push to GitHub, merge the existing entries. Everything we built this session was infrastructure for that moment.

One thing I keep coming back to: the garden health check surfaced 123 entries with empty tags — all the legacy GE-0NNN entries from before the YAML migration. The tags were never backfilled. A rule-based script could infer most of them from the title and stack fields. That's a session's worth of work that would meaningfully improve search quality. Not urgent. But not nothing.
