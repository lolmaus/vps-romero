# Implementation Plan: Manage VPS Baseline

**Branch**: `001-manage-vps-baseline` | **Date**: 2026-08-27 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-manage-vps-baseline/spec.md`

## Summary

Manage the existing Ubuntu 26.04 LTS VPS with ansible-core 2.21.3 running from the operator workstation on a supported control-node Python (3.12–3.14) over the existing SSH transport. Resolve the complete local Python tooling graph from a committed uv lockfile. Provide thin connectivity and bootstrap playbooks over a YAML inventory, with a reusable `vps_baseline` role that validates the target, inventories Docker state through read-only built-in modules, blocks on unexpected or ambiguous state, simulates the exact APT removal transaction, and then removes only verified Docker-specific packages, services, repository configuration, configuration, and data. Preserve SSH and provider-owned facilities, prove outbound and inbound connectivity after application, and require an immediate zero-change second run.

## Technical Context

**Language/Version**: YAML for Ansible content; control-node Python 3.12–3.14; target Python 3.14; `ansible-core==2.21.3`

**Primary Dependencies**: `uv`, locked `ansible-core==2.21.3` and `ansible-lint==26.8.0`, OpenSSH client; target system Python 3.14 and pre-existing `python3-apt`; no external Ansible collections

**Storage**: Version-controlled YAML, TOML, INI, and Markdown files only; no database, secrets, credentials, or target runtime data stored in the repository

**Testing**: `ansible-playbook --syntax-check`, `ansible-lint`, read-only connectivity playbook, `--check --diff`, first explicitly authorized application, post-application SSH/DNS/outbound checks, and immediate second application with `changed=0`

**Target Platform**: Existing Ubuntu 26.04 LTS host at `romero.lolma.us` with Python 3.14, managed from a POSIX local workstation over SSH

**Project Type**: Single-host declarative infrastructure repository using reusable Ansible roles and thin playbook entry points

**Performance Goals**: No throughput target; safety and deterministic convergence take priority for one host. Read-only preflight must finish before any destructive task starts.

**Constraints**: Do not provision the VPS or manage DNS, provider networking, netplan, cloud-init, SSH authentication, or sshd; use root only in bootstrap-era entry points; prefer `ansible.builtin`; fail closed on Docker inspection ambiguity; no deployment during planning or implementation without separate authorization

**Scale/Scope**: One existing VPS, one minimal baseline role, one Docker cleanup workflow, and a repository shape that permits independent future service roles and playbooks

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

### Pre-Research Gate

| Principle | Status | Plan Evidence |
|-----------|--------|---------------|
| Declarative Desired State | PASS | Inventory, tool versions, Docker absence, and any reviewed cleanup approvals are repository declarations; secrets and SSH material remain external. |
| Reproducibility and Idempotence | PASS | The complete Python tooling graph is committed in `uv.lock`; modules declare absence; the validation contract requires an immediate second run with zero changes. |
| Least Privilege and Minimal Exposure | PASS | Existing root access is restricted to the two bootstrap-era playbooks; no global root transport, new listener, port, account, or service is introduced. |
| Safe and Recoverable Change | PASS | All Docker discovery and APT simulation precede changes; ambiguous state blocks; SSH, networking, and cloud-init are explicitly untouched; deployment is separate. |
| Verification Before Completion | PASS | Syntax, lint, connectivity, check mode, post-apply access/network verification, Docker absence, and idempotence are all required. |
| Engineering Simplicity | PASS | One built-in-only role and two thin playbooks avoid external collections, agents, and provider integrations. |
| ADR Compliance | PASS | Accepted ADR-0001 records the user-mandated, cross-cutting Ansible architecture, and this plan conforms to it. |

No constitutional violations require complexity exceptions.

### Post-Design Gate

| Principle | Status | Design Confirmation |
|-----------|--------|---------------------|
| Declarative Desired State | PASS | The host inventory, exact artifact approval manifest, role policy, and operator commands have explicit contracts. |
| Reproducibility and Idempotence | PASS | Package/file/service absence uses idempotent modules; read-only exceptions report no change; check-mode and second-run behavior are specified. |
| Least Privilege and Minimal Exposure | PASS | Root is play-scoped to connectivity/bootstrap and no future service transport is selected by this feature. |
| Safe and Recoverable Change | PASS | The state machine prevents package, service, or data mutation until Docker state and APT impact are conclusively safe. |
| Verification Before Completion | PASS | `quickstart.md` covers all acceptance signals without performing deployment as part of repository validation. |
| ADR Compliance | PASS | Accepted ADR `docs/adr/0001-use-ansible-for-host-configuration.md` records the durable project-wide decision. |

The Phase 1 design introduces no gate failures.

## Project Structure

### Documentation (this feature)

```text
specs/001-manage-vps-baseline/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── operator-interface.md
└── tasks.md                       # Generated by $speckit-tasks
```

### Source Code (repository root)

```text
.ansible-lint
.gitignore
ansible.cfg
pyproject.toml
uv.lock
inventory/
├── hosts.yml
└── host_vars/
    └── romero.yml                 # Non-secret, exact Docker review approvals only
