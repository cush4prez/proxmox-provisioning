# Troubleshooting

Add `-v` (or `-vvv`) to any run for more detail. Use `--limit <fqdn>` to work
on one node at a time.

## Inventory and variables

**A node isn't in the group I expect**

```bash
ansible-inventory --graph
ansible-inventory --host <fqdn> | grep -E 'rhel_version|server_roles|stig_enabled'
```

Groups come from host_vars through `zz_groups.yml`. Check that the host_vars
file name exactly matches the name in `hosts.yml`.

**`'vault_vm_ci_password' is undefined`**

`group_vars/all/vault.yml` is missing, or you didn't pass the vault password.
Create it from `vault.yml.example` and run with `--ask-vault-pass` (or set
`vault_password_file`).

**`Could not match supplied host pattern, ignoring: role_aap`**

Harmless. No node has that role.

## Provisioning (Proxmox)

**`proxmoxer` / `requests` import error**

Install them on the control node, for the same Python that runs Ansible:
`sudo dnf install python3-proxmoxer python3-requests`, or `pip install
proxmoxer requests` inside your venv.

**`401 Unauthorized` / `authentication failure`**

- `proxmox_api_user` must be the full user (`root@pam`), and
  `proxmox_api_token_id` only the token name (`provisioning`).
- Check the secret: `echo $PROXMOX_TOKEN_SECRET`, or
  `ansible-vault view group_vars/all/vault.yml`.

**`403 Permission check failed`**

The token lacks ACLs. A token created with `--privsep 1` has **no**
permissions of its own. Re-create it with `--privsep 0`, or grant ACLs to the
token itself; see [PROXMOX_TEMPLATES.md](PROXMOX_TEMPLATES.md).

**SSL: `CERTIFICATE_VERIFY_FAILED`**

Proxmox is using its self-signed certificate. Install a trusted certificate
(ACME), or set `proxmox_validate_certs: false` in a lab.

**`VM <template> does not exist` / clone fails**

- The name in `proxmox_templates` must exactly match the template's name.
- The template must be on `vm_node`. If not, set `proxmox_template_node`, and
  keep the template on shared storage.

**`VM with vmid N already exists`, but it's a different VM**

`vm_id` collides with an existing VM. Every node needs a unique `vm_id`.

**Disk resize error: `shrinking disks is not supported`**

`vm_disk_size` is smaller than the disk (often the template's disk). Use a
size at least as large as the current one.

**Stuck at "Wait for SSH"**

- Open the VM console in Proxmox. Did cloud-init run? Is the IP right?
- Is the cloud-init drive (`ide2`) present on the template?
- Can the control node reach `vm_ip` on port 22 (routing, VLAN `vm_vlan`)?
- DHCP nodes: the FQDN must resolve to the leased address.

**Hardware changes (CPU, RAM, agent) don't show up**

Proxmox applies these at the next full stop/start of the VM, not a reboot from
inside the guest. Run `qm stop <id> && qm start <id>`, or use the UI.

## Base configuration

**`Failed to connect` / `Permission denied (publickey)`**

- The key in `vm_ssh_public_keys` must match a private key your ssh-agent or
  `~/.ssh` has.
- `ansible_user` defaults to `vm_ci_user` (`cloud-user`).

**`Missing sudo password` after the STIG ran**

The STIG removed passwordless sudo. Make sure `vault_vm_ci_password` is set;
it's used as `ansible_become_password`. If the cloud-init password was changed
on the node, update the vault.

**`is RedHat 9.x but rhel_version is 8`**

The template doesn't match `rhel_version`. Fix `proxmox_templates` (or
`vm_template`), then rebuild the VM.

**Package installs fail: `No repositories available` / `This system is not registered`**

Set `registration_method` and the activation key or credentials, or use
templates that already have repository access.

**RHEL 8: `Ansible requires Python 3.9 or newer on the target` / `Could not import the dnf python module`**

You're configuring RHEL 8 with ansible-core 2.17 or newer. See
[RHEL8.md](RHEL8.md).

## DISA STIG

**`The STIG removes NOPASSWD from sudoers...`**

This is the wrapper's sudo lockout guard. Set `vault_vm_ci_password`, or set
`sudo_remove_nopasswd: false` and document the deviation.

**`the role 'RedHatOfficial.rhel9_stig' was not found`**

Run `ansible-galaxy role install -r requirements.yml`.

**The STIG run takes a long time**

That's normal. The STIG roles contain hundreds of rules, so a first run takes
15–40 minutes.

**A service broke after the STIG**

Check these, in order:

1. `journalctl -u <service>` and `ausearch -m avc -ts recent` (SELinux)
2. fapolicyd: `journalctl -u fapolicyd`, `fapolicyd-cli --list`
3. `umask` (077) affecting files the service creates
4. Crypto policy `FIPS:STIG` rejecting older algorithms

Fix it with a trust rule or configuration change, or disable the specific
STIG rule (see [STIG_COMPLIANCE.md](STIG_COMPLIANCE.md)).

**The scan report shows failures right after hardening**

Some findings are expected, such as partitioning or FIPS on RHEL 10; see
[STIG_COMPLIANCE.md](STIG_COMPLIANCE.md#scan-reports).

## Server roles

**DNS: `failed to validate` on a zone file**

`named-checkzone` rejected the zone, and the message says why. The most common
cause is `NS 'x' has no address records`: an in-zone name server needs an A
record in `records`. The running zone is untouched, and the next run retries.

**DNS: queries from clients are refused**

Add the client network to `dns_listen_networks`.

**NTP: clients don't sync**

The client network must be in `ntp_allowed_networks`. On the client, run
`chronyc sources -v`.

**Satellite: installer failed**

Check `/var/log/foreman-installer/satellite.log` on the node. The usual causes
are:

- the FQDN doesn't resolve forward and reverse;
- less than 20 GB RAM;
- the required repos aren't enabled (`subscription-manager repos --list-enabled`).

Fix the cause, then re-run. The installer runs again because the
`.ansible-installed` marker is only written on success.

**AAP: installer failed**

Check `~/aap-setup/install.log` on the node (in the `aap_install_user`'s home).
Fix the cause and re-run; the `.installed` marker is only written on success.
`podman ps -a` as that user shows the platform containers.

**AAP: `Wait for the platform gateway` times out**

As the install user on the node, run
`podman ps --format '{{.Names}} {{.Status}}'` and
`journalctl --user -u automation-gateway`, and check that port 443 is open.
