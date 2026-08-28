---

description: "Implementation tasks for the managed VPS baseline"
---

# Tasks: Manage VPS Baseline

**Input**: Design documents from `specs/001-manage-vps-baseline/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/operator-interface.md`, `quickstart.md`, and Accepted ADR-0001

**Tests**: The feature requires local syntax/lint validation, read-only connectivity and check-mode runs, an explicitly authorized application, post-application access/network checks, and an immediate zero-change second application. No standalone test suite is planned.

**Organization**: Tasks are grouped by user story. Repository implementation consists of T001–T033 and ends after the final safety/scope audit and diff review. T034–T035 form a separate operator-authorized production acceptance phase; `$speckit-implement` and other implementation runs MUST leave them unexecuted unless the operator gives separate explicit deployment authorization at that time.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel because it changes a different file and has no dependency on an incomplete task
- **[Story]**: Maps the task to a user story from `spec.md`
- Every task names the exact file or files it creates, changes, or validates

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the reproducible local Ansible toolchain and repository configuration.

- [X] T001 Create `pyproject.toml` with `requires-python = ">=3.12,<3.15"`, exact direct requirements for `ansible-core==2.21.3` and `ansible-lint==26.8.0`, and non-package uv project configuration
- [X] T002 Generate and commit the complete cross-platform Python dependency resolution in `uv.lock` from `pyproject.toml`
- [X] T003 [P] Add `.venv/`, Ansible caches, and retry files to `.gitignore` without replacing unrelated ignore rules
- [X] T004 [P] Configure repository-local inventory and role paths while preserving SSH host-key checking and avoiding global connection credentials or `remote_user` in `ansible.cfg`
- [X] T005 [P] Configure syntax and quality rules for `playbooks/` and `roles/` without warning skips in `.ansible-lint`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define the managed host, declarative review input, and non-overridable Docker safety policy used by every story.

**CRITICAL**: No user-story implementation begins until this phase is complete.

- [X] T006 Define the `romero` host, `romero.lolma.us`, and `/usr/bin/python3` using non-secret metadata only in `inventory/hosts.yml`
- [X] T007 [P] Create the host-scoped, non-secret Docker review-approval override location with no initial approvals or SSH material in `inventory/host_vars/romero.yml`
- [X] T008 [P] Define the empty exact-match Docker approval manifest and role defaults in `roles/vps_baseline/defaults/main.yml`
- [X] T009 [P] Validate every approval-manifest collection and reject blanket approval inputs in `roles/vps_baseline/meta/argument_specs.yml`
- [X] T010 [P] Define Docker-only package, service, built-in network, Engine API endpoint, source/key, configuration, runtime, and data-path policy allowlists in `roles/vps_baseline/vars/main.yml`

**Checkpoint**: The locked workstation toolchain, host identity, approval model, and immutable cleanup policy are ready.

---

## Phase 3: User Story 1 - Safely Apply the Managed Baseline (Priority: P1) MVP

**Goal**: Provide the existing root-SSH bootstrap path, fail-fast compatibility checks, and post-run proof that access and provider networking still work without managing those facilities.

**Independent Test**: Against a supported host that already matches the baseline, run `playbooks/connectivity.yml` and `playbooks/bootstrap.yml`; connectivity reports no changes, unsupported targets stop before mutation, the bootstrap completes, and a reset connection establishes a new SSH session while workstation and target network checks pass.

### Implementation for User Story 1

- [X] T011 [P] [US1] Implement the read-only root-SSH connectivity entry point with workstation DNS, Ansible ping, Ubuntu 26.04, target Python 3.14, effective UID 0, target DNS, and outbound HTTPS checks in `playbooks/connectivity.yml`
- [X] T012 [P] [US1] Implement duplicated fail-fast OS, Python, privilege, `python3-apt`, package-fact, and service-fact prerequisites without auto-installing dependencies in `roles/vps_baseline/tasks/preflight.yml`
- [X] T013 [P] [US1] Implement connection reset, new SSH ping, workstation DNS, target DNS, and outbound HTTPS verification with read-only change reporting in `roles/vps_baseline/tasks/verify.yml`
- [X] T014 [US1] Create the ordered role dispatcher for preflight and verification in `roles/vps_baseline/tasks/main.yml`
- [X] T015 [US1] Implement the thin root-SSH role entry point without SSH, user, network, netplan, cloud-init, DNS, or provider management in `playbooks/bootstrap.yml`
- [X] T016 [US1] Run locked syntax checks for `playbooks/connectivity.yml` and `playbooks/bootstrap.yml`, then run the read-only connectivity contract from `specs/001-manage-vps-baseline/contracts/operator-interface.md` when operator SSH credentials are available

