# Confined-`unsafe` MMIO Façade

*A structural pattern for memory-mapped I/O over a Peripheral Access Crate (PAC).*

---

## Also Known As

Unsafe-perimeter newtype · Safe MMIO façade · Single-gate `unsafe` ·
Register-access proof token (the MMIO layer of the *owned-peripheral* family).

## Intent

Confine all `unsafe` required to touch memory-mapped hardware to a **single,
audited construction site**, turning the peripheral's safety obligations into a
**type invariant**, and re-expose the device as a narrow, safe, intent-named
API so that no business logic ever writes `unsafe` or sees PAC types.

## Motivation

`svd2rust`-style PACs expose a register block reached through a raw pointer
(`PERIPH::ptr()`). Used directly, every access site repeats
`unsafe { &*PERIPH::ptr() }` and raw `w.bits(..)`, the unsafety is *diffuse*,
auditing requires reading the whole driver, and PAC types leak everywhere.

The obligations for sound MMIO are actually few and **discharged once**:

1. the base pointer refers to a valid register block for the program lifetime;
2. access to the instance is serialized (no data race on the device).

A newtype takes a pointer in **one** `unsafe fn new` whose doc enumerates
exactly those two obligations. Its caller discharges them once; thereafter the
invariant holds for the value's whole life. Inside, a single private helper
performs the only repeated `unsafe` (`&*ptr`), justified by that invariant.
The public surface is a hand-picked set of *device-vocabulary* methods
(`clear_hash_intflag`, `program_hash_operation`), not the PAC register set.
Audit collapses to: read `new`, read `regs()`, done.

## Applicability

Use this pattern when:

- accessing MMIO through a PAC / raw register-block pointer;
- you want driver and business code to be `unsafe`-free and PAC-agnostic;
- the soundness conditions are *establishable once* (fixed base, externally
  serialized access) rather than per-call;
- you want the threading obligation (`!Send`/`!Sync`) encoded in the type.

Do **not** rely on it (alone) when the type system must *enforce* exclusive
access — see Consequences: this pattern *confines* `unsafe` and *restricts
threading*; it *delegates* serialization. Stack a move/typestate session on top
for exclusivity.

## Structure

```
        caller (discharges the 2 obligations once)
                     │  unsafe fn new(base)
                     ▼
   ┌─────────────────────────────────────────────┐
   │  <Periph>Registers   (Copy newtype)          │
   │   - ptr: *mut RegisterBlock                   │
   │   - _not_send: PhantomData<*mut ()>           │
   │                                               │
   │   fn regs(&self) -> &RegisterBlock  ← only    │  (private, the
   │       └ unsafe { &*ptr }              repeated │   sole interior
   │                                       unsafe)  │   unsafe site)
   │   pub(crate) fn <intent op>(&self, …)  ← SAFE  │
   └─────────────────────────────────────────────┘
                     │  safe, device-named API
                     ▼
        driver / business logic   (no unsafe, no PAC types)
```

## Participants

- **Façade newtype** (`<Periph>Registers`) — owns the raw pointer; holds the
  invariant; sole place `unsafe` appears.
- **Construction gate** (`unsafe fn new` / `new_global`) — the entire unsafe
  perimeter; its doc comment is the safety contract.
- **Interior deref** (`regs()`, private) — the one repeated `unsafe`, reused by
  every operation.
- **Intent operations** (`pub(crate)` methods) — curated device verbs; the only
  thing callers see; raw `.bits()` confined here.
- **Non-shareable marker** (`PhantomData<*mut ()>` or explicit `!Send`/`!Sync`).
- **Client** — driver/typestate layer; entirely safe.

## Collaborations

- The client calls the **construction gate** exactly once per peripheral
  binding, discharging the contract, then holds the façade.
- Every **intent operation** routes through the **interior deref**; none expose
  PAC types or `unsafe`.
- Higher layers (scoped-borrow / owned-move *sessions*, typestate) compose *on
  top of* the façade to add exclusivity or protocol sequencing.

## Consequences

**Benefits**

- *Auditable*: the unsafe perimeter is two functions, not the whole driver.
- *Abstraction reduction*: PAC/`svd2rust` types never leak; code is testable
  and portable across PAC versions.
- *Zero cost*: newtype over a pointer; methods `#[inline]`.
- *Thread-safety encoded*: `!Send`/`!Sync` by construction, not by convention.
- *Readable*: call sites speak the datasheet's language.

**Liabilities / tradeoffs**

- `Copy`/`Clone` is deliberate but means the façade **does not enforce
  exclusive access** — serialization is *delegated* to the gate's caller.
  Aliasing control, if required, must come from a layer above.
