---
name: peripheral-parity-port
description: >-
  Use when porting / reimplementing a hardware peripheral driver to behavioral
  parity with an existing reference, reverse-engineering a behavioral spec from
  vendor/reference code, establishing a pinned normative authority, classifying
  parity deltas, or building KAT/parity test harnesses. Triggers on tasks like
  "reverse engineer the behavior of X", "port driver X to achieve parity",
  "is the port byte-identical to the reference", "write the parity goal/plan".
---

# Peripheral Behavioral-Parity Porting Playbook

A reusable, agent-driven workflow for porting a hardware peripheral driver to
**behavioral parity** with a reference. Distilled from the AST10x0 HACE port
(`target/ast10x0/peripherals/hace/plans/goal.md` is the canonical worked
example — read it as the reference instantiation of every template below).

Work the phases in order. Each phase has **artifacts** (what to produce) and
**guardrails** (the AI failure-mode it prevents — these are the point).

---

## Phase 0 — Establish the authority (do this before reading anything deeply)

There are usually several candidate references (the vendor driver, another
language port, a datasheet). Exactly one is **normative**: *the implementation
the deployed target actually runs*. Everything else is **informative**.

Artifacts:
- A `plans/<peripheral>-reference/` directory: the relevant normative source
  files **vendored verbatim**, plus `PINNED_COMMIT.txt` (exact VCS revision,
  upstream path, freeze date, one line stating why it is authoritative).
- A statement in the goal doc: `Authority = <pinned ref> @ <rev>`; named
  informative refs marked "informative only; where it differs, the authority
  wins and the informative ref is treated as buggy."

Guardrail — **normative-over-convenient**: an agent will anchor on the most
readable/accessible reference (often a sibling port). That reference is
frequently *wrong*. Force an explicit normative-selection step with written
justification. (HACE: `aspeed-rust` was the convenient anchor and was buggy;
the pinned Zephyr driver was authoritative.)

## Phase 1 — Reverse-engineer the behavioral spec (behavior, not structure)

Produce the spec from the **normative** source, language-neutral:
register-write sequence (exact order), command/control word composition,
state machine (init/update/finalize/cancel), memory & DMA model,
padding/endianness, completion/error/timeout model.

Guardrail — **citations-or-it-didn't-happen**: every behavioral claim cites a
normative `file:line`. No inferred behavior. Anything not in the source is
marked `OPEN`, never guessed. Spec describes *behavior*; do not transcribe the
reference's type layout — structure does not port across languages.

## Phase 2 — Decide the parity standard (human owns this fork)

Use `AskUserQuestion`. The standard is one of:
exact byte-for-byte incl. reference bugs / **observable parity, keep fixes** /
exact success-path, fixes flagged. Record verbatim in the goal doc as
`Parity standard (decided): …`.

Guardrail — **human-owns-forks**: do not silently pick a parity philosophy; it
changes every later classification.

## Phase 3 — Build the deltas ledger

Table, one row per behavioral difference:

| ID | Authority behavior (verbatim + `file:line`) | Port behavior | Classification |
|----|---------------------------------------------|---------------|----------------|

Classification ∈ { **conformance** (port == authority; not a delta),
**intentional delta** (port deviates by decision), **out-of-scope by
decision** }. Every *intentional delta* needs a **reachability trace**: real
consumer code proving no reachable input triggers it (or proving it does and
the deviation is accepted). A delta is "discharged" only when both the
authority lines were read *and* consumers were traced.

Guardrail — **no speculative unreachability**: "probably no consumer does
that" is not discharge. Cite the consumer. Re-verify the authority's behavior
against the *pinned* source, not against memory or the informative ref.

## Phase 4 — Separate the three authorities (never conflate)

1. **Parity authority** — the pinned driver (byte-match target).
2. **Correctness authority** — independent published KATs (RFC/NIST), distinct
   and non-overlapping with parity.
3. **Interface authority** — the trait/API the port must satisfy.

State each obligation independently in the goal doc.

Guardrail — **verify the mandate**: before writing "the spec/trait requires
X", *read the trait/spec* and quote it. Interface authorities are usually
algorithm-agnostic; do not import an algorithm requirement from an interface
contract. (HACE: the openprot MAC trait mandates shape, not RFC-2104 — RFC-2104
was *our* declared choice, sourced explicitly.)

## Phase 5 — Implementation plan + done criteria

Plan: numbered items, each ending with an `Acceptance:` line. Done criteria:
testable gates keyed to the **production-dominant workload**, not toy vectors.
Drop plan items that become unnecessary as decisions land (strike them, state
why) — keep the plan honest.

## Phase 6 — Parity / KAT harness

- KAT vectors == authority output (= standard algorithm output where the
  authority is conformant).
- **Gating test = the production-dominant call pattern** (e.g. the real
  streaming chunk size/alignment), not just one-shot toy inputs.
- One explicit test per *intentional delta* asserting the port's chosen
  behavior and documenting the authority's divergence.
- Conformance rows need no divergence test.

## Phase 7 — Convergence protocol (when it stops working)

If black-box edit→rebuild→test iteration stops converging or regresses:
**stop speculative edits.** Record an `OPEN ISSUE` with: green/failing matrix,
isolation sub-tests that localize the fault, and any layout/timing sensitivity
signal. Switch to **instrumented tooling** — emulator memory/DMA tracing,
GDB watchpoints on the corrupted region, or reading the emulator's device
model — before any further edits.

Guardrail — **stop-and-instrument**: cap speculative loops; a deterministic
known mismatch is never "done"; escalate to instrumentation, do not keep
guessing.

## Phase 8 — Hygiene & architecture capture

- Commits: stage **explicit paths**, never build artifacts; branch off the
  non-default branch; honest messages describing the actual diff.
- Capture cross-cutting architectural constraints (vendor limitations,
  SW/HW-accelerator selection, isolation/TCB, user-space service shape) as
  **normative ADR sections** in the goal doc, grounded with the same
  citation discipline as behavior.

---

## goal.md skeleton (instantiate per peripheral)

```
# <PERIPHERAL> Behavioral Parity Goal (<target>)
## Objective
  Authority = <pinned ref> @ <rev> (frozen at plans/<p>-reference/).
  Informative-only: <other refs> (authority wins; treated as buggy on diff).
  Parity standard (decided): <one of the three>.
  Scope: <target> only.
## 1. Reference behavior to replicate   (Phase 1, every claim file:line)
## 2. Deltas vs. the authority          (Phase 3 table + reachability traces)
   2.x Independent correctness/interface authority split (Phase 4)
   2.x OPEN ISSUE: <if any>             (Phase 7 format)
## 3. Implementation plan               (numbered, each + Acceptance:)
## 4. Done criteria                     (testable, production-dominant workload)
## 5. Architecture decisions            (Phase 8 ADR sections, grounded)
```

## The five guardrails, condensed

1. **Normative over convenient** — pin the deployed model; the readable ref is often wrong.
2. **Citations or it didn't happen** — every claim → normative `file:line`; unknowns are `OPEN`.
3. **Human owns forks** — parity standard & scope decisions via `AskUserQuestion`, recorded verbatim.
4. **Verify the mandate** — read the trait/spec before asserting it requires anything.
5. **Stop and instrument** — bounded speculative loops; deterministic mismatch ≠ done.
