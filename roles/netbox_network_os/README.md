# netbox_network_os

Sets `ansible_network_os` for each host from the device's **platform in NetBox**, so the Cisco roles
(`network_facts`, `cisco_snmp`, `cisco_aaa`, `cisco_syslog`) and the `cisco.*` connection plugins pick
the right platform. Run it before anything that connects to the device.

```yaml
- hosts: cisco
  gather_facts: false
  roles:
    - netbox_network_os
    - network_facts
    - cisco_snmp
```

## How it works

1. `GET /api/dcim/devices/?name=<inventory_hostname>` (from the controller, `delegate_to: localhost`).
2. Takes `platform.slug`, lower-cased, and maps it with `netbox_network_os_map`
   (`nx-os`/`nxos` -> `cisco.nxos.nxos`, `ios`/`ios-xe` -> `cisco.ios.ios`, `cisco-wlc` -> `community.network.aireos`, ...).
3. `set_fact: ansible_network_os`. Runs once per host (`netbox_network_os_done`).

## NetBox credentials

Same convention as the rest of the project: `NETBOX_URL` and `NETBOX_API` in the environment, filled from
Vault (see `.env`: `set -a; source .env; set +a`, with an authenticated Vault session). Nothing secret is
stored in the role, and there are no placeholder values; the role stops with a clear message if they are empty.
Requests use `no_log`. TLS verification follows `netbox_validate_certs` (on if unset).

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `netbox_network_os_map` | see `defaults/main.yml` | platform slug -> `ansible_network_os`; extend with your slugs |
| `netbox_network_os_overwrite` | `true` | `false` keeps an `ansible_network_os` that is already set |
| `netbox_network_os_on_unmapped` | `fail` | device missing / no platform / slug not in map: `fail`, `skip`, or `default` |
| `netbox_network_os_default` | `""` | value used with `on_unmapped: default` |
| `netbox_network_os_device_name` | `inventory_hostname` | NetBox device name to look up |
| `netbox_network_os_debug` | `false` | print slug -> value per host |

## Notes

* `netbox_inventory.yml` composes `ansible_network_os: platform[0]`. A role fact set with `set_fact`
  takes precedence over inventory vars, so with `overwrite: true` this role wins. You may want to drop
  that `compose` line (or the `platform[0]` expression, which looks unintended) once this role is in use.
* One NetBox request per host. Fine for normal runs; for thousands of hosts a single bulk lookup would be faster.
* Device names must be unique in NetBox; more than one match stops the host rather than guessing.
