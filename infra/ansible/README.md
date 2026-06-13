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

The development inventory reads its host address and SSH user from environment
variables. Add separate inventory files and group variable files for future
environments rather than mixing environments.

## Environment Variables

Before running Ansible, export the required variables:

```bash
export DEV_SERVER_IP="x.x.x.x"
export GITLAB_RUNNER_TOKEN="glrt-xxxxxxxx"
```

The playbook validates both values before attempting to configure a remote host.
It exits with a clear error if either required variable is empty.

Optional variables have project defaults and can be overridden as needed:

```bash
export ANSIBLE_USER="admin"
export GITLAB_URL="https://gitlab.com/"
export GITLAB_RUNNER_DESCRIPTION="product-catalog-dev-runner"
export GITLAB_RUNNER_EXECUTOR="docker"
export GITLAB_RUNNER_DEFAULT_IMAGE="docker:27-cli"
```

Run the playbook:

```bash
cd infra/ansible

ansible-playbook \
  -i inventory/dev.yml \
  playbooks/gitlab-runner.yml
```

For one-line execution from `infra/ansible`:

```bash
DEV_SERVER_IP="x.x.x.x" \
GITLAB_RUNNER_TOKEN="glrt-xxxxxxxx" \
ansible-playbook -i inventory/dev.yml playbooks/gitlab-runner.yml
```

The optional `.env.example` file documents all supported variables. If a local
`.env` file is used with a shell environment loader, it remains ignored by Git.
Ansible does not load `.env` files automatically.

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

Never commit registration tokens, passwords, private keys, `.env` files, or
other credentials. Sensitive and environment-specific values are read from
environment variables. CI/CD jobs should inject them as protected variables.

Do not commit real secrets to the repository. Be aware that values entered
directly in a shell command can be retained in shell history. Tasks that consume
secrets should use `no_log: true`.
