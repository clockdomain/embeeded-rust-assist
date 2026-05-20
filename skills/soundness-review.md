# Soundness Review Skill

**Trigger phrases:** "soundness review", "audit for unsound code", "review for soundness", "check for unsoundness"

## Purpose
Systematically audit Rust code for memory safety issues, unsafe block violations, and embedded-specific soundness pitfalls. Activates a scoped code review targeting the kinds of bugs that can slip past the compiler.

## Activation workflow
1. User invokes `/soundness-review` or mentions soundness audit
2. Skill asks user to scope the review (default: entire crate):
   - Entire crate / specific file(s) / specific unsafe blocks?
3. Skill scans code for issues in these categories (default focus: interrupts, DMA, volatile access):
   - **Interrupts**: race conditions, re-entrancy, improper disable/restore semantics
   - **DMA**: buffer aliasing, coherence issues, ownership tracking across DMA transfers
   - **Volatile access**: correct ordering, proper typing, read/write parity with hardware spec
   - **Unsafe block hygiene**: missing SAFETY comments, unjustified invariants
   - **Memory safety**: use-after-free, double-free, uninitialized reads
   - **Lifetime/borrowing**: violations that unsafe can hide
4. Returns findings grouped by severity + location
5. For each issue, supplies: description, why it matters, suggested fix

## Implementation checklist
- [ ] Identify all `unsafe` blocks and adjacent `SAFETY:` comments
- [ ] For each unsafe block, verify the invariant is stated and sound
- [ ] Check for race conditions in interrupt handlers or shared state
- [ ] Audit volatile reads/writes for correct ordering and type
- [ ] Verify DMA buffers are not aliased with other mutable references
- [ ] Check transmute usage for size/alignment compatibility
- [ ] Look for lifetime holes in raw pointer casts
- [ ] Verify no use-after-free in drop implementations or cleanup paths
- [ ] Check for soundness gaps in macro expansion (if applicable)
- [ ] Flag any `todo!()` / `unwrap()` in interrupt or critical paths

## Severity levels
- **Critical**: immediate UB, crash, or data corruption risk
- **High**: potential race condition, aliasing violation, or memory safety gap
- **Medium**: unsafe block lacks justification, or invariant is unclear
- **Low**: style/documentation issue, or theoretical (blocked by other safety guardrails)

## Output format
- Grouped by file and severity
- Each finding includes: location, issue type, explanation, and suggested fix
- Summary: total findings by severity

## Known uses
- Auditing new unsafe code in drivers/i2c before merge
- Reviewing HAL changes that expose raw hardware access
- Post-port validation (e.g., after porting a peripheral driver)
