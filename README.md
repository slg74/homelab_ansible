# homelab_ansible

Ansible playbooks for managing a personal homelab running Ubuntu and Kali Linux.

## Inventory

| Host | Group | Description |
|---|---|---|
| ubuntu1 | ubuntu, apt_systems | Ubuntu Linux system |
| kali | kali, apt_systems | Kali Linux rolling release system |

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
Hardens `sshd_config` on all systems: disables root login and password authentication, enforces modern ciphers/MACs/KexAlgorithms, sets idle timeouts, and restricts forwarding. Backs up the original config before making changes and validates with `sshd -t` before restarting to prevent lockouts.

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

### bootstrap_sudo.yml
One-time setup playbook that grants passwordless sudo to the `scott` user on all systems. Must be run with `--ask-become-pass` using a user that already has sudo access. Only needs to be run once per new host.

```bash
ansible-playbook bootstrap_sudo.yml --ask-become-pass
```
