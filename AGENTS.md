# Agent Guide

## Repository purpose

This repository manages the romero.lolma.us VPS and services running on it.

Project governance is defined in `.specify/memory/constitution.md`.
Feature requirements and implementation plans live under `specs/`.

## Repository structure

- `inventory/` — Ansible inventories
- `playbooks/` — Ansible entry-point playbooks
- `roles/` — reusable Ansible roles
- `specs/` — Spec Kit feature artifacts
- `docs/adr/` — project-wide architecture decision records
- `.specify/` — Spec Kit configuration and templates

## Working conventions

Use Ansible built-in modules instead of shell commands where practical.

Do not make ad-hoc configuration changes to the VPS while implementing repository changes. Read-only inspection and diagnostics over SSH are allowed, but any intended persistent state change must be implemented through infrastructure-as-code.

Do not deploy infrastructure unless explicitly asked to do so.

## Validation

Before considering Ansible work complete, run:

    ansible-playbook --syntax-check ...
    ansible-lint ...

When appropriate, also run the playbook in check mode.

## Architecture Decision Records

Project-wide architecture decisions are recorded under `docs/adr/`.

Before making an architectural decision, inspect existing accepted ADRs that may govern it.

Create an ADR when a decision is architecturally significant, durable, difficult to reverse, or expected to constrain future features. Do not create ADRs for routine implementation details.

New ADRs start as `Proposed` and require user review before becoming `Accepted`.

Do not rewrite an accepted ADR to represent a later decision. Supersede it with a new ADR instead.

Feature plans should reference applicable ADRs. When a plan introduces a new ADR-worthy decision, perform ADR review after planning and before generating implementation tasks.

## Spec Kit

For feature development, follow the Spec Kit artifacts under `specs/`.

Do not bypass requirements or architectural decisions recorded by the
current feature specification and plan.
