---
layout: post
title: "Hortora as a Knowledge Methodology"
date: 2026-05-07
entry: "16"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
excerpt: "The session started as a status review. Then I started pulling a thread: casehub's platform protocols and their relationship to the hortora garden. That thread unravelled into something bigger."
---

The session started as a status review. Then I started pulling a thread that had been sitting there: the casehub platform protocols — 33 convention files in `parent/docs/conventions/` — and their relationship to the hortora garden. Do they belong in hortora? And if not, why not?

The obvious answer is that they're project-specific. Quarkus gotchas in the canonical garden are useful to anyone building with Quarkus. A casehub convention about Flyway version range allocation per module is useless to anyone not building casehub. Project knowledge belongs in the project.

But that raised a bigger question. If Hortora's job is to be the place knowledge lives, why is it only doing one thing? The garden captures gotchas and techniques. It says nothing about how to structure ADRs, how to write platform conventions, how to maintain architecture docs. A new project starting from scratch gets none of that guidance.

I brought Claude in and we worked through what "Hortora as a project knowledge methodology" would actually mean. Three layers: discovery gardens (current), platform protocols (normative rules — "when you do X in this platform, do it Y way"), and ADRs (settled decisions with rationale). All three need to be RAG-optimal from day one. All three can be hierarchical — a child platform augments or overrides the parent.

The most interesting thing we designed was DEEP-SCAN. The protocol skill (not built yet) gets a standard sweep that runs at session end alongside forage. But DEEP-SCAN is different — a one-off that reads the actual codebase, finds patterns not yet captured as protocols, checks whether existing protocols are still followed, and scans for violations. You write a protocol saying "never call fireAsync inside @Transactional", add a `violation_hint` field to the YAML frontmatter, and DEEP-SCAN surfaces any code that breaks it. A protocol-aware linter on demand, no build plugin required.

Claude found a concrete example of why this matters. The casehub conventions and the hortora garden have been maintained separately. One convention says use `quarkus-junit`, not `quarkus-junit5`. A garden entry says `quarkus-junit5` is the correct artifact — `quarkus-junit` is the wrong one. One of them is wrong. Two systems, one fact, contradictory answers — and nobody noticed because they're in different places. That's exactly the failure mode we're designing against.

The casehub garden concept follows from all this: a project-scoped knowledge repo at `~/.hortora/casehub/` that sits below the canonical garden in the hierarchy. Discovery entries specific to casehub, protocols, architecture docs — all in one place, all queryable together. Roughly 70 entries in the canonical `quarkus/` garden reference casehub concepts directly. They're in the wrong place.

We ended the session doing something concrete: splitting `PLATFORM.md`. It had grown to include the application tier (devtown, aml, clinical) alongside the foundation repos. Foundation sessions load PLATFORM.md by default — they don't need application detail in their context. Claude carved out `APPLICATIONS.md`, moved the application tier there, left a single pointer in PLATFORM.md. The loading rule is simple: foundation sessions load PLATFORM.md only; application sessions load both.

The design doc is at `docs/design/2026-05-07-hortora-knowledge-methodology.md`. It's a pre-spec — four content audits need to run before anything gets implemented. Most of what we settled is a hypothesis until those audits hit the actual content.
