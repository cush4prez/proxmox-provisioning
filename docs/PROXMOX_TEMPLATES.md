# Preparing Proxmox VE

Do this once per cluster. Run the commands as root on a Proxmox node.

## 1. API token

The playbooks authenticate as `root@pam` with an API token. `root@pam`
already exists and has full rights, so you only need to create the token:

```bash
# Token without privilege separation, so it inherits root@pam's permissions
pveum user token add root@pam provisioning --privsep 0
```

This prints the token secret **once**. Put it in
`group_vars/all/vault.yml` as `vault_proxmox_api_token_secret`, or export it
as `PROXMOX_TOKEN_SECRET`. Then match `group_vars/all/main.yml`:

```yaml
proxmox_api_user: root@pam
proxmox_api_token_id: provisioning
```

The token is as powerful as root on the whole cluster, so keep the vault
encrypted. To limit what the playbooks can do, create a dedicated user
instead and grant it only `PVEVMAdmin` on `/`, `PVEDatastoreUser` on
`/storage` and `PVESDNUser` on `/sdn`. Then set `proxmox_api_user` to that
user.

**TLS:** `proxmox_validate_certs: true` needs a certificate your control node
trusts, such as one from Proxmox's built-in ACME / Let's Encrypt support. In a
lab with the self-signed certificate, set `proxmox_validate_certs: false`.

## 2. Cloud-init templates (one per RHEL version)

The playbooks clone a template and configure it through cloud-init. Each
template needs:

- the RHEL KVM guest image (it already contains cloud-init);
- a cloud-init drive;
- its boot disk on `scsi0`, or set `vm_disk` to match;
- the QEMU guest agent enabled.

### Download the image

From <https://access.redhat.com/downloads>, under *Red Hat Enterprise Linux →
Product Software*, download the **KVM Guest Image** (`.qcow2`) for each
version, for example `rhel-9.6-x86_64-kvm.qcow2`. Copy it to the Proxmox node,
e.g. `/var/lib/vz/template/`.

### Optional: bake in extras

`virt-customize` comes from the `libguestfs-tools` package. Installing the
guest agent at build time makes the VM's IP show up in the Proxmox UI on
first boot, before registration:

```bash
virt-customize -a rhel-9.6-x86_64-kvm.qcow2 \
  --sm-credentials 'USER:password:PASS' --sm-register --sm-attach auto \
  --install qemu-guest-agent \
  --sm-unregister --selinux-relabel
```

### Create the template

Repeat this for each version, changing the VM ID, name and image:

```bash
ID=9000; NAME=rhel9-cloudinit-template; IMG=/var/lib/vz/template/rhel-9.6-x86_64-kvm.qcow2
STORAGE=local-lvm

qm create $ID --name $NAME --ostype l26 --cpu host --cores 2 --memory 2048 \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single \
  --agent enabled=1 --serial0 socket --vga serial0
qm set $ID --scsi0 $STORAGE:0,import-from=$IMG,discard=on
qm set $ID --ide2 $STORAGE:cloudinit --boot order=scsi0
qm template $ID
```

| Version | Suggested ID | Name (must match `proxmox_templates`) |
|---|---|---|
| RHEL 8 | 9000 | `rhel8-cloudinit-template` |
| RHEL 9 | 9001 | `rhel9-cloudinit-template` |
| RHEL 10 | 9002 | `rhel10-cloudinit-template` |

### Multi-node clusters

`proxmox_kvm` clones on the node that holds the template.

- **Per-node templates:** keep a copy of each template on every node. VMs are
  cloned on their own `vm_node`.
- **Shared templates:** put the templates on shared storage (Ceph, NFS) and
  set `proxmox_template_node`. Clones are then created there and placed on
  `vm_node` with `target`.

## 3. RHEL-specific template notes

**RHEL 8.** The guest image's platform-python is 3.6. That's fine for
ansible-core 2.16; see [RHEL8.md](RHEL8.md).

**RHEL 10 and FIPS.** RHEL 10 can only enter FIPS mode at installation, so
`fips-mode-setup` no longer exists. If DISA STIG nodes on RHEL 10 must run in
FIPS mode, build the RHEL 10 template from an image installed with the `fips=1`
kernel argument, e.g. with Image Builder (`[customizations] fips = true`) or a
kickstart install. With the stock image the STIG role warns and the scan
reports the FIPS rule as a finding. Set `stig_require_fips: true` to make it
fail instead.

**Separate partitions for DISA STIG.** The STIG expects separate filesystems
for `/var`, `/var/log`, `/var/log/audit`, `/var/tmp`, `/tmp` and `/home`. The
KVM guest image has a single root filesystem, so those rules show as findings
in the scan. If your accreditation requires them, build STIG templates with
that partition layout. Image Builder can do this with `[[customizations.filesystem]]`,
as can a kickstart.

## 4. Smoke test

```bash
ansible-playbook playbooks/provision_vm.yml --limit util01.myhome.lab
ssh cloud-user@192.168.10.40
```
