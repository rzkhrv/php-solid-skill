---
name: php-solid-skill
description: Apply SOLID to PHP design, refactoring, and code review. Use when evaluating class responsibilities, extension points, polymorphic contracts, interface boundaries, or dependency direction.
---

# PHP SOLID

Read the relevant rules before applying them. For a full SOLID audit, read all five. Base recommendations on a concrete consequence, a broken contract, or a confirmed design requirement. Code shape alone is insufficient.

## Before analysis

- Check the project's PHP version, conventions, callers, and tests. Distinguish evidence from assumptions.
- Preserve observable behavior and public contracts during refactoring unless a change is agreed.
- Prefer the smallest useful fix. Avoid speculative abstractions and unrelated cleanup.
- Run relevant available checks; state what could not be verified.

## Rules

| Rule | Read when |
| --- | --- |
| [SRP](rules/solid-srp.md) | Unrelated stakeholders or requirements change one module independently. |
| [OCP](rules/solid-ocp.md) | Independent variants are a confirmed requirement or already cause repeated edits to stable consumer logic. |
| [LSP](rules/solid-lsp.md) | An implementation breaks an interface's behavioral promises. |
| [ISP](rules/solid-isp.md) | Clients depend on unrelated or unsupported capabilities. |
| [DIP](rules/solid-dip.md) | Policy depends on infrastructure details instead of a suitable boundary. |

## Review output

For each actionable finding, give the location, the concrete consequence, and the smallest useful fix. Rank findings by impact. Include the rule ID, severity, or verification steps when they help the reader act or when requested. Label unverified concerns as hypotheses and say when there are no actionable findings.
