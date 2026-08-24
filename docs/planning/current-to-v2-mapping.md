# Current State → v2 Mapping (gap analysis)

> Companion to `architecture-v2.md` and `migration-plan-v2.md`.
> Purpose: make the migration **estimable** by mapping every known legacy component
> to its v2 disposition, and by surfacing the open questions the planning docs defer.
> Status: **draft for review** — dispositions marked `[PROPOSED]` are not yet decided.

## 1. Legacy inventory (seed from `docs/legacy-ops.md` + prior docs)

| # | Legacy component            | Role today                                  | Source of info        |
|---|-----------------------------|---------------------------------------------|-----------------------|
| L1 | `climate_registry` (SQLite) | DB-first metadata store, weekly sync        | legacy README         |
| L2 | `sources/` directory        | Append-mostly source of truth for articles  | legacy README         |
| L3 | `agentic_wiki`              | (referenced) wiki build/agent               | project map           |
| L4 | `api_server.py`             | Serves web + Obsidian + chat retrieval      | API/CLI audit         |
| L5 | `showcase`                  | (referenced)                                | module map            |
| L6 | `scripts/*.py`              | Scheduled-job runners (incl. 10:30 job)     | scheduled-job bounds  |
| L7 | Obsidian plugin             | Side-panel chat, `contextPath` passthrough  | web surface           |
| L8 | existing pytest suite       | API/CLI audit coverage                      | API/CLI audit         |
| L9 | Obsidian graph precompute   | Notes + Keywords modes, API-computed        | web surface           |

> TODO (owner: maintainer): expand L1–L9 with the full list from
> `docs/project-closeout.md` (legacy repo) before this document is approved.

## 2. Disposition mapping

Disposition legend: **M**igrate · **P**roject (re-implement in v2 shape) ·
**A**rchive · **R**etire.

| # | Component        | v2 disposition | Maps to v2 module            | Notes |
|---|------------------|----------------|------------------------------|-------|
| L1 | `climate_registry` | **A** then **R** | n/a (replaced by `bundle.json`) | Must NOT be a source of truth in v2 (arch §2 Non-goals: "Registry rows ... as alternate sources of report truth"). Archive the SQLite, retire the writer. Open: does any consumer still read it directly? |
| L2 | `sources/`        | **R**          | replaced by `bundle.json`    | Append-only history can be frozen as v1 bundle snapshots. |
| L3 | `agentic_wiki`    | **P** `[PROPOSED]` | Monitor pipeline (builder) | Re-express as a pipeline stage producing bundle content. |
| L4 | `api_server.py`   | **P** `[PROPOSED]` | `modules/web_storage` (read API) + `report.md` projection | Read paths become Storage read APIs (§3 #8); `report.md` projection serves Web/API/UI. |
| L5 | `showcase`        | **P** `[PROPOSED]` | `report.md` projection (Web surface) | Web surface is a projection of `report.md` (arch §5 Canonical bundle → `report.md`). |
| L6 | `scripts/*.py`    | **P** `[PROPOSED]` | Hermes adapter (scheduler) | Scheduling logic moves to `adapters/hermes`; business steps become Monitor pipeline stages. |
| L7 | Obsidian plugin   | **P** `[PROPOSED]` | projection consumer       | Stays a thin client over the read API; `contextPath` passthrough unchanged. |
| L8 | pytest suite      | **M** `[PROPOSED]` | v2 conformance + E2E (#11) | Reuse as the byte-identical `report.md` projection conformance tests (arch §5 `report.md` + §11 Required conformance properties). |
| L9 | graph precompute  | **P** `[PROPOSED]` | Canonical Keyword Graphify projection (#9) | Notes/Keywords modes become projections over the canonical bundle. |

## 3. Open questions the planning docs defer (and must answer before cutover)

1. **Registry read paths** — who currently reads `climate_registry` directly, and
   what do they fall back to during dual-run? (arch §2 Non-goals forbids Registry rows as truth, but says
   nothing about the read API shim.)
2. **Code migration path** — copy from legacy repo `ferryhe/climate_monitor_wiki`
   into this repo, or rewrite? The planning docs imply a fresh repo; confirm.
3. **Evidence of pain** — the docs cite "hidden ownership" and "semantic drift" but
   give no concrete incident. Capture 1–2 real examples to justify the governance
   overhead (see §5).
4. **Bundle size / frequency** — weekly cadence is stated; bundle size, storage
   growth, and retention are unspecified. Needed to size Storage core (#3/#5).
5. **Schema v1 longevity** — arch §9 Compatibility and §10 Versioning define the v1→v2 policy; roadmap #2 Contract starts at v1.
   Confirm v1 is expected to live long enough that versioning machinery earns its keep.

## 4. Note on the dependency-graph numbering (no correction needed)

The §2 mermaid graph and the §3 roadmap in `migration-plan-v2.md` are **already
consistent** — the earlier claim of a "+2 offset" was a misreading and is
withdrawn. The graph numbers implementation prerequisites starting at #4 (the
Contract node); the roadmap numbers workstreams #1–#12. The two share the same
labels. Reference table (authoritative, from `migration-plan-v2.md`):

| # | §3 Roadmap workstream                         | §2 Graph node (if shown)        | v2 module / owner            |
|---|-----------------------------------------------|---------------------------------|------------------------------|
| 1 | Epic: Modular Canonical Pipeline v2          | (tracker only)                  | —                            |
| 2 | Contract                                      | #4 Contract                     | `packages/report_contract`   |
| 3 | Storage                                       | #5 Storage core                 | `modules/web_storage`        |
| 4 | Monitor/WebListening                          | #6 Monitor / WebListening       | Monitor pipeline             |
| 5 | Delivery                                      | #7 Delivery                     | Delivery projector           |
| 6 | Wiki                                          | #8 Wiki                         | Wiki importer/projector      |
| 7 | RAG/Chat                                      | #9 RAG / Chat                   | RAG indexing + chat          |
| 8 | Web/API/UI                                    | #10 Web / API / UI              | `modules/web_storage`        |
| 9 | Standalone Canonical Keyword Graphify        | #11 Canonical Keyword Graphify  | Graphify projection          |
|10 | Publisher adapter                             | #12 Publisher                   | Publisher adapter            |
|11 | E2E / dual-run / cutover / rollback / observe| #13 E2E / cutover / observation | Hermes adapter + gates       |
|12 | Phase 2: Admin Prompt/Taxonomy UI            | #14 Admin UI                    | Admin UI (post-cutover)      |

`Hermes` is the **runtime/scheduler** (arch §8 Adapter boundaries → Hermes, adapter
`adapters/hermes`), not a numbered workstream — it integrates at #11, not as a
standalone build item. The canonical artifact is `bundle.json` (arch §5 Canonical
bundle); `report.md` is its deterministic projection.

## 5. Design-tradeoff note (addresses "over-engineering" concern)

For a single-maintainer, weekly-cadence system, the full v2 governance is heavy.
Recommended **minimum viable** cut that still delivers the core win (single source
of truth + deterministic projection):

- **Keep:** `bundle.json` canonical + `report.md` projection + conformance test
  (byte-identical). This alone kills semantic drift.
- **Defer / trim:**
  - Contract schema v1→v2 matrix → ship v1 only; add versioning only when a second
    consumer needs it.
  - Dual-run full comparison → compare only `report.md` checksum + key counts.
  - Rollback drill + approval gates → keep pointer-only rollback, drop the formal
    drill until a second operator exists.
  - Epic lifecycle overhead → a single tracking issue is enough at this scale.

If the maintainer expects multi-operator or external contributors soon, keep the
full machinery as written.
