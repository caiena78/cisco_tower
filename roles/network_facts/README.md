# network_facts

Shared helper used by `cisco_snmp`, `cisco_aaa` and `cisco_syslog`. Runs once per host per play
(guarded by `network_facts_done`) and replaces `netbox_getsite`, `get_mgnt_vrf`,
`ansible_ios_get_mgnt_vrf` and `nxos_get_mgnt_vrf`.

Run it first in the play so the transport decision is made once with the settings you want:

```yaml
- hosts: cisco
  gather_facts: false
  roles:
    - network_facts
    - cisco_snmp
    - cisco_aaa
    - cisco_syslog
```

## Facts it sets

| Fact | Meaning |
|---|---|
| `network_platform` | `ios`, `nxos` or `aireos` (from `ansible_network_os`, or `network_platform_override`) |
| `network_transport` | `netconf` or `cli` |
| `network_transport_reason` | why (port closed / session failed / model missing / forced / not eligible) |
| `netbox_site_response` | NetBox site of the prefix that contains `ansible_host` (same name the old role used) |
| `mgntINT`, `vrf_name` | interface holding `ansible_host` and its VRF; `NOT_SET` when not found |

## Transport selection (NETCONF first, CLI fallback)

`network_transport_preference`: `auto` (default) | `netconf` (fail the host if unusable) | `cli` (no probe).

For IOS-XE and NX-OS in `auto`:

1. `wait_for` tcp/`network_netconf_port` (830) from the controller, `network_netconf_probe_timeout` seconds.
2. A NETCONF session reads one leaf from the platform model (`Cisco-IOS-XE-native` or
   `cisco-nx-os-device`). An open port with bad credentials, no `ncclient`, or a device without that
   model (e.g. classic IOS) falls back to CLI with the reason recorded.
3. Each role then uses NETCONF for the items it covers and falls back to CLI for that item if the
   NETCONF edit fails (`network_netconf_fallback_on_error`, default true).

AireOS, classic IOS and telnet devices always use CLI. The roles never enable NETCONF on a device.

**What is configured over NETCONF today:** SNMP location and the SNMP management ACL
(`cisco_snmp`). Everything else (SNMPv3 users/groups/communities, TACACS+ keys and AAA, syslog) is
CLI. Secrets such as SNMPv3 and TACACS keys are stored hashed by the device and cannot be read back,
so a NETCONF path could not be idempotent for them. Syslog and AAA structure have YANG models; they are
not implemented because the exact paths must be validated against your IOS-XE/NX-OS versions first
(see "Validation" below).

## Variables

See `defaults/main.yml`. NetBox credentials default to the environment (`NETBOX_API` (URL), `NETBOX_TOKEN`);
no placeholder values are defined anywhere.

## Requirements

* Collections: `cisco.ios`, `cisco.nxos`, `ansible.netcommon` (`collections/requirements.yml`)
* Execution environment: `ncclient`, `xmltodict`
* Devices: `netconf-yang` (IOS-XE) / `feature netconf` (NX-OS) if you want NETCONF

## Validation

The NETCONF XML templates (`cisco_snmp/templates/netconf/`) are well-formed and render correctly, but
they were written from the YANG models without a device available. Before relying on them:

1. Run `ansible-playbook ... --check` against one IOS-XE and one NX-OS lab device with `network_transport_preference: netconf`.
2. Compare `show running-config` for the touched sections with a `cli` run.
3. If a path is wrong the device returns an `rpc-error` and the role falls back to CLI (visible in the
   debug message), so a wrong template costs a fallback, not a bad config. To skip NETCONF entirely
   set `network_transport_preference: cli`.