**Checkpoint**: The management transport and safety preflight are independently usable; no production application has been authorized by completing this phase.

---

## Phase 4: User Story 2 - Remove Unwanted Provider Software (Priority: P2)

**Goal**: Discover all Docker state, require exact review for unexpected or sensitive artifacts, prove the APT transaction is Docker-only, and remove only reviewed Docker-related state.

**Independent Test**: Run `playbooks/bootstrap.yml --check --diff` against the provider Docker installation. A safe installation predicts only the declared Docker cleanup; any container, image, volume, build cache, custom network, swarm object, plugin, ambiguous path, group member, unavailable inspection, mixed APT source, or unrelated APT removal stops before mutation.

### Implementation for User Story 2

- [X] T017 [US2] Discover installed Docker and shared-runtime packages, services, sockets, APT sources/keys, system configuration, systemd overrides, rootful/rootless data, runtime paths, and Docker group membership without reading credential-bearing contents in `roles/vps_baseline/tasks/inspect_docker.yml`
- [X] T018 [US2] Query every discovered relevant rootful or rootless Docker Engine Unix socket with read-only built-in URI calls for all containers, images, volumes, non-built-in networks, build cache, swarm/services/configs/secrets, and plugins, and fail closed without starting Docker when Docker-bearing state exists and any relevant daemon cannot be safely inspected in `roles/vps_baseline/tasks/inspect_docker.yml`
- [X] T019 [US2] Normalize Docker discovery into stable identifier/path sets and enforce exact equality with the host approval manifest, including invalidation when discovered state changes, in `roles/vps_baseline/tasks/inspect_docker.yml`
- [X] T020 [US2] Build the installed Docker-only purge set, protect distribution `containerd`, `runc`, and `podman-docker`, and compare it with one read-only `apt-get --simulate purge` command using `argv`, `changed_when: false`, and check-mode execution in `roles/vps_baseline/tasks/inspect_docker.yml`
- [X] T021 [US2] Stop and disable only approved Docker-specific services and purge only the simulated exact package set with `autoremove: false` and APT dependency auto-installation disabled in `roles/vps_baseline/tasks/remove_docker.yml`
- [X] T022 [US2] Remove only verified dedicated Docker APT sources, unreferenced Docker signing keys, and Docker-specific APT cache files while blocking mixed or noncanonical sources in `roles/vps_baseline/tasks/remove_docker.yml`
- [X] T023 [US2] Remove approved Docker configuration, systemd override, runtime, rootless, and data paths while selecting `containerd.io` only after no unrelated current use is detected and removing an existing `/var/lib/containerd` only through the exact operator-approved path loop in `roles/vps_baseline/tasks/remove_docker.yml`
- [X] T024 [US2] Insert Docker inspection and gated removal between preflight and verification in `roles/vps_baseline/tasks/main.yml`
- [X] T025 [US2] Regather facts and assert Docker-specific packages, services, sources, keys, configuration, runtime paths, and data are absent after a normal apply while preserving the existing access/network checks in `roles/vps_baseline/tasks/verify.yml`
- [X] T026 [US2] Run locked syntax and lint validation plus the read-only check-mode contract for `playbooks/bootstrap.yml` and `roles/vps_baseline/`, and confirm its diff contains no SSH, user, firewall, network, netplan, cloud-init, DNS, or provider-resource changes

**Checkpoint**: Docker cleanup is fully declared, fail-closed, and previewable; check mode does not authorize or perform production cleanup.

---

## Phase 5: User Story 3 - Confirm Convergence and Future Extensibility (Priority: P3)

**Goal**: Prove stable no-op behavior, repository validity, and clean composition boundaries for independent future service features.

**Independent Test**: For repository implementation, local locked setup, both syntax checks, ansible-lint, read-only connectivity, and check mode pass, and adding a separate role/playbook would not require edits to `vps_baseline`, inventory-wide root transport, or manual host configuration. The production-only convergence check is deferred to the separately authorized acceptance phase after repository implementation is complete.

### Implementation and Validation for User Story 3

