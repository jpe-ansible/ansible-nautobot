# server

Installs and configures Nautobot with Redis, application settings and systemd units. With `server_manage_app` true,
the role also runs migrations and creates the first superuser if absent. With `server_manage_app` and
`server_manage_accounts` true, it also reconciles service accounts, object permissions, API tokens and published token
fields.

## Prerequisites

- A Debian-based host with systemd and network access to a PostgreSQL service.
- Privilege escalation for the play.
- The inputs below for the database, the superuser and OpenBao, supplied by the caller from its secrets store at run
  time: `server_db_password`, `server_superuser_password`, `server_openbao_addr`, `server_openbao_token` and
  `server_openbao_ca_file`. None has a role default.
- The `networktocode.nautobot` collection, which provides the permission module.

## Usage

```yaml
- name: Install Nautobot
  hosts: nautobot
  become: true
  roles:
    - role: jpe_ansible.nautobot.server
      vars:
        server_ui_fqdn: "{{ nautobot_ui_fqdn }}"
        server_db_password: "{{ nautobot_db_password }}"
        server_superuser_password: "{{ nautobot_superuser_password }}"
        server_openbao_addr: "{{ openbao_addr }}"
        server_openbao_token: "{{ openbao_token }}"
        server_openbao_ca_file: "{{ openbao_ca_file }}"
```

`meta/argument_specs.yml` lists every input with its type, and `defaults/main.yml` holds every default, including the
Renovate-managed package pins. Set `server_manage_app: false` to render configuration and units without touching the
application. Set `server_manage_accounts: false` to skip service account, permission, token and token-field publication
reconciliation while keeping application management enabled.

## What the role manages

| Item | Behavior |
| --- | --- |
| Settings and units | Templates for `nautobot_config.py`, the private environment file, uWSGI and three systemd units; a change restarts the services. |
| SECRET_KEY | Resolved and persisted using the precedence below; OpenBao updates preserve sibling fields. |
| Packages | The pinned application packages install into a virtual environment; migrations run on each converge and post-upgrade runs when the pins change. |
| Superuser | `createsuperuser --noinput` with the username, email and password in a protected environment; an existing username counts as unchanged. |
| Bootstrap integration | The SSoT `bootstrap` extra and the three Bootstrap settings. |
| Service accounts | One reader per consumer and one writer, each with its own object permissions and API token. |
| Environment file | Each `server_env` entry renders double-quoted with escapes. A bad key or a value with a line break fails the run. |

A non-empty `server_secret_key` takes precedence, followed by the OpenBao field when `server_manage_app` is true, the persisted
host file, and a newly generated key. The role updates the host file only when its content differs and publishes the OpenBao
field only when it differs from the effective key, using KV v2 check-and-set against the version just read while preserving
sibling fields. An unchanged converge creates no new KV version, and a rebuilt host reuses the stored key. With
`server_manage_app` false, the role does not read or write OpenBao.

## Secrets and command lines

Migration, post-upgrade and superuser commands source their environment from protected files instead of receiving secrets
through an Ansible task environment or command arguments. `.nautobot-cmd-env.sh` contains the sorted `server_env` entries
and stays on disk with mode `0600`, under the same trust boundary as `nautobot.env`; it is rendered on each run.
`.nautobot-superuser-env.sh` contains the three `DJANGO_SUPERUSER_*` values with mode `0600` and is removed after the
superuser command, including when that command fails.

## Running in stages

A run with no tag filter does everything in one pass. Tags let a caller run the role as separate stages, for a runner
with a time limit per task:

- **1 install**, tag `server_install`: system packages, service account, directories, Redis, virtual environment and
  pinned packages. To run only this stage, skip `server_secret,server_migrate,server_config`.
- **2 migrate**, tag `server_migrate`: migrations, post-upgrade and the first superuser. These need the SECRET_KEY, so
  `server_secret` also runs. To run only this stage, skip `server_install,server_config`.
- **3 config**, tag `server_config`: settings, environment file, uWSGI, units, handler flush, service start, web wait
  and accounts. These need the SECRET_KEY, so `server_secret` also runs. To run only this stage, skip
  `server_install,server_migrate`.

Run the stages in this order on an upgrade: the new settings and the service restart in stage 3 take effect only after
stage 1 has installed the packages and stage 2 has migrated the database. Stage 1 replaces package files while the
running services still hold the old code, and the restart in stage 3 loads the new code. Redis changes restart Redis
inside stage 1, and configuration changes restart the services inside stage 3. Stage 3 also restarts the services when
the package pins differ from the pins recorded at the last service restart, so a package-only upgrade loads the new
code. A fresh host needs one run with no tag filter, because migrations need the settings file.

