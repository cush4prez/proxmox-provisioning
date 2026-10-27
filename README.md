# proxmox-provisioning

Ansible playbooks and roles that build RHEL 8, 9 and 10 virtual machines on
Proxmox VE and turn them into finished servers. Each node is defined in one
file, which sets:

- **The RHEL version:** 8, 9 or 10.
- **The VM's provisioning parameters:** VM ID, CPU, memory, disk, extra disks,
  IP address, VLAN, Proxmox node.
- **Whether the node is DISA STIG compliant**, using Red Hat's official
  `RedHatOfficial.rhelN_stig` roles plus an OpenSCAP report.
- **Which server roles it gets:** `dns` (BIND), `ntp` (chrony), `satellite`
  (Red Hat Satellite) and/or `aap` (Ansible Automation Platform). A node can
  have none, one or several.

```yaml
# inventory/production/host_vars/ns01.myhome.lab.yml
rhel_version: 9
stig_enabled: true
server_roles: [dns, ntp]

vm_id: 9101
vm_cores: 2
vm_memory: 4096
vm_disk_size: 40G
vm_ip: 192.168.10.11
```

```bash
ansible-playbook site.yml --limit ns01.myhome.lab
```

## What happens when you run `site.yml`

| Phase | Playbook | Runs against | What it does |
|---|---|---|---|
| 1 | `playbooks/provision_vm.yml` | Proxmox API | Clones the RHEL template, sets CPU, memory, disks, NIC and cloud-init (user, SSH keys, IP), starts the VM and waits for SSH |
| 2 | `playbooks/configure_base.yml` | every node | Hostname, registration (RHSM or Satellite), packages, chrony client, firewalld, sshd hardening, admin users, extra filesystems |
| 3 | `playbooks/apply_stig.yml` | nodes with `stig_enabled: true` | FIPS mode, DISA STIG role, one reboot, OpenSCAP scan with the HTML report saved to `reports/` |
| 4 | `playbooks/configure_{ntp,dns,satellite,aap}.yml` | nodes with that server role | Installs and configures the service, opens its firewall ports and checks that it answers |

The STIG goes on before the server roles. That way services are installed on a
hardened baseline, and their configuration isn't reverted by the next STIG run.

## Repository layout

```
site.yml                      # full build = provision_vm.yml + configure.yml
ansible.cfg
requirements.yml              # collections + STIG roles (ansible-core >= 2.17)
requirements-core216.yml      # for configuring RHEL 8 (see docs/RHEL8.md)
playbooks/
  provision_vm.yml            # phase 1 - Proxmox
  configure.yml               # phases 2-4
  configure_base.yml  apply_stig.yml
  configure_ntp.yml   configure_dns.yml  configure_satellite.yml  configure_aap.yml
  destroy_vm.yml              # delete VMs (needs -e confirm_destroy=yes)
  preflight.yml               # helper imported by the playbooks above
inventory/
  production/  staging/
    hosts.yml                 # list of nodes
    zz_groups.yml             # auto-groups: rhelN, role_X, stig/nostig
    group_vars/all/main.yml   # site-wide defaults (Proxmox, network, NTP...)
    group_vars/all/vault.yml  # secrets (you create it from vault.yml.example)
    group_vars/stig.yml       # settings for every STIG node
    host_vars/<fqdn>.yml      # ONE FILE PER NODE - what the node is
roles/
  proxmox_vm/  common/  stig_compliance/  dns/  ntp/  satellite/  aap/
docs/
```

## Requirements

**Control node (where you run Ansible)**

- ansible-core 2.17 or newer
- The Python Proxmox client: `sudo dnf install python3-proxmoxer python3-requests`
  (or `pip install proxmoxer requests`)
- Collections and STIG roles:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ansible-galaxy role install -r requirements.yml
  ```
- **RHEL 8 nodes are the exception.** ansible-core 2.17 and newer can't
  manage RHEL 8's Python 3.6, so after the VM is created you configure RHEL 8
  nodes from an ansible-core 2.16 environment. See
  [docs/RHEL8.md](docs/RHEL8.md). RHEL 9 and 10 need nothing extra.

**Proxmox VE**

- An API token for a user with VM, datastore and SDN permissions
- One cloud-init template per RHEL version you use

[docs/PROXMOX_TEMPLATES.md](docs/PROXMOX_TEMPLATES.md) covers both.

**Red Hat**

- Subscriptions or activation keys for the nodes
- For `satellite`: a Satellite subscription (a manifest is optional)
- For `aap`: the AAP containerized setup bundle, downloaded from
  access.redhat.com

## Quick start

```bash
# 1. Dependencies
ansible-galaxy collection install -r requirements.yml
ansible-galaxy role install -r requirements.yml

# 2. Secrets
cp inventory/production/group_vars/all/vault.yml.example inventory/production/group_vars/all/vault.yml
$EDITOR inventory/production/group_vars/all/vault.yml
ansible-vault encrypt inventory/production/group_vars/all/vault.yml

# 3. Site settings: Proxmox host/token, templates, network, NTP
$EDITOR inventory/production/group_vars/all/main.yml

# 4. Nodes: list them in hosts.yml, and give each one a host_vars/<fqdn>.yml
$EDITOR inventory/production/hosts.yml

# 5. Check, then build
ansible-inventory --graph
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --limit ns01.myhome.lab --ask-vault-pass
```

[docs/QUICKSTART.md](docs/QUICKSTART.md) walks through a first deployment
step by step.

## Everyday commands

```bash
# Build or update one node end to end
ansible-playbook site.yml --limit util01.myhome.lab

# Only create the VMs
ansible-playbook playbooks/provision_vm.yml

# Re-run configuration only (VMs already exist)
ansible-playbook playbooks/configure.yml --limit ns01.myhome.lab

# Only one phase, using tags: provision, base, stig, ntp, dns, satellite, aap
ansible-playbook site.yml --tags dns
ansible-playbook playbooks/apply_stig.yml          # every STIG node

# Target nodes by what they are (these groups are built automatically)
ansible-playbook playbooks/configure.yml --limit role_dns
ansible-playbook playbooks/configure.yml --limit 'rhel9:&stig'

# Use the staging inventory
ansible-playbook -i inventory/staging site.yml

# Delete a VM
ansible-playbook playbooks/destroy_vm.yml --limit util01.myhome.lab -e confirm_destroy=yes
```

Automatic groups: `rhel8`, `rhel9`, `rhel10`, `role_dns`, `role_ntp`,
`role_satellite`, `role_aap`, `stig` and `nostig`.

## Documentation

| Document | Covers |
|---|---|
| [docs/QUICKSTART.md](docs/QUICKSTART.md) | First deployment, step by step |
| [docs/PROXMOX_TEMPLATES.md](docs/PROXMOX_TEMPLATES.md) | Proxmox API token and RHEL 8/9/10 cloud-init templates |
| [docs/INVENTORY_GUIDE.md](docs/INVENTORY_GUIDE.md) | Every node parameter, groups, variable precedence, adding nodes |
| [docs/SERVER_ROLES.md](docs/SERVER_ROLES.md) | DNS, NTP, Satellite and AAP options and supported RHEL versions |
| [docs/STIG_COMPLIANCE.md](docs/STIG_COMPLIANCE.md) | Turning DISA STIG on or off, FIPS, exceptions, scan reports |
| [docs/RHEL8.md](docs/RHEL8.md) | Configuring RHEL 8 nodes with ansible-core 2.16 |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Common errors and fixes |