- [X] T027 [US3] Make every discovery and verification operation report unchanged, preserve read-only safety checks in check mode, skip only impossible post-removal assertions in check mode, and make the Docker-absent path a no-op in `roles/vps_baseline/tasks/inspect_docker.yml`, `roles/vps_baseline/tasks/remove_docker.yml`, and `roles/vps_baseline/tasks/verify.yml`
- [X] T028 [US3] Verify role imports remain ordered and self-contained and that root transport remains scoped only to bootstrap-era entry points, correcting composition leaks in `roles/vps_baseline/tasks/main.yml`, `playbooks/connectivity.yml`, `playbooks/bootstrap.yml`, `inventory/hosts.yml`, and `ansible.cfg`
- [X] T029 [US3] After T027 and T028, reconcile final locked commands, safety-stop behavior, deployment separation, post-apply checks, and idempotency procedure with the final implementation in `specs/001-manage-vps-baseline/quickstart.md` and `specs/001-manage-vps-baseline/contracts/operator-interface.md`
- [X] T030 [US3] Run `uv sync --locked`, both locked playbook syntax checks, and locked ansible-lint against `pyproject.toml`, `uv.lock`, `playbooks/connectivity.yml`, `playbooks/bootstrap.yml`, and `roles/vps_baseline/`
- [X] T031 [US3] Run the read-only connectivity and `--check --diff` procedures in `specs/001-manage-vps-baseline/quickstart.md`; treat a non-mutating fail-closed `blocked-for-review` result caused by unexpected or ambiguous Docker state as a valid safety outcome requiring operator review before cleanup, and resolve only incorrect implementation, incompatibility with the approved design, or incorrect change reporting as defects in the referenced inventory, playbooks, and role files

**Checkpoint**: Repository implementation and all non-mutating validation are complete. No production application is authorized by completing this phase.

---

## Phase 6: Final Repository Safety and Scope Review (Implementation Completion)

**Purpose**: Perform the final repository-wide safety/scope audit and diff review after implementation, local validation, read-only target validation, and check mode are complete.

- [X] T032 Audit `inventory/`, `playbooks/`, `roles/`, `pyproject.toml`, and `uv.lock` for secrets, private SSH configuration, mutable dependency references, non-built-in Ansible collections, accidental ownership of excluded provider or SSH/network state, and any installation, configuration, or other introduction of Unreal Tournament 2004 state
- [X] T033 Confirm `git diff --check` passes and review the final changes against `specs/001-manage-vps-baseline/spec.md`, `specs/001-manage-vps-baseline/plan.md`, and `docs/adr/0001-use-ansible-for-host-configuration.md`

**Implementation Completion Boundary**: `$speckit-implement` and any other repository implementation run MUST stop after T033 and may report repository implementation complete with T034–T035 still unchecked. Proceeding beyond this boundary requires a new, separate, explicit operator authorization to deploy at that time.

---

## Phase 7: Operator-Authorized Production Acceptance (Not an Implementation Phase)

**Purpose**: Perform the production application and convergence acceptance checks only after repository implementation is complete and the operator separately authorizes deployment.

**OPERATOR GATE**: T034–T035 MUST NOT be executed by `$speckit-implement` or any implementation run unless the operator gives separate explicit deployment authorization at that time. Their presence in this file records production acceptance work; it does not grant deployment authority.

- [ ] T034 [US3] OPERATOR ONLY — After separate explicit deployment authorization, run the first normal application and post-application SSH/DNS/outbound/Docker-absence checks exactly as documented in `specs/001-manage-vps-baseline/quickstart.md`
- [ ] T035 [US3] OPERATOR ONLY — Immediately after T034, run the documented second application and require `changed=0`; if it reports any change, stop production acceptance without modifying or redeploying implementation under the existing authorization, return to repository implementation to correct the non-convergent behavior, rerun the applicable validation and final review tasks through T033, and obtain fresh explicit deployment authorization before another T034–T035 production acceptance sequence

**Production Acceptance Checkpoint**: The feature's production acceptance criteria are complete only after both operator-authorized tasks pass.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 — Setup**: Starts immediately; T002 depends on T001.
- **Phase 2 — Foundational**: Depends on Phase 1 and blocks every user story.
- **Phase 3 — User Story 1**: Depends on Phase 2 and establishes the safe transport, compatibility, and verification path.
- **Phase 4 — User Story 2**: Depends on User Story 1 because cleanup must reuse its preflight and post-change access checks.
- **Phase 5 — User Story 3**: Depends on User Stories 1 and 2 and completes repository implementation plus local/read-only/check-mode validation.
- **Phase 6 — Final Repository Review**: Depends on Phase 5 and completes the implementation-run scope at T033.
- **Phase 7 — Production Acceptance**: Depends on Phase 6 but is outside ordinary implementation execution; it starts only after separate explicit deployment authorization.

