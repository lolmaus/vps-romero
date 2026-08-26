# ADR-0001: Use Ansible for Host Configuration

- Status: Proposed
- Date: 2026-08-27

## Context

The repository needs a declarative, repeatable way to configure the existing `romero.lolma.us` VPS from an operator workstation over its existing SSH access. The configuration architecture must support independently developed future services, preserve a separation between repository changes and deployment, and avoid taking ownership of VPS lifecycle or provider-managed infrastructure.

This choice is durable and cross-cutting: it determines how future host and service features express desired state, compose changes, validate content, and connect to managed hosts.

## Decision

Use a pinned, supported ansible-core toolchain running from the operator's local workstation over SSH for host configuration management.

Organize Ansible content as reusable roles with thin playbook entry points. Prefer fully qualified `ansible.builtin` modules; use command execution only when no built-in module provides the required behavior and document each exception. Keep host inventory and non-secret desired-state variables in the repository while keeping credentials and private SSH configuration external.

Do not use Ansible to provision or manage VPS lifecycle, DNS, provider networking, cloud-init, or provider resources unless a later accepted decision explicitly expands that boundary.

## Consequences

- Infrastructure and service desired state becomes reviewable, repeatable Ansible content in Git.
- Future service features add independent roles and thin entry points instead of extending a monolithic bootstrap script.
- The repository must pin and periodically review ansible-core and validation-tool versions.
- Each supported target must provide a Python version compatible with the selected ansible-core release.
- SSH remains the transport, so connection credentials, host-key trust, and recovery access remain operator/provider responsibilities.
- Safe use of root or privilege escalation must be scoped by each feature; this decision does not make root the default for future services.
- Provider lifecycle and networking remain outside the configuration-management boundary.

## Alternatives Considered

### Ad-hoc shell scripts

Rejected because they provide weaker state modelling, check-mode behavior, linting, module-level idempotence, and reusable composition for future services.

### Agent-based configuration management

Rejected because it would introduce a persistent management service, credentials, and lifecycle on a single existing VPS without a requirement that justifies that complexity.

### Provider-native provisioning as the source of truth

Rejected because VPS provisioning and provider infrastructure are explicitly outside repository scope, while the required changes concern configuration inside an already-existing host.

## References

- [Feature specification](../../specs/001-manage-vps-baseline/spec.md)
- [Implementation research](../../specs/001-manage-vps-baseline/research.md)
- [Implementation plan](../../specs/001-manage-vps-baseline/plan.md)
