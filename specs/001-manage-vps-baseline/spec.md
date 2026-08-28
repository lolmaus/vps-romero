# Feature Specification: Manage VPS Baseline

**Feature Branch**: `001-manage-vps-baseline`

**Created**: 2026-08-26

**Status**: Draft

**Input**: User description: "Manage the existing Ubuntu 26.04 LTS VPS at romero.lolma.us as declarative infrastructure, establish a safe repeatable baseline, remove unused provider-installed Docker, preserve administrative access and provider networking, and prepare the repository for independent future services."

## Clarifications

### Session 2026-08-26

- Q: What persistent host state should the initial baseline manage beyond removing Docker and establishing the infrastructure-management workflow? → A: Manage Docker's absence and only the host declarations needed for connectivity, application, and validation; leave other OS state unmanaged.
- Q: After confirming that Docker has no containers or other operator-owned data, how completely should Docker-specific residual state be removed? → A: Remove all Docker-specific software, services, installation sources, configuration, and data after the safety check passes.
- Q: Should this feature remove only the known provider-installed Docker environment, or also introduce a generalized way to remove other provider-installed software? → A: Remove only Docker in this feature; handle other provider-installed software through later declarative changes.
- Q: Which observable checks must pass after baseline application to prove that provider networking still functions? → A: Verify workstation DNS resolution and a new SSH connection to the VPS, plus outbound name resolution and connectivity from the VPS.
- Q: If Docker images, volumes, build cache, or custom networks are found despite there being no containers, should cleanup stop or remove them automatically? → A: Stop cleanup when any such artifacts are found and require operator review.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Safely Apply the Managed Baseline (Priority: P1)

As the operator, I can verify that the existing VPS is reachable through the infrastructure management workflow and apply the declared baseline from my local workstation without replacing or redesigning the access path that already works.

**Why this priority**: A repeatable and safe management path is the foundation for every current and future configuration change. Losing administrative access would prevent recovery and invalidate the feature.

**Independent Test**: Starting with the already-provisioned VPS and valid operator credentials, run the documented connectivity check and baseline application. The test delivers value when the target is identified correctly, the baseline completes, and a new SSH session can still be established afterward without any manual persistent host configuration.

**Acceptance Scenarios**:

1. **Given** the operator can reach `romero.lolma.us` over the existing SSH path, **When** the operator runs the infrastructure connectivity check, **Then** the check identifies the managed VPS and succeeds without changing its persistent state.
2. **Given** connectivity and required administrative privileges have been verified, **When** the operator applies the declared baseline, **Then** the application completes successfully and reports a clear outcome.
3. **Given** the baseline has completed, **When** the operator opens a new SSH connection using the pre-existing administrative method, **Then** the connection and required administrative privileges still work.
4. **Given** the target is not the supported Ubuntu 26.04 LTS environment, **When** the operator attempts to apply the baseline, **Then** the workflow stops before making managed changes and explains the incompatibility.

---

### User Story 2 - Remove Unwanted Provider Software (Priority: P2)

As the operator, I can bring the identified provider-installed Docker environment under declarative control so that it is removed without ad-hoc host cleanup.

**Why this priority**: Unwanted provider software creates unmanaged state, consumes resources, and obscures what the repository actually promises to maintain.

**Independent Test**: On the existing VPS with the provider-installed Docker environment and no active workloads, apply the baseline and verify that Docker Engine and its provider-added supporting artifacts are absent while unrelated provider facilities continue to operate.

**Acceptance Scenarios**:

1. **Given** the provider-installed Docker environment has no containers, other active workloads, or operator-owned data, **When** the operator applies the baseline, **Then** all Docker-specific software, services, installation sources, configuration, and data are removed.
2. **Given** Docker is declared absent from the baseline, **When** cleanup is needed, **Then** the operator can perform it through the same infrastructure management process without making a manual persistent change on the VPS.
3. **Given** containers, images, volumes, build cache, custom networks, other unexpected workloads, or operator-owned Docker data are detected before removal, **When** the operator applies the baseline, **Then** removal stops for operator review before Docker-specific data is deleted.

---

### User Story 3 - Confirm Convergence and Future Extensibility (Priority: P3)

As the operator, I can prove that the managed host converges to a stable desired state and that later services can be introduced as independent features without rewriting the common VPS baseline.

**Why this priority**: Stable reapplication prevents drift and surprise changes, while clear feature boundaries keep future service work reviewable and independently maintainable.

**Independent Test**: Immediately reapply the baseline after a successful application and run all repository validation. The test succeeds when no unintended host changes are proposed or made and validation passes.

**Acceptance Scenarios**:

1. **Given** the VPS already matches the declared baseline, **When** the operator immediately reapplies it, **Then** the run completes successfully with zero changes to managed state.
2. **Given** the baseline declarations are present, **When** repository validation is run, **Then** every required validation check passes.
3. **Given** a separately defined future service, **When** it is added to or removed from the managed desired state, **Then** the common baseline and unrelated service declarations do not need to change, and no manual persistent host configuration is required.

