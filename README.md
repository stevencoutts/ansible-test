# ansible-test

Ansible playbooks for managing a small fleet of Debian servers.

## Layout

| File | Purpose |
| --- | --- |
| `ansible.cfg` | Points Ansible at the local `inventory` file and sets the `remote_user`. |
| `inventory` | The live host list (group `[servers]`). |
| `inventory.example` | Sanitised template to copy when setting up elsewhere. |
| `nmap_output` | A ping-sweep of the local subnet (reference only, not used by Ansible). |
| `updateServers.yml` | Apt upgrade workflow with safety checks and conditional reboot. |
| `debian-clean.yml` | Idempotent disk-space cleanup. |
| `set_dns.yml` | Configure DNS nameservers via resolvconf. |
| `ntpServers.yml` | Configure NTP time servers. |

## Inventory

Hosts live under the `[servers]` group in the `inventory` file. Copy
`inventory.example` to `inventory` and edit the host list for your environment.

## Requirements

- Ansible (a virtualenv with it installed is included under `venv/`).
- SSH access to the target hosts with sudo/`become` privileges.
- Debian-family targets (`debian-clean.yml` asserts this and aborts otherwise).

Activate the bundled virtualenv if you want to use it:

```sh
source venv/bin/activate
```

## Playbooks

### updateServers.yml

Full apt upgrade across the `servers` group. It asserts at least 500 MB free on
`/` before starting, updates the package cache, shows the pending upgrades, runs
a full upgrade, and restarts affected services with `needrestart`. Failures are
caught by a rescue block (dpkg repair) and an always block always runs
autoremove and autoclean. The host reboots only if `/var/run/reboot-required`
exists.

```sh
ansible-playbook updateServers.yml
```

### debian-clean.yml

Safe, idempotent disk-space cleanup. Reports `df` usage before and after, then
reclaims space from the apt cache, orphaned packages, the systemd journal,
rotated logs, old `/tmp` and `/var/tmp` files, and stale crash dumps. Nothing it
removes touches application data or live config.

Tunables (override with `-e`):

| Variable | Default | Meaning |
| --- | --- | --- |
| `journal_max_age` | `7d` | Keep this much journal history. |
| `journal_max_size` | _(unset)_ | Cap the journal at this size; wins over age if set (e.g. `200M`). |
| `apt_autoremove` | `true` | Remove orphaned dependencies. |
| `clean_old_logs` | `true` | Delete rotated logs under `/var/log`. |
| `clean_tmp` | `true` | Remove old files in `/tmp` and `/var/tmp`. |
| `tmp_age` | `7d` | Age threshold for temp cleanup. |

```sh
# Remote hosts
ansible-playbook debian-clean.yml -i inventory

# Local host only
sudo ansible-playbook debian-clean.yml -i localhost, -c local

# Example with overrides
ansible-playbook debian-clean.yml -e journal_max_size=200M -e clean_tmp=false
```

### set_dns.yml

Installs `resolvconf` and writes the nameservers and search domains to
`/etc/resolvconf/resolv.conf.d/head`. One host is intentionally excluded via a
`when` condition.

```sh
ansible-playbook set_dns.yml
```

### ntpServers.yml

Installs the NTP package and points it at the configured upstream NTP servers,
removing the default Debian pool entries.

```sh
ansible-playbook ntpServers.yml
```

## Notes

- `set_dns.yml` and `ntpServers.yml` use the legacy `ntp` package, which has been
  replaced by `systemd-timesyncd` on Debian 12 and later. These playbooks may
  need updating for newer hosts.

## Licence

MIT. See [`LICENSE`](LICENSE).
