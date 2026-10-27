# Server roles

List roles in a node's `server_roles`. A node can have any combination, but
`satellite` and `aap` each expect a dedicated node.

| Role | Service | RHEL 8 | RHEL 9 | RHEL 10 | Playbook |
|---|---|:-:|:-:|:-:|---|
| `ntp` | chrony server | ✔ | ✔ | ✔ | `playbooks/configure_ntp.yml` |
| `dns` | BIND (named) | ✔ | ✔ | ✔ | `playbooks/configure_dns.yml` |
| `satellite` | Red Hat Satellite | 6.15–6.16 | ✔ | ✘ | `playbooks/configure_satellite.yml` |
| `aap` | Ansible Automation Platform (containerized) | ✘ | ✔ | AAP 2.6 | `playbooks/configure_aap.yml` |

The Satellite and AAP rows come from editable tables
(`satellite_supported_rhel`, `aap_supported_rhel`). The role refuses
combinations that aren't in its table. Check Red Hat's lifecycle pages when a
new release comes out and update the table.

---

## `ntp`: chrony time server

The node syncs from `ntp_upstream_servers` and serves time to
`ntp_allowed_networks`. Every other node in the inventory automatically uses
the ntp-role nodes as its time sources (`chrony_servers`).

| Variable | Default | Description |
|---|---|---|
| `ntp_upstream_servers` | RHEL pool | Upstream sources (site-wide, in `group_vars/all`) |
| `ntp_allowed_networks` | `[]` | CIDRs allowed to query, e.g. `[192.168.10.0/24]` |
| `ntp_serve_when_unsynced` | `true` | Keep serving (`local stratum`) when upstreams are unreachable, for air-gapped sites |
| `ntp_local_stratum` | `10` | Stratum advertised in that case |

The role opens the `ntp` firewalld service. On STIG nodes the STIG's
"chrony client only" rule is turned off automatically so the server can
listen on UDP/123. Log that as a deviation; see
[STIG_COMPLIANCE.md](STIG_COMPLIANCE.md).

To verify, run `chronyc sources` on the server (it should show upstreams) and
on a client (it should show the server). chrony doesn't allow remote
`chronyc` monitoring, so run it locally on each node.

---

## `dns`: BIND authoritative + recursive DNS

| Variable | Default | Description |
|---|---|---|
| `dns_zones` | `[]` | Zones and their records (below) |
| `dns_listen_networks` | `[]` | CIDRs allowed to query and recurse, in addition to localhost and local nets |
| `dns_recursion` | `true` | Act as a recursive resolver for those networks |
| `dns_forwarders` | `[]` | Upstream resolvers for recursion |
| `dns_forward_only` | `false` | `true` means never fall back to the root servers |
| `dns_dnssec_validation` | `auto` | `auto`, `yes` or `no` |
| `dns_listen_ipv6` | `false` | Listen on IPv6 |
| `dns_zone_ttl` | `3600` | Default `$TTL` |
| `dns_zone_admin` | `hostmaster.<zone>` | SOA contact (no trailing dot) |

```yaml
dns_zones:
  - name: myhome.lab                # primary zone (default type)
    nameservers: [ns01.myhome.lab.] # default: this host
    allow_transfer: [192.168.10.12]      # optional
    records:
      - {name: ns01, type: A, value: 192.168.10.11}
      - {name: www, type: CNAME, value: ns01}
      - {name: "@", type: MX, value: "10 mail.myhome.lab."}
      - {name: _ldap._tcp, type: SRV, value: "0 100 389 idm.myhome.lab.", ttl: 300}

  - name: 10.168.192.in-addr.arpa        # reverse zone
    records:
      - {name: "11", type: PTR, value: ns01.myhome.lab.}

  - name: corp.example.com               # secondary zone
    type: secondary
    primaries: [10.0.0.53]
```

How the role behaves:

- **Validation.** `named.conf` is checked with `named-checkconf` and every
  zone with `named-checkzone` before it's installed. A typo fails the play and
  leaves the running config untouched. A common example is an NS name inside
  the zone that has no A record.
- **Serials.** The SOA serial is a Unix timestamp. It changes only when a
  zone's records change, so re-runs report no changes and secondaries always
  see the serial increase.
- **Keywords.** RHEL 8 (BIND 9.11) gets `master`/`slave`; RHEL 9/10 get
  `primary`/`secondary`. You always write `primary`/`secondary` in inventory.
- **Firewall.** The `dns` firewalld service is opened, and every primary zone
  is checked by querying its SOA.

**Bootstrapping.** Until the DNS node exists, other nodes must use your
existing resolver (`vm_dns_servers`). Once it's up, point `vm_dns_servers` at
it for new VMs.

---

## `satellite`: Red Hat Satellite

**Prerequisites. The role checks these and fails early.**

- RHEL 9, or RHEL 8 for Satellite 6.15/6.16
- At least 4 vCPU and 20 GB RAM (32 GB recommended): `vm_cores: 4`,
  `vm_memory: 32768`
