# GitLab Runner role

This role installs, registers, and manages GitLab Runner on Ubuntu 24.04 hosts.

## Responsibilities

1. Add the official GitLab Runner package repository and install the runner.
2. Grant the runner service account access to an existing Docker installation.
3. Register the Docker executor unless a runner with the configured description
   already exists in `/etc/gitlab-runner/config.toml`.
4. Restart and enable the GitLab Runner service when its configuration or group
   membership changes.
5. Verify the installed binary and registered runner.

## Variables

Non-secret defaults are documented in `defaults/main.yml`. Environment-specific
values belong in `group_vars/`. `gitlab_runner_token` is sensitive and is read
from the `GITLAB_RUNNER_TOKEN` environment variable. Never commit a real token.

The development environment currently supplies:

- `gitlab_url`
- `gitlab_runner_description`
- `gitlab_runner_executor`
- `gitlab_runner_default_image`
- `gitlab_runner_token` (read from the local environment)

Docker and the `docker` group must exist before this role runs. Registration
commands and configuration inspection use `no_log: true` so authentication
tokens stored by GitLab Runner are not exposed in Ansible output.
