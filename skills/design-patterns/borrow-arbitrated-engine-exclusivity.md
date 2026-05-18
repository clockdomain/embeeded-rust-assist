# Borrow-Arbitrated Engine Exclusivity

*A structural concurrency pattern: make a shared single-instance hardware
engine mutually-exclusive by language ownership, not a runtime flag. (Rust
`&mut`-borrow mechanism — language-specific; stated honestly below.)*

---

## Also Known As

Borrow-as-arbiter · Ownership-arbitrated shared engine · Owned-engine
exclusivity token · Compile-time `in_use` (the *exclusivity* member of the
*owned-peripheral* family — the layer the Confined-`unsafe` MMIO Façade and
Cooperative-Yield Bounded-Poll Device entries both defer "to a layer above").

## Intent

Model one physical engine instance as exactly **one owned value**; derive
every operation handle from an **exclusive `&mut` borrow** (or move) of it, so
that two mutually-exclusive operations of that engine **cannot overlap by
construction** — the borrow checker is the arbiter. This replaces the
reference's fragile runtime `in_use`/`-EBUSY` flag and removes any need for a
hardware "busy/available" predicate.

## Motivation

A single hardware engine often serves several mutually-exclusive operations
(e.g. one Secure-Boot-Controller public-key engine doing both ECDSA verify and
RSA modexp; one HACE engine doing SHA and HMAC). The reference firmware
arbitrates with a **per-driver software flag**:

```c
static struct aspeed_ecdsa_drv_state drv_state;          /* ecdsa_aspeed.c */
if (drv_state.in_use) return -EBUSY;  drv_state.in_use = true;
static struct aspeed_rsa_drv_state   drv_state;          /* rsa_aspeed.c   */
if (drv_state.in_use) return -EBUSY;  drv_state.in_use = true;
```

Two defects: (1) the flag is **per driver** — the ECDSA and RSA `in_use`
booleans are *independent statics*, so the reference does **not** coordinate
the two operations on the one shared engine at all; safety rests on
single-threaded usage discipline. (2) check-then-set is a runtime race window.
Reaching for a *hardware* busy bit instead is worse: it deviates from an
authority that does not use one, and its real-silicon semantics are usually
unverified.

The obligation is actually just "≤1 operation touches this engine at a time
within this owner." In an ownership language that is a **type invariant**: if
the engine is one non-`Copy` value and each operation borrows it `&mut`, the
compiler refuses a second concurrent operation — across *all* operation kinds —
with zero runtime cost and no race window.

## Applicability

Use when:

- one engine instance is shared by ≥1 mutually-exclusive operations **within a
  single trust domain / address space**, layered on a Confined-`unsafe` Façade;
- the reference arbitrates with a runtime `in_use`/`-EBUSY` (or you are tempted
  to read a HW busy bit) and you want exclusivity made structural and
  cross-operation-correct;
- operations are run-to-completion in-process (bounded loop / blocking).

Do **not** use it (alone) when:

- the engine is genuinely reached from multiple processes / cores / DMA
  masters concurrently — ownership is a language fiction, not a hardware lock;
  use a real HW semaphore / OS lock;
- many clients must enqueue work answered asynchronously — that is a
  service-layer queue above the device, not in-process borrow exclusivity;
- you must reproduce the reference's exact `-EBUSY`-on-overlap as *observable*
  behavior — then add the optional non-blocking `try_acquire` variant
  (software, typed `Busy`) on top; pure borrowing makes overlap a *compile*
  error, not a runtime return.

## Structure

```
        owns exactly one (non-Copy, constructed at the confined-unsafe gate)
   caller ─────────────► EngineDevice
                              │  &mut self  (exclusive borrow / move)
            ┌─────────────────┼──────────────────┐
            ▼                                     ▼
   Op_A<'a> = from_device(&'a mut dev)   Op_B<'a> = from_device(&'a mut dev)
   (ECDSA verify)                        (RSA modexp)
            │                                     │
            └──────────── borrow checker ─────────┘
              a 2nd live Op (any kind) while one is alive  ⇒  COMPILE ERROR
              Op dropped ⇒ &mut released ⇒ engine free again
   (no `in_use` bool anywhere · no HW busy-bit read · no TOCTOU)
```

## Participants

- **Engine device** (`<Engine>Device`) — the single owned binding over the
  Confined-`unsafe` façade; **not `Copy`/`Clone`**; constructed once.
- **Operation handle(s)** (`<Engine>Op`, `<Engine>Digest`, …) — each built
  *only* by an exclusive `&mut`/move borrow of the device (borrow-split); short
  lifetime `'a` tied to that borrow.
- **Borrow checker** — the arbiter. Not a runtime component.
- **(Optional) service/queue layer** — above the device, for multi-client
  runtime serialization; the principled replacement for the reference flag
  when overlap is *reachable* across owners.

