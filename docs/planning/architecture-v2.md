# Climate Monitor modular canonical pipeline v2

## 1. Purpose

Climate Monitor v2 separates monitoring, delivery, knowledge publishing, retrieval, and web access into modules that communicate through immutable, versioned artifacts. The design removes hidden ownership from the scheduler and prevents downstream systems from independently reconstructing report meaning.

The central architectural decision is:

> `bundle.json` is the sole business truth for a completed report run.

Every human or machine-facing representation is either part of that bundle or a deterministic projection whose input digest is recorded.

## 2. Goals and non-goals

### Goals

- Make every report attributable to exact code, configuration, prompt, taxonomy, inputs, evidence, and model execution metadata.
- Give each module a narrow ownership boundary and a versioned input/output contract.
- Make replay, validation, dual-run comparison, rollback, and audit normal operations.
- Preserve the Wiki as an independent product while changing its upstream interface to the bundle.
- Support immutable storage and multiple read-only consumers without semantic drift.
- Keep deployment and scheduling concerns outside business logic.

### Non-goals

- Reimplementing the old system's accidental coupling.
- Treating Markdown, email, PDF, Registry rows, Wiki files, or UI state as alternate sources of report truth.
- Moving prompts, taxonomy, selection rules, or rendering logic into Hermes.
- Combining RAG/Chat, Wiki, Delivery, or keyword graph generation into Monitor.
- Performing production cutover as part of core module implementation.

## 3. Repository boundaries

```text
packages/
  report_contract/
  artifact_protocol/
modules/
  monitor/
  delivery/
  wiki/
  rag_chat/
  web_storage/
adapters/
  hermes/
  publisher/
tests/
  conformance/
  e2e/
```

| Component | Owns | Must not own |
| --- | --- | --- |
| `packages/report_contract` | Versioned bundle schema, semantic invariants, canonical serialization, validation, types, fixtures, compatibility policy | Storage, scheduling, model calls, delivery, UI |
| `packages/artifact_protocol` | Artifact paths, digest rules, content-addressed evidence, atomic commit protocol, run-receipt schema, immutable manifest rules | Report semantics or consumer behavior |
| `modules/monitor` | WebListening integration, collection, normalization, relevance, dedupe, final selection, deterministic statistics, one authoring pass, bundle validation | Scheduling, delivery, Wiki rendering, RAG, publication |
| `modules/delivery` | Bundle-driven message/document rendering, recipient policy, send receipts, idempotent delivery | Monitoring or report reinterpretation |
| `modules/wiki` | Existing Wiki structure, navigation, projections, Wiki-specific validation and publication preparation | Canonical monitoring rules, RAG internals, keyword graph mutation |
| `modules/rag_chat` | Bundle/evidence indexing, retrieval, answer generation, citations, chat-facing policy | Report production, Wiki ownership, canonical mutation |
| `modules/web_storage` | Immutable artifact persistence, indexes/read models, API, access control, web UI | Canonical report authoring or legacy semantic fallback |
| `adapters/hermes` | Process invocation, schedule integration, secrets injection, time/resource limits, runtime logs, exit status | Prompts, taxonomy, report logic, schemas, projections, storage semantics |
| `adapters/publisher` | Controlled promotion between validated artifact states and external publication targets | Editing bundle meaning, invoking hidden generation, bypassing approval gates |
| `tests/conformance` | Producer/consumer contract fixtures and compatibility suites | Module implementation |
| `tests/e2e` | Full runs, dual-run comparisons, deployment/cutover/rollback drills, scheduled observation | Business logic |

## 4. Canonical run flow

Monitor owns one explicit pipeline:

1. **Collect** through a versioned WebListening port. Record the query, source adapter version, retrieval window, and raw response references.
2. **Normalize** URLs, timestamps, publishers, titles, and evidence references using deterministic rules.
3. **Evaluate relevance** using repository-owned prompt/configuration and record decision metadata.
4. **Deduplicate** with a versioned deterministic identity algorithm. Keep resolution records rather than silently dropping conflicts.
5. **Select final articles** under explicit policy. The selected set is frozen before authoring.
6. **Compute statistics deterministically** from the frozen selected set.
7. **Run one authoring pass** to produce only authored fields such as executive synthesis and canonical per-article summary/category/keyword fields. The pass receives the frozen facts and statistics; it does not recollect or reselect.
8. **Assemble and validate `bundle.json`** against the report contract and semantic invariants.
9. **Project `report.md` deterministically** from the validated bundle.
10. **Commit artifacts atomically** with content-addressed evidence and `run-receipt.json`.

Failures before atomic commit produce diagnostics and an unsuccessful receipt or runtime record, but never a partially published canonical report.

## 5. Canonical bundle

The concrete schema belongs to `packages/report_contract`. At minimum, the contract must represent:

- contract name and semantic version;
- stable report/run identity and reporting window;
- generation timestamp and locale/time-zone policy;
- monitoring configuration, prompt, taxonomy, and selection-policy references by digest;
- normalized candidate and final-selection records with stable article identities;
- canonical article fields, including source metadata, evidence digests, summary, categories, and keywords;
- deterministic aggregate statistics and the input fields from which they were computed;
- authored report sections from the single authoring pass;
- warnings, exclusions, and structured decision reasons;
- provenance links to the corresponding run receipt and artifact manifest.

Canonical serialization must be deterministic: stable field ordering, normalized timestamps and Unicode, explicit null/omission rules, and no environment-dependent values. The bundle digest is calculated only after validation.

### `report.md`

