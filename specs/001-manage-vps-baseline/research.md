# Phase 0 Research: Manage VPS Baseline

**Date**: 2026-08-27

## ansible-core and Python compatibility

**Decision**: Use `ansible-core==2.21.3` on any supported control-node Python from 3.12 through 3.14. Require Python 3.14 only on the managed Ubuntu 26.04 host.

**Rationale**: ansible-core 2.21 is the current generally available feature line as of 2026-08-27. Its published support matrix covers Python 3.12–3.14 on the control node and Python 3.9–3.14 on target nodes, so requiring Python 3.14 on the workstation would add no feature benefit while excluding supported operator environments. The selected release directly supports the Ubuntu 26.04 target's required Python 3.14 runtime. Version 2.21.3, released 2026-08-10, is the latest stable patch in that line. The 2.21 line receives general fixes until November 2026, security fixes until May 2027, and reaches end of life in November 2027.

**Alternatives considered**:

- `ansible-core==2.20.8`: supports target Python 3.14, but it has already moved beyond general bug-fix maintenance and has a shorter remaining lifecycle.
- ansible-core 2.22: would extend Python coverage and lifecycle, but it is not scheduled for GA until November 2026 and is therefore not a current supported release for this plan.
- The full Ansible community package: unnecessary because the feature is deliberately limited to `ansible.builtin` content.

**Sources**:

