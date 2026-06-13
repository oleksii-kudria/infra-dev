# Ansible infrastructure management

This directory contains configuration management for infrastructure provisioned
by Terraform. Terraform remains responsible for creating Hetzner Cloud
resources; Ansible configures the resulting Ubuntu 24.04 hosts and deploys
services on them.

GitLab Runner is the first role scaffolded here. Future roles are expected for
Nginx, Kafka, Prometheus, and Grafana.

## Layout

```text
ansible/
├── ansible.cfg              # Project-level Ansible configuration
├── inventory/               # Environment inventories
│   └── dev.yml
├── group_vars/              # Variables grouped by inventory group
│   └── dev.yml
├── playbooks/               # Service entry-point playbooks
│   └── gitlab-runner.yml
└── roles/                   # Reusable component automation
    └── gitlab_runner/
```

Each future component should have a role under `roles/` with tasks, defaults,
handlers, templates, files, and role-specific documentation. Keep playbooks
small: select hosts, set privilege escalation, and compose roles.

## Inventory

Inventories are organized by environment under `inventory/`. The `dev` group in
`inventory/dev.yml` contains development hosts, while `group_vars/dev.yml`
provides variables for every host in that group.

The development inventory contains a static host entry for the single
development server. Add separate inventory files and group variable files for
future environments rather than mixing environments.

## Manual execution

An administrator runs Ansible manually from a local workstation. There is no
CI deployment, generated inventory, or Terraform-to-Ansible integration.

### Prerequisites

Install Ansible locally and ensure the workstation can connect to the
development server over SSH.

### Configure GitLab Runner token

Export the GitLab Runner token before execution:

```bash
export GITLAB_RUNNER_TOKEN="glrt-xxxxxxxx"
```

The playbook validates this value before running the GitLab Runner role and
exits with a clear error if it is missing.

### Verify inventory

Edit:

```text
inventory/dev.yml
```

Replace `SERVER_IP` with the actual server address. Update `ansible_user` in
that file if the server's SSH administrator account is not `admin`.

### Run playbook

```bash
cd infra/ansible

ansible-playbook \
  -i inventory/dev.yml \
  playbooks/gitlab-runner.yml
```

## Running a playbook

Run commands from this directory so that Ansible automatically loads
`ansible.cfg`:

```bash
cd infra/ansible
ansible-playbook --syntax-check -i inventory/dev.yml playbooks/gitlab-runner.yml
ansible-playbook -i inventory/dev.yml playbooks/gitlab-runner.yml
```

To preview changes once the role has real implementation tasks, use check mode:

```bash
ansible-playbook --check --diff -i inventory/dev.yml playbooks/gitlab-runner.yml
```

The development inventory is configured by default. An explicit inventory can
be selected with `-i inventory/<environment>.yml`.

## Secrets

Never commit registration tokens, passwords, private keys, or other
credentials. The GitLab Runner token is the only secret read from an environment
variable. Be aware that exported values may be retained in shell history. Tasks
that consume secrets should use `no_log: true`.
