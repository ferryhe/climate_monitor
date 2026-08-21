# Modular Canonical Pipeline v2 planning

This directory is the decision source for the Climate Monitor v2 architecture and migration. It describes the intended system; implementation details must conform to the versioned contracts created during the roadmap.

## Documents

- [`architecture-v2.md`](architecture-v2.md) defines the target architecture, canonical artifact model, module ownership, and integration rules.
- [`migration-plan-v2.md`](migration-plan-v2.md) defines sequencing, dependencies, acceptance gates, dual-run, cutover, rollback, and Phase 2.

## Decision hierarchy

When implementation choices conflict, use this order:

1. The versioned `report_contract` and `artifact_protocol` packages once implemented.
2. Accepted architecture decisions in this directory.
3. Module-local implementation details.
4. Legacy behavior from `ferryhe/climate_monitor_wiki`, which is reference material only.

Legacy behavior cannot override a v2 contract. Any intentional contract change requires a reviewed contract version change, conformance fixtures, consumer compatibility evidence, and a migration note.

## Current status

The repository is in architecture and migration planning. No document in this directory authorizes a production cutover or destructive data migration. Those actions belong to the final deployment issue and require explicit approval.
