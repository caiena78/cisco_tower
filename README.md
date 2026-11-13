# Cisco management baseline (Ansible / Tower)

Ansible roles and playbooks that configure the management plane of Cisco devices from the NetBox
inventory: **SNMPv3**, **TACACS+ AAA** (plus the matching Cisco ISE device entry) and **remote syslog**.
They consolidate nine single-purpose repos into three roles and two helpers.

Platforms: Cisco IOS / IOS-XE, NX-OS, and (AAA only) AireOS WLC; IOS over telnet for old images.

> **Status:** the roles were checked with `ansible-playbook --syntax-check`, `ansible-lint --profile min`
> and rendering the generated commands/XML against sample data. They have **not** yet been run against real
> devices, NetBox or ISE. Run `--check --diff` on one lab device per platform first (see [Validation](#validation)).

## Layout

```
site_baseline.yml          # everything: platform + facts, then snmp, aaa, syslog (tagged)
snmp_acl_update.yml        # only the SNMP management ACL, refreshed from NetBox
netbox_inventory.yml       # NetBox dynamic inventory
collections/requirements.yml
.env                       # Vault location and field names only - no secrets
roles/
  netbox_network_os/       # sets ansible_network_os from the NetBox platform
  network_facts/           # platform, NETCONF/CLI transport, NetBox site, management interface/VRF
  cisco_snmp/              # SNMPv3, management ACL, communities, ISE sync
  cisco_aaa/               # TACACS+ AAA on IOS/IOS-XE (ssh/telnet), NX-OS, AireOS + ISE sync
  cisco_syslog/            # remote syslog on IOS/IOS-XE and NX-OS
```

Each role has its own `README.md` with variables, behavior and an old-to-new variable migration table.

## Requirements

* **Collections:** `ansible-galaxy collection install -r collections/requirements.yml`
  (`cisco.ios`, `cisco.nxos`, `ansible.netcommon`, `ansible.utils`, `community.network`, `netbox.netbox`)
* **Execution environment (Tower):** Python `ncclient` and `xmltodict` (NETCONF)
* **NetBox:** devices with a **platform** set (see [NetBox data](#netbox-data))
* **Devices:** SSH reachable. NETCONF is optional: `netconf-yang` (IOS-XE) / `feature netconf` (NX-OS).
  The roles never enable NETCONF on a device.

## Credentials

Nothing secret is stored in this repo. NetBox values are fetched from Vault (see `.env` for the Vault
address, mount, path and field names):

```bash
vault login                      # or export VAULT_TOKEN
set -a; source .env; set +a      # exports NETBOX_URL / NETBOX_API for the run
```

In Tower, supply them through a credential or the job template environment. The roles read
`NETBOX_URL` and `NETBOX_API` from the environment and stop with a clear message if they are empty.

Also keep these in Vault / Tower credentials (never in `group_vars`):

| Needed by | Variables |
|---|---|
| `cisco_snmp` | SNMPv3 user passwords (`cisco_snmp.users[*].auth_pass`, `priv_pass`) |
| `cisco_aaa` | `cisco_aaa.key` (TACACS+ shared secret), local user/enable passwords |
| ISE sync (`cisco_snmp`, `cisco_aaa`) | `ise_hostname`, `ise_username`, `ise_password`, `ise_verify`, `ise_port` |

## Running

```bash
# everything, one host, no changes
ansible-playbook -i netbox_inventory.yml site_baseline.yml --limit <host> --check --diff

# one function
ansible-playbook -i netbox_inventory.yml site_baseline.yml --tags snmp   --limit <host>
ansible-playbook -i netbox_inventory.yml site_baseline.yml --tags aaa    --limit <host>
ansible-playbook -i netbox_inventory.yml site_baseline.yml --tags syslog --limit <host>

# only the SNMP management ACL
ansible-playbook -i netbox_inventory.yml snmp_acl_update.yml --limit <host>
```

### Tags

| Tag | Runs |
|---|---|
| `snmp` | `cisco_snmp` |
| `aaa` | `cisco_aaa` |
| `syslog` | `cisco_syslog` |
| `always` | `netbox_network_os` + `network_facts` (needed by every role; never skip it) |

### Suggested Tower job templates

One template per function is safer than one big template filtered by tags.

| Template | Playbook | Job tags | Notes |
|---|---|---|---|
| Cisco Baseline (all) | `site_baseline.yml` | | prompt on launch: Limit |
| Cisco SNMP | `site_baseline.yml` | `snmp` | |
| Cisco AAA | `site_baseline.yml` | `aaa` | AAA mistakes can lock devices out: require a Limit, restrict who can launch, keep a check-mode copy |
| Cisco Syslog | `site_baseline.yml` | `syslog` | |
| Cisco SNMP ACL refresh | `snmp_acl_update.yml` | | prompt on launch: Limit; updates devices in batches (`acl_update_batch`, default 25%) |

Make a "(check)" copy of each with run mode set to check for first use.

## How it fits together

1. **`netbox_network_os`** looks up the device by name in NetBox and sets `ansible_network_os` from its platform.
2. **`network_facts`** runs once per host and sets `network_platform`, `network_transport` (`netconf` or `cli`),
   `netbox_site_response`, and the management interface/VRF (`mgntINT`, `vrf_name`).
3. The function roles configure the device and use those facts.

### NETCONF first, SSH fallback

For IOS-XE and NX-OS, `network_facts` tests tcp/830 and opens a NETCONF session that reads one leaf from the
platform YANG model. If any step fails the host uses SSH and the reason is recorded. Override with
`network_transport_preference: auto | netconf | cli`.

NETCONF currently covers only **SNMP location** and the **SNMP management ACL**. Everything else (SNMPv3
users/communities, TACACS+ and AAA, syslog) is CLI: users' and TACACS+ secrets are stored hashed by the device
and cannot be read back, and the YANG paths for syslog/AAA structure still need validating against your
software versions. See `roles/network_facts/README.md`.

## NetBox data

* Devices need a **platform**. `netbox_network_os_map` (in `roles/netbox_network_os/defaults/main.yml`) maps
  platform slugs to `ansible_network_os`; add yours if a slug is missing.
* The site custom fields used: `SNMP_LOCATION` (SNMP), `syslog` (a JSON list of `{ip, port, protocol}`),
  `ISE_LOCATION` (AAA / ISE).
* IPs tagged `snmp_acl_allow` (configurable with `cisco_snmp.acl.tag`) become the SNMP management ACL.
* `netbox_inventory.yml` sets `ansible_network_os: platform[0]`, which looks unintended and is overridden by
  `netbox_network_os`; consider removing it.

## Safety behavior worth knowing

* The SNMP management ACL is **never** changed when NetBox returns no IPv4 addresses (or more than 1000); the host
  fails instead, because an empty IOS ACL denies all management access. On the CLI path ACL entries are pushed one at a
  time (large ACLs break `ios_config` when sent as one block).
* Cleanup of stale SNMP users/hosts (`cisco_snmp_cleanup`) and TACACS servers (`cisco_aaa_cleanup`) is **off** by default.
* Tasks that carry secrets use `no_log`; `cisco_aaa_debug: true` turns that off for troubleshooting.
* The telnet AAA path has no running-config diffing, so it always reports changed and saves with `write memory`.

## Validation

1. `--check --diff` on one IOS, one IOS-XE, one NX-OS (and one AireOS / telnet device if you have them).
2. Run each role twice; the second run should report no changes (except keys/passwords the device stores hashed, and the telnet path).
3. For NETCONF: run once with `network_transport_preference: netconf` and once with `cli`, and compare `show running-config`
   for the touched sections. A wrong YANG path makes the device return an error and the role falls back to CLI for that item.
4. ISE: confirm the device's TACACS key and SNMPv3 settings in ISE after a run.

## Origin

| Role | Replaces |
|---|---|
| `network_facts` | `netbox_getsite`, `get_mgnt_vrf`, `ansible_ios_get_mgnt_vrf`, `nxos_get_mgnt_vrf` |
| `cisco_snmp` | `ansible_nxos_snmp`, `Sapphire-Health/ansible-role-cisco-ios-snmp`, `mgnt_acl` |
| `cisco_aaa` | `ios_aaa`, `ios_aaa_telnet`, `nxos_aaa`, `airos_aaa` (Cisco AireOS) |
| `cisco_syslog` | `ios_syslog`, `ansible_nxos_syslog` |
| `netbox_network_os` | new |

The original GitHub repos are unchanged. Repointing `ansible-cisco-management/roles/requirements.yml` at this
repo and archiving the old repos are follow-ups once the roles are validated.
