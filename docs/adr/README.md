# Architecture Decision Records

This directory contains project-wide Architecture Decision Records (ADRs).

ADRs capture architecturally significant decisions that are durable, cross-cutting, difficult to reverse, or expected to constrain future work. They complement Spec Kit feature artifacts: feature research and plans explain decisions in the context of one feature, while ADRs preserve decisions that future features must continue to understand.

## Workflow

1. During feature planning, inspect existing accepted ADRs that may constrain the design.
2. After `$speckit-plan` and before `$speckit-tasks`, review `research.md` and `plan.md` for new ADR-worthy decisions.
3. Create a new ADR from `0000-template.md` when a decision is architecturally significant.
4. New ADRs start as `Proposed` and require user review before becoming `Accepted`.
5. Reference applicable accepted ADRs from the feature plan.
6. Do not rewrite an accepted ADR to represent a later decision. Create a new ADR that supersedes it.

Routine implementation details do not require ADRs.

## Statuses

- `Proposed` — under consideration and not yet binding.
- `Accepted` — approved and applicable to future work.
- `Superseded` — replaced by a newer ADR; retain for history.
- `Deprecated` — no longer recommended or applicable, without a direct replacement.
- `Rejected` — considered but explicitly not adopted.

## Naming

Use zero-padded sequential numbers and a concise kebab-case title:

```text
0001-use-ansible-for-host-configuration.md
0002-manage-secrets-with-example-tool.md
```

`0000-template.md` is reserved as the template and is not itself an architectural decision.
