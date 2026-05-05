---
layout: post
title: "Phase 5 Done — and Two Bugs That Hid"
date: 2026-05-05
entry: "15"
type: phase-update
entry_type: note
subtype: diary
projects: [hortora]
excerpt: "Phase 5 landed: validate_schema.py, init_garden.py, 120 tests passing. Then a code review found a drift counter that silently defaulted to zero — and 29 MCP test failures that were blaming the wrong layer."
---

Phase 5 landed in a previous session: `validate_schema.py` validates SCHEMA.md
federation configs, `init_garden.py` initialises a new canonical, child, or peer
garden from scratch. Both came in with 120 tests — all passing. What I wanted to
do this session was verify the work properly and review it before calling it done.

I brought Claude in to do a systematic code review against the spec. It came back
with four issues, one of which was a textbook silent failure.

## The drift counter that never counted

`init_garden.py` writes a fresh `GARDEN.md` with this line:

```
Entries merged since last sweep: 0
```

Plain text. But `validate_garden.py --dedupe-check` reads the counter with:

```python
re.search(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', content)
```

That regex expects Markdown bold markers. Without them it never matches,
the counter is treated as `None`, and the dedupe threshold check is silently
skipped. Every garden initialised with `init_garden.py` would permanently
report zero drift regardless of actual state.

Both files had tests. Both tests passed. The writer test used
`assertIn('Entries merged since last sweep: 0', content)` — a substring
match that succeeds whether or not the bold markers are there. The fix
was one character: `**Entries merged since last sweep:** 0` in the
template, and a tighter assertion in the test to match what the parser
actually expects.

Claude also caught that `description` was listed as required in the spec
but never enforced in `validate_schema.py`, and that schema validation
warnings were routed through `log_info` — invisible without `--verbose` —
rather than `log_warning`. Both straightforward.

## When the error lies about where it lives

There were 29 failing MCP protocol tests I'd been deferring. The error
was `McpError: Connection closed`. That looks like a protocol initialisation
failure — the server not responding correctly to the MCP handshake.

The actual cause was this line in the test setup:

```python
StdioServerParameters(command='python3', ...)
```

`python3` here is the system Python — Homebrew-managed, no `mcp` package
installed. The test suite runs under a pyenv Python that does have it.
So the subprocess spawned the server with the wrong interpreter, which
crashed immediately on `import mcp` and closed the pipe. The error
surfaced at the MCP client layer as a connection failure; the real cause
was a `ModuleNotFoundError` buried in stderr.

The fix is always `sys.executable` when you need a test subprocess to use
the same Python environment as the test runner. One line, 29 tests flip
from failing to passing. All 836 pass now.
