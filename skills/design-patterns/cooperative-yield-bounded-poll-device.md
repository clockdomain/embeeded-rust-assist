# Cooperative-Yield Bounded-Poll Device

*A behavioral pattern for completion-wait on memory-mapped hardware, layered on a Confined-`unsafe` MMIO Façade.*

---

## Also Known As

Injected-wait device binding · Strategy-injected completion poll · Bounded spin
with yield seam · Decoupled-wait driver (the *wait-policy* layer of the
*owned-peripheral* family).

## Intent

Drive a peripheral to completion by **polling a safe façade predicate inside an
explicitly bounded loop**, while the *wait policy* (busy-spin, RTOS sleep,
async-executor yield, instrumented backoff) is **injected as a closure at
construction** rather than hardcoded — so the same driver runs unchanged across
execution models, and a stuck device fails as a typed timeout rather than an
unbounded hang.

## Motivation

A façade (see *Confined-`unsafe` MMIO Façade*) gives a safe `is_done()`
predicate, but *something* must wait for it. The naïve driver hardcodes the
wait: `loop { if done { break } delay.delay_ns(5000) }`. That couples the
driver to one execution model (a blocking `DelayNs`), and an unbounded `loop`
turns a wedged engine into a hang.

The reference AST10x0 ECDSA/HACE drivers show the hardcoded form:
`AspeedEcdsa<D: DelayNs>` spins `while retry > 0 { … delay.delay_ns(5000) }`.
Two concerns are entangled there: *how long to wait before giving up* and *what
to do while waiting*.

This pattern separates them. The wait is bounded by an explicit **poll budget**
whose exhaustion is a **typed timeout error** (observable failure, no hang). The
"what to do between polls" is a **caller-injected `FnMut(u32)` strategy**: tests
pass `|_| core::hint::spin_loop()`; an RTOS build passes a task-yield; an async
build passes an executor poll-yield; a bring-up build passes a cycle counter.
The strategy is **type-erased** (`&mut dyn FnMut(u32)`) at the adapter that owns
the loop, so the protocol/trait implementations driving the device need not be
generic over it — the generic lives only at the construction gate.

## Applicability

Use this pattern when:

- a peripheral signals completion by a pollable status bit (no usable IRQ, or
  IRQ deliberately not used), reached through a safe façade predicate;
- the driver must run under more than one execution model (bare-metal spin,
  RTOS, async) without per-model forks;
- a wedged device must surface as a *typed timeout*, never an unbounded spin;
- the completion wait sits *below* a trait/protocol layer that you do **not**
  want to make generic over the wait strategy.

Do **not** use it when:

- the device is genuinely interrupt-driven and you want event-driven (not
  polled) completion — use an interrupt/waker pattern instead;
- the wait policy is fixed for the program's life and portability is a non-goal
  (a hardcoded bounded loop is simpler and sufficient);
- completion timing must be *enforced* by the type system — this pattern bounds
  and decouples the wait; it does not prove real-time deadlines.

## Structure

```
   caller ── injects wait strategy ──► unsafe fn new(base, yield_fn: Y)
                                              │  Y: FnMut(u32)  (generic gate)
                                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │  <Periph>Device<Y: FnMut(u32)>                              │
   │    regs: <Periph>Registers   (the façade)                   │
   │    yield_fn: Y               poll_budget: u32               │
   └──────────────────────────────────────────────────────────┘
            │  borrow-split: Copy regs+budget, &mut yield_fn
            ▼   type-erased here
   ┌──────────────────────────────────────────────────────────┐
   │  <Periph>Op  (runs the loop; NOT generic over Y)            │
   │    yield_fn: &mut dyn FnMut(u32)                            │
   │                                                             │
   │   for _ in 0..poll_budget {                                 │
   │       if regs.is_done() { return Ok(..) }   ← safe façade   │
   │       (yield_fn)(SUGGESTED_NS);             ← injected      │
   │   }                                                          │
   │   return Err(Timeout)                       ← typed, bounded │
   └──────────────────────────────────────────────────────────┘
```

## Participants

- **Façade** (`<Periph>Registers`) — supplies the safe completion predicate;
  the loop body touches nothing else (no `unsafe`, no PAC types).
- **Device binding** (`<Periph>Device<Y: FnMut(u32)>`) — the construction gate;
  owns the façade handle, the injected strategy `Y`, and the poll budget.
