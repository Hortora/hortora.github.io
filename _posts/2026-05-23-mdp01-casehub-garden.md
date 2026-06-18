---
layout: post
title: "A Garden for CaseHub"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: log
projects: [hortora, casehub]
entry: "39"
excerpt: "A broken protocol ref turned into a design problem. The protocol store needed its own repo."
---

The session started with a simple question: what is the "quarkus/ → jvm/ merge" mentioned in the handover? It turned out to be two things conflated into one label — a YAML domain field migration (`quarkus` → `jvm`) decided in April but never executed, and a physical directory rename the design explicitly said not to do. The directory stays for GitHub browsability; the field migration is the real work.

Before touching it, I wanted to verify the underlying decision was still right.

## The coarse domain decision, revisited

We ran an exhaustive evidence review — every design doc, the ADRs, git history across both repos, the retrieval service spec. The coarse Qdrant partition approach (eight categories: `jvm`, `python`, `tools`, `web`, etc.) is theoretically sound. Fine-grained attribution at capture time is unreliable — a Drools bug running on Quarkus can't reliably be attributed to either technology at the moment you file it.

The problem is the decision moved technology precision to `tags` and `stack` as a replacement, and 51% of entries still have `tags: []`. You've removed the precision without delivering the substitute. Right now the coarse approach gives worse retrieval than the fine-grained one it replaced.

I'd propose adding a `technology` field — best-effort, not authoritative, one or two values. Different from `root_cause_layer`, which requires certainty. If you were working in Quarkus, `technology: quarkus` is enough even if the root cause turns out to be Drools. Gives you filtering without demanding reliable attribution. We held that decision pending.

## Broken refs and where they came from

A broken ref report from casehub/platform: three files referenced in ADRs didn't exist locally. We traced them — `typed-preference-keys.md`, `platform-spi-contract.md`, `module-tier-structure.md` were all in casehub/parent, not casehub/platform. The ADRs pointed to local paths that were never created there.

Symptom, not root cause. The root cause was that the protocol store was split across repos depending on which session captured the protocol. The protocol skill resolves `$PROJECT_ROOT/docs/protocols` from whatever repo is the current working directory. A casehub/platform session creates protocols locally. A casehub/parent session creates them in parent. Neither is consistent; both happen silently.

The structural problem runs deeper. casehub/parent has feature branches. When parent is on a work branch, protocol commits from any peer session land on that branch — invisible to every other session until the branch merges. Protocol stores can't live in repos with development cycles.

The skill itself had the same gap. Its resolution order checked `$PROJECT_ROOT/docs/protocols`, then a child `parent/` subdirectory, then created the directory if nothing was found. Sibling layouts — where each module sits beside a `parent/` at the same level — weren't handled. Any peer session without an explicit routing override in CLAUDE.md would silently create a local `docs/protocols/` and deposit protocols there. That's exactly what happened in casehub/platform.

## casehub/garden

We created `casehubio/garden` — a private GitHub repo, always on main, no branches ever. Moved the full protocol store (68 files across `casehub/` and `universal/`) from parent to garden. Updated every peer repo's CLAUDE.md to point to `../garden/docs/protocols/`. Updated the skill's resolution order: CLAUDE.md routing row first, then local, then `../parent/`, then create as last resort. Fixed the commit step to resolve the git root from the resolved protocols directory, not the current project — otherwise commits go to the wrong repo.

The bigger point is that this is the federated garden model from the Hortora spec, applied concretely. Hortora garden holds universal JVM, Python, and tools knowledge. casehub/garden holds platform-specific protocols and — eventually — casehub-specific gotchas currently scattered through the hortora garden under the `quarkus/` domain.

The two audiences were already implicit in the index structure: FOUNDATION-INDEX for developers building the platform, HARNESS-INDEX for developers building applications on it. The tiering just needed its own repo.
