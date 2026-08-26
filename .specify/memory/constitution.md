<!--
Sync Impact Report
- Version change: 1.0.1 → 1.1.0
- Modified principles: none
- Added governance requirements:
  - Architecturally significant decisions must be recorded as ADRs.
  - Plans must comply with applicable accepted ADRs and should reference them.
- Added sections: none
- Removed sections: none
- Follow-up TODOs: none
-->

# VPS Romero Constitution

## Core Principles

### I. Declarative Desired State

- Git MUST be the source of truth for infrastructure and service desired state.
- Intended changes MUST be represented declaratively in the repository before they are applied to
  a managed host.
- Secrets, credentials, private keys, and mutable runtime data MUST remain outside Git and use an
  appropriate secure mechanism.
- Emergency manual changes MUST be reconciled afterward with the intended repository state.
- Ad-hoc production changes MUST NOT become undocumented long-term state.

These rules keep managed state reviewable and prevent production from drifting away from its
recorded intent.

### II. Reproducibility and Idempotence

- Automation MUST repeatedly converge a compatible fresh host toward the declared desired state.
- Reapplying an already-converged configuration MUST NOT cause unintended changes.
- External software and dependencies SHOULD be pinned and verified with checksums where practical.
- Mutable references such as `latest` MUST NOT silently change production software when a stable
  version can reasonably be pinned.
- Important state that cannot be reproduced from the repository MUST be identified, persisted
  appropriately, and covered by backup or recovery planning.

These rules make rebuilding and recovering a host predictable.

### III. Least Privilege and Minimal Exposure

- Services MUST use only the privileges reasonably required.
- Long-running services SHOULD use dedicated unprivileged identities unless a documented technical
  requirement prevents it.
- Public inbound network access MUST be unavailable unless an explicitly supported use case requires it.
- Only required public ports and interfaces MAY be exposed.
- Administrative interfaces SHOULD remain private and use secure administrative channels.
- Secrets MUST NOT enter Git or generated artifacts intended for version control.

These rules reduce the impact of mistakes and compromise.

### IV. Safe and Recoverable Change

- A repository change and its production deployment MUST be separate actions.
- Implementation work MUST NOT implicitly authorize production deployment.
- Changes to recovery-critical facilities MUST preserve a viable administrative recovery path.
  This includes authentication, privilege escalation, firewalling, networking, boot behavior,
  storage, and remote access.
- Destructive or difficult-to-reverse operations MUST be explicit and justified.
- Infrastructure MUST favor changes that can be reviewed, tested, rolled back, or reconstructed.

These rules protect access and recovery while changes are introduced.

### V. Verification Before Completion

- Infrastructure work MUST have objective verification appropriate to the change.
- Static validation, linting, syntax checks, dry runs, check modes, idempotency checks, service
  health checks, and post-deployment checks MUST be used where technically meaningful.
- Installation or an active process alone MUST NOT count as proof that a feature works.
- Feature specifications MUST define observable success criteria from the user or operator view.
- Validation MUST emphasize boundaries that can cause outages, lockouts, data loss, unwanted public
  exposure, or silent drift.

These rules define completion through observed behavior rather than assumed success.

## Engineering Constraints

- The simplest architecture that satisfies the requirements SHOULD be preferred.
- Additional infrastructure layers MUST solve a concrete requirement and MUST NOT be added only
  because they are fashionable or commonly considered modern.
- Native, well-supported operating-system facilities SHOULD be preferred when they are simpler and
  sufficiently robust.
- Mutable application state SHOULD be separate from declarative configuration where practical.
- Long-running services SHOULD use the host's normal service management and logging facilities
  unless a feature documents a justified alternative.
- Observability MUST let an operator assess health and diagnose common failures remotely.
- Specifications SHOULD describe outcomes and constraints without prematurely choosing technology.
  Technology choices belong primarily in planning unless they are themselves requirements.

## Development and Deployment Workflow

- Feature work MUST follow a Spec Kit workflow proportionate to its size and risk.
- Specifications MUST define required behavior and acceptance criteria.
- Plans MUST define implementation architecture and technology choices.
- Architecturally significant decisions that are durable, cross-cutting, difficult to reverse, or
  expected to constrain future work MUST be recorded as Architecture Decision Records.
- Plans MUST comply with applicable accepted ADRs and SHOULD reference the ADRs that materially
  constrain their design.
- Tasks MUST divide an approved plan into reviewable units of work.
- Implementation MUST follow the current specification, plan, this constitution, and applicable
  repository agent guidance.
- Infrastructure changes SHOULD be implemented and validated in small, reviewable increments.
- Before production deployment, relevant repository validation MUST pass and the intended change
  MUST be reviewable.
- After production deployment, observable acceptance criteria MUST be verified on the managed host.
- Discoveries that invalidate requirements or architectural assumptions MUST update the relevant
  specification or plan rather than remain hidden in implementation.

## Governance

- This constitution governs project-wide engineering decisions and supersedes conflicting
  feature-specific conventions.
- `AGENTS.md` provides operational guidance but MUST NOT redefine or weaken these principles.
- Feature specifications and plans MAY impose stricter requirements.
- Exceptions to a constitutional MUST or MUST NOT require explicit documented justification.
- Recurring exceptions SHOULD prompt an amendment instead of becoming customary workarounds.
- Amendments MUST be deliberate, reviewed as project-level governance changes, and accompanied by
  required updates to affected guidance, specifications, or implementation.
- Reviews MUST check relevant work for constitutional compliance. Unjustified noncompliance MUST be
  resolved before the work is considered complete.
- Constitution versions MUST use semantic versioning:
  - MAJOR for removing or incompatibly redefining a governing principle.
  - MINOR for adding a principle or materially expanding governance.
  - PATCH for clarifications that do not change meaning.

**Version**: 1.1.0 | **Ratified**: 2026-08-26 | **Last Amended**: 2026-08-26