- **Wait strategy** (`Y: FnMut(u32)`) — caller-supplied; argument is an
  *advisory* suggested wait window (ns). May ignore it (`spin_loop`) or honor
  it (sleep/yield-for).
- **Operation adapter** (`<Periph>Op`) — runs the bounded poll loop; holds the
  strategy **type-erased** as `&mut dyn FnMut(u32)` so it (and any trait impls
  on it) stay non-generic.
- **Poll budget + timeout error** — the bound and its observable failure.
- **Client** — protocol/trait layer; sees only `Ok(output)` / `Err(timeout)`.

## Collaborations

- The client constructs the device **once**, injecting the strategy and (option-
  ally) overriding the budget; it then drives operations through the adapter.
- The adapter is built from the device by a **borrow split**: `regs` and
  `poll_budget` are `Copy`d out; `yield_fn` is reborrowed as
  `&mut dyn FnMut(u32)` for the adapter's lifetime.
- Each operation routes its completion wait through the bounded loop; the
  strategy is invoked once per non-completing poll.
- Composes strictly **on top of** the façade and **below** the typestate/
  protocol layer.

## Consequences

**Benefits**

- *Execution-model portable*: one driver; spin / RTOS / async / instrumented
  differ only by the closure passed at `new`.
- *No hangs*: a wedged device is a typed `Timeout`, bounded by `poll_budget`.
- *Non-generic protocol layer*: type erasure at the adapter keeps trait impls
  (`DigestOp`, …) free of a `Y` parameter — no generic blow-up across the API.
- *Testable*: inject `|_| spin_loop()` for deterministic KAT runs; inject a
  counter to measure real poll counts.
- *Façade-clean*: the loop body is safe; all `unsafe`/PAC stays in the façade.

**Liabilities / tradeoffs**

- The suggested-wait argument is **advisory**: a `spin_loop` strategy ignores it,
  so the budget is in *poll iterations*, not wall-clock time — the timeout is
  not a real-time guarantee.
- Type erasure costs one indirect call per poll (`&mut dyn FnMut`); negligible
  beside MMIO reads, but it is not zero.
- Choosing `poll_budget` is a tuning obligation: too low → spurious timeouts on
  slow silicon; too high → long stalls before a wedged device is detected.
- The pattern *decouples and bounds* the wait; it does not make it event-driven.
  If the device has a usable IRQ, polling is leaving efficiency on the table by
  choice.

## Implementation

1. Put the wait strategy on the **construction gate** as a generic
   `Y: FnMut(u32)` parameter, alongside the façade handle and a `poll_budget`
   with a sane default constant; offer a `with_timeout_polls` override.
2. Document the **non-reentrant** contract at the gate (one live device at a
   time) — the same exclusivity the façade delegates upward.
3. Build the operation adapter by **borrow-splitting** the device: `Copy` the
   façade handle and budget, reborrow `yield_fn` as `&mut dyn FnMut(u32)`.
   **Type-erase here** so the adapter and its trait impls are not generic.
