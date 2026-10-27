# RHEL 8 nodes

RHEL 8 is fully supported, but it needs a second Ansible environment for the
configuration phase.

## Why

RHEL 8's system Python (`/usr/libexec/platform-python`) is **3.6**, and its
`dnf` Python bindings exist only for that interpreter.

- ansible-core 2.17 and newer require Python 3.9+ on the managed node.
- Installing a newer Python on the node doesn't help. The `dnf` module still
  re-launches itself under platform-python 3.6, so package installs fail.
- ansible-core 2.16 is the last release that manages Python 3.6 nodes.

Provisioning is different: it only talks to the Proxmox API, and
`community.proxmox` needs ansible-core 2.17 or newer.

So a RHEL 8 node is built in two steps:

| Step | Playbook | ansible-core |
|---|---|---|
| Create the VM | `playbooks/provision_vm.yml` | your normal one (2.17+) |
| Configure it | `playbooks/configure.yml` | 2.16 |

If you run `configure.yml` against a RHEL 8 node with ansible-core 2.17 or
newer, the preflight check stops with a message pointing here.

## One-time setup: an ansible-core 2.16 environment

ansible-core 2.16 runs on Python 3.10–3.12. Pick one option.

**Option A: Python virtual environment**

```bash
# Fedora: sudo dnf install python3.12     RHEL 9: sudo dnf install python3.12
python3.12 -m venv ~/.venvs/ansible216
source ~/.venvs/ansible216/bin/activate
pip install 'ansible-core>=2.16,<2.17'

# Collections pinned to versions that still support 2.16, kept apart from your main ones
export ANSIBLE_COLLECTIONS_PATH=~/.venvs/ansible216/collections
ansible-galaxy collection install -r requirements-core216.yml
ansible-galaxy role install -r requirements-core216.yml
deactivate
```

**Option B: ansible-navigator with an execution environment**

Use any execution environment image that ships ansible-core 2.16, e.g. the
AAP 2.4 `ee-supported-rhel8` image from registry.redhat.io:

```bash
ansible-navigator run playbooks/configure.yml --mode stdout \
  --eei registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest \
  --limit rhel8
```

## Building a RHEL 8 node

```bash
# 1. Create the VM (normal environment)
ansible-playbook playbooks/provision_vm.yml --limit ntp01.myhome.lab

# 2. Configure it (2.16 environment)
source ~/.venvs/ansible216/bin/activate
export ANSIBLE_COLLECTIONS_PATH=~/.venvs/ansible216/collections
ansible-playbook playbooks/configure.yml --limit ntp01.myhome.lab
deactivate
```

To configure every RHEL 8 node at once, use `--limit rhel8`.

## Role compatibility

All the host-side roles (`common`, `stig_compliance`, `ntp`, `dns`,
`satellite`) are written to run on both ansible-core 2.16 and 2.17+. They use
only `ansible.builtin` and `ansible.posix` modules. The `ntp`, `dns` and
`common` roles were tested on EL8 under ansible-core 2.16, including a second
run to confirm nothing changes.

`aap` doesn't apply: AAP 2.5 and later don't support RHEL 8.

`satellite` on RHEL 8 is limited to Satellite 6.15 and 6.16. Set
`satellite_version` to match.
