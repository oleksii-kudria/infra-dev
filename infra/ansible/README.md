# Ansible infrastructure management

This directory contains configuration management for infrastructure provisioned
by Terraform. Terraform remains responsible for creating Hetzner Cloud
resources; Ansible configures the resulting Ubuntu 24.04 hosts and deploys
services on them.

The development playbook configures host-level Nginx reverse proxying, obtains
Let's Encrypt certificates with Certbot, and deploys GitLab Runner.

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
    ├── certbot/
    ├── gitlab_runner/
    └── nginx/
```

Each component should have a role under `roles/` with tasks, defaults,
handlers, templates, files, and role-specific documentation. Keep playbooks
small: select hosts, set privilege escalation, and compose roles.


## Required configuration before deployment

Before running Ansible against a real server, review and customize these files:

- `inventory/dev.yml`: contains the inventory hosts for the development environment. Replace the placeholder host address (`ansible_host`) with the target server IP or DNS name and update `ansible_user` if your SSH account is different.
- `group_vars/dev.yml`: contains environment-wide variables for the `dev` group, including `base_domain`, `nginx_sites`, Certbot settings, and GitLab Runner defaults. Replace the placeholder `base_domain` value and the Certbot email address before first deployment.
- Additional `group_vars/<environment>.yml` files: if you add another environment, create or update its group variables with the same required values instead of reusing development-specific settings.

`nginx_sites` is intentionally an environment variable only. Do not set
an empty `nginx_sites` value in role defaults, playbooks, or role vars: it can
hide missing environment configuration and cause Nginx and Certbot site loops to
skip. The GitLab Runner playbook and the Nginx and Certbot roles validate the
effective value before making changes, and both roles consume the same
`nginx_sites` list loaded from the selected inventory group variables.

At minimum, change these variables before first deployment:

- `base_domain`: replace `example.com` with the real base domain that points to the server. The public service hostnames are generated from this single value.
- `certbot_email`: replace `admin@example.com` with the administrator email for Let's Encrypt registration and renewal notices.
- `ansible_host`: replace `SERVER_IP` in the inventory or provide the real address with `-e ansible_host=<SERVER_IP>`.
- `ansible_user`: update the inventory value or provide `-e ansible_user=<SSH_USER>` if the SSH username is not `admin`.
- `gitlab_runner_token`: provide a real token through `GITLAB_RUNNER_TOKEN` or `-e gitlab_runner_token=...`; do not commit tokens.

These variables are usually safe to leave unchanged unless your services listen on different local ports or you intentionally want different behavior:

- `nginx_sites[*].name`: local Nginx config filename prefix for each site.
- `nginx_sites[*].upstream_host`: defaults to `127.0.0.1` so only local services are proxied.
- `nginx_sites[*].upstream_port`: keep the documented ports if your local services use the default catalog API and Grafana ports.
- `certbot_agree_tos`: must remain `true` for non-interactive certificate issuance.
- `certbot_redirect_http_to_https`: keep `true` to enforce HTTP-to-HTTPS redirects.
- `gitlab_runner_executor` and `gitlab_runner_default_image`: keep these defaults unless you need a different runner executor or base image.

Example `group_vars/dev.yml` values after customization:

```yaml
base_domain: example.com

nginx_sites:
  - name: catalog-api
    server_name: "catalog-api.{{ base_domain }}"
    upstream_host: 127.0.0.1
    upstream_port: 8082

  - name: catalog-grafana
    server_name: "catalog-grafana.{{ base_domain }}"
    upstream_host: 127.0.0.1
    upstream_port: 3000

certbot_email: admin@example.com
```

## Inventory and variables

Inventories are organized by environment under `inventory/`. The `dev` group in
`inventory/dev.yml` contains development hosts, while `group_vars/dev.yml`
provides variables for every host in that group.

The development inventory contains a static host entry for the single
development server. Add separate inventory files and group variable files for
future environments rather than mixing environments.

### Nginx domains and upstreams

Public Nginx reverse proxy sites are configured in `group_vars/dev.yml` with a
single `base_domain` value plus `nginx_sites`. Before running Ansible, change
only the placeholder base domain to your real domain:

```yaml
base_domain: example.com
```

For example:

```yaml
base_domain: your-domain.example
```

Each `nginx_sites` entry derives its public hostname from `base_domain` and
defines the local backend address that Nginx proxies to:

```yaml
base_domain: example.com

