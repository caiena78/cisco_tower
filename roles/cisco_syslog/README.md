# cisco_syslog

Remote syslog on **Cisco IOS/IOS-XE** and **NX-OS**. Combines `caiena78/ios_syslog` and
`caiena78/ansible_nxos_syslog`. Servers come from the NetBox site custom field (default `syslog`,
a JSON list of `{ip, port, protocol}`) with `cisco_syslog.servers` as fallback. Management interface
and VRF come from `network_facts`.

```yaml
- hosts: cisco
  gather_facts: false
  roles:
    - cisco_syslog
```

## What is pushed

| | IOS / IOS-XE | NX-OS |
|---|---|---|
| origin | `logging origin-id hostname` | `logging origin-id string <origin_id_prefix><hostname>` |
| source | `logging source-interface <int> [vrf <vrf>]` | `logging source-interface <int>` |
| servers | `logging host <ip> [vrf <vrf>] transport <udp\|tcp> port <n>` | `logging server <ip> <level> [port <n>] [use-vrf <vrf>]` |
| level | `logging trap <keyword>` | `logging level all <n>` |

IOS-XE with tcp/830 open: one NETCONF edit-config (candidate + commit when advertised, else running), CLI fallback on error. NX-OS and IOS without NETCONF: all lines in one `*_config` call. See `network_facts`.

## Variables (see `defaults/main.yml`)

| Variable | Purpose |
|---|---|
| `cisco_syslog_platform` | `ios` / `nxos`; auto-detected if empty |
| `cisco_syslog_netbox_enabled`, `cisco_syslog_netbox_field` | take servers from the NetBox site custom field |
| `cisco_syslog.servers` | fallback list `{ip, port=514, protocol=udp}` |
| `cisco_syslog.level` | 0-7 (6 = informational) |
| `cisco_syslog.source_interface`, `.vrf` | override the detected management interface / VRF (`vrf: none` forces the global table) |
| `cisco_syslog.origin_id_prefix` | NX-OS origin-id prefix (default `123456:`, the old hardcoded value) |

## Changes from the merged repos

* **IOS:** the VRF is now on the `logging host` line (before, only the source-interface had it, so logs left through the wrong routing table).
* **NX-OS:** the severity is on each `logging server` line, so remote servers get the configured level; the default port 514 is omitted and there is no double space, so reruns report no change (the old lines never matched `show run`).
* `port` defaults to 514 and `protocol` to udp; entries without an `ip` are skipped; a site without the NetBox field no longer fails the run (fallback list, or only global settings).
* NX-OS no longer depends on `ansible_net_hostname` (uses `inventory_hostname`); level is one setting (`cisco_syslog.level`) instead of `syslog_logging_level` (IOS) and `syslog_logging_nxos`.
* One task per platform instead of one per line.
* Config is saved only when `network_save_config: true`.

## Migrating

| Old | New |
|---|---|
| `syslog_logging_level` (name) / `syslog_logging_nxos` (number) | `cisco_syslog.level` (number) |
| `syslog_debug` | `cisco_syslog_debug` |
| role `netbox_getsite`, `get_mgnt_vrf`, `nxos_get_mgnt_vrf` | `network_facts` |
| `ios_syslog`, `ansible_nxos_syslog` | `cisco_syslog` |
