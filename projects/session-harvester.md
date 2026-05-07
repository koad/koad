---
name: session-harvester
title: Session harvester — collect, scrub, summarize all harness transcripts
status: active
created: 2026-05-06
updated: 2026-05-06
owners: [juno, vulcan]
tags: [sessions, harvester, scrubber, walker, archive, findings, aegis, vesta]
relates-to:
  - session-indexer
  - drift-atlas
  - control-tower-postgres
---

# Session harvester

## What

A scheduled walker that collects session transcripts from every harness
storage location, deposits them in the canonical archive
(`~/.forge/archive/sessions/<YYYY-MM-DD>/`), then scrubs each session
to extract structured summaries: key findings, accomplishments,
decisions, corrections, drift moments, and entity calibration signals.

The scrubber output feeds two consumers:
1. A daemon-interface UI that lets koad browse summarized sessions and
   dispatch entities (Vesta, Aegis, etc.) back into specific valuable
   conversations for deeper review.
2. Entity memory enrichment — Aegis gets calibration data (which past
   findings koad confirmed vs dismissed), Vesta gets protocol decisions,
   every entity gets accumulated operational wisdom that currently
   evaporates when sessions end.

## Why

Sessions are the richest signal source in the kingdom. Every correction
koad makes, every architectural pivot, every "yes exactly" and "no not
that" lives in session transcripts — but dies there. Entities dispatch
with thin memory because nobody harvests the gold from past
conversations. Aegis specifically needs past-findings calibration to
avoid rediscovering the same drift patterns and to know which of her
assessments landed vs got dismissed.

The archive corpus is stale (39 files, mostly from April 10) because
no harvester runs. Claude Code stores sessions in
`~/.claude/projects/*/`, opencode in SQLite, and neither flows into the
canonical archive path the indexer watches.

## Where it lives

| Surface | Path |
|---------|------|
| Harvester script (target) | `~/.forge/commands/session/harvest/command.sh` |
| Scrubber (target) | `~/.forge/commands/session/scrub/command.sh` |
| Canonical archive | `~/.forge/archive/sessions/<YYYY-MM-DD>/<session-id>.jsonl` |
| Scrubber output | `~/.forge/archive/sessions/<YYYY-MM-DD>/<session-id>.summary.md` |
| Scrubber index | `~/.forge/archive/sessions/INDEX.jsonl` (append-only, one line per scrubbed session) |
| Daemon-interface UI (target) | `~/.forge/daemon-interface/src/client/pages/session-review/` |
| Scheduling | astro-project-templates (when available) |

## Source locations (what the harvester collects from)

| Harness | Storage path | Format | Notes |
|---------|-------------|--------|-------|
| Claude Code (current) | `~/.<entity>/projects/-home-koad--*/*.jsonl` | JSONL | Primary source since ~2026-04-25 |
| Claude Code (legacy) | `~/.claude/projects/-home-koad--*/*.jsonl` | JSONL | Pre-2026-04-25 sessions |
| Claude Code subagents | `~/.<entity>/projects/**/subagents/*.jsonl` | JSONL | Nested dispatches |
| opencode | `~/.local/share/opencode/opencode.db` | SQLite | Needs extraction (future) |
| Legacy workbench | `~/.forge/archive/legacy/**/*.jsonl` | JSONL | Already in archive |

Known entity project dirs: `~/.juno/projects/` (475), `~/.vesta/projects/` (41),
`~/.sibyl/projects/` (40), `~/.iris/projects/` (9), `~/.vulcan/projects/` (1).

## Scrubber output shape

Per session, produce a sidecar `.summary.md`:

```yaml
---
session_id: <id>
harness: claude-code | opencode
entity: juno | vulcan | aegis | ...
date: YYYY-MM-DD
turns: N
duration_estimate: Xm
status: productive | exploratory | blocked | corrective
---

## Key findings
- <finding 1>
- <finding 2>

## Accomplishments
- <what landed>

## Decisions
- <decision made, with rationale if stated>

## Corrections (koad → entity)
- <what koad pushed back on, what the correction was>

## Drift signals
- <any patterns worth watching>

## Valuable for review by
- <entity>: <why>
```

## Status (right now)

