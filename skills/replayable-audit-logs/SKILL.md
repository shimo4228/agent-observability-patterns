---
name: replayable-audit-logs
description: Design pattern for observability-by-default in LLM agents — every feature that performs external I/O, calls an LLM, or makes non-deterministic/heuristic decisions ships a replayable append-only JSONL audit log in the same change. Use when adding or reviewing such a feature (the review question "which log answers why, and can we replay it offline?"), when designing a new audit record schema, when a recurring failure needs corpus-driven repair (replay harness, positive/negative ground truth, regression fixtures from real traffic), or when deciding how to store untrusted text in a log.
compatibility: Developed and tested on Claude Code; portable to other Agent Skills-compatible agents. Examples are abstract worked examples, not tied to any harness.
user-invocable: true
origin: shimo4228
---

# Replayable Audit Logs (observability by default)

> Sibling pattern: an **instrument** reads across the whole store at query time
> (distributions, calibration) — see
> [`read-only-instruments`](../read-only-instruments/SKILL.md). The log is the
> corpus; the instrument is one lens over it. A third sibling,
> [`shadow-mode-validation`](../shadow-mode-validation/SKILL.md), uses a log of
> this kind as the record of a candidate mechanism's would-be decisions.

The pattern sits in the lineage of append-only logs and event sourcing: the
record of what happened is written as an immutable sequence at the moment it
happens, and every later analysis is a read over that sequence.

Principle: **a feature with external I/O, LLM calls, or heuristic decisions
ships its audit log in the same change** — because the corpus must predate the
failure it will one day explain. Worked example: an agent that submits
generated answers to an external verifier logged every attempt as an ordinary
side effect for weeks; when its parser needed repair, hundreds of real logged
challenges could be replayed offline against a zero-wrong hard gate, and the
repair was validated against real traffic instead of synthetic cases. Ad-hoc
logging added at investigation time can never provide that.

## Record schema checklist

Design the record so the run can be **replayed offline**, not merely read:

- [ ] **Raw input, recoverable** — the exact input the decision saw. Untrusted
      text (API responses, user/content text) as **base64 + sha256**, never free
      text: a raw log read must not become a prompt-injection path (base64 is
      encoding, not sanitization — anything that decodes the payload must treat
      it as untrusted again). Bound the stored size with a `truncated` flag when
      cut; a truncated record is thereby marked non-replayable rather than
      silently partial, so the replay gate can excuse it instead of trusting it.
- [ ] **Decision path** — which branch/tier handled it (e.g. `solver_path:
      code_parse | llm_extract | llm_reason | none`).
- [ ] **Reason codes, categorical** — every abstain / fallback / failure gets a
      machine-groupable code (`abstain_reason: reasoning_self_inconsistent`),
      not prose. **A silent fallback is a defect**, not a style choice.
- [ ] **Outcome** — what happened downstream (accepted / rejected / error),
      plus a sanitized error message (stripped to printable characters,
      length-capped).
- [ ] **Timestamps + stable keys** — `ts` (ISO, UTC) and a content hash
      (sha256) so records dedupe and join across retries.

Writer: append-only JSONL under the agent's data directory, written with
restricted file permissions, best-effort — the feature must not fail because
logging failed.

## Ground truth discipline

A log becomes a labeled corpus when outcomes are recorded honestly. This
discipline presupposes that the external system's accept/reject is an
**authoritative verdict** for that exact input — a flaky verifier, a
policy-driven rejection, or a stale catalog does not pin truth, and outcomes
from such sources belong in the manual-label lane below, not the automatic
ones:

- **Positive truth** — an externally *accepted* result pins the correct answer
  for that exact input (keyed by input hash).
- **Negative truth** — an externally *rejected* result is durably wrong for
  that input, with no manual labeling. Rejections are data; log them with the
  same fidelity as successes.
- **Manual labels** — for inputs the outside world never confirmed, keep a
  hand-labeled file next to the replay script (`manual_labels.json`), each
  label with provenance (hand-solved / twin-confirmed against an accepted
  same-shape input). A **null answer** marks known-unresolvable cases
  (external inconsistency) so the gate can excuse rather than ignore them.

## Replay harness pattern

1. Pure-code script (no LLM, no network) that re-runs the deterministic layer
   over every unique logged input.
2. **Hard gate: zero wrong vs known truth** (positive + negative). Coverage is
   a soft metric — report it, don't gate on it.
3. Unlabeled parses are printed for labeling; the gate fails until they are
   labeled or the behavior abstains.
4. Exit code 0 only on gate PASS, so the harness can sit in a verification
   chain.

## Corpus-driven repair loop

When a feature "keeps failing at the same rate":

1. **Aggregate before hypothesizing** — success/failure per decision path over
   the log; the failing tier is rarely the one you suspect.
2. **Decode and classify** every failure into named classes.
3. **Twin-confirm intended fixes** — before changing a rule, find a *twin*:
   an already-accepted logged record whose input has the same structural
   shape as the failing one, proving how the external system actually treats
   that shape. A fix without a twin is a guess about external semantics.
4. Fix behind the replay hard gate (no lost previously-correct cases), and pin
   the failure round as regression fixtures cut from real traffic (base64 in
   tests, with provenance comments).
5. Record the round as a decision-record amendment; leave irreducible cases
   explicitly labeled (the floor is part of the finding).

## The review-gate question

At the verification step of any change in scope:

> When this feature misbehaves, which log will explain why?
> Can the behavior be reproduced (replayed) offline from that log?

No answer → back to design, in the same change. Instruments worth imitating
in any agent: a verification audit log (per-attempt decisions of a solver), an
API-drift log (request/response shapes over time), an approval-gate audit
trail (accept/reject with reasons), and LLM telemetry with caller tags (so a
retune can be justified by measured failure-reason rates).
