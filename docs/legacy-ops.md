# Legacy Operations Reference (archived)

> Archived from `README.md` on `origin/main` (pre-v2 rewrite). Kept so operational
> knowledge is not lost when the v2 planning README replaces the old one.
> This is **not** the target design — see `docs/planning/` for that.

## What the system exposes (as of main)

Three web tabs plus an Obsidian plugin:

- **Historical Reports** — default operator archive for weekly narrative briefings,
  monitoring snapshots, PDFs, and their source articles.
- **Chat** — minimal single-column conversation layout (inspired by
  `ferryhe/c-ross-2`), recolored to match the Obsidian workspace.
- **Obsidian** — browsing workspace with `Dataview`, `Note Detail`, and `Graph View`.
  - Page order: `Dataview + Note Detail` first, then `Graph View`.
  - Graph supports `Notes` and `Keywords` modes (file links vs. source-backed
    concept map). Both modes are **precomputed by the API** so the client renders
    without rebuilding the graph.
- `.obsidian/plugins/climate-agent-chat/` — Obsidian side-panel chat plugin that
  calls the same local API.

The active note chosen in the web Obsidian tab or the Obsidian plugin is sent as
`contextPath`, so retrieval can prioritize the current page during chat.

## Deployment / run state (as of main, commit cf19da8)

- Production Registry and Article Detail behavior reported healthy.
- Legacy Publisher record repaired and validates with its formal identity.
- Two controlled captures observed 21 successes and 4 deterministic publisher-wall
  403 failures; candidates were **not** promoted and the live DB remained
  byte-for-byte unchanged.
- Remaining gate: deploy validated fallback coverage, run a controlled exact sync,
  and create the (disabled) 10:30 scheduled job.

> Confirm the deployed commit in the controlled deployment runbook; do not infer it
> from this document.

## Known components mentioned in prior docs (inventory seed)

Used as input to `docs/planning/current-to-v2-mapping.md`:

| Component (legacy)            | Mentioned in                                  |
|-------------------------------|-----------------------------------------------|
| `climate_registry` (SQLite)   | README: "DB-first", weekly sync               |
| `sources/` directory          | README: "append-mostly source of truth"       |
| `agentic_wiki`                | project map (referenced)                       |
| `api_server.py`               | API/CLI audit (referenced)                     |
| `showcase`                    | module map (referenced)                        |
| `scripts/*.py`                | scheduled-job boundaries (referenced)          |
| Obsidian plugin               | web surface                                    |
| Obsidian graph precompute     | web surface (Notes/Keywords modes, API-computed) |
| existing pytest suite         | API/CLI audit (referenced)                     |

See `docs/project-closeout.md` (not tracked in this repo) for the full operator
guide, module map, API/CLI audit, scheduled-job boundaries, and closeout record.
If that file still exists in the legacy repo, migrate the relevant sections into
`docs/planning/` before retiring this archive.
