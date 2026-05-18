---
name: stop-and-instrument
description: >-
  Debugging-escalation discipline for when iterative fixing stops converging.
  Use when several fix→rebuild→test attempts have failed, a bug is
  deterministic-wrong or flaky, behavior is sensitive to memory layout /
  alignment / timing / optimization level, a fault's victim moves when
  unrelated code changes, or you're tempted to make another speculative edit.
  Triggers on "I've tried N things and it still fails", "still wrong after the
  fix", "works in debug not release", "moving the buffer changed the symptom",
  "intermittent / heisenbug", memory/DMA/aliasing corruption, mystery
  HardFault. Forces a stop, an evidence matrix, and instrumented observation
  before any further edits.
---

# Stop and Instrument

Speculative edit→rebuild→test loops have a failure mode: they stop converging,
start regressing, and burn iterations while *learning nothing*. This skill is
the circuit breaker. It is debugging-method-agnostic and not specific to any
language or domain.

## Tripwires — invoke the moment any of these is true

- ≥ ~3 fix attempts on the same symptom without a confirmed root cause.
- A fix changed the symptom but didn't eliminate it (you're chasing, not fixing).
- The symptom is **deterministic-wrong** (same wrong answer every run) — that is
  *information*, not noise; guessing wastes it.
- The symptom moves with something that *shouldn't* matter: alignment, buffer
  placement (`static` vs stack), struct field order, `-O` level, adding a
  `println`, unrelated code. **This is the strongest tell — see Signals.**
- A previously-green test broke from an unrelated change.
- You're about to edit again "to see if it helps."

## The protocol

### 1. STOP. No more edits until step 4.

State explicitly that black-box iteration is suspended and why. Do not "try one
more thing."

### 2. Capture the evidence matrix

Write it down (in the issue / plan doc, not just in your head):

- **Green / Failing table**: every variant tried and its exact outcome
  (input, config, build mode, the *specific* wrong vs right value).
- **Isolation sub-tests**: minimal experiments that bisect the fault. Each must
  *split* the hypothesis space, e.g.:
  - reproduce the suspect computation through a *known-good independent path*
    (different API, reference impl, hand math) — does it agree?
  - feed a *pre-reduced / simplified* input that skips the suspect stage.
  - run the suspect stage *standalone* with a known-answer.
  The HACE worked example: `SHA-512(key)` standalone correct + manual HMAC math
  correct + pre-reduced-key correct ⇒ fault localized to one branch's chained
  sub-hashes, not the algorithm.

### 3. Classify the signal

| Signal | Most likely class | Don't waste time on |
|--------|-------------------|---------------------|
| Deterministic-wrong, stable | Logic/spec error in a specific path | Concurrency, "flakiness" |
| Victim **moves with memory layout / alignment / `static` vs stack** | Memory/DMA/aliasing/lifetime fault — *not* an algorithm error | Rewriting the algorithm |
| Works debug, fails release (or vice-versa) | UB, uninit, aliasing, timing assumption | Re-deriving the math |
| Intermittent, order-dependent | Race, shared mutable state, reuse-before-init | Single-step logic audits |
| Fault appears far from the trigger edit | Corruption upstream; the edit only moved the victim | "Fixing" the victim site |

The layout-sensitivity row is the key insight from the HACE HMAC-SHA512 case:
*forcing alignment moved the failing case; moving buffers to `static` turned a
green case into a HardFault.* A wandering victim ⇒ memory/DMA/aliasing, full
stop — algorithm rewrites will only relocate it.

### 4. Choose the instrument matched to the class — *then* observe

| Class | Instrument |
|-------|------------|
| Memory/DMA/aliasing corruption | Watchpoint on the corrupted region; emulator memory/DMA trace; read the **emulator/device model source** for addressing/length/alignment constraints; ASan/Miri/valgrind where available |
| UB / release-only | Miri, sanitizers, `-Z`/UBSan, inspect optimized IR/asm of the hot spot |
| Race / order-dependent | Logging with timestamps/thread ids, TSan, deterministic single-thread repro |
| Spec/logic error | Diff observed vs reference *intermediate* values at each stage, not just the final output |

Observe the *actual mechanism* before forming the next hypothesis. The goal of
this step is a **root cause stated in terms of observed evidence**, not a guess.

### 5. Only now edit — one change, targeting the proven cause

Re-run the full green set (step 2) to prove no regression. If the cause isn't
proven yet, return to step 4 — not step 5.

## Anti-patterns this prevents

- "Shotgun debugging" — many edits, no isolation, regressions.
- Treating a deterministic wrong answer as mysterious instead of as a precise
  clue.
- Rewriting algorithms to fix what is actually memory/DMA corruption.
- Declaring done with a known deterministic mismatch ("probably fine now").
- Throwing away the green baseline so regressions go unnoticed.

## Exit criteria

You may resume normal editing when **either**:
- the root cause is stated in terms of *observed* evidence (a watchpoint hit, a
  trace, a divergent intermediate value), and the next edit targets exactly
  that; **or**
- the matrix proves the issue is out of the current layer's scope and is handed
  off with the evidence attached.

"It passes now" after a speculative edit is **not** an exit criterion unless the
mechanism was observed.
