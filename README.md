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

> Supply real secrets via the vault / extra-vars.

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
