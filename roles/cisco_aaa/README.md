# cisco_aaa

TACACS+ AAA on **Cisco IOS/IOS-XE** (SSH or telnet), **NX-OS** and **AireOS WLC**, plus the matching
Cisco ISE network-device sync. Combines `caiena78/ios_aaa`, `ios_aaa_telnet`, `nxos_aaa` and `airos_aaa`.
(`airos` here is Cisco AireOS, not Ubiquiti.)

```yaml
- hosts: cisco
  gather_facts: false
  roles:
    - cisco_aaa
```

## How it works

`main.yml`: `network_facts` -> map legacy variables (`compat.yml`) -> `ise.yml` -> one platform file.

| File | Platform |
|---|---|
| `ios.yml` | IOS/IOS-XE over SSH (`ios_config`) |
| `ios_telnet.yml` | IOS over telnet (`ansible.netcommon.telnet`), `cisco_aaa_transport: telnet` |
| `nxos.yml` | NX-OS |
| `aireos.yml` | AireOS WLC (`ansible_network_os: community.network.aireos`) |

`ios.yml` and `ios_telnet.yml` both use `ios_build.yml`, which builds the ordered config blocks once,
so the two transports cannot drift apart. The telnet path has no running-config diffing, so it always
reports changed and always ends with `write memory`.

Transport is CLI for every platform: TACACS keys and local passwords are stored hashed by the device
and cannot be diffed over NETCONF.

## Variables (see `defaults/main.yml`)

Key ones: `cisco_aaa.key` (one shared secret for device and ISE - keep in Vault), `cisco_aaa.servers`
(`name`, `address`, optional `key`/`timeout`/`port`), `cisco_aaa.group_name`, `cisco_aaa.local_user`,
`cisco_aaa_cleanup`, `cisco_aaa_transport`, `cisco_aaa_ise_enabled`. ISE connection variables
(`ise_hostname`, `ise_username`, `ise_password`, `ise_verify`, `ise_port`) are shared with `cisco_snmp`
and have no placeholder defaults; the role stops with a clear message if ISE sync is on and they are missing.

## Migrating from the old repos

The old names still work (mapped in `compat.yml`), so existing Tower surveys keep working:

| Old | New |
|---|---|
| `tacacs` list (`name, address, key, host_timeout, server_type, port, encrypt_type`) | `cisco_aaa.servers` (`host_timeout` -> `timeout`) |
| `tacacspwd` (ISE) + `tacacs[].key` (device) | `cisco_aaa.key` (one secret; `cisco_aaa.ise.shared_secret` to keep them different) |
| `tacacs_group_name` | `cisco_aaa.group_name` |
| `LOCAL_USER` / `LOCAL_PWD` / `LOCAL_ENABLE` (`"NONE"` = skip) | `cisco_aaa.local_user.{username,password,enable_secret}` (empty = skip) |
| `remove_undefined_servers` | `cisco_aaa_cleanup` |
| `ISE_LOCATION`, `ISE_DESCRIPTION`, `ISE_DEVICE_TYPE`, `ise_device_name`, `ise_skip_log_file` | `cisco_aaa.ise.*` |
| `ios_telnet_use_legacy_tacacs_syntax`, `telnet_*` | `cisco_aaa.ios.legacy_tacacs_syntax`, `cisco_aaa.telnet.*` |
| role `ios_aaa_telnet` | `cisco_aaa_transport: telnet` |

## Changes from the merged repos

* **One ISE sync** (4 copies, 2 diverging versions): always the newer one, which also updates the Device Type group on existing devices. The ISE device type is per platform (`cisco_aaa.ise.device_type`).
* **IOS:** `user` -> `username` in the scrypt branch (it always fell through to the fallback); the three overlapping source-interface tasks became one VRF-aware block; modern-vs-legacy TACACS syntax is judged over the whole run (was only the first server) and can be forced with `legacy_tacacs_syntax`; removed a broken `var: netbox_getsite` debug.
* **NX-OS:** `use-vrf <vrf>` is now applied to the server group (the VRF was detected but never used); group name is not hardcoded; `ip tacacs source-interface` is skipped when no management interface is found (it pushed `NOT_SET`); `LOCAL_ENABLE` is no longer required.
* **AireOS:** only slots that differ are changed (the old role deleted and re-added all of them on every run, dropping TACACS briefly each time); `mgmtuser add` is skipped when the user exists. Keys cannot be read back, so a key-only change is not detected - use `cisco_aaa.aireos.force_rewrite: true` for that.
* **Secrets:** `no_log` on every task that carries keys or passwords (set `cisco_aaa_debug: true` to see output while debugging).
* **Telnet:** shares the IOS command builder instead of a 28 KB copy.

## Behaviors kept as they were

* Command-authorization (`aaa authorization commands ...`) and named accounting lists are removed on IOS before the default lists are configured.
* Keys and local passwords are re-sent every run (the device stores them hashed), so those tasks report changed.
* AireOS: three TACACS servers maximum, port 49, `aaa auth mgmt tacacs local`.
