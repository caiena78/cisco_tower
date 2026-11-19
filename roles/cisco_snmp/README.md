# cisco_snmp

Single role that configures SNMPv3 on **Cisco IOS/IOS-XE** and **Cisco NX-OS**.
It combines `Sapphire-Health/ansible-role-cisco-ios-snmp`, `caiena78/ansible_nxos_snmp` and
`caiena78/mgnt_acl` (the management ACL built from NetBox tags). Site lookup and
management VRF detection come from the shared `network_facts` role.

## How it works

`tasks/main.yml` runs (platform comes from `network_facts`; override with `cisco_snmp_platform: ios|nxos`):

1. `init.yml`     - `network_facts` (platform, NETCONF/CLI transport, NetBox site)
2. `netbox.yml`   - `SNMP_LOCATION` fact from the site custom field
3. `acl.yml`      - management ACL from NetBox IPs tagged `cisco_snmp.acl.tag` (NETCONF, CLI fallback)
4. `location.yml` - `snmp-server location` (NETCONF, CLI fallback)
5. `ios.yml` **or** `nxos.yml` - views/groups (IOS), users, hosts, community ACL retrofit (IOS), optional cleanup (CLI)
6. `ise.yml`      - sync SNMPv3 creds to Cisco ISE for users with `cisco_ise: true`

`acl.yml` also works alone (replaces the old `mgnt_acl` role):

```yaml
- ansible.builtin.include_role: {name: cisco_snmp, tasks_from: acl.yml}
  vars: {mgnt_acl: {name: MANAGEMENT, tag: snmp_acl_allow}}
```

## NETCONF

When `network_facts` finds NETCONF usable (tcp/830 open, session works, YANG model present) the
location and the management ACL are pushed over NETCONF (the ACL as a single atomic `replace`, so
there is no window with the ACL missing); on any NETCONF error the role falls back to CLI for that
item. SNMPv3 users, views, groups, hosts and v2c communities stay on CLI: the device stores user
secrets localized/hashed, so they cannot be read back and diffed over NETCONF. See `network_facts`.

## Variables (see `defaults/main.yml`)

| Variable | Purpose |
|---|---|
| `cisco_snmp_platform` | `ios` or `nxos`; auto-detected if empty |
| `cisco_snmp.acl` | `{name, tag}` passed to `mgnt_acl` |
| `cisco_snmp.views` / `groups` / `group_context` | IOS only (views + group context); `groups` used by both |
| `cisco_snmp.users` | list of users: `username, access (ro/rw), auth_proto, auth_pass, priv_proto, priv_bits, priv_pass, [host], [cisco_ise]` |
| `cisco_snmp_cleanup` | remove SNMPv3 users/hosts not in `cisco_snmp.users` |
| `cisco_snmp_protected_users` | never removed by cleanup (default `[admin]`) |
| `cisco_snmp_netbox_enabled` / `cisco_snmp_ise_enabled` | toggle those sections |
| `cisco_snmp_acl_debug` | print the NetBox addresses used for the ACL |
| `ise_hostname, ise_port, ise_username, ise_password, ise_verify` | Cisco ISE ERS API (provide via vault/group_vars) |
| `netbox_url, netbox_token` | default to env `NETBOX_API` / `NETBOX_TOKEN` (no placeholders) |

Keep passwords in Ansible Vault. Tasks that carry secrets use `no_log`.

## Example

```yaml
- hosts: cisco
  gather_facts: false
  roles:
    - cisco_snmp
```

## Platform differences

* **IOS**: views, groups (+vlan context), `snmp-server user ... v3 auth/priv ... access ACL`, v2c community ACL retrofit.
* **NX-OS**: `globalEnforcePriv`, `snmp-server user ... auth/priv`, `use-ipv4acl`, admin ACL binding; cleanup parses `show snmp user`.

## Notes vs. the original roles

* Variables unified: `ios_snmp`/`nxos_snmp` → `cisco_snmp`; `*_cleanup`/`*_protected_users` likewise.
* Removed unused `CAT_SNMP_*` vars, a dummy no-op `set_fact`, duplicate NX-OS ACL task, and password-printing debug task.
* NX-OS cleanup no longer trims a fixed 6 lines from the `show snmp user` output; it parses the "SNMP USERS" section.
* Regexes avoid backslashes so Jinja escape handling cannot break them.

## Changes from the merged repos

* `mgnt_acl` folded in (`acl.yml`). Fixes: refuses an empty tag result and a truncated (>1000) NetBox
  result instead of deleting/emptying the ACL; IPv4 only (IPv6 skipped); NX-OS detection no longer
  silently skips `cisco.nxos.nxos`; NX-OS existence check is an exact match (was a substring); ACEs are
  pushed one per call on CLI (large ACLs break `ios_config` when sent as one block); NetBox token has `no_log`.
* v2c communities: **every** RO/RW community is moved onto the management ACL, replacing an older ACL
  (original `ios_snmp` behavior; `cisco_snmp` 2.0 only touched communities with no ACL). Communities
  already on the right ACL are skipped so reruns stay unchanged.
* Placeholder `netbox_url` / `netbox_token` defaults removed - they shadowed the env values `netbox_getsite` used.
* `netbox_getsite` replaced by `network_facts`, which picks the most specific prefix by length (was a string sort).

## Migrating from the old repos

| Old | New |
|---|---|
| `ios_snmp` / `nxos_snmp` (dict) | `cisco_snmp` |
| `ios_snmp_cleanup` / `nxos_snmp_cleanup` | `cisco_snmp_cleanup` |
| `ios_snmp_protected_users` / `nxos_snmp_protected_users` | `cisco_snmp_protected_users` (default `[admin]`, so admin is now protected on NX-OS too) |
| role `mgnt_acl`, var `mgnt_acl` | `cisco_snmp` `tasks_from: acl.yml`, var `mgnt_acl` (same shape) |
| role `netbox_getsite` | `network_facts` (still registers `netbox_site_response`) |