- The contract is real: a wrong base pointer or unserialized access is UB. The
  pattern *localizes* that risk; it does not remove it.
- Curated API must be maintained as new device operations are needed (by
  design — breadth is intentional, not free).

## Implementation

1. **One** `unsafe fn new(base)`. Its `# Safety` doc lists *exactly* the
   obligations (valid pointer for program lifetime; serialized access) and
   nothing else is `unsafe`-callable. Provide `new_global()` for the singleton.
2. **One** private `regs()` doing the sole `unsafe { &*ptr }`, with a `SAFETY:`
   comment citing the constructor invariant.
3. **No** `unsafe` above the façade: every public method is safe; raw `.bits()`
   and register writes confined inside.
4. Add `PhantomData<*mut ()>` (or `impl !Send`/`!Sync`) unless the peripheral is
   genuinely shareable.
5. Public API = **intent-named device operations only**, minimal set, each
   documenting any register-ordering / completion assumption it depends on.
6. Keep it `const`-constructible where the PAC allows, for static binding.

## Sample Code

```rust
use core::marker::PhantomData;
use ast1060_pac as device;

/// Safe façade over the AST10x0 HACE register block.
#[derive(Copy, Clone)]
pub struct HaceRegisters {
    ptr: *mut device::hace::RegisterBlock,
    _not_send: PhantomData<*mut ()>,
}

impl HaceRegisters {
    /// # Safety  — the entire unsafe perimeter, discharged once here.
    /// - `base` points to a valid HACE register block for the program lifetime.
    /// - Access to the returned instance is serialized by the caller.
    pub const unsafe fn new(base: *const device::hace::RegisterBlock) -> Self {
        Self { ptr: base as *mut _, _not_send: PhantomData }
    }

    /// # Safety — caller coordinates singleton access.
    pub const unsafe fn new_global() -> Self {
        unsafe { Self::new(device::Hace::ptr()) }
    }

    /// The only repeated `unsafe`; justified by `new`'s invariant.
    #[inline]
    fn regs(&self) -> &device::hace::RegisterBlock {
        // SAFETY: constructor guarantees a valid pointer; access serialized.
        unsafe { &*self.ptr }
    }

    // --- curated, intent-named, safe operations ---
    #[inline]
    pub(crate) fn clear_hash_intflag(&self) {
        self.regs().hace1c().write(|w| w.hash_intflag().set_bit());
    }
    #[inline]
    pub(crate) fn hash_intflag_is_set(&self) -> bool {
        self.regs().hace1c().read().hash_intflag().bit_is_set()
    }
    #[inline]
    pub(crate) fn program_hash_operation(
        &self, src: u32, digest: u32, len: u32, cmd: u32,
    ) {
        // raw bits() confined to the façade
        self.regs().hace20().write(|w| unsafe { w.bits(src) });
        self.regs().hace24().write(|w| unsafe { w.bits(digest) });
        self.regs().hace28().write(|w| unsafe { w.bits(digest) });
        self.regs().hace2c().write(|w| unsafe { w.bits(len) });
        self.regs().hace30().write(|w| unsafe { w.bits(cmd) }); // starts engine
    }
}
```

## Known Uses

- `target/ast10x0/peripherals/hace/registers.rs` — `HaceRegisters` (the
  reference instance this entry is drawn from).
- The scoped (`HaceDigest`) and owned (`OwnedDigestContext`) session types
  compose **on top of** this façade to add exclusivity and protocol sequencing.
- Idiomatic embedded-Rust HAL register wrappers and the `svd2rust` "owned
  peripheral" model are the same idea at a coarser granularity.

## Related Patterns

- **Owned Peripheral / Proof Token** — this pattern is its MMIO layer; the
  proof discharged at `new` is the token.
- **Typestate Session** (scoped-borrow / owned-move) — layered above to enforce
  exclusive access and operation ordering the façade deliberately does not.
- **Newtype / Facade (GoF)** — the structural mechanism: a thin type narrowing
  and renaming a wider, unsafe interface.
- **RAII Guard** — a complementary teardown variant (idle/zeroize the device on
  `Drop`).

## Checklist (a conforming instance has all of these)

- [ ] Exactly one `unsafe fn` constructor; its doc enumerates *only* the
      pointer-validity + serialization obligations.
- [ ] Exactly one private `unsafe` deref helper, used by every method.
- [ ] No `unsafe` and no PAC types above the façade; `.bits()` confined inside.
- [ ] `!Send`/`!Sync` unless the device is genuinely shareable.
- [ ] Public API is minimal, intent-named device verbs; ordering assumptions
      documented per method.
- [ ] Exclusivity (if required) provided by a layer above — explicitly noted.
