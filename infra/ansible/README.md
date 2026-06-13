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

Run the commands from `infra/ansible`. Export the runner authentication token,
then pass the token and target SSH connection details as runtime variables:

```bash
export GITLAB_RUNNER_TOKEN="glrt-xxxxxxxx"

ANSIBLE_CONFIG=./ansible.cfg \
ansible-playbook \
  -i inventory/dev.yml \
  playbooks/gitlab-runner.yml \
  -e gitlab_runner_token="$GITLAB_RUNNER_TOKEN" \
  -e ansible_host=<SERVER_IP> \
  -e ansible_user=<SSH_USER>
```

Replace `<SERVER_IP>` and `<SSH_USER>` with the target server's address and SSH
account. The playbook validates the token before running the GitLab Runner role
and exits with a clear error if it is missing.

| Parameter | Description |
| --- | --- |
| `GITLAB_RUNNER_TOKEN` | GitLab Runner authentication token generated in GitLab |
| `ansible_host` | Target server IP address |
| `ansible_user` | SSH user used to connect to the server |
| `ANSIBLE_CONFIG` | Explicit path to `ansible.cfg` |

### WSL and `ansible.cfg`

When running Ansible from WSL with the repository stored under `/mnt/c/...`,
Ansible may display this warning:

```text
Ansible is being run in a world writable directory
```

In this situation, Ansible may not automatically load the project configuration.
Specify `ANSIBLE_CONFIG=./ansible.cfg` explicitly, as shown in the deployment
command above.

## Verification and troubleshooting

Confirm that the runner token is set in the current shell:

```bash
echo $GITLAB_RUNNER_TOKEN
```

Inspect the resolved development inventory:

```bash
ansible-inventory -i inventory/dev.yml --list
```

Check the playbook syntax before deployment:

```bash
ansible-playbook \
  --syntax-check \
  -i inventory/dev.yml \
  playbooks/gitlab-runner.yml
```

If Ansible does not use the repository configuration, prefix verification
commands with `ANSIBLE_CONFIG=./ansible.cfg`. To preview changes once the role
has real implementation tasks, add `--check --diff` to the deployment command.

## Security notes

- GitLab Runner tokens must never be committed to Git.
- Do not store real tokens or other secret values in inventory files.
- Do not store real tokens or other secret values in `group_vars` files.
- Provide tokens through environment variables during execution.

Also never commit passwords, private keys, or other credentials. Be aware that
exported values may be retained in shell history. Tasks that consume secrets
should use `no_log: true`.
