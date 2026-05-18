---
name: userspace-driver-client
description: >-
  Use when building or porting a CLIENT API for a userspace driver where the
  consumer depends on an abstract trait seam (e.g. an embedded-hal 1.0 master
  trait like SpiDevice/I2c, or another HAL trait) and the actual transport is
  pluggable — mapped to cross-process IPC in production and to an in-process
  "loopback" for host testability. Covers the layered api/server/client(+traits)
  crate split, the whole-object transaction discipline, the Transport
  abstraction (IpcTransport + LoopbackTransport), keeping IPC private behind the
  trait, the platform-agnostic guardrail, and the runtime-dispatch-vs-typestate
  impedance trap. Triggers on "client api for a userspace driver", "wrap this
  driver behind IPC", "make the transport pluggable / mockable", "loopback for
  testing the driver protocol", "expose an embedded-hal device over IPC",
  "userspace driver server/client like the usart/crypto one". Language-agnostic;
  worked instantiation is Rust/embedded (openprot).
---

# Pluggable-Transport Userspace Driver Client

A reusable workflow for giving a userspace driver a **client API the consumer
depends on without knowing the transport**, where the transport is swappable
between real cross-process IPC and an in-process loopback used for host tests.

Distilled from the openprot `drivers/usart` (template), the crypto driver
(`crypto-driver-sketch.md` ADR-C1/C2/C5), and `services/mctp` (`IpcMctpClient` +
generic facade). Those are the canonical worked examples — read them as the
reference instantiation of every section below.

Work the sections roughly in order. Each lists **artifacts** (what to produce)
and **guardrails** (the failure-mode it prevents — the guardrails are the
point).

## Use when

- A driver runs in its own protection domain/process and consumers must reach
  it without linking it or knowing the IPC mechanism.
- The consumer-facing contract is (or should be) an existing abstract trait —
  ideally **embedded-hal 1.0** for bus masters (`SpiDevice`, `SpiBus`, `I2c`),
  or another canonical HAL trait — not a bespoke API.
- You want the protocol + marshalling path host-testable without a kernel/QEMU.

## Don't use when

- The driver is a pure in-process library with no isolation boundary (just call
  it; no transport, no client crate).
- The interaction is inherently a long-lived stream that cannot be expressed as
  bounded whole-object requests (rethink the boundary first — see Invariant 1).

## Core invariants (non-negotiable)

1. **Whole-object, run-to-completion across the boundary.** The unit marshalled
   is one complete operation that the server runs start-to-finish before
   replying. No `begin/update/finish`, no held lock, no per-step round-trip
   across the transport. For embedded-hal masters this means **an entire
   `transaction(&mut [Operation])` is one request/response** — the server
   performs the whole transaction atomically on the real bus and replies;
   per-`Operation` or bus lock/unlock is never exposed across the boundary
   (that is what preserves `SpiDevice`'s exclusive-atomic contract between
   processes). Guardrail: prevents recreating a held-across-yield lock spanning
   processes.
2. **Consumers depend only on the abstract seam, never the transport.** The
   client crate may be soaked in IPC syscalls — that is its single job — but it
   exposes itself **only** as an implementation of the abstract trait (the
   `IpcMctpClient : MctpClient` pattern). No public inherent transport methods.
   Guardrail: a consumer cannot become IPC-coupled by accident; backends stay
   swappable (real / loopback / mock).
3. **Platform-agnostic boundary.** Nothing in the agnostic driver crates names
   a SoC/vendor/silicon/bus-instance; platform specifics live only in a
   per-target backend/binding crate. Guardrail: one vendor name leaking in
   couples every consumer to one platform.
4. **`Transport::transact` is whole-object by construction.** The transport
   trait is bytes-in → bytes-out, one shot, for *every* impl. Invariant 1 is
   then enforced by the transport signature itself, not by convention.

## Layered crate map (mirror `drivers/usart/`)

