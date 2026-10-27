# Inventory guide

Everything about a node is data in the inventory. The playbooks themselves
never need editing.

```
inventory/production/
  hosts.yml                 # which nodes exist (group rhel_nodes)
  zz_groups.yml             # builds rhelN / role_X / stig / nostig groups from host_vars
  group_vars/all/main.yml   # site-wide defaults
  group_vars/all/vault.yml  # secrets (ansible-vault encrypted)
  group_vars/stig.yml       # applies to every node with stig_enabled: true
  host_vars/<fqdn>.yml      # the node definition
```

`inventory/staging/` has the same layout. Select it with `-i inventory/staging`.

## Adding a node

1. Add the FQDN under `rhel_nodes.hosts` in `hosts.yml`.
2. Create `host_vars/<fqdn>.yml` with at least `rhel_version`, `vm_id` and
   `vm_ip`, plus `stig_enabled` and `server_roles` if you want them.
3. Run `ansible-inventory --host <fqdn>` to review the merged variables.
4. Build it with `ansible-playbook site.yml --limit <fqdn>`.

The inventory hostname must be the node's FQDN. It becomes the VM name, the
OS hostname, the default DNS NS name, and the name that Satellite and AAP
require to resolve.

## Node parameters

### What the node is

| Variable | Default | Description |
|---|---|---|
| `rhel_version` | `9` | `8`, `9` or `10`. Picks the template, and the play fails if the guest reports a different version |
| `stig_enabled` | `false` | Apply the DISA STIG ([STIG_COMPLIANCE.md](STIG_COMPLIANCE.md)) |
| `server_roles` | `[]` | Any of `dns`, `ntp`, `satellite`, `aap` ([SERVER_ROLES.md](SERVER_ROLES.md)) |

### Proxmox VM

| Variable | Default | Description |
|---|---|---|
| `vm_id` | **required** | Proxmox VMID, unique in the cluster |
| `vm_name` | inventory hostname | Name shown in Proxmox |
| `vm_node` | `proxmox_default_node` | Cluster node the VM runs on |
| `vm_template` | `proxmox_templates[rhel_version]` | Override the template for this node only |
| `vm_full_clone` | `true` | `false` makes a linked clone (faster, but tied to the template) |
| `vm_cores` / `vm_sockets` | `2` / `1` | vCPU topology |
| `vm_cpu_type` | `host` | QEMU CPU model. Use e.g. `x86-64-v3` if VMs must live-migrate between different CPUs |
| `vm_memory` | `4096` | MiB |
| `vm_disk` | `scsi0` | Boot disk in the template |
| `vm_disk_size` | `40G` | Boot disk size. It **only grows**; Proxmox refuses to shrink |
| `vm_storage` | `local-lvm` | Storage for the clone and extra disks |
| `vm_extra_disks` | `[]` | `[{disk: scsi1, size: 200, storage: local-lvm}]`, with size in GiB |
| `vm_bridge` / `vm_vlan` | `vmbr0` / `""` | NIC bridge and optional VLAN tag |
| `vm_onboot` | `true` | Start with the Proxmox host |
| `vm_tags` | `ansible`, `rhelN`, `stig`, roles | Proxmox tags |
| `vm_state` | `present` | `absent` deletes the VM. Use `playbooks/destroy_vm.yml` instead |

### Network and cloud-init

| Variable | Default | Description |
|---|---|---|
| `vm_ip` | *undefined* | Static IPv4 address. If omitted, the VM uses DHCP and the FQDN must resolve |
| `vm_prefix` | `24` | Prefix length |
| `vm_gateway` | site value | Default gateway |
| `vm_dns_servers` | site value | Resolvers written by cloud-init |
| `vm_search_domains` | site value | DNS search list |
| `vm_ci_user` | `cloud-user` | Cloud-init user; Ansible connects as this user |
| `vm_ci_password` | `vault_vm_ci_password` | That user's password, also used as the sudo password |
| `vm_ssh_public_keys` | `~/.ssh/id_ed25519.pub` | Keys authorized for `vm_ci_user` |

Cloud-init settings only apply at first boot. After that, change the network
on the guest itself.

### Base OS (`roles/common`)

