# Ansible Basic

This module teaches the foundational skills for using Ansible in real-world infrastructure automation. You will start by understanding what Ansible is and how it connects to remote hosts, then progressively build your knowledge through inventory management, writing practical tasks with standard modules, working with variables and templates, controlling execution flow, organising code into reusable roles, and finally securing credentials with Ansible Vault while following professional workflow practices.

## Curriculum

<!-- 7 sections · 35 themes -->

### Introduction to Ansible and Automation Concepts

Introduces Ansible's purpose, agentless architecture, and core concepts, then establishes the YAML skills and setup required to run your first commands.

- [What Is Ansible?](./ansible-intro/ansible-what-is.md)
- [Ansible Architecture and Key Components](./ansible-intro/ansible-architecture.md)
- [YAML Syntax for Ansible](./ansible-intro/ansible-yaml-syntax.md)
- [Installation, SSH, and Privilege Escalation](./ansible-intro/ansible-installation-ssh.md)
- [Running Ad-Hoc Commands](./ansible-intro/ansible-ad-hoc.md)

### Inventory and Project Structure

Explains how Ansible discovers and targets hosts using static and dynamic inventory, organises variables with group_vars/host_vars, configures behaviour via ansible.cfg, and lays out a scalable project directory.

- [Static Inventory](./ansible-inventory/ansible-static-inventory.md)
- [Dynamic Inventory](./ansible-inventory/ansible-dynamic-inventory.md)
- [group_vars and host_vars](./ansible-inventory/ansible-group-vars.md)
- [ansible.cfg](./ansible-inventory/ansible-cfg.md)
- [Project Layout and Playbook Structure](./ansible-inventory/ansible-project-layout.md)

### Core Modules and Writing Real Tasks

Walks through the most commonly used Ansible modules for everyday automation: managing files, installing packages, controlling services, running commands, editing files in-place, and using handlers.

- [File, Copy, and Fetch Modules](./ansible-core-modules/ansible-file-copy-fetch.md)
- [Package and Service Modules](./ansible-core-modules/ansible-package-service.md)
- [Command, Shell, and Script Modules](./ansible-core-modules/ansible-command-shell-script.md)
- [lineinfile and blockinfile](./ansible-core-modules/ansible-lineinfile.md)
- [Handlers, flush_handlers, and debug](./ansible-core-modules/ansible-handlers-debug.md)

### Variables, Facts, and Runtime Data

Covers the different ways data flows through a playbook at runtime: capturing task output, reading system state with facts, computing derived values, rendering dynamic content with Jinja2, and lookup plugins.

- [Registering Variables](./ansible-variables-facts/ansible-register.md)
- [Gathering and Using Facts](./ansible-variables-facts/ansible-facts.md)
- [Jinja2 Basics — Filters, Tests, and Expressions](./ansible-variables-facts/ansible-jinja2.md)
- [The template Module and .j2 Files](./ansible-variables-facts/ansible-template-module.md)
- [Lookup Plugins](./ansible-variables-facts/ansible-lookups.md)

### Flow Control and Error Handling

Teaches how to make playbooks adaptive and resilient: running tasks conditionally, iterating over data, grouping tasks with block/rescue/always, and delegating tasks to specific hosts.

- [Conditionals — when](./ansible-flow-control/ansible-conditionals.md)
- [Loops](./ansible-flow-control/ansible-loops.md)
- [Blocks — block, rescue, always](./ansible-flow-control/ansible-blocks.md)
- [Error Handling — failed_when and changed_when](./ansible-flow-control/ansible-error-handling.md)
- [delegate_to and run_once](./ansible-flow-control/ansible-delegate.md)

### Roles and Playbook Reuse

Introduces roles as the primary unit of reuse in Ansible: creating them with ansible-galaxy, understanding the directory structure, dynamic vs static inclusion, role dependencies, and variable precedence.

- [What Are Roles?](./ansible-roles/ansible-what-are-roles.md)
- [Creating and Using Roles](./ansible-roles/ansible-creating-roles.md)
- [include_tasks and import_tasks](./ansible-roles/ansible-include-import.md)
- [Role Dependencies](./ansible-roles/ansible-role-dependencies.md)
- [Variable Precedence](./ansible-roles/ansible-variable-precedence.md)

### Vault, Tags, and Professional Workflow

Closes the module with production-readiness skills: encrypting sensitive data with Ansible Vault, selective execution with tags, check mode, linting with ansible-lint, and performance best practices.

- [Ansible Vault — Protecting Sensitive Data](./ansible-vault-workflow/ansible-vault.md)
- [Tags — Selective Execution](./ansible-vault-workflow/ansible-tags.md)
- [Check Mode, Diff, and Limiting Scope](./ansible-vault-workflow/ansible-check-limit.md)
- [ansible-lint](./ansible-vault-workflow/ansible-lint.md)
- [Best Practices and Performance](./ansible-vault-workflow/ansible-best-practices.md)