nginx_sites:
  - name: catalog-api
    server_name: "catalog-api.{{ base_domain }}"
    upstream_host: 127.0.0.1
    upstream_port: 8082

  - name: catalog-grafana
    server_name: "catalog-grafana.{{ base_domain }}"
    upstream_host: 127.0.0.1
    upstream_port: 3000
```

Only these public domain patterns are generated and configured:

- `catalog-api.<base_domain>`
- `catalog-grafana.<base_domain>`

For the default placeholder, these resolve to:

- `catalog-api.example.com`
- `catalog-grafana.example.com`

The `nginx` role installs Nginx, removes the default virtual host, creates one
configuration in `/etc/nginx/sites-available/` for each `nginx_sites` entry,
enables each site with a symlink in `/etc/nginx/sites-enabled/`, validates the
configuration with `nginx -t`, and reloads Nginx only after validation succeeds.

### Certbot configuration

Certbot settings are also defined in `group_vars/dev.yml`:

```yaml
certbot_email: admin@example.com
certbot_agree_tos: true
certbot_redirect_http_to_https: true
```

Update `certbot_email` before deployment if certificates should be registered
to a different administrator address. Certificate issuance uses the domains from
`nginx_sites`; do not add domain names directly to role tasks or templates.

The `certbot` role installs `certbot` and `python3-certbot-nginx`, obtains a
certificate for every configured Nginx site, enables HTTP-to-HTTPS redirects,
ensures the `certbot.timer` renewal unit is enabled and started, and installs a
renewal deploy hook that validates and reloads Nginx after successful renewal.

## Manual execution

An administrator runs Ansible manually from a local workstation. There is no
CI deployment, generated inventory, or Terraform-to-Ansible integration.

### Prerequisites

Install Ansible locally and ensure the workstation can connect to the
development server over SSH. DNS for `catalog-api.<base_domain>` and `catalog-grafana.<base_domain>` must
point at the target server before running Certbot, and Hetzner Firewall must
allow TCP/80 and TCP/443.

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

The playbook applies roles in this order:

1. `nginx`
2. `certbot`
3. `gitlab_runner`

Nginx is installed and validated before Certbot attempts certificate issuance.
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

Check the playbook syntax before deployment:

```bash
ansible-playbook \
  --syntax-check \
  -i inventory/dev.yml \
  playbooks/gitlab-runner.yml
```

After the playbook completes, connect to the server and verify Nginx, Certbot,
GitLab Runner, and Docker access:

```bash
ssh <SSH_USER>@<SERVER_IP>

sudo nginx -t
sudo systemctl status nginx
sudo certbot renew --dry-run

gitlab-runner --version
systemctl status gitlab-runner
sudo gitlab-runner verify
sudo -u gitlab-runner docker ps
```

The public HTTPS endpoints should be available at the generated public hostnames:

- `https://catalog-api.<base_domain>`
- `https://catalog-grafana.<base_domain>`

With the default placeholder, these are:

- `https://catalog-api.example.com`
- `https://catalog-grafana.example.com`

HTTP requests to the same hosts should redirect to HTTPS.

Confirm that the runner token is set in the current shell:

```bash
echo $GITLAB_RUNNER_TOKEN
```

Inspect the resolved development inventory:

```bash
ansible-inventory -i inventory/dev.yml --list
```

If Ansible does not use the repository configuration, prefix local Ansible
commands with `ANSIBLE_CONFIG=./ansible.cfg`. To preview changes, add
`--check --diff` to the deployment command. Do not use check mode for the final
Certbot issuance run because Let's Encrypt certificate creation requires live
network calls and host validation.

## Security notes

- GitLab Runner tokens must never be committed to Git.
- Do not store real tokens or other secret values in inventory files.
- Do not store real tokens or other secret values in `group_vars` files.
- Provide tokens through environment variables during execution.
- Keep internal services off public firewall rules; expose only Nginx on TCP/80
  and TCP/443 for HTTP and HTTPS traffic.

Also never commit passwords, private keys, or other credentials. Be aware that
exported values may be retained in shell history. Tasks that consume secrets
should use `no_log: true`.
