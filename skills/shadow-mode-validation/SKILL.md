---
name: shadow-mode-validation
description: Design pattern for shadow-mode validation — running a candidate decision mechanism (typically an LLM judgment) in observe-only parallel with the live path, recording what it WOULD have decided per event, and letting the accumulated record decide enforcement. Use when an unvalidated stochastic mechanism is about to replace or filter a live behavior (a one-way door for output quality), when a selector/classifier/gate has no published reliability evidence for the model class in play, or when designing the isolation, kill-switch, and exit criteria for a shadow deployment. NOT for aggregate readings over stored state (that is read-only-instruments), NOT for the audit-log record schema itself (that is replayable-audit-logs — a shadow log IS one of those logs), and NOT a substitute for unit tests — shadow mode validates decision quality in production traffic, not code correctness.
compatibility: Developed and tested on Claude Code; portable to other Agent Skills-compatible agents. Examples are abstract worked examples, not tied to any harness.
user-invocable: true
origin: shimo4228
---

# Shadow-Mode Validation

**Shadow mode** runs a candidate decision mechanism alongside the live path,
records its would-be decision per event, and changes nothing. The name and
the stance come from the established deployment lineage — *shadow
deployment* / *shadow testing* / *dark launch* in ML and service operations,
where a new system receives production traffic whose outputs are recorded
but never served. This skill applies that lineage to in-agent decision
mechanisms: the live behavior stays exactly as it was; the only output is an
append-only record of "here is what the new mechanism would have done."
Enforcement — actually letting the mechanism steer behavior — is a separate,
later decision made from the accumulated record.

Worked example: an agent adds a pass-1 LLM judgment that selects which of
its skills apply to the current situation. Instead of wiring the selection
into the prompt on day one, the selector runs shadow-only for 2–4 weeks,
recording per event what it would have injected; the enforcement decision is
then made from hallucination rates, fail-open rates, and realized-reduction
distributions in the record.

**How it composes with the sibling skills.** The shadow record is a
replayable audit log ([`replayable-audit-logs`](../replayable-audit-logs/SKILL.md)
owns the schema: b64+sha256 untrusted text, verdict reason codes, no silent
fallbacks). The go/no-go reading over that log is an instrument
([`read-only-instruments`](../read-only-instruments/SKILL.md) owns
instrument-first sequencing and signal-first). What THIS skill owns is the
middle layer: the discipline of running a live-path mechanism in
observe-only mode without perturbing what it observes.

## When to reach for shadow mode

- **The mechanism's decision quality is the unknown, not its code.** Unit
  tests prove parsing and wiring; they cannot prove that a small local model
  selects the right skills, or that a heuristic gate fires on the right
  events. When the failure mode is "plausible but wrong judgment," only real
  traffic answers.
- **Enforcement is a one-way door for behavior.** If wiring the mechanism in
  changes what the model reads or publishes, a bad mechanism degrades output
  *silently* — and with the old path gone, there is no baseline left to
  notice the degradation against. Shadow-first preserves the baseline while
  the evidence accumulates.
- **A prior attempt died wired-but-unobserved.** The anti-pattern this
  inverts: a mechanism shipped straight into the live path with no usable
  failure signal, which was eventually sunset without ever having informed a
  decision. Shadow mode is the exact inversion: observability ships first
  and alone; the mechanism earns its wiring.

## Design elements (each earned the hard way)

1. **Observe-only entry point, guaranteed by type.** The shadow hook returns
   nothing (`-> None`): callers *cannot* consume the selection even by
   accident. The record is the only output.
2. **Isolation from shared failure machinery — the observer must not
   suppress what it observes.** Run the shadow call inside its own failure
   shield. In one deployment, a repeatedly failing shadow call incremented a
   circuit breaker *shared with the live path*, and the very generation the
   shadow preceded was skipped as circuit-open — the observer silently
   suppressed the behavior it existed to observe (a cross-model reviewer
   caught it; two same-model reviewers had missed it). Audit every piece of
   shared state the shadow path touches: circuit breakers, rate-limit
   budgets, caches, retry counters. "Degrade never abort" inside the shadow
   function is not enough if its failures leak out through a side channel.
3. **Validate, don't trust, the stochastic output.** Match the mechanism's
   answer against the closed set of legal values (catalog names);
   out-of-set answers are rejected from every downstream field but
   *recorded* (`rejected_names`) — hallucination rate is first-class
   enforcement-decision data, distinct from parse failure. Keep the verdict
   taxonomy honest: "the answer didn't parse" (`fail_open_parse`) and
   "every pick was wrong" (`judged` + empty selected) are different events.
4. **Bake would-be metrics at record time.** Any metric computed against
   mutable state (token totals against a store that later maintenance will
   change) must be written into the record, not recomputed at report time —
   otherwise the record cannot replay what the decision would have meant
   *then*.
5. **Kill switch = configuration absence.** Leaving the shadow config unset
   disables the whole path (no LLM call, no file). No feature flag to
   maintain; tests and one-shot CLI paths stay clean by default.
6. **Reserve exit criteria at launch, decide later.** Name the readings the
   enforcement decision will consume (hallucination rate, fail-open rate,
   stability of never-selected, realized reduction distribution) and the
   window (2–4 weeks) in the decision record — but do not pick numeric
   thresholds before the data exists; one clean run is not evidence, and a
   threshold invented pre-data is a guess wearing a number.

## Pitfalls

- **Observer cost is real and must be recorded.** A shadow LLM call adds
  latency per event (several seconds per action in one deployment). Record
  it in telemetry so the enforcement decision weighs cost against benefit
  from the same corpus — and so "the instrument is too expensive to keep" is
  itself a data-backed verdict.
- **The shadow input must be what the live path sees.** Pass the same
  untrusted-wrapped input to the candidate mechanism that the live
  generation reads. Trimming the shadow input to save tokens makes the
  recorded decisions unfaithful to the enforcement-time decisions — if you
  must trim, record that as an open question in the decision record, don't
  silently diverge.
- **First-run enthusiasm.** One clean session (all events judged, zero
  hallucinations) proves the wiring, not the mechanism. The recurring
  question the window must answer: are ever-present picks genuinely
  general-purpose, or "generally useful" drift the prompt forbids?
- **Shadow mode that never ends is a zombie.** The record exists to be
  consumed by a dated decision — put an entry with a date in the task
  ledger at launch. A shadow path with no consumer and no exit date is
  exactly the wired-but-unread scaffolding this pattern was built to avoid —
  signal-first applies to the shadow itself.