| Variable | Default | Description |
|---|---|---|
| `registration_method` | `none` | `none`, `rhsm` (Red Hat CDN) or `satellite` |
| `rhsm_org_id` / `rhsm_activation_key` | `""` | Activation key registration (works with RHSM and Satellite) |
| `vault_rhsm_username` / `vault_rhsm_password` | `""` | Username/password registration (RHSM only) |
| `satellite_registration_server` | | Satellite FQDN when `registration_method: satellite` |
| `timezone` | `UTC` | |
| `ntp_upstream_servers` | RHEL pool | Time sources for ntp-role nodes, and for everyone if there are none |
| `chrony_servers` | ntp-role nodes | Time sources for all other nodes (computed) |
| `admin_users` | `[]` | `[{name: jdoe, ssh_keys: ["ssh-ed25519 ..."]}]`, added to `wheel` |
| `common_extra_packages` | `[]` | Additional packages |
| `common_update_packages` | `false` | `dnf update` everything, then reboot if needed |
| `common_sshd_settings` | root login, password auth and X11 off | sshd options |
| `common_firewall_services` | `[ssh]` | firewalld services opened on every node |
| `extra_filesystems` | `[]` | LVM + filesystem + mount on an extra disk, see below |

```yaml
vm_extra_disks:
  - {disk: scsi1, size: 500}                 # appears in the guest as /dev/sdb
extra_filesystems:
  - {device: /dev/sdb, vg: vg_pulp, lv: lv_pulp, size: 100%FREE, mount: /var/lib/pulp, fstype: xfs}
```

`size` accepts LVM extents (`100%FREE`, `50%VG`) or absolute sizes (`200G`).

### Proxmox connection (site-wide)

| Variable | Description |
|---|---|
| `proxmox_api_host` / `proxmox_api_port` | API endpoint (port 8006) |
| `proxmox_api_user` / `proxmox_api_token_id` | e.g. `root@pam` / `provisioning` |
| `proxmox_api_token_secret` | From `vault_proxmox_api_token_secret` or the `PROXMOX_TOKEN_SECRET` environment variable |
| `proxmox_validate_certs` | Verify the Proxmox TLS certificate |
| `proxmox_default_node` | Default `vm_node` |
| `proxmox_template_node` | Node holding shared templates (optional) |
| `proxmox_templates` | `{8: name, 9: name, 10: name}` |

Role-specific settings (`dns_*`, `ntp_*`, `satellite_*`, `aap_*`) are in
[SERVER_ROLES.md](SERVER_ROLES.md). STIG settings (`stig_*`) are in
[STIG_COMPLIANCE.md](STIG_COMPLIANCE.md).

## Automatic groups

`zz_groups.yml` uses Ansible's `constructed` inventory plugin to put each node
into groups based on its own variables:

| Group | Members |
|---|---|
| `rhel8`, `rhel9`, `rhel10` | by `rhel_version` |
| `role_dns`, `role_ntp`, `role_satellite`, `role_aap` | by `server_roles` |
| `stig`, `nostig` | by `stig_enabled` |

You never edit these groups. They update when you change a node's host_vars.
Use them to:

```bash
ansible-playbook playbooks/configure.yml --limit role_ntp      # all NTP servers
ansible-playbook playbooks/configure.yml --limit 'rhel9:&stig'  # STIG'd RHEL 9 only
ansible stig -m command -a 'cat /proc/sys/crypto/fips_enabled'
```

They also take `group_vars/<group>.yml` files. `group_vars/stig.yml` comes
with the project; you can add others, e.g. `group_vars/rhel8.yml` or
`group_vars/role_dns.yml`.

## Variable precedence

From lowest to highest; later entries win:

1. Role defaults: `roles/*/defaults/main.yml`
2. `group_vars/all/*.yml`
3. Other groups: `group_vars/rhel9.yml`, `group_vars/stig.yml`, ...
4. `host_vars/<fqdn>.yml`
5. `-e` extra vars on the command line

Some role-internal values are higher than inventory: the per-version
`roles/*/vars/RedHat-N.yml` files (BIND keywords, package names), and the
automatic STIG exceptions, which are set with `set_fact`. Those are
deliberate; see the role docs.

## Secrets

`group_vars/all/vault.yml` holds every secret, with names starting `vault_`.
Plaintext `vault.yml` files are git-ignored. If you want them in git, encrypt
them with `ansible-vault encrypt` and add them with `git add -f`.

```bash
ansible-vault edit inventory/production/group_vars/all/vault.yml
ansible-playbook site.yml --ask-vault-pass        # or vault_password_file in ansible.cfg
```
