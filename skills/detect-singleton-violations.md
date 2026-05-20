# Singleton Violation Detector Skill

**Trigger phrases:** "detect singleton violations", "find cross-coupling", "audit for singleton creation", "check singleton discipline"

## Purpose
Identify unauthorized instantiation of singletons — peripherals, drivers, or shared resources that should exist in exactly one place but are being constructed in multiple modules without coordination. Detects when modules bypass the singleton boundary and create their own instances, leading to resource conflicts, lost updates, or duplicate initialization.

## Activation workflow
1. User invokes `/detect-singletons` or mentions singleton violations
2. Skill asks user to define the singletons:
   - Which types should be singletons? (e.g., `SpiController`, `I2cBus`, `TimerManager`)
   - Where are they *supposed* to be created? (e.g., `main`, `init.rs`, `board.rs`)
3. Skill scans for:
   - Calls to `new()`, constructors, or factory functions on singleton types
   - Instantiation patterns: `SpiController::new()`, `let uart = Uart::init()`, etc.
   - Files/modules that create instances *outside* the designated singleton module
4. Returns findings grouped by:
   - **Violation type**: direct instantiation, factory call, unsafe construction
   - **Location**: file + line where violation occurs
   - **Severity**: critical (multiple instances possible), high (creates resource leak or duplication risk)
   - **Expected authority**: which module should have created this

## Patterns to detect
- **Direct constructor calls** in non-singleton modules: `SpiController::new(regs)` outside the designated authority
- **Factory function calls**: `SpiController::from_pac(pac::SPI)` in unauthorized locations
- **Lazy initialization**: `static mut UART: Option<Uart> = None;` then `UART = Some(Uart::new())` in multiple places
- **Implicit singletons**: `&'static mut` bindings that could race or reinitialize
- **Type duplication**: multiple structs wrapping the same PAC register block (e.g., two `SpiController` instances over the same `pac::SPI`)
- **Zero-authority construction**: any `new()` call on a peripheral in library code (should be in bin/app only)

## Severity levels
- **Critical**: multiple modules can independently create instances of a shared resource (race condition, resource exhaustion)
- **High**: singleton created in library code instead of bin/board (breaks library reusability)
- **Medium**: singleton created in an unauthorized module (violates architecture, but single point of failure)
- **Low**: redundant construction (harmless but wasteful, or expected in testing)

## Output format
- Grouped by violation type
- Each finding includes: location (file:line), the constructor call, which module should have authority, risk (resource conflict, duplication, etc.)
- Summary: total findings by severity

## Implementation notes
- Focus on *constructor calls* not trait implementations (a module can impl a trait without owning a singleton)
- Flag `new()`, `init()`, `from_pac()`, and other factory patterns as violations
- Distinguish between:
  - **Legitimate**: test modules creating test doubles (usually marked with `#[cfg(test)]`)
  - **Illegitimate**: production code in multiple modules creating the same peripheral
- Parameterize singleton authority by module path (e.g., `board::` or `init::` only)
- Check for `&'static mut` patterns that could hide multiple initializations

## Known uses
- Detecting multiple I2C bus instantiations in embedded projects
- Preventing SPI controller duplication when FAT filesystem + bootloader both need access
- Catching UART instances spawned in library code instead of board init
- Enforcing single-instance constraint on interrupt handlers, clock managers, power domains
