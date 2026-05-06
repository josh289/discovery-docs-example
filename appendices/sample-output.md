# Appendix E — Sample Output

For new readers: start with this representative artifact to understand the audit's depth and citation style.

**Recommended starting artifact**: [C4 Component — service-bug](../c4/component-bug.md)

Why this one: it shows the C4 component view of the system's central aggregate (`service-bug`) — one Mermaid diagram, ~20+ components (command handlers, query handlers, read models, event subscriptions, policies), each with `[source:]` citations back to the architecture phase. Reading this first gives a non-engineer reader a sense of how the audit weaves diagrams + prose + traceability without being overwhelming.

## Reading order for the full audit

1. `audit-output/executive-summary.md` — 1-page brief (three surprises, recommended path forward)
2. [C4 Component — service-bug](../c4/component-bug.md) — feel for depth + format (this artifact)
3. `audit-output/sow.md` — phased plan, story points, and cost estimates
4. `audit-output/risk-register.md` — what could go wrong (20 risks, grouped by tribal/hidden/wire-format/operational)
5. `audit-output/permission-matrix.md` — who can do what (role × operation × policy)
6. The remaining artifacts as needed (see [Citation Index](./citation-index.md) for the full map)