4. The completion wait is `for _ in 0..poll_budget { if regs.is_done() { … }
   (yield_fn)(SUGGESTED_NS); }` — invoke the strategy **once per non-completing
   poll**, on the not-done path, with an advisory ns window (a documented
   constant, ideally mirroring the reference driver's poll interval).
5. On budget exhaustion, perform any device stop/cleanup the façade exposes,
   then return a **typed timeout error** — never fall through or spin forever.
6. The loop body calls **only safe façade predicates** — no `unsafe`, no PAC
   types above the façade.
7. When the adapter is rebuilt (e.g. an `init()` returning a fresh op context),
   **forward the strategy by reborrow** (`&mut *self.yield_fn`), not by copy.

## Sample Code

Real instance — AST10x0 HACE digest path.

Construction gate, `target/ast10x0/peripherals/hace/device.rs`:

```rust
pub struct HaceDevice<Y: FnMut(u32)> {
    pub(crate) regs: HaceRegisters,          // the façade
    /// Cooperative yield hook invoked between completion polls.
    /// Argument is a suggested wait window in nanoseconds.
    pub(crate) yield_fn: Y,
    pub(crate) poll_budget: u32,
}

impl<Y: FnMut(u32)> HaceDevice<Y> {
    /// # Safety — same contract as `HaceRegisters::new`.
    /// This type is non-reentrant: only one `HaceDevice` may be active at a time.
    pub unsafe fn new_global(yield_fn: Y) -> Self { /* … */ }

    #[must_use]
    pub const fn with_timeout_polls(mut self, timeout_polls: u32) -> Self {
        self.poll_budget = timeout_polls;
        self
    }
}
```

Borrow-split + type erasure, `target/ast10x0/peripherals/hace/digest.rs`:

```rust
pub struct HaceDigest<'a, T: DigestAlgorithm> {
    pub(crate) regs: HaceRegisters,
    pub(crate) ctx: &'a mut HashContext,
    pub(crate) poll_budget: u32,
    pub(crate) yield_fn: &'a mut dyn FnMut(u32),   // type-erased: not generic over Y
    _algo: PhantomData<T>,
}

pub unsafe fn from_device<Y: FnMut(u32)>(
    device: &'a mut super::device::HaceDevice<Y>,
) -> Self {
    // regs/poll_budget are Copy; only the disjoint yield_fn field is borrowed.
    let regs = device.regs;
    let poll_budget = device.poll_budget;
    let yield_fn: &'a mut dyn FnMut(u32) = &mut device.yield_fn;
    Self::new(regs, /* ctx */ …, poll_budget, yield_fn)
}
```

Bounded poll with injected yield (same file, `DigestOp::update`):

```rust
let mut done = false;
for _ in 0..self.poll_budget {
    if self.regs.hash_intflag_is_set() {     // safe façade predicate
        done = true;
        break;
    }
    (self.yield_fn)(POLL_YIELD_NS);           // injected strategy, advisory ns
}
if !done {
    self.regs.stop_hash_operation();          // façade cleanup
    return Err(HaceError::Timeout);           // typed, bounded failure
}
```

`POLL_YIELD_NS` (`target/ast10x0/peripherals/hace/constants.rs`) mirrors the
reference HACE driver's 1 µs poll interval and is documented as advisory.

## Known Uses

- `target/ast10x0/peripherals/hace/{device.rs,digest.rs}` — `HaceDevice<Y>` +
  `HaceDigest` (the reference instance this entry is drawn from). Tests inject
  `|_| core::hint::spin_loop()`; verified against the SHA-2 / HMAC RFC-4231 KAT
  suite (the wiring is behavior-preserving — only the pre-existing, tracked
  HMAC-SHA512 long-key case fails, unrelated to the wait policy).
- The AST10x0 ECDSA port (planned, under `peripheral-parity-port`) maps the
  reference driver's `AspeedEcdsa<D: DelayNs>` + hardcoded `delay.delay_ns` /
  `retry` loop onto this pattern: `EcdsaDevice<Y: FnMut(u32)>` + `poll_budget`
  polling the `secure014` status predicate via the `EcdsaRegisters` façade.
- The hardcoded `reg_read_poll_timeout(..., 1, 3000)` idiom in vendor C HALs is
  the *coupled* ancestor this pattern factors apart.

## Related Patterns

- **Confined-`unsafe` MMIO Façade** — the layer directly below; supplies the
  safe completion predicate this loop polls. This pattern is meaningless
  without it.
- **Typestate Session** — layered above; sequences operations and enforces
  exclusivity the device binding only documents.
- **Strategy (GoF)** — the injected `FnMut(u32)` *is* a Strategy; the novelty
  is erasing it at the adapter so the protocol layer stays non-generic.
- **Owned Peripheral / Proof Token** — the non-reentrant device binding is the
  wait-policy member of that family.

## Checklist (a conforming instance has all of these)

- [ ] Wait strategy is **injected at the construction gate** as a generic
      `FnMut`-style closure — not a hardcoded delay/spin/`DelayNs`.
- [ ] Completion wait is bounded by an **explicit budget**; exhaustion returns
      a **typed timeout error** after any façade cleanup — no unbounded loop.
- [ ] The injected strategy is invoked **once per non-completing poll**, on the
      not-done path, with an advisory wait-window argument (documented constant).
- [ ] The strategy is **type-erased** (`&mut dyn FnMut(_)`) at the adapter that
      owns the loop, so that adapter and its protocol/trait impls are **not**
      generic over the strategy type.
- [ ] The loop body calls **only safe façade predicates** — no `unsafe`, no PAC
      types; it composes on a Confined-`unsafe` MMIO Façade.
- [ ] The **non-reentrant** contract is documented at the construction gate.
- [ ] Adapter rebuilds (e.g. `init()` → fresh op context) **forward the
      strategy by reborrow**, not by dropping or copying it.
