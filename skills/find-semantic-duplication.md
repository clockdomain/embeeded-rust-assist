# Semantic Duplication Finder Skill

**Trigger phrases:** "find duplication", "audit for duplication", "semantic duplication", "code duplication review"

## Purpose
Identify repeated *logic patterns* rather than mere copy-paste. Semantic duplication detects identical control flow, register access sequences, or algorithmic structures across multiple functions that could benefit from abstraction, loop extraction, or helper functions.

## Activation workflow
1. User invokes `/find-duplication` or mentions finding duplicate code
2. Skill asks user to scope the search:
   - Entire crate / specific files / specific module?
3. Skill scans for patterns like:
   - Match/if arms with identical or near-identical bodies (differing only in constants/indices)
   - Register access sequences repeated across multiple device indices (e.g., GPIO, SPI)
   - Loop bodies that repeat with minor variations (off-by-one, constant swaps)
   - Trait impl blocks repeating the same operations for different types
   - Error handling chains duplicated across functions
4. Returns findings grouped by:
   - **Duplication type**: match-arm, register-sequence, loop-body, trait-impl, error-chain
   - **Severity**: high (5+ repetitions), medium (3-4), low (2)
   - **Location**: file + line range
   - **Suggested abstraction**: helper function, macro, iterator, type parameter, or loop

## Duplication patterns to detect
- **Register index loops**: same operations on GPIO[0], GPIO[1], GPIO[2], GPIO[3] (parameterize index)
- **Match arm duplication**: arms in a match statement that differ only in constants or register names
- **Error propagation chains**: repeated `match err { ... }` blocks in multiple functions
- **Iterator bodies**: loop bodies that iterate over the same collection with identical logic
- **Trait impl repetition**: multiple impl blocks implementing the same trait for similar types
- **Offset/stride calculations**: repeated buffer indexing or pointer arithmetic patterns

## Severity levels
- **High**: 5+ instances of the same logic; significant consolidation gain
- **Medium**: 3-4 instances; consolidation is worthwhile
- **Low**: 2 instances; only worth consolidating if refactor is trivial

## Output format
- Grouped by duplication type
- Each finding includes: locations (file:line), current pattern, suggested refactor, estimated line-count savings
- Summary: total findings by severity

## Implementation notes
- Focus on *logic* not syntax; `foo(x)` and `bar(x)` are different even if the body is the same
- Flag index-parameterized loops (0..4, 0..5) as candidates for abstraction
- Prioritize high-impact patterns (register access, error chains) over low-variance matches
- Suggest specific refactoring techniques (loop, macro, helper fn, iterator)

## Known uses
- Identifying register-access patterns in aspeed-rust SPI (spim_proprietary_pre/post_config)
- Consolidating error handling in device driver traits
- Extracting trait impl boilerplate for peripheral variants
