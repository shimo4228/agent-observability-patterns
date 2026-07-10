---
name: read-only-instruments
description: Design pattern for read-only instruments — aggregate readings over an agent's stored state (distributions, compositions, cluster structure) that inform the operator before an intervention. Use when quantifying an open observation before intervening (instrument-first sequencing), when a design floor/threshold would otherwise be guessed, when calibrating an embedding-based reading (three-point scale), or when deciding whether to build OR remove an instrument (signal-first both ways). NOT for per-event audit logs that replay a decision offline (that is replayable-audit-logs) and NOT for metrics that feed gates, ranking, or retrieval — instruments are observability, never intervention.
compatibility: Developed and tested on Claude Code; portable to other Agent Skills-compatible agents. Examples are abstract worked examples, not tied to any harness.
user-invocable: true
origin: shimo4228
---

# Read-Only Instruments

An **instrument** — in the aviation-panel sense: a gauge you read, not a
control you pull — is a read-only aggregate reading over stored state: a
distribution, composition tally, or cluster structure, built so the operator
can choose interventions from data instead of intuition. Worked example: an
agent that accumulates a text corpus ships a module reporting supply per
consumed view and pairwise-cosine diversity over the pool — an echo-chamber
detector the operator reads before deciding whether to rebalance the corpus.

**Not the same thing as an audit log.** An audit log records one event per
decision for offline replay
([`replayable-audit-logs`](../replayable-audit-logs/SKILL.md)). An instrument
reads across the whole store at query time. They compose: the log is the
corpus, the instrument is one lens over it.

## When to build one (instrument-first sequencing)

- **Before intervening.** Quantify current state → choose the intervention
  from readings → re-measure. There is an extra reason worth checking every
  time: if a confounder is in play (say, an embedding or generation model was
  swapped days earlier), any uninstrumented change becomes unattributable —
  you cannot tell your intervention's effect from the confounder's.
- **When a floor/threshold would otherwise be guessed.** A rescue-lane floor,
  a similarity cutoff, a cap — pick it from the measured distribution, never
  as a bare number invented at the keyboard.
- **When an open observation has no data.** "An echo effect seems to be
  forming" stays anecdote until pairwise homogeneity and supply concentration
  turn it into a reading.

## Signal-first — the gate for building AND removing

Build an instrument only if its reading **changes a named action**. The same
test governs removal:

- An instrument whose reading is *constant* under the current pipeline changes
  no action — remove it, even if it shipped days earlier. Sunk cost does not
  keep a gauge on the panel.
- An instrument nothing consumes yet is not built early. Defer it, with an
  explicit note of which future decision would consume the reading, until
  that decision is actually on the table.

## Invariants

1. **Observability, never intervention.** Readings feed the operator; none
   feed gates, ranking, retrieval, or promotion *directly or automatically*.
   An operator who reads the gauge and then sets a threshold is the intended
   loop; code that consumes the number at runtime is not. Visibility without
   intervention is the design stance; a module docstring and a decision
   record are the only guard — resist wiring numbers back into behavior.
2. **Carry an ambiguity note in the output itself.** Empty/low supply can mean
   "missing patterns" OR "a stale seed" — undecidable by the instrument.
   State that in the rendered output. A systematically biased instrument is
   worse than none.
3. **Degrade, never abort.** A malformed or wrong-dimension row is skipped
   with a warning; an instrument must never crash its host command.
4. **Bound your own cost.** Pairwise statistics over n² pairs get a
   deterministic stride-sample cap; constants that cannot be imported
   (import cycles) are pinned by a cross-check test instead of a comment.
5. **Production stays byte-identical.** Wire into dry-run / opt-in report
   paths first; longitudinal always-on wiring is a follow-up once readings
   prove useful.

## Calibration: the three-point scale

An embedding-based reading is meaningless without anchors. Measure all three
(values below are illustrative, from one deployment; measure your own):

| Anchor | How | Illustrative values |
|---|---|---|
| Floor | cosine of deliberately unrelated texts vs corpus/seeds | ~0.33–0.46 |
| Corpus mean | pairwise mean over the live pool | ~0.55 |
| Top band | best matches of the consumed views | ~0.68–0.77 |

A reading is a *signal* only relative to these (in the worked example, the
echo signature was a corpus mean sitting ~0.1–0.2 above the floor).
**Re-measure the scale whenever the geometry changes** — embedding model
swap, seed rewrite, normalization change.

## Reading pitfalls

- **A flat metric does not mean nothing changed.** A pairwise mean can hold
  steady while the corpus register visibly transforms — the change was
  orthogonal to the instrument's axis ("axis rotation"). Corroborate a flat
  reading with a qualitative sample before concluding no-effect.
- **Survivors-only data is not a distribution.** Scores recorded only for
  items that passed a gate cannot be analyzed as the population distribution.
- **Coarse scorers support thresholds only at populated bucket boundaries.**
  If the scorer emits a few discrete values, a threshold between buckets is
  fiction.
- **Don't measure orphans.** A distribution over a seed/axis nothing consumes
  measures staleness of the seed, not structure of the corpus — scope the
  instrument to what is actually consumed.

## Placement

A read-only module in the core of the codebase, passed into pipelines via an
explicit parameter, rendered through dry-run / opt-in report paths. Record
the baseline readings and the calibration scale in the decision record or
evidence directory that motivated the instrument, so the next reader knows
what the numbers meant when built.