- An FQDN that resolves. The base role adds it to `/etc/hosts` when `vm_ip`
  is set; reverse DNS should resolve too.
- `registration_method: rhsm`, with a subscription that includes Satellite
- Disk space: at least 100 GB for `/`, plus a separate disk for
  `/var/lib/pulp` (synced content, often 500 GB or more)

| Variable | Default | Description |
|---|---|---|
| `satellite_version` | `6.17` | Selects the `satellite-<ver>-for-rhel-<N>-x86_64-rpms` repos |
| `satellite_organization` / `satellite_location` | `Default Organization` / `Default Location` | Initial org and location |
| `satellite_admin_username` | `admin` | |
| `satellite_admin_password` | `vault_satellite_admin_password` | |
| `satellite_installer_extra_args` | `[]` | e.g. `["--foreman-proxy-dns true", "--enable-foreman-plugin-remote-execution"]` |
| `satellite_manage_repos` | `true` | Enable the BaseOS, AppStream, Satellite and Maintenance repos |
| `satellite_firewall_ports` | 80, 443, 5647, 8000, 8140, 9090/tcp | Add `53/udp`, `67/udp`, `69/udp` and so on if you enable DNS, DHCP or TFTP on the integrated capsule |
| `satellite_manifest_file` | `""` | Path on the control node to a subscription manifest `.zip` to upload |
| `satellite_repositories` | `[]` | Red Hat repositories to enable after the manifest upload |
| `satellite_sync_repositories` | `false` | Start an async sync of those products |

```yaml
satellite_manifest_file: files/manifest.zip
satellite_repositories:
  - product: Red Hat Enterprise Linux for x86_64
    name: Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
    releasever: "9"
  - product: Red Hat Enterprise Linux for x86_64
    name: Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
    releasever: "9"
satellite_sync_repositories: true
```

The installer takes 20–60 minutes. It runs once: the marker file
`/etc/foreman-installer/.ansible-installed` stops it running again. To
re-run it with new options, delete the marker, or run `satellite-installer`
by hand. The admin password is hidden from Ansible output. If the install
fails, read `/var/log/foreman-installer/satellite.log`.

**Registering other nodes to this Satellite.** Create an activation key in
Satellite, then set this on the client nodes:

```yaml
registration_method: satellite
satellite_registration_server: sat01.myhome.lab
rhsm_org_id: Lab
rhsm_activation_key: rhel9-base
```

---

## `aap`: Ansible Automation Platform (containerized, all-in-one)

This installs the platform gateway, automation controller, and optionally
automation hub and Event-Driven Ansible on one node. It uses Red Hat's
containerized installer, which is the supported method for AAP 2.5 and later.

**Prerequisites. The role checks these and fails early.**

- RHEL 9 (AAP 2.5/2.6) or RHEL 10 (AAP 2.6)
- At least 4 vCPU and 16 GB RAM, plus at least 60 GB of disk (more with hub)
- The **setup bundle** from access.redhat.com (*Ansible Automation Platform →
  Containerized Setup Bundle*), saved on the control node, e.g.
  `files/ansible-automation-platform-containerized-setup-bundle-2.6-1-x86_64.tar.gz`
- A registered node (`registration_method: rhsm` or `satellite`) to install
  `ansible-core` and `podman`

| Variable | Default | Description |
|---|---|---|
| `aap_version` | `2.6` | |
| `aap_setup_bundle` | `""` | Bundle path on the control node; it's copied to the node |
| `aap_setup_bundle_url` | `""` | ...or a URL the node can download it from |
| `aap_bundle_install` | `true` | `false` means use the online installer, which needs `vault_registry_username`/`vault_registry_password` for registry.redhat.io |
| `aap_install_hub` / `aap_install_eda` | `true` | Include automation hub and Event-Driven Ansible |
| `aap_admin_password` | `vault_aap_admin_password` | Admin password for all components |
| `aap_pg_password` | `vault_aap_pg_password` | PostgreSQL passwords |
| `aap_install_user` | `ansible_user` | The installer runs rootless as this user (a Red Hat requirement) |
| `aap_controller_memory_capacity` | `0.5` | |
| `aap_installer_extra_vars` | `{}` | Extra `key: value` pairs for the installer inventory |
| `aap_firewall_ports` | 80, 443, 27199/tcp | |

The installer takes 30–90 minutes. Its output goes to
`~cloud-user/aap-setup/install.log` on the node, and it runs once, guarded by
the `.installed` marker. When it finishes, browse to
`https://<fqdn>/` and log in as `admin`. Then upload a subscription manifest
(the Red Hat docs walk through it).

The generated installer inventory, `~/aap-setup/inventory-provisioning`
(mode 0600), contains the passwords. For anything beyond all-in-one
(enterprise topology, external database), write that inventory yourself and
run the installer as the Red Hat docs describe.
