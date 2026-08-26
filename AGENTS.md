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
- `.specify/` — Spec Kit configuration and templates

## Working conventions

Use Ansible built-in modules instead of shell commands where practical.

Do not make ad-hoc changes to the VPS while implementing repository changes.

Do not deploy infrastructure unless explicitly asked to do so.

## Validation

Before considering Ansible work complete, run:

    ansible-playbook --syntax-check ...
    ansible-lint ...

When appropriate, also run the playbook in check mode.

## Spec Kit

For feature development, follow the Spec Kit artifacts under `specs/`.

Do not bypass requirements or architectural decisions recorded by the
current feature specification and plan.
