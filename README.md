# ansible-role-app-compose

[![CI](https://github.com/pluggero/ansible-role-app-compose/actions/workflows/ci.yml/badge.svg)](https://github.com/pluggero/ansible-role-app-compose/actions/workflows/ci.yml) [![Ansible Galaxy downloads](https://img.shields.io/ansible/role/d/pluggero/app_compose?label=Galaxy%20downloads&logo=ansible&color=%23096598)](https://galaxy.ansible.com/ui/standalone/roles/pluggero/app_compose)

Render a Docker Compose stack (`compose.yml`, `.env`, and optional extra config
files) into a directory on a target host and deploy it with
`community.docker.docker_compose_v2`. Built for GitOps pipelines: one
possible way among several to deploy an application, generic and agnostic of
any specific project's directory layout, secrets backend, or compose
topology — all of that is resolved by the caller and passed in as plain role
variables.

## Requirements

**Collections:**

- `community.docker` (provides `docker_compose_v2`)

**On the target host:**

- Docker Engine and the Docker Compose v2 CLI plugin must already be
  installed and usable. This role deliberately does not install Docker
  itself — that is the caller's responsibility (or another role's), keeping
  `app_compose` focused on rendering + deploying.

## Role Variables

Available variables are listed below, along with default values (see
`defaults/main.yml`) and full type/choice constraints (see
`meta/argument_specs.yml`).

### Required

| Variable | Type | Description |
| --- | --- | --- |
| `app_compose_app_name` | str | Application name; used to derive `app_compose_dir` and `app_compose_project_name`. |
| `app_compose_source_compose_file` | path | Path to a compose file or Jinja2 template, rendered into `<app_compose_dir>/compose.yml` (mode `0640`). Use `app_compose_secrets` for secrets, not this file. |
| `app_compose_secrets` | dict | Secret key/value pairs merged into `.env`. Wins over `app_compose_env_vars` on key collision. `no_log: true`. |

### Optional

| Variable | Default | Description |
| --- | --- | --- |
| `app_compose_dir` | `/opt/{{ app_compose_app_name }}` | Deploy directory on the target host. |
| `app_compose_env_vars` | `{}` | Plain (non-secret) key/value pairs merged into `.env`. |
| `app_compose_extra_templates` | `[]` | List of `{src, dest, mode}`. Each `dest` is relative to `app_compose_dir`. Lets a caller add e.g. a reverse-proxy config without this role knowing anything about reverse proxies. |
| `app_compose_rollback` | `false` | Roll back `.env` to the previous deploy instead of rendering a new one. See [Rollback](#rollback). |
| `app_compose_pull_policy` | `always` | One of `always`, `missing`, `never`, `build`. Passed to `docker_compose_v2`'s `pull`. |
| `app_compose_wait_timeout` | `120` | Seconds to wait for the stack to report healthy. |
| `app_compose_project_name` | `{{ app_compose_app_name }}` | Compose project name passed to `docker_compose_v2`. |

### Vault Variables

This role takes no vault variables directly — `app_compose_secrets` is a
plain dict and the caller decides where its values come from. The expected
pattern is to source it from Ansible Vault in your own inventory/group vars:

```yaml
app_compose_secrets:
  DB_PASSWORD: "{{ vault_myapp_db_password }}"
  API_KEY: "{{ vault_myapp_api_key }}"
```

## Example Playbook

```yaml
- hosts: app_servers
  roles:
    - role: pluggero.app_compose
      vars:
        app_compose_app_name: myapp
        app_compose_source_compose_file: files/myapp/compose.yml.j2
        app_compose_env_vars:
          LOG_LEVEL: info
        app_compose_secrets:
          DB_PASSWORD: "{{ vault_myapp_db_password }}"
        app_compose_extra_templates:
          - src: files/myapp/caddy.conf.j2
            dest: caddy/Caddyfile
            mode: "0640"
```

## Rollback

Set `app_compose_rollback: true` to skip rendering a new `.env` and instead
swap the current `.env` with the previously deployed `.env.previous`:

```yaml
- hosts: app_servers
  roles:
    - role: pluggero.app_compose
      vars:
        app_compose_app_name: myapp
        app_compose_source_compose_file: files/myapp/compose.yml.j2
        app_compose_secrets: "{{ app_compose_secrets_unused_in_rollback | default({}) }}"
        app_compose_rollback: true
```

Notes:

- `app_compose_secrets` is still required by the role's argument spec even
  in rollback mode (pass `{}` or your usual secrets dict — it is not used to
  render anything when rolling back).
- The role fails loudly if there is no `.env.previous` to roll back to —
  deploy normally at least once first.
- The swap is a true toggle: running with `app_compose_rollback: true` twice
  in a row swaps back to the config you just rolled back from.
- `compose.yml` and any `app_compose_extra_templates` are still re-rendered
  during a rollback run; only `.env` handling changes.

## License

MIT

## Author Information

This role was created in 2026 by pluggero.
