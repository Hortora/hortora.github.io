---
layout: post
title: "Three Kinds of Knowledge"
date: 2026-05-13
entry: "18"
type: phase-update
entry_type: note
subtype: diary
projects: [Hortora]
excerpt: "Classifying garden entries forced a question I hadn't asked: what exactly is a convention, and should the garden know the difference?"
---

The taxonomy question came up halfway through Audit 2.

I was classifying 36 casehub protocol entries into "move to garden" or "keep in protocols" when the binary broke down. Maven submodule naming conventions aren't gotchas — they're deliberate style choices where alternatives exist. Quartz RAM store isn't a fact about how Quartz works — it's a decision casehub made, with JDBC store as a valid alternative. These weren't errors or non-obvious techniques. They were preferences.

I'd been treating the garden as a library of universal truths. It isn't, entirely. Some entries say "this is how the technology behaves." Others say "this is how we've chosen to use it." The difference matters when an LLM is deciding whether something applies.

Working through the implications with Claude, we landed on three categories. **Universal** — objective facts about technology behaviour. `@IfBuildProperty` is evaluated at augmentation, not runtime. Panache statics bypass CDI alternatives. True for anyone using the technology. **Convention** — a style or structural choice where alternatives are equally valid. Three-tier module structure, Maven submodule naming, Quartz RAM store. Another project could choose differently and be right. **Casehub-specific** — only meaningful in casehub's domain. `WorkItemStatus.EXPIRED.isTerminal()` returning false. Qhorus protocol facts. No useful life outside the project.

From there came a garden schema proposal: add `type: convention` alongside `gotcha`, `technique`, and `undocumented`. And for conventions with alternatives, a `variant:` field — two entries covering the same decision space, linked under a shared title. Maven submodule naming, with one entry for the Quarkus extension convention and another for the Spring layered approach. RAG hits the parent heading and loads only the variant the project uses. The convention entries wait for that schema change to land.

The audit itself produced concrete numbers. 22 universal gotchas and techniques from the casehub protocol files moved to the canonical garden — CDI async transaction boundaries, `@IfBuildProperty` build-time evaluation, `@QuarkusTest` naming traps, SQL type portability. Then 33 entries reclassified across `quarkus/` and `jvm/` to `casehub-engine/`, `casehub-work/`, `casehub-ledger/`, and a new `casehub-qhorus/` domain. We discarded 8 duplicates that covered the same ground as entries we'd just submitted.

The garden's post-commit hook created one genuine complication. We staged 29 `git mv` operations to move entries between domains. The hook auto-committed them before we could intervene. Four moves had failed silently — `git mv` requires the target directory to exist first, which is easy to miss. Those four files ended up deleted from the index without landing anywhere. Recovery: `git show <commit>:original/path > new/path`, then re-add. It works, but it shouldn't have been necessary.

The casehub protocols INDEX.md now has two new sections: garden references for the 22 universal entries, and casehub domain entries for the 33 reclassified ones — both there for LLM discoverability until proper garden RAG lands.