- ✅ Harvester built: `~/.forge/commands/session/harvest/command.sh`
- ✅ First harvest run: 2,574 sessions across 35 date dirs (2026-03-16 → 2026-05-06)
- ✅ Scrubber built: `~/.forge/commands/session/scrub/command.sh`
- ✅ Scrubber dispatches to fourty4 ollama (`qwen2.5-coder:7b` at `10.10.10.11:11434`) via mesh API
- ✅ Turn extractor: `~/.forge/commands/session/scrub/extract-turns.py` — extracts user/assistant turns, captions tool calls, skips noise
- ✅ Single-session scrub tested end-to-end (this very session, 145 turns, 22min, ~30s scrub time)
- ✅ Sidecar `.summary.md` format working: frontmatter + 7 standard sections
- ✅ Idempotent — re-runs skip already-scrubbed; `--force` to override
- ✅ Batch mode wired: `--batch [--date YYYY-MM-DD] [--entity NAME]`
- ✅ Session indexer watches the archive path and works when files appear
- ✅ Wired to entity launcher: `koad-io session harvest` and `koad-io session scrub`
- ⏸ Daemon-interface review UI not built (Vulcan dispatch)
- ⏸ opencode SQLite extraction not built

## Architecture — scrubber via fourty4 + koad:io harness

The scrubber is NOT a batch job that silently processes everything.
It's an **on-demand, click-through system**:

```
daemon-interface UI (browse sessions)
  → koad clicks a session (or selects batch)
  → UI fires scrub request to daemon
  → daemon dispatches to fourty4 via koad:io harness
  → fourty4 runs local LLM (ollama, ~8B model) against the transcript
  → structured summary returns to daemon
  → daemon stores sidecar .summary.md + updates Sessions collection
  → UI renders the summary inline, offers "dispatch entity for review"
```

**Why fourty4:** No API credits burned. Scrubbing 2,500 sessions at
even $0.01/session is $25 — fourty4 does it for electricity. The 8B
model is plenty for extracting findings/corrections from structured
JSONL transcripts. Heavier review (dispatching Vesta or Aegis back
into a specific conversation) uses the full harness on wonderland.

**Why koad:io harness:** The scrub job runs as a proper entity dispatch
(likely `juno session scrub <session-id>`), meaning it gets cascade
env, emission tracking, flight lifecycle. The work is visible in the
kingdom's nervous system, not hidden in a cron log.

**UI flow in daemon-interface:**
1. Session list page — filterable by entity, date, status (scrubbed/unscrubbed)
2. Click a session → metadata card (turns, entity, date, size)
3. "Scrub" button → fires dispatch to fourty4, progress shown via emission updates
4. Once scrubbed → summary rendered inline (findings, accomplishments, corrections)
5. "Review with..." dropdown → dispatch Aegis/Vesta/any entity into that session for deeper analysis
6. Batch scrub option for "scrub all unscrubbed from this week"

**Dispatch-for-review pattern:** When koad clicks "Review with Vesta"
on a session, the system:
- Reads the transcript + existing summary
- Dispatches the chosen entity via the harness with the transcript as context
- The entity produces a structured review (protocol implications for Vesta, calibration data for Aegis, etc.)
- Review stored as a second sidecar: `<session-id>.review.<entity>.md`

## What's next

1. ~~Harvester command~~ ✅ DONE — `koad-io session harvest`
2. **Scrubber command** — `koad-io session scrub [session-id|--batch]`
   dispatches to fourty4 via harness. Produces sidecar summaries.
   Needs: ollama prompt template for extraction, SSH dispatch to
   fourty4, result collection back to wonderland.
3. **Daemon-interface session-review page** — Vulcan builds the UI
   surface. Reads from Sessions collection (already indexed), renders
   scrubbed/unscrubbed state, fire-button for on-demand scrub.
4. **Schedule** — harvester runs daily via astro-project-templates.
   Scrubber stays on-demand (UI-triggered) with optional batch mode
   for backfill.
5. **Entity review dispatch** — "Review with..." button dispatches an
   entity to read a specific session transcript and produce calibration
   output for their own memory.
6. **Entity calibration feed** — Aegis, Vesta, and others can query
   scrubber output for their own calibration data. "Show me every
   correction koad made to Aegis assessments" becomes a structured
   query against the sidecar files.

## References

- `~/.koad-io/me/projects/session-indexer.md` (the indexer this feeds)
- `~/.juno/briefs/2026-05-05-sessions-indexer-and-atlas-extension.md`
- `~/.juno/projects/-home-koad--juno/memory/project_opencode_cost_capture.md` (opencode db schema)
- `~/.forge/control-tower/src/server/indexers/sessions.js` (the indexer that consumes the archive)
- fourty4 — Mac Mini M2 8GB, ollama, ~8B models (scrub inference target)
- `~/.forge/daemon-interface/` (UI host for session-review page)
