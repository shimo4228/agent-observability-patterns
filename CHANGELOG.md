# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-07-11

Initial release as a standalone skill repository.

### Added

- `skills/replayable-audit-logs/SKILL.md` — observability by default: every feature with external I/O, LLM calls, or heuristic decisions ships an append-only, offline-replayable JSONL audit log in the same change; record schema checklist (base64 + sha256 untrusted text, categorical reason codes, decision paths), ground-truth discipline, zero-wrong replay-gate harness, corpus-driven repair loop.
- `skills/read-only-instruments/SKILL.md` — aggregate readings over stored state that inform the operator before an intervention: instrument-first sequencing, the signal-first test for building and removing instruments, five invariants (observability never intervention, ambiguity notes, degrade never abort, bounded cost, byte-identical production), three-point calibration scale, reading pitfalls.
- `skills/shadow-mode-validation/SKILL.md` — a candidate decision mechanism runs observe-only beside the live path, in the shadow deployment / dark launch lineage: observe-only entry points guaranteed by type, isolation from shared failure machinery, hallucination-rate accounting, would-be metrics baked at record time, kill-switch by configuration absence, exit criteria reserved at launch.

### Notes

- The paired decision records [AAP ADR-0006 Causal Traceability and ADR-0005 Human Approval Gate](https://github.com/shimo4228/agent-attribution-practice/tree/main/docs/adr) stay in the [Agent Attribution Practice](https://github.com/shimo4228/agent-attribution-practice) research repository; this repository is their installable "how" counterpart.
- This repository is manually curated: it generalizes an operational variant that runs inside the [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) project. There is intentionally no automated sync script.
