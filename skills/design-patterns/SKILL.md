---
name: design-patterns
description: >-
  A curated catalog of reusable software design patterns (GoF-style entries:
  Intent / Motivation / Applicability / Structure / Participants /
  Consequences / Implementation / Sample Code / Known Uses / Related). Use
  when applying a known pattern, deciding which pattern fits a problem,
  reviewing code against catalog patterns, describing/naming a recurring
  design, or recording a newly distilled pattern. Triggers on "how would you
  describe this pattern", "is there a pattern for X", "turn this into a
  pattern/template", "add this to the pattern catalog", "what pattern is this".
  Language-agnostic; current entries are Rust/embedded-leaning.
---

# Design Pattern Catalog

A shared, cross-repo catalog. Each pattern is one Markdown file in this skill
directory, written as a formal catalog entry.

## How to use this skill

**Applying / identifying a pattern:** scan the index below, open the matching
entry, follow its *Implementation* and *Checklist*. Cite the entry by name when
recommending it. An instance is only "conforming" if it satisfies every box in
that entry's Checklist.

**Recording a new pattern:** add a new `<kebab-name>.md` here using the
**entry template** below, add a line to the index, and ground the *Sample Code*
in a real instance (not a toy). Keep one pattern per file. A pattern earns a
catalog entry only once it has ≥1 real *Known Use* and a verifiable *Checklist*.

## Index

| Pattern | File | One-line intent |
|---------|------|-----------------|
| Confined-`unsafe` MMIO Façade | [confined-unsafe-mmio-facade.md](confined-unsafe-mmio-facade.md) | Push all MMIO `unsafe` to one audited construction site; re-expose hardware as a narrow safe API; obligations become a type invariant. |
| Cooperative-Yield Bounded-Poll Device | [cooperative-yield-bounded-poll-device.md](cooperative-yield-bounded-poll-device.md) | Poll a façade predicate in an explicitly bounded loop; inject the wait policy as a closure; a wedged device fails as a typed timeout, not a hang. |
| Borrow-Arbitrated Engine Exclusivity | [borrow-arbitrated-engine-exclusivity.md](borrow-arbitrated-engine-exclusivity.md) | One owned non-`Copy` device per engine; every operation is an exclusive `&mut` borrow — overlap is a compile error, replacing a runtime `in_use` flag / HW busy-bit. |

## Entry template (use verbatim for new patterns)

```
# <Pattern Name>
*<one-line classification>*
## Also Known As
## Intent
## Motivation
## Applicability            (when to use — and when NOT to)
## Structure                (ASCII diagram)
## Participants
## Collaborations
## Consequences             (benefits AND liabilities/tradeoffs)
## Implementation           (numbered, actionable)
## Sample Code              (a REAL instance, with its source path)
## Known Uses
## Related Patterns
## Checklist                (auditable: a conforming instance has all of these)
```

## Conventions

- *Sample Code* is drawn from a real, named source location — never invented.
- *Consequences* must state liabilities/tradeoffs, not only benefits.
- *Related Patterns* situates the entry in its family so callers compose rather
  than duplicate.
- The *Checklist* is the conformance oracle: it must be mechanically verifiable.
- Keep entries language-honest: if the mechanism is language-specific, say so.
