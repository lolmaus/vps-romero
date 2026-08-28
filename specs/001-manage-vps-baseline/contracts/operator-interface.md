# Operator Interface Contract

The repository exposes local commands and two Ansible playbook entry points. Commands are run from the repository root. Credentials and SSH client configuration remain outside the repository.

## Toolchain Contract

Use uv with any control-node Python from 3.12 through 3.14 to synchronize the exact dependency graph recorded in the committed lockfile:

```bash
uv sync --locked
```

The command must fail if `uv.lock` is missing or inconsistent with `pyproject.toml`, rather than updating it. The installed environment must use control-node Python 3.12–3.14 and report ansible-core 2.21.3 and ansible-lint 26.8.0:

```bash
uv run --locked python --version
uv run --locked ansible --version
uv run --locked ansible-lint --version
```

## Static Validation Contract

These commands must exit zero and must not connect to the VPS:

```bash
uv run --locked ansible-playbook --syntax-check playbooks/connectivity.yml
uv run --locked ansible-playbook --syntax-check playbooks/bootstrap.yml
uv run --locked ansible-lint
```

## Connectivity Contract

```bash
uv run --locked ansible-playbook playbooks/connectivity.yml --limit romero
```

This command is read-only. It must:

- resolve `romero.lolma.us` from the workstation;
- establish the existing root SSH connection without modifying SSH configuration;
- verify effective root access, Ubuntu 26.04, target Python 3.14, and pre-existing APT module support;
- verify target-side external name resolution and outbound HTTPS connectivity;
- exit nonzero before persistent change if any prerequisite fails.

## Dry-Run Contract

```bash
uv run --locked ansible-playbook playbooks/bootstrap.yml --limit romero --check --diff
```

Read-only Docker API inventory and APT transaction simulation still execute and report unchanged. Mutating built-in modules predict their changes. The command stops nonzero if Docker state is unexpected or ambiguous, the exact approval manifest does not match discovery, or APT proposes removing anything outside the Docker-only request.

Every discovered relevant rootful or rootless Docker daemon socket is queried.
Docker-bearing state with no usable socket, or with any socket that cannot be
fully inspected, is a non-mutating `blocked-for-review` result; the workflow
does not start a daemon to bypass that stop.

The workflow first reads each daemon's `/version` response, proves support for
Engine API v1.52, and then uses explicitly versioned endpoints. Its verbose
build-cache response must contain an unambiguous v1.52 `BuildCacheUsage`
object. When `containerd.io` is installed, fixed-argument read-only `ctr`
inventory must prove that every namespace belongs to Docker and that no active
content transfer exists before the package or `/var/lib/containerd` can be
selected for removal.

Check mode does not prove that post-removal absence checks pass because predicted changes are not applied.

## Docker Review Contract

The host-scoped approval input has this shape and defaults to empty sets:

```yaml
vps_baseline_docker_approved_state:
  containers: []
  images: []
  volumes: []
  custom_networks: []
  build_cache: []
  swarm_objects: []
  plugins: []
  paths: []
  group_members: []
```

When inspection reports unexpected state, the operator reviews it and records only the exact identifiers or paths approved for removal. Every nonempty approval set must exactly equal current discovery. There is no force flag or wildcard approval. File contents and credentials must not be placed in this manifest.

## Apply Contract

Deployment is a separate, explicitly authorized action:

```bash
uv run --locked ansible-playbook playbooks/bootstrap.yml --limit romero --diff
```

A successful apply must:

- pass every safety gate before the first mutation;
- remove only the approved Docker-specific package and filesystem state;
- avoid APT autoremove and unrelated package removal;
- leave SSH, sshd, users, authorized keys, provider networking, netplan, DNS ownership, cloud-init, and VPS lifecycle untouched;
- reset the connection and establish a new SSH session;
- pass workstation DNS and target outbound DNS/HTTPS checks;
- exit zero.

## Idempotency Contract

Immediately repeat the apply command. It must exit zero with `changed=0` for `romero`. Read-only inspection and verification tasks must always report unchanged.

## Failure Contract

A nonzero exit before cleanup is a safe stop. The operator reviews ordinary Ansible task output and the normalized identifiers/paths emitted by the safety assertion. The workflow must not suggest manual persistent changes; any authorized cleanup input is added declaratively and reviewed before rerunning.

An expected `blocked-for-review` result in check mode is not by itself an
implementation defect. It requires operator review before cleanup can proceed.