playbooks/
├── connectivity.yml              # Read-only bootstrap transport and target checks
└── bootstrap.yml                 # Thin entry point for vps_baseline
roles/
└── vps_baseline/
    ├── defaults/
    │   └── main.yml               # Empty approval-manifest defaults
    ├── meta/
    │   └── argument_specs.yml     # Approval-manifest shape validation
    ├── tasks/
    │   ├── main.yml               # Ordered task imports only
    │   ├── preflight.yml          # OS, Python, access, package/service/file facts
    │   ├── inspect_docker.yml     # Read-only API, filesystem, source, and APT inspection
    │   ├── remove_docker.yml      # Gated service, package, source, config, and data absence
    │   └── verify.yml             # Absence, new SSH session, DNS, and outbound checks
    └── vars/
        └── main.yml               # Non-overridable package/path/service policy allowlists
```

**Structure Decision**: Keep inventory identity separate from play-scoped connection identity. Both current playbooks set `remote_user: root` because they are the initial bootstrap interface, while `ansible.cfg` and inventory set no global remote user. Future features add independent roles and thin playbooks and may compose them without modifying `vps_baseline` or inheriting root transport.

## Design

### Local toolchain

- Declare ansible-core 2.21.3 and ansible-lint 26.8.0 in `pyproject.toml` with `requires-python = ">=3.12,<3.15"`; do not add a `.python-version` that selects one workstation Python minor.
- Generate and commit `uv.lock` so the complete Python tooling dependency graph is reproducible across its supported markers. Use `uv sync --locked` and `uv run --locked` in operator workflows so validation cannot silently update the lockfile.
- Configure only repository-local inventory and role paths in `ansible.cfg`; retain SSH host-key checking and do not weaken SSH options.
- Ignore `.venv/`, repository-local Ansible caches, and retry files without changing unrelated ignore rules.
- Configure ansible-lint to lint both playbooks and roles, use fully qualified module names, and introduce no warning skips for planned content.

### Connectivity and target preflight

- Inventory one host named `romero` with `ansible_host: romero.lolma.us` and `/usr/bin/python3`; store no key path, password, or SSH option.
- `playbooks/connectivity.yml` is read-only and play-scopes `remote_user: root`. It verifies workstation DNS resolution, Ansible ping, fact gathering, root effective UID, Ubuntu distribution/version, target Python 3.14, remote name resolution, and outbound HTTPS connectivity.
- `playbooks/bootstrap.yml` repeats all destructive prerequisites inside the role so a caller cannot bypass safety by skipping the connectivity playbook.
- A missing `python3-apt` dependency blocks before cleanup. APT module dependency auto-installation is disabled so the minimal baseline does not acquire incidental packages.

### Docker discovery and approval

- Gather installed-package and service facts and find candidate Docker repository, key, configuration, systemd override, data, runtime, rootless, and per-user paths without reading secret file contents.
- If Docker is already completely absent, skip daemon inspection and all mutations and report no change.
- When a Docker Unix socket is available, query `/version`, prove the daemon supports Engine API v1.52, and use explicitly versioned read-only `ansible.builtin.uri` requests for all containers, images, volumes, custom networks, build-cache usage, swarm state, services, configs, secrets, and plugins. Require the v1.52 verbose `BuildCacheUsage` schema and fail closed rather than treating a missing or ambiguous field as empty. Read-only inspection tasks run even under Ansible check mode and always report `changed: false`.
- When Docker-related packages or files exist but no daemon can be inspected, stop. Do not start Docker for inspection.
- Normalize discovered identifiers and paths into the model in `data-model.md`. Any nonempty unexpected-state set must equal the host's exact approval manifest before cleanup can proceed. The default manifest is empty; no blanket force option exists.
- Treat the canonical built-in networks (`bridge`, `host`, `none`) as installation scaffolding, not custom network state. Any other network blocks unless exactly reviewed.

### Package and service safety

- Build the removal request from the intersection of installed packages and an explicit Docker-only allowlist. Never pass wildcards.
- Classify distribution `containerd`, `runc`, and `podman-docker` as shared-risk and leave them untouched. Include Docker's bundled `containerd.io` only after fixed-argument, read-only `ctr` commands successfully enumerate every containerd namespace and relevant state category, every namespace is one reported by Docker, no active transfer exists, and the APT simulation finds no unrelated dependent package. Failure to inspect does not start containerd and blocks cleanup.
- Run `apt-get --simulate purge` through one read-only `ansible.builtin.command` task using `argv`; parse the proposed `Remv` set and assert it contains no package outside the approved Docker removal set.
- Stop and disable only present Docker-specific services after all gates pass. Stop `containerd` only when its selected package and data have been proved Docker-owned.
- Purge the exact package list with `ansible.builtin.apt`, `autoremove: false`, and module dependency auto-installation disabled.

### Repository, configuration, and data cleanup

- Remove canonical dedicated Docker APT source files only after verifying that they contain no unrelated stanza. A Docker URI in a mixed/noncanonical source blocks for review rather than causing whole-file deletion.
- Remove a Docker key only after no remaining source refers to it; remove Docker-specific cached APT list files without refreshing or upgrading unrelated packages.
- Remove `/etc/docker`, Docker-only systemd overrides/default files, `/var/lib/docker`, and Docker runtime paths after approval. Remove `/var/lib/containerd` only when the containerd namespace/state inventory and APT transaction together prove `containerd.io` is exclusively Docker-owned; nonempty Docker-owned residual containerd state also requires exact path review.
- Inspect Docker group membership and rootless/per-user paths. Nonempty membership or user-owned paths block for exact review; content and credentials are never logged.
- Do not call firewall, netplan, network, cloud-init, user, authorized-key, or sshd management modules. Docker runtime networking may disappear when Docker stops, but the role declares no provider/network configuration.

### Verification and convergence

- In normal mode, regather facts and assert that all selected Docker packages, services, source/key files, configuration, runtime paths, and approved data artifacts are absent.
- In check mode, rely on module change predictions and skip post-removal absence assertions that cannot be true until an apply occurs.
- Reset the Ansible connection and run a new ping to prove a new SSH session works.
- Repeat workstation hostname resolution and verify target-side DNS plus outbound HTTPS connectivity.
- Reapplying the bootstrap playbook after a successful apply must report `changed=0`; read-only discovery and verification tasks always report unchanged.

## ADR Review

Using Ansible from the operator workstation over SSH, storing non-secret desired state and inventory in Git, and limiting its boundary to configuration inside already-provisioned hosts is a durable, cross-cutting choice recorded in Accepted ADR `docs/adr/0001-use-ansible-for-host-configuration.md`. Role/playbook layout and module-selection conventions remain implementation-plan and repository-guidance concerns.