Under `--check`, the migration and post-upgrade return-code checks and the web wait are skipped, because their
commands and request do not run.

## Network exposure

The web service listens on plain HTTP at `server_web_bind` and `server_web_port`. `server_web_bind` defaults to
`127.0.0.1`. When a TLS proxy runs on another host, set `server_web_bind` to the address that proxy reaches, and keep
that address off untrusted networks. With `server_trust_proxy_ssl_header` true, the settings trust the forwarded scheme
header and mark the session and CSRF cookies secure. Set it to false only when clients reach the service without a
proxy.

## Service accounts

Each account has a username, an OpenBao path, a token field and a URL field, all inputs named
`server_<account>_username`, `_token_path`, `_token_key` and `_url_key`. The accounts are `inventory`, `mcp`,
`pipeline` and `write`.

| Account | Object types | Actions | Constraint | Live assertion |
| --- | --- | --- | --- | --- |
| `inventory` | `server_inventory_object_types` | `view` | None | VirtualMachines are invisible; a Device POST is denied. |
| `mcp` | `server_mcp_object_types` | `view` | None | A Device POST is denied; token sweep removes an extra token. |
| `pipeline` | `server_pipeline_object_types` | `view` | None | Pre-flight, export and drift run; saved-query `run` is denied. |
| `write` | `server_write_object_types` | `view`, `add`, `change` | None | Inbound sync and flow-owned VirtualMachine create/delete succeed. |
| `write` | `server_write_delete_types` | `delete` | None | Sync prune succeeds; Location and Prefix DELETE are denied. |
| `write` | `server_write_reference_types` | `view`, `add`, `change` | None | Inbound sync creates and reads shared references. |
| `write` | `server_write_lookup_types` | `view` | None | Inbound sync resolves existing lookup records. |
| `write` | `extras.gitrepository` | `view`, `add`, `change` | `name` equals `server_bootstrap_repository_name` | Only the named repository changes. |
| `write` | `extras.job` | `view`, `run` | `name` equals `server_bootstrap_job_name` | A stock Job run is denied. |
| `write` | `extras.jobresult` | `view` | None | Bootstrap job results are readable. |
| All accounts | `users.token` | `view`, `add`, `delete` | The token user is `$user` | Each account authenticates; model writes by readers are denied. |

API tokens are write-enabled so each service account can manage its own token. Object permissions still limit the
three readers to `view` on their Nautobot data types. The writer's data mutations are split between the write types
and the exact delete types used by sync pruning.

Each run follows these steps:

1. Probe each stored token by listing the tokens it owns; an account is unchanged only when the token answers and the
   list contains that exact token key with the account description and expected write flag.
2. Delete leftover temporary superuser tokens, then create a new one with HTTP Basic authentication. It expires after
   `server_temporary_token_minutes`.
3. For accounts without a working stored token, create or update the account with a one-time random password.
4. Apply the object permissions of every account.
5. For accounts without a working stored token, create the account's token by authenticating as that account, and
   publish it to OpenBao with a KV v2 write that keeps the sibling fields of the secret and fails when the secret
   changed since it was read.
6. List each account's own tokens with its published token and delete every token whose key differs from the key
   published in OpenBao.
7. Replace passwords for accounts whose token was recreated with a new random value that nothing stores, then delete
   the temporary superuser token in an `always` section.

The role sweeps each account's token list on every run using that account's published token, deleting tokens whose key
does not match the key in OpenBao. A holder of a service token can mint more tokens for that account until the next
account run removes them, so the residual exposure window is one run interval. The role does not set expirations on
service tokens because the scheduled consumers run unattended.

All Nautobot and OpenBao calls run on the controller. No password or token is written to disk or sent to the managed
host. OpenBao and Nautobot are reached with certificate validation enabled, and the role fails when
`server_nautobot_validate_certs` is false.

## Known limits

- The role does not reset the superuser password after the first run; rotate it through Nautobot.
- A failed run can leave a service account without a working token; the next run repeats the reconcile for that
  account.
- Redis accepts connections from loopback only and has no password; the role assumes a single guest.

## References

- [Django createsuperuser](https://docs.djangoproject.com/en/6.1/ref/django-admin/#createsuperuser)
- [networktocode.nautobot admin_permission](https://docs.nautobot.com/projects/ansible/en/latest/networktocode.nautobot/module/admin_permission.html)
- [SSoT Bootstrap setup](https://docs.nautobot.com/projects/ssot/en/latest/admin/integrations/bootstrap_setup/)
