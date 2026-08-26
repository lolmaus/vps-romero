# Operator Interface Contract

The repository exposes local commands and two Ansible playbook entry points. Commands are run from the repository root. Credentials and SSH client configuration remain outside the repository.

## Toolchain Contract

Create an isolated Python 3.14 environment and install the exact repository requirements:

```bash
python3.14 -m venv .venv
.venv/bin/python -m pip install --requirement requirements.txt
```

The installed versions must report ansible-core 2.21.3 and ansible-lint 26.8.0:

```bash
.venv/bin/ansible --version
.venv/bin/ansible-lint --version
```

## Static Validation Contract

These commands must exit zero and must not connect to the VPS:

```bash
.venv/bin/ansible-playbook --syntax-check playbooks/connectivity.yml
.venv/bin/ansible-playbook --syntax-check playbooks/bootstrap.yml
.venv/bin/ansible-lint
```

## Connectivity Contract

```bash
.venv/bin/ansible-playbook playbooks/connectivity.yml --limit romero
```

This command is read-only. It must:

- resolve `romero.lolma.us` from the workstation;
- establish the existing root SSH connection without modifying SSH configuration;
- verify effective root access, Ubuntu 26.04, target Python 3.14, and pre-existing APT module support;
- verify target-side external name resolution and outbound HTTPS connectivity;
- exit nonzero before persistent change if any prerequisite fails.

## Dry-Run Contract

```bash
.venv/bin/ansible-playbook playbooks/bootstrap.yml --limit romero --check --diff
```

Read-only Docker API inventory and APT transaction simulation still execute and report unchanged. Mutating built-in modules predict their changes. The command stops nonzero if Docker state is unexpected or ambiguous, the exact approval manifest does not match discovery, or APT proposes removing anything outside the Docker-only request.

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
.venv/bin/ansible-playbook playbooks/bootstrap.yml --limit romero --diff
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
