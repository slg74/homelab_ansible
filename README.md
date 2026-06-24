# homelab_ansible

Ansible playbooks for managing a personal homelab running Ubuntu and Kali Linux.

## Inventory

### Static (homelab)

| Host | Group | Description |
|---|---|---|
| ubuntu1 | ubuntu, apt_systems, loki_server | Ubuntu Linux system — runs Loki and Grafana |
| kali | kali, apt_systems | Kali Linux rolling release system |

### Dynamic (AWS EC2)

Managed via `inventory/aws_ec2.yml` using the `amazon.aws.aws_ec2` plugin. Includes running and stopped instances in `us-east-1`. Credentials must be available in the environment (`~/.aws/credentials` or env vars).

| Host | Group | User | Description |
|---|---|---|---|
| bobasoft_ubuntu1 | bobasoft_ubuntu1 | ubuntu | Ubuntu 24.04 EC2 instance |

## Roles

Reusable, self-contained units of Ansible logic. Each role lives under `roles/<name>/` and splits its tasks, handlers, and default variables into separate files so they can be tested in isolation and called from any playbook.

```
roles/
  harden_ssh/
    defaults/main.yml   # variables and their defaults (override per host/group)
    tasks/main.yml      # the actual work
    handlers/main.yml   # triggered on change (e.g. restart sshd after config change)
    molecule/default/   # automated tests for this role
```

### Testing roles with Molecule

Molecule spins up a Docker container, runs the role against it, checks idempotency, runs the verify assertions, then destroys the container.

```bash
# From the role directory:
cd roles/harden_ssh

molecule test          # full lifecycle: create → converge → idempotence → verify → destroy
molecule converge      # apply the role to the container (without destroying it)
molecule idempotence   # re-run the role and assert zero changes
molecule verify        # run the verify.yml assertions only
molecule login         # open a shell inside the test container for manual inspection
molecule destroy       # tear down the container
```

The quickest iteration loop when writing or fixing a role:

```bash
molecule converge && molecule idempotence
```

### harden_ssh

Hardens `sshd_config`: disables root login and password auth, enforces modern ciphers/MACs/KexAlgorithms, sets idle timeouts, restricts forwarding. Backs up the original config before any changes and validates with `sshd -t` before restarting sshd to prevent lockouts. Called by `harden_ssh.yml`.

## Playbooks

### patch_ubuntu.yml
Applies security patches to Ubuntu systems using `unattended-upgrades`, which limits updates to security-only packages. Syncs the system clock via NTP before patching to avoid apt release file timestamp errors. Reboots automatically if `/var/run/reboot-required` is present after patching.

```bash
ansible-playbook patch_ubuntu.yml
```

### patch_kali.yml
Performs a full `dist-upgrade` on Kali Linux systems. Kali is a rolling release distro, so all packages are upgraded rather than security-only. Syncs the system clock before patching and reboots automatically if required.

```bash
ansible-playbook patch_kali.yml
```

### cleanup_var.yml
Cleans up disk space under `/var` on all apt-managed systems. Removes the apt package cache, orphaned packages, rotated log files older than 14 days, and files in `/var/tmp` not accessed in 7 days. Age thresholds are configurable via `var_tmp_age_days` and `log_rotated_age_days`.

```bash
ansible-playbook cleanup_var.yml
```

### sync_ntp.yml
Configures NTP on all apt-managed systems to sync against NIST time servers (`time-a-g.nist.gov`, `time-b-g.nist.gov`, `time.nist.gov`). Automatically detects whether chrony or systemd-timesyncd is in use and updates the appropriate config file, then forces an immediate time sync.

```bash
ansible-playbook sync_ntp.yml
```

### get_date.yml
Fetches and displays the current date and time from all apt-managed systems. Useful for quickly verifying clock sync after running `sync_ntp.yml`.

```bash
ansible-playbook get_date.yml
```

### harden_ssh.yml
Runs the `harden_ssh` role against all apt-managed systems. See [Roles → harden_ssh](#harden_ssh) for full details.

```bash
ansible-playbook harden_ssh.yml
```

### audit_lynis.yml
Installs Lynis and runs a full system security audit on each host. Fetches the log and report files back to `./reports/<hostname>/` on the controller. Displays each host's hardening index score in the play output so you can compare ubuntu1 vs kali side by side.

```bash
ansible-playbook audit_lynis.yml
```

### backup_configs.yml
Pulls critical system config files from each host back to `./backups/<hostname>/<date>/` on the controller. Covers SSH config, sudoers, hosts, fstab, crontab, passwd/group, and apt sources. Optional files (fail2ban, ufw, etc.) are included if present. Run this before and after any major change.

```bash
ansible-playbook backup_configs.yml
```

### setup_fail2ban.yml
Installs fail2ban and deploys a `jail.local` that enables the SSH jail, bans offending IPs for 1 hour after 3 failed attempts within 10 minutes, and whitelists the local `192.168.1.0/24` subnet. Displays the active jail status at the end of the run.

```bash
ansible-playbook setup_fail2ban.yml
```

### setup_loki.yml
Deploys a centralized log aggregation stack. Installs Loki (log storage) and Grafana (UI) on `ubuntu1`, and Promtail (log shipper) on all hosts. Promtail collects `/var/log/**/*.log` and the systemd journal from each host and forwards them to Loki. The Loki datasource is provisioned in Grafana automatically.

```bash
ansible-playbook setup_loki.yml --ask-become-pass
```

After the run, Grafana is available at `http://192.168.1.57:3000` (default credentials `admin / admin`). To query logs, go to **Explore → Loki** and use LogQL:

```logql
# All logs from a specific host
{host="ubuntu1"}

# SSH auth logs across all hosts
{job="varlogs"} |= "sshd"

# All systemd journal entries for a unit
{job="systemd-journal", unit="fail2ban.service"}
```

**Ports used:**
| Port | Service | Host |
|------|---------|------|
| 3100 | Loki HTTP API | ubuntu1 |
| 3000 | Grafana | ubuntu1 |
| 9080 | Promtail metrics | all hosts |

### start_ec2.yml
Starts the two stopped EC2 instances (`bobasoft_ubuntu1`, `k8s-infographics`) in `us-east-1` and waits until they are fully running. Requires AWS credentials available in the environment.

```bash
ansible-playbook start_ec2.yml
```

### stop_ec2.yml
Gracefully stops the same two EC2 instances and waits until they reach the stopped state.

```bash
ansible-playbook stop_ec2.yml
```

### bootstrap_sudo.yml
One-time setup playbook that grants passwordless sudo to the `scott` user on all systems. Must be run with `--ask-become-pass` using a user that already has sudo access. Only needs to be run once per new host.

```bash
ansible-playbook bootstrap_sudo.yml --ask-become-pass
```