## Collaborations

- The device is constructed once (Confined-`unsafe` Façade gate) and held by
  one owner. Each operation is a borrow-split of it (the same split the
  Cooperative-Yield entry uses to type-erase the wait strategy).
- All operation kinds route through the *same* device value, so mutual
  exclusion holds **across operations**, not just within one — closing the
  reference's per-driver-flag gap.
- Composes strictly **above** the Confined-`unsafe` Façade and Cooperative-
  Yield Bounded-Poll Device; **below** any Typestate Session sequencing.

## Consequences

**Benefits**

- *Compile-time mutual exclusion, zero runtime cost*: no `in_use` byte, no
  flag-set/flag-check, no TOCTOU window.
- *Cross-operation correctness by construction*: every op needs the one
  `&mut`; ECDSA-verify vs RSA-modexp overlap is unrepresentable — the exact
  bug the reference's independent per-driver flags do **not** prevent.
- *No hardware-busy read*: no deviation from a reference that doesn't use one;
  no dependence on unverified HW semantics.
- *Drop = release*: scope ends → engine free; no "forgot to clear the flag".

**Liabilities / tradeoffs**

- *Language fiction, not a hardware lock*: arbitrates only within one
  owner/address space. A second process, core, or DMA master touching the same
  MMIO is **not** stopped — say so explicitly; cross-domain sharing still needs
  a real lock or service queue.
- *Singleton plumbing*: exactly one device value must be threaded to whoever
  needs it; "who owns it / how it's handed between tasks" becomes an explicit
  design question (intentional, but real).
- *Over-serialization across yields*: a borrow held across an `.await`/scheduler
  yield pins the engine for that whole interval (same hazard the Cooperative-
  Yield entry notes) — exclusivity is real but can serialize more than wanted
  if an op straddles a yield.
- *Not a queue*: gives mutual exclusion, not fairness/ordering among waiting
  clients; if overlap is reachable across owners you still must add the
  service layer.

## Implementation

1. Make the engine binding **one owned, non-`Copy`/`Clone` value**
   (`<Engine>Device`), constructed once at the Confined-`unsafe` façade gate.
   The façade handle inside may be `Copy` (it confines `unsafe`, not
   exclusivity) — the *device* must not be.
2. Create every operation handle **only** via an exclusive borrow of the
   device: `fn from_device(&'a mut <Engine>Device) -> <Engine>Op<'a>` (or a
   consuming move for an owned-session variant). Bind the op's lifetime to that
   borrow.
3. Route **all** operation kinds (every distinct op of the engine) through that
   one device, so the single `&mut` arbitrates across operations, not per kind.
4. Do **not** add an `in_use`/busy boolean and do **not** read a hardware
   "busy/available" register to gate mutually-exclusive ops — those are the
   rejected ancestor; delete them if porting from such a reference and record
   it as the intentional structural delta.
