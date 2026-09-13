# Ansible Linux Automation

Ansible-based Linux infrastructure automation lab focused on **server patching, pre/post health checks, credential management, and controlled execution**.

This project was built as a hands-on DevOps learning project to understand how Ansible can be structured for a realistic Linux maintenance workflow using reusable roles rather than a single large playbook.

## Overview

The automation performs a controlled Linux patching workflow:

```text
                    ┌─────────────────────┐
                    │      site.yml       │
                    │    Orchestrator     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Linux Pre-check  │
                    │                     │
                    │ • Disk              │
                    │ • Memory            │
                    │ • Kernel            │
                    │ • System readiness  │
                    └──────────┬──────────┘
                               │
                 ┌─────────────▼─────────────┐
                 │   Application Pre-check   │
                 │                           │
                 │ Docker │ PostgreSQL │ API │
                 └─────────────┬─────────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Linux Patching   │
                    │                     │
                    │ • Update packages   │
                    │ • Upgrade packages   │
                    │ • Reboot if needed  │
                    └──────────┬──────────┘
                               │
                 ┌─────────────▼─────────────┐
                 │  Application Post-check  │
                 │                           │
                 │ Docker │ PostgreSQL │ API │
                 └───────────────────────────┘
```

The playbook is designed so that the **main playbook acts as the workflow/orchestrator**, while individual roles contain reusable operational tasks.

## What This Project Demonstrates

### Ansible fundamentals

* Inventory management
* `ansible.cfg` configuration
* Variables using `group_vars` and `host_vars`
* Conditional task execution
* Facts and registered variables
* `serial` for controlled host execution
* `forks` for Ansible parallelism
* `block` / `rescue` for error handling
* Roles and reusable task components
* Pre-tasks and post-tasks

### Security

* Ansible Vault for sensitive credentials
* Separate host-specific and group-level variables
* Local inventory and secrets excluded from Git

### Infrastructure operations

* Linux system pre-checks
* Disk utilisation validation
* Memory and kernel information checks
* Package updates and upgrades
* Reboot detection
* Docker service health checks
* PostgreSQL connectivity checks
* Flask API health checks

### Automation

* Scheduled Ansible execution using Linux `cron`
* Controlled patching workflow using `serial`
* Application-specific checks using inventory groups

## Project Structure

```text
.
├── ansible.cfg
├── site.yml
├── inventory.example.ini
│
├── group_vars/
│
├── host_vars/
│
├── linux_precheck/
│   └── tasks/
│
├── linux_patching_role/
│   └── tasks/
│
├── docker_check/
│   └── tasks/
│
├── postgres_check/
│   └── tasks/
│
└── flask_check/
    └── tasks/
```

### Main components

| Component               | Purpose                                             |
| ----------------------- | --------------------------------------------------- |
| `site.yml`              | Main orchestration playbook                         |
| `linux_precheck`        | Common Linux readiness checks                       |
| `linux_patching_role`   | Package update, upgrade and reboot handling         |
| `docker_check`          | Docker service health validation                    |
| `postgres_check`        | PostgreSQL availability and connectivity validation |
| `flask_check`           | Flask API health validation                         |
| `group_vars`            | Variables shared by inventory groups                |
| `host_vars`             | Host-specific configuration and credentials         |
| `inventory.example.ini` | Example inventory for the environment               |

## Execution Flow

The main playbook separates **workflow logic** from reusable roles.

For example:

```yaml
pre_tasks:
  - Linux pre-check
  - Application pre-check

roles:
  - linux_patching_role

post_tasks:
  - Application post-check
```

Application checks are executed conditionally based on inventory membership.

For example, PostgreSQL checks only run against hosts in:

```text
[postgres_servers]
```

while Docker checks run against:

```text
[docker_servers]
```

This allows different servers to have different application requirements without creating separate patching playbooks.

## Controlled Execution

The lab uses Ansible's `serial` feature to control how many hosts enter the patching workflow at once.

This is useful for maintenance operations where patching every server simultaneously would create unnecessary risk.

Ansible `forks` is also configured separately to control how many hosts Ansible can process concurrently at the execution level.

These two settings serve different purposes:

* **`serial`** controls the rollout batch size of the play.
* **`forks`** controls Ansible's parallel execution capacity.

## Error Handling

Application validation is wrapped using Ansible `block` / `rescue`.

A failed validation can be handled without necessarily terminating the entire automation run.

The project uses this pattern to demonstrate host-level failure handling and the concept of marking a host as unsuitable for subsequent operations.

The intention is to distinguish between:

```text
Task failure
    ↓
Can this host recover / be handled?
    ↓
Yes → rescue / continue appropriately
No  → fail the host / stop according to workflow requirements
```

## Ansible Vault

Sensitive PostgreSQL credentials are stored using **Ansible Vault** rather than committed as plaintext.

Example:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Vault-protected variables are consumed by the PostgreSQL role during connectivity validation.

> The actual inventory, Vault files, and Vault password are intentionally excluded from this repository.

## Inventory

A sanitized example inventory is provided:

```text
inventory.example.ini
```

The actual local inventory is kept outside Git:

```text
inventory.local.ini
```

This allows the same playbook structure to be used without exposing private infrastructure information.

## Running the Playbook

Clone the repository:

```bash
git clone https://github.com/aidil-syakirin/ansible-linux-automation.git
cd ansible-linux-automation
```

Create your local inventory based on:

```text
inventory.example.ini
```

Then run:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Before execution, verify the inventory:

```bash
ansible-inventory --graph
```

Test connectivity:

```bash
ansible linux_servers -m ping
```

## Scheduled Execution

The playbook can also be executed automatically using Linux `cron`.

Example maintenance window:

```cron
0 2 * * 0
```

This runs the playbook every **Sunday at 02:00**.

For unattended execution, the Vault password can be supplied using a protected password file rather than interactive input. The password file must remain outside version control.

## Design Principles

The project follows a few simple principles:

**1. Playbook as orchestrator**

The main playbook defines the overall workflow and execution order.

**2. Roles as reusable automation**

Roles encapsulate tasks that can be reused in different parts of the workflow.

**3. Inventory-driven behaviour**

Host groups determine which application-specific checks are executed.

**4. Secrets separated from code**

Sensitive credentials are managed using Ansible Vault and are not stored in plaintext in the repository.

**5. Controlled change execution**

Patching is performed in controlled batches rather than blindly updating every server simultaneously.

## Learning Outcomes

This lab was built to move beyond basic Ansible syntax and understand how Ansible can be structured for an operational workflow.

Key concepts explored:

* Designing an Ansible project structure
* Separating orchestration from reusable roles
* Managing host and group variables
* Handling secrets with Ansible Vault
* Designing pre-check and post-check workflows
* Applying conditional execution based on inventory
* Understanding `serial` vs `forks`
* Handling task failures with `block` / `rescue`
* Scheduling infrastructure automation with `cron`

## Future Improvements

Potential improvements to the lab include:

* More robust recovery procedures
* Additional application health checks
* Centralised logging
* CI validation for Ansible syntax and linting
* Automated testing
* Integration with a CI/CD pipeline
* Terraform-based infrastructure provisioning

---

### Disclaimer

This is a **personal learning and portfolio project** built to demonstrate practical Ansible and infrastructure automation concepts.

The environment is a lab environment and is not intended to represent a complete production patch-management platform.

