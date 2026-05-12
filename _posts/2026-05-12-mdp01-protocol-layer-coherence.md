---
layout: post
title: "Protocol Layer and Coherence"
date: 2026-05-12
entry: "17"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
excerpt: "Three weeks of design work, then implementation. The protocol layer is now real — and figuring out what protocols actually are turned out to be the interesting part."
---

Three weeks of design work, then implementation. The protocol layer is now real.

The immediate priorities from the design session: migrate 33 casehub convention files to YAML frontmatter, rename `conventions/` to `protocols/`, build a `protocol` skill, hook it into the handover wrap. We did all three. The conventions are now proper protocol entries — `id`, `title`, `type`, `scope`, `applies_to`, `severity`, `violation_hint`, `created` in the frontmatter, one file per rule, retrievable independently.

The quarkus-junit contradiction I'd flagged last session got resolved on the way. One casehub protocol said use `quarkus-junit`. A garden entry said use `quarkus-junit5`. We inspected the actual Maven POMs in `~/.m2` and settled it: `quarkus-junit5` has been a relocation stub since Migration Guide 3.31, pointing to `quarkus-junit`. The garden entry had the direction backwards. We deleted it.

Most of what followed turned into something more interesting than implementation — working out what protocols actually are.

My first attempt: protocols are casehub-specific rules. Wrong. Too broad. `quarkus-test-naming-convention` (*Test.java not *IT.java) applies to every Quarkus project; a casehub protocol shouldn't be the canonical source for that. Second attempt: gotchas only a casehub developer would hit. Also wrong — any gotcha encountered while developing casehub is either a generic Quarkus issue (goes to the garden) or design guidance (stays as a protocol). There's no residual casehub gotcha category distinct from both.

The right frame came from asking what protocols are for, not what they contain: platform coherence and consistency. A protocol answers "how do we do this consistently across the platform?" — not "how do I avoid this library trap?" Once that landed, the audit criteria simplified. A protocol earns its place if removing it would let an LLM or developer make a locally reasonable choice that quietly diverges from the platform's established patterns. Maven submodule naming conventions qualify. CDI fireAsync transaction boundaries don't — that's a Quarkus gotcha, it goes to the garden.

That definition pulled several things up from casehub protocols into `java-dev` — consolidation across module and repo boundaries, prioritising clean APIs, the right framing for commit discipline. The old "minimize changes" rule was conflating two things: don't scatter unrelated changes across a commit, and don't touch more code than necessary. The first is right; the second was blocking legitimate refactoring. We separated them.

The other structural problem: `config-architecture.md` — a document mapping where every piece of Claude guidance lives — was sitting in `cc-praxis`, a generic skill repository. Half of it was casehub-specific: the Platform Coherence Protocol section, prompt snippet paths, `PLATFORM.md` references. We split it. `cc-praxis` keeps a generic version. `casehub/parent/docs/` gets the casehub version. `update-claude-md` now reads a `**Config architecture:**` URL from the project's CLAUDE.md before falling back to the generic one — each project owns its own configuration map.

That's the principle that surfaced three times in one day: nothing casehub-specific lives in Hortora repos unless it can be generalised. The protocol skill, the config architecture map, the development posture guidance — all three started casehub-specific, all three got either generalised or moved.

Audit 2 — classifying which of the 33 protocols are genuine platform coherence rules vs Quarkus gotchas that belong in the garden — is half-done but not yet actioned. Roughly 9 of the 33 look like real protocols. The rest are going to the garden.