`report.md` is rendered from `bundle.json` by a versioned deterministic projector. It cannot contain independently generated facts or prose. A conformance test must prove that rerendering the same bundle produces byte-identical output.

### Evidence

Evidence blobs are addressed by a cryptographic content digest and stored immutably. Bundle records reference the digest plus media type, capture metadata, original locator, and any extraction metadata. A repeated blob is stored once; a changed blob gets a new address.

### `run-receipt.json`

The receipt is provenance, not business truth. It records at least:

- run ID, start/end time, outcome, and environment class;
- repository URL and exact commit;
- contract and artifact-protocol versions;
- dependency/runtime identity;
- WebListening adapter and request identifiers;
- prompt, taxonomy, configuration, and policy digests;
- model/provider identifiers and reproducibility parameters where available;
- input, evidence, bundle, projection, and manifest digests;
- validation results, warnings, and failure stage;
- Hermes invocation/job identity when applicable.

Secrets, full credentials, and unsafe private model traces are never stored in the receipt.

## 6. Storage and read model

The artifact protocol defines an immutable logical layout. An implementation may map it to a filesystem, object store, or database-backed catalog without changing the contract.

```text
runs/<run-id>/run-receipt.json
reports/<report-id>/<bundle-digest>/bundle.json
reports/<report-id>/<bundle-digest>/report.md
evidence/<algorithm>/<digest>
manifests/<manifest-digest>.json
```

Mutable concepts such as `latest` are read-model pointers, never mutable canonical artifacts. Publishing updates a pointer or catalog entry only after all referenced immutable objects validate and exist.

The API and UI query the read model and return canonical artifact references. They may cache or index data, but caches are disposable and cannot become an alternate source of truth.

## 7. Consumer rules

### Delivery

Delivery accepts a validated bundle reference and produces deterministic message/PDF projections plus delivery receipts. It must not read Monitor work directories, query WebListening, or infer missing report fields from old Markdown.

### Wiki

Wiki remains an independent whole and preserves its current internal structure, navigation, and local validation where useful. A bundle importer/projector is its only v2 monitoring input. Wiki output is a publication/read surface, not canonical report state.

### RAG/Chat

RAG/Chat is a separate module. It indexes versioned bundles and referenced evidence, records index provenance, and returns citations to canonical article/evidence identities. It can use Wiki content as an explicitly identified secondary corpus, but Wiki cannot silently replace canonical bundle fields.

### Web/API/UI

Web/API/UI belongs with immutable storage and read models in the same `modules/web_storage` module. Persistence/read-model work and API/UI work may land as separate implementation increments, but they must share one artifact identity resolver and must not become sibling storage and web services. The module exposes report history, report detail, evidence, receipts, and module status without mutating canonical artifacts.

### Canonical Keyword Graphify

Canonical Keyword Graphify is a standalone read-only projection. Its nodes and edges derive only from canonical article keyword fields and stable article identities in bundles. It neither reads the Wiki as an input nor writes to the Wiki. Rebuilding the same set of bundle versions must produce the same graph.

## 8. Adapter boundaries

### Hermes

Hermes performs operational orchestration only:

- schedule a deployed commit;
- provide the runtime, credentials, resource limits, and timeout;
- invoke a repository-owned adapter/CLI;
- retain runtime logs and job outcome;
- expose an operational job identity to the run receipt.

Hermes must be deliberately boring. A typical job checks out an approved deployed commit and invokes `adapters/hermes/run_weekly_monitor`; all prompts, taxonomy, policy, schemas, and pipeline steps remain in Git.

### Publisher

The publisher adapter promotes already validated immutable artifacts to explicitly selected publication targets. It depends only on storage and the interfaces of those selected targets; RAG/Chat and unrelated consumers are not implicit prerequisites. It verifies digests and preconditions, is idempotent, records publication receipts, and never regenerates or edits the bundle. Human-review or environment-approval gates remain outside the adapter's semantic behavior.

## 9. Compatibility and provenance policy

The old `ferryhe/climate_monitor_wiki` project is a migration input only. Allowed uses include:

- inventorying existing structures and behavior;
- creating fixed historical fixtures;
- comparing v1 and v2 outputs during dual-run;
- validating that the independent Wiki preserves agreed presentation and navigation behavior;
- documenting the source and transformation of migrated historical artifacts.

After cutover, v2 code must not parse old report, Registry, annotation, manifest, or Wiki formats as a semantic fallback. Historical content is migrated once into an explicitly versioned canonical representation or remains in a clearly labeled legacy archive.

## 10. Versioning and change control

- Contract changes use semantic versioning and compatibility tests.
- Producers write one declared contract version; consumers declare the versions they accept.
- Breaking changes require new fixtures, migration tooling, a dual-read period only when explicitly designed, and a reviewed removal date.
- Prompt, taxonomy, policy, and deterministic algorithm changes are repository changes reviewed through pull requests and identified by digest in receipts.
- No merge, production deployment, destructive migration, or cutover occurs without explicit human approval.

## 11. Required conformance properties

The conformance suite must eventually verify:

- valid/invalid bundle fixtures and semantic invariants;
- canonical serialization and stable digest vectors;
- article identity and dedupe vectors;
- byte-identical `report.md` projection;
- content-addressed evidence verification;
- receipt-to-artifact digest closure;
- atomic commit and idempotent replay;
- every consumer operates from bundle fixtures with Monitor and legacy files absent;
- Keyword Graphify reads only canonical keywords and never changes Wiki files;
- Hermes adapter contains no business configuration or authoring logic.
