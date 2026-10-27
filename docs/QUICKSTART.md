# Quick start: your first node

This walks through building one RHEL 9 node with DNS and NTP and the DISA
STIG applied. It takes about 20 minutes once the Proxmox templates exist.

## 1. Prepare the control node

```bash
# ansible-core 2.17+ (Fedora/RHEL: dnf install ansible-core)
ansible --version

# Proxmox API client used by the community.proxmox modules
sudo dnf install python3-proxmoxer python3-requests     # or: pip install proxmoxer requests

# Collections and the RedHatOfficial STIG roles
cd proxmox-provisioning
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml
```

You also need an SSH key pair. By default the playbooks inject
`~/.ssh/id_ed25519.pub` into every VM; change `vm_ssh_public_keys` to use a
different key.

## 2. Prepare Proxmox (once)

Follow [PROXMOX_TEMPLATES.md](PROXMOX_TEMPLATES.md) to:

1. create the `provisioning` API token for `root@pam`;
2. build a cloud-init template for each RHEL version you plan to use, named
   `rhel9-cloudinit-template` (and the 8/10 equivalents).

## 3. Store the secrets

```bash
cd inventory/production/group_vars/all
cp vault.yml.example vault.yml
$EDITOR vault.yml          # Proxmox token secret, cloud-init password, ...
ansible-vault encrypt vault.yml
cd -
```

`vault_vm_ci_password` matters. It becomes the cloud-init user's password and
the sudo password Ansible uses once the STIG removes passwordless sudo.

To avoid typing the vault password on every run, put it in `.vault_pass`
(git-ignored) and uncomment `vault_password_file` in `ansible.cfg`.

## 4. Set the site defaults

Edit `inventory/production/group_vars/all/main.yml`. At a minimum, set:

| Setting | Example |
|---|---|
| `proxmox_api_host` | `pve01.myhome.lab` |
| `proxmox_api_user` / `proxmox_api_token_id` | `root@pam` / `provisioning` |
| `proxmox_default_node` | `pve01` |
| `proxmox_templates` | names of your templates |
| `vm_storage`, `vm_bridge` | `local-lvm`, `vmbr0` |
| `vm_gateway`, `vm_prefix`, `vm_dns_servers`, `vm_search_domains` | your network |
| `registration_method` + `rhsm_org_id` / `rhsm_activation_key` | how nodes get RHEL content |

Nodes need RHEL repositories to install packages. Either register them
(`registration_method: rhsm` or `satellite`) or build templates that already
have repo access.

## 5. Define the node

Add the host to `inventory/production/hosts.yml`:

```yaml
all:
  children:
    rhel_nodes:
      hosts:
        ns01.myhome.lab:
```

Create `inventory/production/host_vars/ns01.myhome.lab.yml`:

```yaml
rhel_version: 9
stig_enabled: true
server_roles: [dns, ntp]

vm_id: 9101
vm_cores: 2
vm_memory: 4096
vm_disk_size: 40G
vm_ip: 192.168.10.11

ntp_allowed_networks: [192.168.10.0/24]
dns_listen_networks: [192.168.10.0/24]
dns_forwarders: [1.1.1.1]
dns_zones:
  - name: myhome.lab
    records:
      - {name: ns01, type: A, value: 192.168.10.11}
```

Every parameter is listed in [INVENTORY_GUIDE.md](INVENTORY_GUIDE.md).

## 6. Check before you build

```bash
ansible-inventory --graph                       # ns01 should be in rhel9, role_dns, role_ntp, stig
ansible-inventory --host ns01.myhome.lab   # every variable the node will get
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --limit ns01.myhome.lab --list-tasks
```

## 7. Build it

```bash
ansible-playbook site.yml --limit ns01.myhome.lab --ask-vault-pass
```

What you will see:

1. **Provision:** the template is cloned to VM 9101, then sized,
   given its IP and started. The playbook waits for SSH and cloud-init.
2. **Base:** hostname, packages, chrony, firewalld and sshd settings.
3. **STIG:** FIPS mode is enabled (with a reboot), the RHEL 9 STIG role runs,
   the node reboots once more, and an OpenSCAP report is saved to
   `reports/ns01.myhome.lab-stig-report.html`.
4. **NTP, then DNS:** chrony starts serving `192.168.10.0/24`, BIND loads the
   zones, and both firewall ports are opened.

## 8. Verify

```bash
dig @192.168.10.11 ns01.myhome.lab +short
ansible ns01.myhome.lab -m command -a 'chronyc sources'
ansible ns01.myhome.lab -m command -a 'firewall-cmd --list-services'
xdg-open reports/ns01.myhome.lab-stig-report.html
```

## Next steps

- **Add more nodes:** one `host_vars` file each, then
  `ansible-playbook site.yml --limit <fqdn>`.
- **Change a node:** edit its `host_vars` and re-run. The playbooks are
  idempotent. A larger `vm_disk_size` grows the disk; CPU and memory changes
  apply at the next VM restart.
- **Satellite or AAP nodes:** read [SERVER_ROLES.md](SERVER_ROLES.md) first.
  They have size and subscription prerequisites.
- **RHEL 8 nodes:** read [RHEL8.md](RHEL8.md).
