# pulsar_relay (Ansible role)

Deploys [pulsar-relay](https://github.com/mvdbeek/pulsar-relay) as a systemd
service, backed by **Valkey** (Redis-compatible) so credentials and in-flight
messages survive a restart.

The relay is the HTTP broker between Galaxy and a remote Pulsar running in
relay mode: both sides reach it by long-polling, so the compute site needs no
inbound ports and no AMQP broker.

## What it does

- Installs Valkey (snap) when the Valkey backend is selected
- Installs Python 3 + venv and pulsar-relay into a virtualenv
- Renders and enables the `pulsar-relay` systemd unit
- Creates the relay accounts listed in `pulsar_relay_users`

## Role variables

See `defaults/main.yml`. Override at least the secrets:

| Variable | Default | Notes |
|----------|---------|-------|
| `pulsar_relay_bind` | `127.0.0.1` | listen address, put nginx in front |
| `pulsar_relay_port` | `9000` | Listen port |
| `pulsar_relay_service_name` | `galaxy-pulsar-relay` | systemd unit name |
| `pulsar_relay_user` | `pulsar-relay` | system user, created by the role |
| `pulsar_relay_venv` | `/opt/pulsar-relay/venv` | Virtualenv path |
| `pulsar_relay_jwt_secret` | `change_me_in_production` | **override** |
| `pulsar_relay_admin_username` | `admin` | |
| `pulsar_relay_admin_password` | `change_me_in_production` | **override** |
| `pulsar_relay_storage_backend` | `valkey` | `valkey` or `memory` |
| `pulsar_relay_valkey_host` | `localhost` | |
| `pulsar_relay_valkey_port` | `6379` | |
| `pulsar_relay_valkey_password` | `CHANGE_ME` | **override** |
| `pulsar_relay_manage_valkey` | `true` | `false` if valkey comes from another role (e.g. `usegalaxy-eu.valkey`) |
| `pulsar_relay_allowed_origins` | `["https://usegalaxy.eu"]` | |
| `pulsar_relay_trusted_hosts` | `["*"]` | set to the public hostname |

| `pulsar_relay_users` | `[]` | relay accounts to create, see below |
| `pulsar_relay_users_default_permissions` | `[read, write]` | for entries without `permissions` |
| `pulsar_relay_users_update_password` | `false` | also set passwords of existing users |
| `pulsar_relay_api_url` | `http://127.0.0.1:<port>` | how the role reaches the relay (from the relay host) |
| `pulsar_relay_api_host` | first non-wildcard trusted host | `Host` header for those calls |

> Supply real secrets via the vault / extra-vars.

## Relay accounts

Each Pulsar logs in to the relay with its own account. List them and the role
creates the missing ones through the relay API, logged in as the admin:

```yaml
pulsar_relay_users:
  - username: nemo-3dtrees
    password: "{{ vault_pulsar_nemo_3dtrees_relay_password }}"
    # permissions: [read, write]   # default; never give a Pulsar admin
```

- create-only: existing users are not changed, users that aren't listed are
  never touched (the relay also has OIDC / BYOC accounts)
- the list is what you rebuild from if Valkey loses its data
- rotating a password: change it in the vault and run once with
  `-e pulsar_relay_users_update_password=true` (always reports changed, the
  relay doesn't hand out hashes to compare)
- one admin login per run (`/auth/login` allows 5/minute per client); its
  tokens are revoked at the end
- skipped in check mode

## Example

```yaml
- hosts: relay
  become: true
  roles:
    - role: pulsar_relay
      vars:
        pulsar_relay_jwt_secret: "{{ vault_pulsar_relay_jwt_secret }}"
        pulsar_relay_admin_password: "{{ vault_pulsar_relay_admin_password }}"
```

## Notes

- needs Valkey/Redis >= 6.2, the role checks it
- valkey runs with appendonly. if valkey already has data, do `CONFIG SET appendonly yes` before applying the role, otherwise it can start empty
- coming from the old `pulsar-relay` unit: the role stops and removes it, incl. `/etc/pulsar-relay.env`

## Author

Created by Dmitrijs Sizovs in 2026.
