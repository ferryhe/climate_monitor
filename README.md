# Climate Monitor

Climate Monitor is being redesigned as a modular monorepo whose components exchange one versioned, canonical report bundle. The repository is currently in the planning and migration stage; the v2 boundaries and delivery sequence are defined in [`docs/planning/`](docs/planning/README.md).

## Canonical architecture

The v2 pipeline has one business source of truth: `bundle.json`.

```text
Hermes (schedule/runtime/secrets/logs only)
    |
    v
Monitor + WebListening
collect -> normalize -> relevance -> dedupe -> final selection
        -> deterministic statistics -> one authoring pass
    |
    v
canonical bundle + content-addressed evidence + run receipt
    |
    +--> Delivery
    +--> Wiki
    +--> RAG/Chat
    +--> Web/Storage API and UI
    +--> Canonical Keyword Graphify
```

The core rules are:

- `bundle.json` is the sole business truth shared between modules.
- `report.md` is a deterministic projection of the bundle, never an independently authored source.
- Evidence is immutable and content-addressed.
- `run-receipt.json` records the code, configuration, prompt, taxonomy, inputs, and output digests needed to reproduce and audit a run.
- Monitor owns acquisition through canonical bundle production. It calls WebListening, performs deterministic selection and statistics, and uses exactly one authoring pass for authored report fields.
- Delivery, Wiki, RAG/Chat, Web/API/UI, and Canonical Keyword Graphify consume the canonical bundle. They do not reconstruct business truth from legacy files.
- Hermes is only the runtime and scheduler. It does not own prompts, taxonomy, relevance rules, deduplication, schemas, rendering, storage, or publishing behavior.
- Wiki remains an independent whole and preserves its current internal structure while changing its input boundary to the bundle.
- Canonical Keyword Graphify is a standalone projection of canonical article keywords. It does not read or modify the Wiki.

## Planned monorepo layout

```text
packages/
  report_contract/       # Versioned bundle schema, types, validation, fixtures
  artifact_protocol/     # Immutable layout, digests, evidence, run receipts
modules/
  monitor/               # WebListening -> canonical bundle producer
  delivery/              # Bundle-only email/document delivery
  wiki/                  # Independent wiki projection and UI
  rag_chat/               # Retrieval, citations, and chat
  web_storage/            # Artifact store, read model, API, and web UI
adapters/
  hermes/                 # Runtime/scheduler integration only
  publisher/              # Controlled promotion/publication integration
tests/
  conformance/            # Contract and cross-module compatibility suites
  e2e/                    # Dual-run, cutover, rollback, and scheduled-run tests
```

## Planning documents

- [Architecture](docs/planning/architecture-v2.md) defines module ownership, contracts, artifact flow, and prohibited coupling.
- [Migration plan](docs/planning/migration-plan-v2.md) defines the dependency order, minimum vertical slice, dual-run, cutover, rollback, and Phase 2 work.
- [Planning index](docs/planning/README.md) records the status and decision hierarchy.

## Migration policy

The old [`ferryhe/climate_monitor_wiki`](https://github.com/ferryhe/climate_monitor_wiki) project is provenance and migration reference only. It may supply fixtures, historical artifacts, behavioral comparisons, and structure-preservation evidence. Post-cutover v2 modules must not introduce semantic fallbacks that parse old report, metadata, registry, or Wiki representations as alternate business truth.

All implementation and migration work follows GitHub branch, pull-request, automated-check, and review workflows. Production cutover, destructive migration, rollback execution, and merge remain explicit human-approved actions.