5. If the reference's `-EBUSY`-on-overlap must be observable, add a software
   `try_acquire(&mut self) -> Result<Op, Busy>` *on the owned device* (typed
   `Busy`, mirrors the reference flag's semantics) — still no HW predicate.
6. Push any multi-client runtime serialization to a **service/queue layer
   above** the device; keep the driver a single owned engine.
7. State the language dependency: this is Rust `&mut`-exclusivity (or
   move-linearity); in a non-ownership language the runtime flag may be
   unavoidable — then this pattern does not apply, say so.

## Sample Code

Real instance — AST10x0 SBC public-key engine (one engine, two operations:
ECDSA verify implemented, RSA modexp planned — same hardware).

Owned device, `target/ast10x0/peripherals/sbc/device.rs` — note: **no
`#[derive(Copy/Clone)]`**, unlike the façade it wraps:

```rust
pub struct SbcDevice<Y: FnMut(u32)> {     // one owned binding per engine
    pub(crate) regs: SbcRegisters,        // façade may be Copy; the device is not
    pub(crate) yield_fn: Y,
    pub(crate) poll_budget: u32,
}
```

Borrow-split = the arbiter, `target/ast10x0/peripherals/sbc/op.rs`:

```rust
pub unsafe fn from_device<Y: FnMut(u32)>(
    device: &'a mut SbcDevice<Y>,         // EXCLUSIVE &mut — the whole pattern
) -> Self {                               // ⇒ ≤1 live SbcOp of ANY kind
    let regs = device.regs;
    let poll_budget = device.poll_budget;
    let yield_fn: &'a mut dyn FnMut(u32) = &mut device.yield_fn;
    Self::new(regs, poll_budget, yield_fn)
}
// ECDSA verify (op #1) and the planned RSA modexp (op #2) both require
// `&mut SbcDevice` — a concurrent second op is a borrow-check error, with
// no `in_use` flag and no hardware busy-bit anywhere.
```

The rejected ancestor (the *coupled* form this factors apart) — pinned Zephyr
`drivers/crypto/{ecdsa_aspeed.c:150, rsa_aspeed.c:129}`: two independent
`static drv_state.in_use` booleans + `-EBUSY`, which do **not** coordinate the
two operations of the one shared engine.

## Known Uses

- `target/ast10x0/peripherals/sbc/{device.rs,op.rs}` — `SbcDevice` shared by
  ECDSA verify (`SbcOp` + `verify_raw`, implemented) and RSA modexp (planned,
  same engine; see that port's goal.md ADR-5). One `&mut` borrow arbitrates
  both operations — the design decision this entry was distilled from.
- Vendor Zephyr `aspeed_ecdsa`/`aspeed_rsa` per-driver `in_use`/`-EBUSY`
  (`ecdsa_aspeed.c:150`, `rsa_aspeed.c:129`) — the uncoordinated runtime-flag
  ancestor; replaced (and its cross-operation gap closed) by the borrow.

**Counter-example — non-conforming look-alike** (instructive):
`target/ast10x0/peripherals/hace/{device.rs,digest.rs}`. It *looks* like this
pattern (`from_device(&mut HaceDevice) -> HaceDigest` borrow-split) but **does
not conform**: (1) `HaceDevice` is `#[derive(Copy, Clone)]` (`device.rs:9`),
so the `&mut` arbitration is bypassable by copying the device — **fails
Checklist box 1**; (2) operation state is a process-global `HashContext`
reached via `unsafe { &mut *shared_ctx_ptr() }` (`digest.rs:128`), aliased
outside the device — **fails the no-global-op-state box**. Net: HACE's mutual
exclusion actually rests on the *documented* `unsafe` non-reentrancy contract
— the reference's "caller must serialize" discipline in Rust clothing, not the
structural guarantee. (Note: `HaceDevice` being `Copy` is fine for the
*Cooperative-Yield Bounded-Poll Device* entry, whose Sample Code it is — that
pattern needs the borrow-split for type-erasure, not exclusivity. Same code,
conforms there, fails here: the Checklists differ for a reason.)

## Related Patterns

- **Confined-`unsafe` MMIO Façade** — the layer below; supplies the safe
  engine handle. It deliberately stays `Copy` and does *not* enforce
  exclusivity — this pattern is the layer it defers that to.
- **Cooperative-Yield Bounded-Poll Device** — sibling layer; its non-`Copy`
  device binding *is* the owned value this pattern arbitrates on, and its
  borrow-split is the same mechanism (here used for exclusivity, there for
  wait-strategy type-erasure).
- **Typestate Session** — may sit above to additionally order operations; this
  pattern only guarantees mutual exclusion, not sequencing.
- **Owned Peripheral / Proof Token** — this is that family's *exclusivity*
  member (the façade entry is its MMIO member, the bounded-poll entry its
  wait-policy member).
- **RAII Guard (GoF-adjacent)** — drop-releases-the-engine is the guard aspect;
  the borrow *is* the guard.

## Checklist (a conforming instance has all of these)

- [ ] The engine-device value is **not `Copy`/`Clone`** (a `Copy` device makes
      `&mut` arbitration bypassable — the single hardest, most common failure;
      see the HACE counter-example).
- [ ] Operation state is **owned by, or passed through, the borrowed device**
      — **no `static`/global mutable operation buffer reached via raw pointer**
      and aliased outside the device. (HACE's `shared_ctx_ptr()` fails this.)
- [ ] Single-instance-per-physical-engine is upheld by the Confined-`unsafe`
      gate's documented `unsafe fn new*` contract (the family mechanism) —
      the entry/instance **states** this is gate-delegated, not pretended to
      be borrow-enforced. (This pattern's own guarantee is *cross-operation*
      exclusivity **given** one such device.)
- [ ] Every operation handle is produced **only** by an exclusive `&mut`
      borrow (or move) of that device; its lifetime is tied to that borrow.
- [ ] **All** operation kinds of the engine route through the same single
      device value (mutual exclusion holds *across* operations, not per kind).
- [ ] **No** runtime `in_use`/busy boolean and **no** hardware
      "busy/available" register read is used to arbitrate mutually-exclusive
      operations (and if porting from such a reference, the removal is recorded
      as the intentional structural delta).
- [ ] Any multi-client/runtime serialization lives in a service/queue layer
      **above** the device, not in the driver and not as a HW predicate.
- [ ] The entry/instance **states the language dependency** explicitly (Rust
      `&mut`-exclusivity / move-linearity is the arbiter).
- [ ] An optional non-blocking API, if provided, is a **software**
      `try_acquire`-style typed `Busy` on the owned device — never a hardware
      busy-bit predicate.