### Edge Cases

- If the VPS is unreachable, presents the wrong host identity, or rejects the existing credentials, the workflow must stop without attempting persistent changes.
- If the operator lacks the administrative privileges required by the baseline, the workflow must fail before any cleanup or partially privileged change begins.
- If an application is interrupted, a later application must be able to resume convergence without requiring undocumented repair steps and without having altered SSH or provider networking configuration.
- If Docker components are already absent, their declared absence must be treated as a successful no-op.
- If containers, images, volumes, build cache, custom networks, other unexpected workloads, or operator-owned Docker data are present, destructive cleanup must stop for operator review.
- If provider networking or cloud-init artifacts coexist with managed files and packages, the baseline must leave those provider-owned facilities outside its managed scope.
- If removing Docker reveals a package shared with unrelated provider functionality, removal must not silently remove that unrelated functionality.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The repository MUST identify `romero.lolma.us` as an already-existing managed host using non-secret connection metadata.
- **FR-002**: The operator MUST be able to run a read-only connectivity check from a local workstation before applying any desired state.
- **FR-003**: Baseline application MUST use the operator's existing SSH access and privilege path without changing SSH accounts, authorized keys, authentication policy, daemon policy, or the administrative access design.
- **FR-004**: The management workflow MUST verify that the target is compatible with Ubuntu 26.04 LTS before applying managed changes.
- **FR-005**: The repository MUST declare a common VPS baseline that can be applied from a local workstation without manual persistent configuration on the VPS. The baseline MUST manage no persistent VPS state beyond Docker's absence; support for connectivity, application, and validation MUST leave other operating-system state unmanaged.
- **FR-006**: The workflow MUST verify connectivity and required administrative privileges before beginning cleanup or other persistent baseline changes.
- **FR-007**: The baseline MUST preserve the provider-managed networking needed for reachability and MUST NOT provision or reprovision the VPS or manage DNS, provider networking, cloud-init, provider bootstrap mechanisms, provider resources, or VPS lifecycle.
- **FR-008**: This feature's provider-software cleanup MUST be limited to declaring the identified Docker Engine environment absent. Other provider-installed software MUST require a separate future declarative change once identified as unwanted.
- **FR-009**: After the Docker cleanup safety check passes, applying the baseline MUST remove all Docker-specific software, services, installation sources, configuration, and data.
- **FR-010**: Docker removal MUST stop for operator review before deleting Docker-specific data if containers, images, volumes, build cache, custom networks, other unexpected workloads, or operator-owned Docker data are detected.
- **FR-011**: Docker cleanup MUST avoid removing software or configuration required by unrelated provider functionality.
- **FR-012**: Reapplying the baseline to an already-converged VPS MUST complete successfully and report zero changes to managed state.
- **FR-013**: The repository MUST provide documented operator procedures for prerequisites, connectivity verification, baseline application, repository validation, convergence verification, and post-application SSH verification.
- **FR-014**: The repository MUST provide objective validation that checks the declared configuration for structural, syntax, and quality errors before it is considered ready to apply.
- **FR-015**: Common host baseline concerns MUST remain separable from service-specific desired state so that additional services can be declared, validated, included, and removed independently.
- **FR-016**: This feature MUST NOT install or configure Unreal Tournament 2004.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The operator completes one connectivity verification and one baseline application from a local workstation with zero manual persistent configuration steps on the VPS.
- **SC-002**: Immediately before and after baseline application, the workstation resolves `romero.lolma.us` and establishes a new SSH connection using the pre-existing administrative method; after application, the VPS also resolves external names and establishes an outbound network connection.
- **SC-003**: After baseline application, the managed VPS has zero Docker-specific software, active or enabled services, installation sources, configuration, or data.
- **SC-004**: A second baseline application immediately after the first completes successfully and reports zero managed-state changes.
- **SC-005**: One hundred percent of the repository's required validation checks pass before the feature is considered ready.

## Assumptions

- The target is Ubuntu 26.04 LTS at implementation and acceptance time.
- The currently installed Docker Engine environment is provider-installed, is not part of the desired baseline, and is reported to contain no containers. Any other Docker artifact discovered during application is a safety stop requiring operator review, not authorization to destroy it.
- The existing SSH access design is acceptable for this feature. Security hardening or redesign may be addressed by a separate future feature.
- Future hosted services, including Unreal Tournament 2004, will be specified separately and are not required to demonstrate this baseline feature.

## Dependencies

- Continued availability of the already-provisioned VPS, its DNS resolution, provider-managed networking, and provider recovery mechanisms.
- A local operator workstation that can reach the VPS over SSH and has the project-supported infrastructure tooling available.
- Existing operator SSH credentials and authorization to inspect and modify system-level software on the VPS.