### User Story Dependencies

```text
Setup
  └── Foundational
        └── US1 Safe management path
              └── US2 Docker discovery and cleanup
                    └── US3 Convergence and extensibility
```

- **US1 (P1)**: Independently testable against a compatible host already matching the baseline; no dependency on another story for its connectivity and access-safety behavior.
- **US2 (P2)**: Extends US1 with the only persistent baseline state in scope: Docker absence.
- **US3 (P3)**: Measures the completed US1+US2 baseline and confirms its composition boundary.

### Within Each User Story

- Implement facts and assertions before role dispatchers or playbook integration.
- Complete all discovery, exact-approval, and APT-simulation gates before any removal task.
- Complete removal declarations before post-apply absence assertions.
- Run local validation before target check mode.
- Target check mode remains read-only; repository implementation stops after T033.
- T034 requires separate explicit deployment authorization given after repository implementation is complete.
- Run T035 immediately after T034 under the same explicit authorization; if it reports any change, stop production acceptance and require corrected implementation, applicable validation and review through T033, and fresh explicit deployment authorization before retrying T034.

### Parallel Opportunities

- T003–T005 can proceed in parallel after T001 while T002 resolves the lockfile.
- T007–T010 can proceed in parallel after T006 establishes the inventory shape.
- T011–T013 modify different files and can proceed in parallel.
- Docker inspection tasks T017–T020 and removal tasks T021–T023 are deliberately sequential within their respective files and safety state machine.

---

## Parallel Example: User Story 1

```text
Task T011: Implement playbooks/connectivity.yml
Task T012: Implement roles/vps_baseline/tasks/preflight.yml
Task T013: Implement roles/vps_baseline/tasks/verify.yml
```

## Parallel Example: Foundational Policy

```text
Task T007: Create inventory/host_vars/romero.yml
Task T008: Create roles/vps_baseline/defaults/main.yml
Task T009: Create roles/vps_baseline/meta/argument_specs.yml
Task T010: Create roles/vps_baseline/vars/main.yml
```

---

## Implementation Strategy

### MVP First — User Story 1

1. Complete Setup and Foundational phases.
2. Implement and validate User Story 1.
3. Stop and review the connectivity, compatibility, SSH-preservation, and provider-network boundaries.
4. Do not apply this partial increment to the current Docker-bearing VPS; it does not yet implement Docker absence.

### Incremental Delivery

1. Setup + Foundational establishes the reproducible toolchain and safety model.
2. US1 establishes the read-only connectivity, fail-fast preflight, and recovery-critical verification path.
3. US2 adds the complete fail-closed Docker absence workflow and check-mode preview.
4. US3 completes repository implementation, local validation, read-only target validation, and check mode.
5. Phase 6 performs the final repository safety/scope audit and diff review; `$speckit-implement` stops after T033.
6. T034 and T035 occur only in a later operator-authorized production acceptance run.

### Safe Execution Boundary

- Repository implementation and local validation do not authorize production deployment.
- Connectivity and check-mode tasks are read-only target operations.
- T001–T033 are the complete `$speckit-implement` execution scope unless separate deployment authorization is given later.
- T034 is the first persistent host change and requires a separate explicit operator instruction given at that time; T035 follows immediately under the same authorization, but any nonzero change in T035 ends that authorization's acceptance sequence and requires a return through implementation, applicable validation and review through T033, and fresh authorization.
- If discovery or APT simulation blocks, update only the exact non-secret approval manifest or implementation after review; do not make manual persistent VPS changes.

---

## Notes

- `[P]` tasks change different files and have no dependency on unfinished work.
- `[US1]`, `[US2]`, and `[US3]` map directly to the three prioritized user stories in `spec.md`.
- Prefer fully qualified `ansible.builtin` modules; command exceptions are limited to the read-only APT purge simulation and fixed-argument read-only `ctr` inventory needed to detect unrelated current containerd use.
- Do not install external Ansible collections or add Docker SDK dependencies.
- Do not log or commit file contents that may contain registry credentials.
- Commit after each task or logical group, but never treat a repository commit as deployment authorization.
- Leave T034–T035 unchecked when completing `$speckit-implement` unless the operator has separately and explicitly authorized deployment during that run.
