# Validation Quickstart: Manage VPS Baseline

This guide describes how to validate and, only when separately authorized, apply the implemented feature. Generating this plan does not run any command against `romero.lolma.us`.

## Prerequisites

- A POSIX workstation with Python 3.14, OpenSSH, and network access to `romero.lolma.us`.
- Existing root SSH access whose host key has been verified by the operator.
- The target is the existing Ubuntu 26.04 LTS VPS with `/usr/bin/python3` and `python3-apt` already available.
- Review [the operator interface](contracts/operator-interface.md) and [the state model](data-model.md), especially the fail-closed Docker approval rules.

## 1. Prepare the Local Toolchain

```bash
python3.14 -m venv .venv
.venv/bin/python -m pip install --requirement requirements.txt
.venv/bin/ansible --version
.venv/bin/ansible-lint --version
```

Expected: ansible-core is 2.21.3, ansible-lint is 26.8.0, and the active configuration resolves to this repository's `ansible.cfg`.

## 2. Run Repository Validation

```bash
.venv/bin/ansible-playbook --syntax-check playbooks/connectivity.yml
.venv/bin/ansible-playbook --syntax-check playbooks/bootstrap.yml
.venv/bin/ansible-lint
```

Expected: all three commands exit zero without contacting or changing the VPS.

## 3. Verify Connectivity and Compatibility

```bash
.venv/bin/ansible-playbook playbooks/connectivity.yml --limit romero
```

Expected: workstation DNS, the existing root SSH transport, Ubuntu 26.04/Python 3.14 compatibility, administrative access, remote DNS, and outbound HTTPS checks all pass. The recap reports no changes.

Stop here if the command fails. Do not compensate with manual host configuration.

## 4. Preview the Baseline

```bash
.venv/bin/ansible-playbook playbooks/bootstrap.yml --limit romero --check --diff
```

Expected when Docker is safe to remove:

- read-only Docker discovery completes;
- no containers, images, volumes, build cache, custom networks, or other unexpected state is found, or current discovery exactly matches a reviewed approval manifest;
- APT simulation proposes only the explicit Docker package set;
- package, source, configuration, and data removal is predicted;
- no SSH, networking, netplan, cloud-init, user, or provider state is predicted to change.

If the run stops on Docker state, inspect the normalized identifiers and paths. After deciding that specific artifacts may be removed, record those exact values in `inventory/host_vars/romero.yml`, review the repository change, and repeat check mode. Never use wildcard or blanket approval.

## 5. Apply Only with Separate Authorization

Do not infer deployment approval from successful repository validation or check mode. When the operator explicitly authorizes application, run:

```bash
.venv/bin/ansible-playbook playbooks/bootstrap.yml --limit romero --diff
```

Expected: the role passes all safety gates, removes the declared Docker environment, reconnects over SSH, and verifies workstation DNS plus target outbound DNS/HTTPS connectivity.

## 6. Verify Observable Outcomes

Confirm from the play recap and verification tasks that:

- a new root SSH session succeeded after cleanup;
- Docker-specific packages, services, sources, keys, configuration, runtime paths, and data are absent;
- no unrelated package was removed;
- provider networking still supports workstation DNS/SSH and target outbound DNS/HTTPS;
- no task managed SSH authentication, sshd, netplan, cloud-init, DNS, or provider resources.

## 7. Prove Idempotence

Immediately run the same apply command again:

```bash
.venv/bin/ansible-playbook playbooks/bootstrap.yml --limit romero --diff
```

Expected: exit zero and `changed=0` for `romero`. Any change or safety failure means convergence has not been demonstrated and must be resolved in repository code before completion.