| Crate | Role |
|---|---|
| `*_api` | Wire protocol (zerocopy headers) + the abstract seam contract / re-export of the embedded-hal trait used. |
| `*_server` | Pure `dispatch_request(backend, req, resp)` translator + a topology-agnostic runtime loop. Generic over the backend trait. No IPC policy beyond read/dispatch/respond. |
| `*_client` | `Client<T: Transport>` that **implements the abstract trait** by marshalling each whole-object call over `T`. All marshalling private. |
| `*_traits` (optional) | If the seam needs bounding/bundling or a generic facade, put it here; otherwise depend on the HAL crate directly. Don't reinvent the trait. |
| `target/<plat>/backend/*` | The real platform driver implementing the same abstract trait (the server's backend). The **only** crate that names silicon. |
| `target/<plat>/tests/*` | system image + host/QEMU smoke wiring it together. |

## The Transport abstraction (the heart of this skill)

Artifacts:

- `trait Transport { fn transact(&mut self, req: &[u8], resp: &mut [u8]) -> Result<usize, TransportError>; }`
- **`IpcTransport`** — the production cross-process path (e.g. Pigweed channel
  `channel_transact`). One impl, isolated.
- **`LoopbackTransport`** — first-class, *not* deferred: calls the server's
  `dispatch_request` directly against an in-process backend (the real platform
  driver, or a mock embedded-hal bus such as `embedded-hal-mock`). This makes
  consumer → client → protocol → server → backend host-testable with **no
  kernel/QEMU**, and is also the correct early-boot path (before IPC exists).
- `Client<T: Transport>` implements the abstract seam; for each seam call it
  serializes a whole-object request, `T::transact`s it, parses the response.

Guardrail: the same `Client` marshalling code is exercised by tests (loopback)
and production (IPC) — the protocol is tested for real, and "in-process vs IPC"
is a wiring choice, never a code fork.

## embedded-hal 1.0 master mapping (worked shape)

- Seam = `embedded_hal::spi::SpiDevice` (or `i2c::I2c`). Consumer depends on
  that trait only.
- Wire op = one `transaction`: encode the `&[Operation]` (Read/Write/Transfer/
  DelayNs) into the request; response carries the read buffers + status.
- `Client<T>: SpiDevice` — `transaction()` builds the request, one
  `T::transact`, scatters results back into the caller's read slices.
- Server backend = the platform's real `SpiBus`/`SpiDevice`; `dispatch_request`
  replays the operation list atomically on it and returns the reads.
- Bounds: cap operation count / buffer sizes in the protocol; reject
  over-large transactions with a typed status (don't fragment across the
  boundary — Invariant 1).

## The runtime-dispatch vs typestate trap (learn this)

If the chosen seam uses **compile-time typestate / opaque associated types**
(GAT contexts, algorithm-as-type, backend-chosen `Key`/`Output` types — common
in crypto HALs), a *generic* server cannot drive it from a runtime-decoded wire
request: associated types have no constructors and outputs no byte view. Don't
hack the seam. Either (a) add a **thin byte-oriented dispatch trait** the
backends implement by *delegating to* their existing seam impls (two honest
contracts: typed in-process vs runtime-byte — see the crypto `RuntimeCrypto`
decision), or (b) have the server take a concrete backend. Quarantine all
runtime↔typed impedance in **one shim module**, never smeared across the server.
embedded-hal master traits are byte/slice-oriented and usually avoid this — but
check before assuming the seam is directly server-drivable.

## Separation of duties (when the driver gates a resource)

The server must never accept caller-supplied addresses/handles into a
more-privileged domain (confused-deputy). Orchestration that needs another
driver's data (e.g. "hash a flash image") belongs in the *consumer*, composing
two clients — not as a cross-domain op that hands one server another's
authority. A call-scoped memory lease (if the platform has one, e.g. Hubris)
is a zero-copy optimization of the hand-off, not a change in trust.

## Operating discipline

- **Faithful reporting.** Build/test for real; report exact pass/fail and every
  deviation. Kernel-tagged crates incompatible on the host platform is expected
  — verify against the analogous reference crate, don't claim a false failure.
- **Hygiene.** Source files only; stage explicit paths; never `git add -A`;
  never commit build artifacts; branch/commit only when asked.
- **Don't reinvent the seam.** Reuse embedded-hal 1.0 / the canonical HAL. A
  bespoke `Read/Write/...` trait that duplicates an existing one is the smell.
- **Guardrail grep before "done":** no vendor/SoC tokens in the agnostic
  crates; no public inherent transport methods on the client; `Transport`
  impls are the only place a channel/syscall is named.

## Done criteria

- Consumer compiles against the abstract seam alone (no transport in its deps).
- `Client<IpcTransport>` and `Client<LoopbackTransport>` both build; a host
  test drives consumer→client→server→backend via loopback with no kernel.
- Server is generic over the backend trait; platform names confined to the
  per-target backend crate.
- Whole-object invariant holds: one seam call ⇒ one `transact` ⇒ one
  server-side run-to-completion ⇒ one reply.
