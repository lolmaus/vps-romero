# State Model: Manage VPS Baseline

This feature has no application database. Its data model consists of version-controlled desired-state records and ephemeral facts gathered during each Ansible run.

## Managed Host

Represents the already-existing VPS selected by inventory.

| Field | Type | Source | Validation |
|-------|------|--------|------------|
| `inventory_name` | string | repository inventory | Exactly `romero`; unique within inventory |
| `ansible_host` | hostname | repository inventory | Exactly `romero.lolma.us`; non-secret |
| `bootstrap_user` | string | playbook | Exactly `root`; defined only on bootstrap-era entry points |
| `python_interpreter` | path | repository inventory | `/usr/bin/python3`; must report Python 3.14 |
| `distribution` | fact | target | Must be Ubuntu |
| `distribution_version` | fact | target | Must be 26.04 |
| `effective_uid` | fact | target | Must be 0 before cleanup |

The host record contains no password, private key, SSH key path, provider identifier, DNS-management setting, or network configuration.

## Docker Discovery

Ephemeral normalized snapshot collected before any destructive task.

| Field | Type | Empty-state meaning |
|-------|------|---------------------|
| `installed_docker_packages` | set of package names | No Docker package is installed |
| `shared_runtime_packages` | set of package names | No ambiguous runtime package needs review |
| `docker_services` | set of service names/states | No Docker-specific unit is present or active |
| `daemon_inspectable` | boolean | False is safe only when all Docker packages and paths are absent |
| `containers` | set of IDs | No running or stopped container exists |
| `images` | set of IDs | No image exists |
| `volumes` | set of names | No volume exists |
| `custom_networks` | set of IDs/names | Only built-in `bridge`, `host`, and `none` networks exist |
| `build_cache` | set of IDs or usage records | No build cache exists |
| `swarm_objects` | set of IDs/names | No swarm, service, config, or secret state exists |
| `plugins` | set of plugin identifiers | No non-built-in plugin state exists |
| `apt_sources` | set of paths and Docker-only classification | No Docker repository entry exists |
| `apt_keys` | set of paths and references | No Docker-only signing key exists |
| `configuration_paths` | set of paths | No Docker-specific system or user configuration exists |
| `data_paths` | set of paths | No Docker-specific data exists |
| `runtime_paths` | set of paths | No Docker socket/PID/runtime directory exists |
| `docker_group_members` | set of user names | No user association must be preserved or reviewed |

Discovery records never include file contents that could contain registry credentials. Only identifiers, counts, states, and paths needed for safety decisions are retained for the current run.

## Docker Approval Manifest

Non-secret host-scoped desired-state input used only after a run stops for operator review.

| Field | Type | Default | Rule |
|-------|------|---------|------|
| `containers` | set of exact IDs | empty | Must exactly equal discovered unexpected containers |
| `images` | set of exact IDs | empty | Must exactly equal discovered images |
| `volumes` | set of exact names | empty | Must exactly equal discovered volumes |
| `custom_networks` | set of exact IDs/names | empty | Must exactly equal discovered custom networks |
| `build_cache` | set of exact IDs/records | empty | Must exactly equal discovered build cache |
| `swarm_objects` | set of exact IDs/names | empty | Must exactly equal discovered swarm-related state |
| `plugins` | set of exact identifiers | empty | Must exactly equal discovered non-built-in plugins |
| `paths` | set of exact absolute paths | empty | Must exactly equal reviewed configuration/data paths selected for removal, including an existing `/var/lib/containerd` whenever its removal is intended |
| `group_members` | set of exact user names | empty | Must exactly equal reviewed Docker group membership |

An empty discovered set needs no approval. A nonempty discovered set must match the corresponding approval set exactly. A blanket boolean approval is invalid. Approval authorizes the repository-declared cleanup on a later run; it does not authorize manual host mutation.

## Package Removal Transaction

Ephemeral comparison between requested and simulated package changes.

| Field | Type | Rule |
|-------|------|------|
| `installed_candidates` | set of package names | Intersection of installed packages and the internal Docker-only allowlist |
| `requested_purge` | set of package names | Exact Docker packages requested absent |
| `simulated_removals` | set of package names | Parsed from the read-only APT simulation |
| `unexpected_removals` | set difference | `simulated_removals - requested_purge`; must be empty |
| `shared_runtime_state` | classification | Must detect no unrelated current use before `containerd.io` is selected; an existing `/var/lib/containerd` also requires exact path approval before removal |

Package wildcards and automatic dependency removal are invalid inputs.

## State Transitions

```text
unknown
  └─ connectivity + target preflight succeeds
       └─ discovered
            ├─ Docker fully absent ────────────────> converged
            ├─ inspection unavailable/ambiguous ──> blocked-for-review
            ├─ unexpected state, no exact approval > blocked-for-review
            ├─ discovery differs from approval ───> blocked-for-review
            └─ safe or exactly reviewed
                 └─ APT simulation has extra removals > blocked-for-review
                 └─ APT simulation is exact
                      └─ services/packages/files/data removed
                           └─ post-checks pass ─────> converged
                           └─ post-checks fail ─────> failed-recoverable
```

Every rerun starts again at `unknown` and rediscovers actual state. No cached approval bypasses equality checks, and an interrupted run is recovered by re-entering the same state machine.

## Convergence Invariants

- SSH authentication, sshd, netplan, provider networking, DNS ownership, cloud-init, and VPS lifecycle state are never modelled as desired state by this feature.
- Converged means Docker-specific packages, services, APT sources/keys, configuration, runtime paths, and data are absent.
- Shared provider/runtime packages remain unless proven to be part of the exact Docker-only removal transaction.
- A second run over a converged host produces no managed-state changes.
