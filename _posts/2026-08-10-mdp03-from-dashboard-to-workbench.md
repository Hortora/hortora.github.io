---
layout: post
title: "From Dashboard to Workbench"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [grove]
tags: [grove, curation, workbench, filtering, split-pane, sparge]
series: issue-24-curation-workbench-ui
---

The intelligence dashboard I built two days ago already tells you what's wrong with your garden. 1,206 entries with health scores below 40. 449 never retrieved. Topic clusters with no tags. The numbers are right there. The problem is you can't do anything with them — it's a static dump. You scroll, you see a triage queue sorted by worst score, you click an entry, you leave the dashboard entirely, you act, you navigate back, you've lost your place. The dashboard tells you what to think about. It doesn't help you work through it.

That's the gap the workbench fills.

![Grove curation workbench — split-pane detail view with signal-driven filters, entry list, and entry detail with health breakdown bars](workbench-split-pane.png)

The core idea is signal-driven filtering. Instead of organising by domain and drilling down (which forces you to already know where the problems are), the workbench starts with the question: what kind of problem am I looking for? Stale entries. Duplicates. Outliers. Low health scores. Never retrieved. Each filter is a button with a count — you see the shape of the problem before you click. The pattern comes directly from Sparge, where the same `.fb` toggle buttons and `.itr` breakdown rows drive the review workflow. Hover over the filter zone and secondary signals appear: never retrieved (449), version aging (0), untagged (2,642). Click a signal and per-domain breakdown rows show you where the problems concentrate — jvm has 829 low-health entries, quarkus 133, web 80.

Two views share the same filter state. List mode gives you a sortable table with checkboxes for bulk actions — select twelve stale entries, confirm freshness in one click. Detail mode is a split-pane: the entry list on the left (320px, compact rows with GE-ID, title, domain badge, composite score), the full entry detail on the right with health breakdown bars, metadata, and curation actions. Click an entry in the left panel, the right panel loads its content. The URL tracks the selected entry (`#workbench/GE-20260504-ae76f6`) so you can share links.

The data architecture behind this ended up more interesting than I expected. The workbench needs three things: basic entry metadata, health scores, and boolean flags (is this a duplicate? an outlier? never retrieved?). Those come from three independent endpoints — `GET /api/entries` for metadata, `GET /api/analysis/health` for the composite scores, and a new `GET /api/analysis/flags` for the booleans derived from cached analysis results. The client loads all three in parallel via `Promise.all`, joins them by Qdrant UUID, and defaults missing fields to null or false. Each endpoint degrades independently: no health data? Scores show "—". No analysis cache? Signal filters show 0. The entry list always works because metadata is always available.

This three-way join is what makes the filter counts instant. Domain and type dropdowns scope everything client-side — no server round-trips for filter state changes. With 2,642 entries the joined dataset is about 400KB in memory. Clicking between filters is immediate.

Browser testing caught three bugs that wouldn't have surfaced from type checking alone. The first was a ghost selection state: select three entries in list mode, switch to detail mode, switch back — the bulk bar says "3 selected" but no checkboxes are checked. The entry table component gets unmounted when you leave list mode and remounted fresh when you return. Its internal selection `Set` starts empty. But the parent workbench shell's `@state` property holding the selection survives the round trip. Claude caught this during Playwright testing — the fix is clearing `bulkSelected` on every path that exits list mode.

The second was simpler: the plan called for reusing `grove-entry-table` in list mode, but the table has its own built-in Type and Staleness dropdown filters. With the workbench's filter box also providing those controls, users would see two sets of filter controls — workbench filters on top, table filters below. A new `showControls` property on the table suppresses its built-in filters when embedded.

Third: once you've clicked "Low Health" to see 1,206 entries, how do you get back to seeing all 2,642? If you don't notice the "All" button — easy to miss when secondary filters are expanded — you're stuck. The fix: clicking an already-active filter button toggles it off, returning to "All". Standard radio buttons don't do this, but Sparge's `.fb` buttons do, and for the same reason.

The workbench doesn't run analyses itself — duplicates, outliers, and topic clusters still need to be triggered from their respective domain views or the intelligence dashboard. What it does is make the results of those analyses actionable. Once you've run the analyses, the workbench is where you work through them — filtering by signal, scanning the breakdown, picking entries, acting, and moving on without losing context.

What this opens up is the question of whether the workbench should be the primary view. Right now it's a parallel route alongside Domains and Intelligence. But a curator's daily workflow is probably: open workbench, filter to stale, scope to a domain, work through the list. The domain map and intelligence dashboard become reference views you check occasionally, not the place you spend your time. I'll see how the workflow feels once the analyses are populated and real curation sessions start generating data.