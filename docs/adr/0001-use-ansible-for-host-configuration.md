# ADR-0001: Use Ansible for Host Configuration

- Status: Accepted
- Date: 2026-08-27

## Context

The repository needs a declarative, repeatable way to configure existing hosts from an operator workstation over existing SSH access. It must keep host configuration ownership distinct from provisioning and provider-managed infrastructure.

This choice is durable and cross-cutting: it establishes the configuration-management mechanism, transport, source of truth, credential boundary, and infrastructure ownership boundary for future host and service features.

## Decision

Use Ansible as the host configuration-management mechanism. Run it from the operator workstation over SSH.

Keep non-secret desired state and inventory in Git. Keep credentials and private SSH configuration external to the repository.

Use Ansible to manage configuration inside already-provisioned hosts. Do not use it to provision or manage VPS lifecycle, DNS, provider networking, cloud-init, or provider resources.

## Consequences

- Non-secret infrastructure and service desired state and inventory become reviewable Ansible content in Git.
- SSH remains the transport, so connection credentials, host-key trust, and recovery access remain operator/provider responsibilities.
- Credentials and private SSH configuration are not version-controlled by this repository.
- Ansible changes begin at the operating-system configuration boundary of an already-provisioned host.
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