- [ansible-core release and maintenance matrix](https://docs.ansible.com/projects/ansible-core/devel/reference_appendices/release_and_maintenance.html)
- [ansible-core 2.21.3 release metadata](https://pypi.org/project/ansible-core/2.21.3/)
- [ansible-core 2.21 changelog](https://github.com/ansible/ansible/blob/stable-2.21/changelogs/CHANGELOG-v2.21.rst)

## Reproducible local toolchain

**Decision**: Declare ansible-core 2.21.3 and ansible-lint 26.8.0 in `pyproject.toml`, set the control-Python compatibility range to 3.12–3.14, and commit the complete dependency resolution in `uv.lock`. Use `uv sync --locked` and `uv run --locked` so local setup and validation fail rather than silently rewriting a missing or stale lockfile.

**Rationale**: Version 26.8.0 is the current stable ansible-lint release as of 2026-08-27. Its published metadata accepts maintained ansible-core versions and supports the selected control-Python range. Exact direct requirements alone do not constrain transitive dependencies, while uv's cross-platform lockfile records the exact resolved package versions for applicable Python and platform markers and is intended to be committed. This provides a small, modern local workflow without a container build or separately maintained requirements export. The repository does not pin one workstation Python minor with `.python-version`; `requires-python = ">=3.12,<3.15"` preserves the full control-node range supported by ansible-core 2.21.

**Alternatives considered**:

- An unpinned latest ansible-lint: rejected because validation behavior could change without a repository change.
- Exact top-level pins in `requirements.txt`: rejected because transitive dependencies could still change between workstation installations.
- Hand-maintained fully pinned requirements: rejected because it duplicates resolver output and is harder to update safely across supported Python versions and platforms.
- A containerized execution environment: rejected because it adds a build and distribution layer that this one-host baseline does not require.
- ansible-lint as a remote or hosted-only check: rejected because the operator must be able to validate locally before deployment.

**Sources**:

- [ansible-lint 26.8.0 release](https://github.com/ansible/ansible-lint/releases/tag/v26.8.0)
- [ansible-lint 26.8.0 package metadata](https://pypi.org/project/ansible-lint/26.8.0/)
- [uv project structure and lockfile](https://docs.astral.sh/uv/concepts/projects/layout/)
- [uv locked command behavior](https://docs.astral.sh/uv/reference/cli/)

## Ansible content boundaries

**Decision**: Use fully qualified `ansible.builtin` modules for facts, assertions, Docker API reads, package removal, service control, filesystem cleanup, and network verification. Use `ansible.builtin.command` only for two narrowly scoped read-only inspections: the APT purge simulation, because no built-in module exposes the complete dependent-removal set for an absent-state operation, and `ctr` inventory commands, because Ansible has no built-in containerd API module.

**Rationale**: Built-in modules provide native idempotence and check-mode behavior. `package_facts`, `service_facts`, `uri`, `apt`, `systemd_service`, `file`, `find`, `stat`, `getent`, `assert`, and `meta` cover the workflow except for inspecting APT's exact proposed removal transaction and querying containerd's gRPC API. Both command exceptions use fixed `argv`, never a shell, and set `changed_when: false` and `check_mode: false` so the read-only safety gates run during dry runs. The APT purge itself remains an `ansible.builtin.apt` task with `autoremove: false`.

The role will set `auto_install_module_deps: false` for APT operations. If the target lacks `python3-apt`, preflight fails rather than silently installing a package outside the minimal baseline.

**Alternatives considered**:

- Shell pipelines for package and Docker discovery: rejected because structured facts and API responses are safer and easier to validate.
- `community.docker` modules: rejected because they add a collection and Python SDK dependency solely to remove Docker.
- Trusting APT to remove only named packages: rejected because dependent removals could affect unrelated provider functionality.

**Sources**:

- [`ansible.builtin.package_facts`](https://docs.ansible.com/projects/ansible-core/devel/collections/ansible/builtin/package_facts_module.html)
- [`ansible.builtin.apt`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/apt_module.html)
- [`ansible.builtin.command`](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/command_module.html)
- [Ansible check-mode behavior](https://docs.ansible.com/projects/ansible-core/devel/playbook_guide/playbooks_checkmode.html)

## Docker state inspection and review gate

**Decision**: Inspect the running Docker daemon through its local Unix socket with read-only `ansible.builtin.uri` calls. Query `/version` first, require every daemon to support Engine API v1.52, and use explicitly versioned URLs for all later requests. Inventory containers, images, volumes, custom networks, build cache, swarm state, and other daemon-visible state before stopping services or deleting anything. Read build cache from the v1.52 verbose `BuildCacheUsage` schema and fail closed when that schema is missing or ambiguous. Fail closed when state is present, inspection is unavailable, or any result is ambiguous.

**Rationale**: The Docker Engine API returns structured data and `ansible.builtin.uri` supports Unix sockets, avoiding both shell parsing and an external Docker collection. The inspection will cover at least:

- all containers, including stopped containers;
- all images;
- all volumes;
- networks other than Docker's built-in `bridge`, `host`, and `none` networks;
- build-cache usage;
- swarm, plugin, secret, config, and service state where exposed by the installed daemon;
- rootful and rootless Docker sockets, configuration paths, and data paths discoverable without reading secret contents.

If unexpected state is found, the role stops before service or package changes. There is no blanket force flag. To continue after review, the operator records the exact approved artifact identifiers or paths in the host-scoped declarative approval manifest. A later run requires the discovered set to match that manifest exactly; added, removed, or changed artifacts invalidate the approval and stop the run. Paths that may contain registry credentials are reported by path only, never read into logs or committed.

If Docker packages or data exist but no usable daemon socket is available, the role does not start Docker merely to inspect it. It stops for review because offline data cannot be classified safely.

**Alternatives considered**:

- Checking only `docker ps`: rejected because it misses stopped containers, images, volumes, networks, and build cache.
- Treating “no containers” as authorization to delete all Docker data: rejected by the clarified specification.
- Automatically starting an inactive Docker daemon for inspection: rejected because startup can alter runtime networking and state.
- A blanket operator confirmation boolean: rejected because it could authorize deletion of state that appeared after review.

**Sources**:

- [Docker Engine API reference](https://docs.docker.com/reference/api/engine/)
- [Docker Engine API system data-usage endpoint](https://docs.docker.com/reference/api/engine/version/v1.52/)
- [Docker custom-network filtering](https://docs.docker.com/reference/cli/docker/network/ls/)
- [`ansible.builtin.uri` Unix-socket support](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/uri_module.html)

## Docker package and filesystem removal

**Decision**: Purge only an explicit allowlist of installed Docker-specific packages, after comparing a read-only APT simulation with that allowlist. Never use package wildcards or APT autoremove. Remove only verified Docker-owned repository, key, configuration, cache, runtime, and data paths after the state-review gate passes. Before selecting `containerd.io` or `/var/lib/containerd`, use the installed `ctr` client read-only to enumerate containerd namespaces and state. Require every namespace to be one reported by Docker, reject active transfers, and fail closed if the daemon is unavailable or exclusive Docker ownership cannot be proved without starting it.

**Rationale**: Docker's Ubuntu documentation distinguishes Docker Engine packages from distribution packages and documents that package purge does not remove images, volumes, configuration, `/var/lib/docker`, `/var/lib/containerd`, the Docker APT source, or its key. The initial allowlist will cover the official Docker Engine packages and unambiguously Docker-specific Ubuntu package variants. Shared runtimes such as distribution `containerd` and `runc`, and compatibility packages such as `podman-docker`, are not automatically removed. Docker's bundled `containerd.io` and `/var/lib/containerd` are removable only when inspection and the APT transaction prove that no unrelated consumer is affected.

Repository cleanup will remove a canonical dedicated Docker source file only after verifying its contents. A Docker entry found in a mixed or noncanonical source file stops for review instead of deleting unrelated repository configuration. A Docker signing key is removed only after no remaining APT source references it. The same exact-ownership rule applies to configuration, systemd overrides, group membership, rootless state, and per-user Docker paths.

The role stops existing Docker services only after all discovery, approval, and APT-transaction gates pass. It does not manipulate firewall, netplan, provider networking, SSH, or cloud-init. Docker-created runtime networking is allowed to disappear as a consequence of stopping and removing Docker, but no firewall or provider-network configuration is declared or edited directly.

**Alternatives considered**:

- Docker's documented unconditional purge and recursive deletion commands: rejected as direct implementation because they do not check for operator state or unrelated consumers first.
- `autoremove: true`: rejected because it can remove provider dependencies outside the declared Docker allowlist.
- Broad filename or package-name patterns: rejected because they can capture unrelated state.

**Source**:

- [Docker Engine installation and uninstall instructions for Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [containerd namespaces and multi-tenancy](https://github.com/containerd/containerd/blob/main/docs/namespaces.md)

## Repository composition and transport

**Decision**: Use a YAML inventory, thin connectivity and bootstrap playbooks, and one reusable `vps_baseline` role split into preflight, inspection, removal, and verification task files. Put `remote_user: root` only on the two bootstrap-era playbook entry points; do not set a global or inventory-wide root user.

**Rationale**: This keeps the existing root SSH path available for the initial baseline without making root the transport contract for future service features. Later services can add independent roles and thin entry points without editing the baseline role. SSH keys and SSH client configuration remain operator-managed outside the repository.

**Alternatives considered**:

- A monolithic playbook: rejected because future service features would become coupled to bootstrap cleanup.
- Global `remote_user = root`: rejected because it would silently bind future services to the bootstrap identity.
- Creating a replacement administrative user now: rejected because SSH access redesign is explicitly out of scope.

## Verification and deployment separation

**Decision**: Provide local syntax and lint checks, a read-only connectivity playbook, check mode with diff, an explicitly invoked apply, post-apply network/access checks, and an immediate second apply requiring zero changes. Planning and implementation do not invoke the apply command.

**Rationale**: These layers prove repository validity, target compatibility, cleanup safety, continued access, and convergence while preserving the constitutional separation between repository changes and production deployment. Read-only Docker/API and APT-simulation tasks are explicitly allowed to execute during check mode; post-mutation absence checks are skipped in check mode because the predicted changes have not actually occurred.

**Alternatives considered**:

- Syntax and lint only: rejected because they cannot prove host safety or idempotence.
- Automatic deployment after validation: rejected because deployment requires separate explicit authorization.
- Treating check mode as proof of convergence: rejected because check mode predicts rather than applies state.
