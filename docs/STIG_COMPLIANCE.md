# DISA STIG compliance

Each node chooses for itself:

```yaml
stig_enabled: true     # DISA STIG hardened
stig_enabled: false    # standard build (default)
```

Nodes with `stig_enabled: true` join the `stig` group automatically. They get
`group_vars/stig.yml` and the `playbooks/apply_stig.yml` phase. Nodes in the
`nostig` group are never touched by it.

## What gets applied

`roles/stig_compliance` wraps Red Hat's official Ansible roles, which are
generated from the ComplianceAsCode project for the DISA STIG profile:

| RHEL | Role (installed by `requirements.yml`) |
|---|---|
| 8 | `RedHatOfficial.rhel8_stig` |
| 9 | `RedHatOfficial.rhel9_stig` |
| 10 | `RedHatOfficial.rhel10_stig` |

These are the server profiles. For a workstation with a GUI there is
`RedHatOfficial.rhel10_stig_gui`; set `stig_role_name` to use it.

The wrapper runs these steps, in order:

1. **Guards against sudo lockout.** The STIG removes `NOPASSWD` from sudoers,
   after which Ansible can only become root with a password. The play stops
   unless `ansible_become_password` is set (from `vault_vm_ci_password`) or
   you've disabled that rule.
2. **Enables FIPS mode.** On RHEL 8/9 it runs `fips-mode-setup --enable` and
   reboots. RHEL 10 can't switch after install (see below).
3. **Applies role exceptions.** Rules that would break a node's server roles
   are turned off (see below).
4. **Runs the STIG role** for the node's RHEL version.
5. **Reboots once,** after the first application only. The marker file
   `/etc/ansible-stig-applied` records it.
6. **Scans with OpenSCAP** using the DISA STIG profile and saves the HTML
   report to `reports/<fqdn>-stig-report.html` on the control node.

Order in `site.yml`: base OS, then STIG, then server roles. Services are
installed onto an already-hardened system, so they don't get reconfigured
behind your back on the next STIG run.

## Settings

| Variable | Default | Description |
|---|---|---|
| `stig_enabled` | `false` | Per node: apply the STIG |
| `stig_enable_fips` | `true` | Enable FIPS mode on RHEL 8/9 |
| `stig_require_fips` | `false` | Fail RHEL 10 nodes that aren't in FIPS mode (instead of warning) |
| `stig_auto_exceptions` | `true` | Apply the role-conflict exceptions below |
| `stig_reboot` | `true` | Reboot after the first application |
| `stig_scan` | `true` | Run an OpenSCAP scan and fetch the report |
| `stig_report_dir` | `<project>/reports` | Where reports are saved |
| `stig_role_name` | `RedHatOfficial.rhel<N>_stig` | Override the STIG role |

## Turning individual STIG rules off

Each rule in the RedHatOfficial roles is a boolean variable named after the
rule, and all of them default to `true`. Set a rule to `false` in
`group_vars/stig.yml` to change it for every STIG node, or in
`host_vars/<fqdn>.yml` for a single node:

```yaml
sudo_remove_nopasswd: false        # keep passwordless sudo for Ansible
mount_option_tmp_noexec: false
package_tuned_removed: false
```

Some rules also take values:

```yaml
var_accounts_tmout: '900'
var_system_crypto_policy: FIPS:STIG
login_banner_contents: "..."
var_multiple_time_servers: "{{ chrony_servers | join(',') }}"   # already set in group_vars/stig.yml
```

The full list is in the installed role, e.g.
`~/.ansible/roles/RedHatOfficial.rhel9_stig/defaults/main.yml`. Every rule you
disable is a deviation from the STIG, so record it in your SSP/POA&M.

## Automatic role exceptions

When `stig_auto_exceptions: true`, the wrapper forces these rules off on nodes
with the matching server role. The list is the `stig_role_exceptions`
dictionary in `roles/stig_compliance/defaults/main.yml`. To extend it,
redefine the whole dictionary in `group_vars/stig.yml`.

| Server role | Rule | Why |
|---|---|---|
| `ntp` | `chronyd_client_only: false` | An NTP server has to listen on UDP/123 |

Notes on the other roles:

- **`satellite`.** The STIG sets a login umask of 077, which breaks the
  installer, so the role runs `satellite-installer` under `umask 0022`.
  `foreman-maintain` and later re-runs of `satellite-installer` that you do by
  hand also need `umask 0022` first. If the STIG removes `tftp-server` and you
  want to provision over PXE from the integrated capsule, set
  `package_tftp_server_removed: false`.
- **`aap`.** The containerized installer runs rootless podman as the SSH user
  and needs a become password on STIG nodes. That's the same
  `vault_vm_ci_password` the guard checks for.
- **`fapolicyd`.** The STIG enables fapolicyd in deny-by-default mode. If a
  service fails to start with permission errors, check
  `fapolicyd-cli --list` and `journalctl -u fapolicyd`, then add trust rules.

## FIPS mode

| RHEL | How the playbooks handle it |
|---|---|
| 8, 9 | `fips-mode-setup --enable`, then reboot, the first time only |
| 10 | FIPS can only be enabled at install time. Build the RHEL 10 template with `fips=1` ([PROXMOX_TEMPLATES.md](PROXMOX_TEMPLATES.md)). Otherwise the role warns, or fails if `stig_require_fips: true` |

Check FIPS on every STIG node:

```bash
ansible stig -m command -a 'cat /proc/sys/crypto/fips_enabled'   # 1 = on
```

## Scan reports

After every STIG run, each node gets
`reports/<fqdn>-stig-report.html`. Open it in a browser to see pass/fail for
each rule, with the STIG ID and how to fix it. Re-scan without re-applying:

```bash
ansible-playbook playbooks/apply_stig.yml --limit ns01.myhome.lab   # applies (idempotent) + scans
```

Expect some failures even on a correctly hardened VM:

- **Separate partitions** (`/var`, `/var/log`, `/var/log/audit`, `/tmp`,
  `/var/tmp`, `/home`). The KVM guest image has one root filesystem. Build
  STIG templates with that layout if you need these rules.
- **FIPS on RHEL 10** if the template wasn't installed with `fips=1`.
- **Organization-specific items:** AIDE baseline review, banners, account
  policies tied to your IdM, audit log offloading, and so on.
- **Rules you disabled on purpose,** including the automatic exceptions.

For an official result, scan with DISA's SCC tool or upload the ARF to your
assessment system. The ARF results stay on the node at
`/var/tmp/stig-arf.xml`.

## Turning the STIG off later

Setting `stig_enabled: false` stops new STIG runs. It does **not** undo
hardening that's already applied; the STIG roles have no rollback. To get an
unhardened node, rebuild it:

```bash
ansible-playbook playbooks/destroy_vm.yml --limit <fqdn> -e confirm_destroy=yes
ansible-playbook site.yml --limit <fqdn>
```
