# GitLab Runner role

This role will install, register, and manage GitLab Runner on Ubuntu 24.04
hosts. The current tasks and handler are safe placeholders that define the
intended workflow without changing a target host.

## Planned responsibilities

1. Add the official GitLab Runner package repository and install the runner.
2. Install or validate Docker and grant the runner account Docker access.
3. Register the runner idempotently with the configured GitLab instance.
4. Restart and enable the GitLab Runner service when its configuration changes.

## Variables

Non-secret defaults are documented in `defaults/main.yml`. Environment-specific
values belong in `group_vars/`. `gitlab_runner_token` is sensitive and is read
from the `GITLAB_RUNNER_TOKEN` environment variable. Never commit a real token.

The development environment currently supplies:

- `gitlab_url`
- `gitlab_runner_description`
- `gitlab_runner_executor`
- `gitlab_runner_default_image`
- `gitlab_runner_token` (placeholder only)

## Extending the role

Put generated configuration files in `templates/` and static payloads in
`files/`. Tasks should remain idempotent, use fully qualified collection names,
avoid logging secrets, and notify handlers only when a managed resource changes.
