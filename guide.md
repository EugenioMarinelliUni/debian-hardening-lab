---
layout: default
title: Complete Hardening Procedure
---

# Complete Debian 13 Hardening Procedure

This procedure combines the two source labs into one sequence. Role-specific steps are marked where relevant.

> **Safety:** Keep an existing SSH session open while changing SSH or firewall settings. Prefer VM console access and a snapshot before making disruptive changes.

## 1. Establish the server role

Before changing anything, write down what the machine is supposed to do. Typical required services might be SSH for administration, HTTP only if a web service is needed, and SMB only for a file server. Anything not required should be considered for removal.

## 2. Create a rollback point

Create a VM snapshot before modifying authentication, networking, firewall rules, services, filesystem permissions, or kernel parameters.

Example snapshot names:

```text
01-INSECURE-BASELINE
01-INSECURE-FILESERVER
```

A snapshot is a rollback mechanism, not a substitute for a tested backup.

## 3. Identify the operating system and network

```bash
cat /etc/os-release
uname -a
hostnamectl
ip addr
ip route
```

These commands confirm the distribution, kernel, hostname, interfaces, IP addresses, and routing table before any hardening decisions are made.

## 4. Capture a before-state

```bash
mkdir -p ~/assessment-before

ss -lntup > ~/assessment-before/ss.txt
systemctl --type=service --state=running \
    > ~/assessment-before/services.txt
sudo nft list ruleset \
    > ~/assessment-before/nftables.txt
sudo sshd -T \
    > ~/assessment-before/sshd-effective.txt
sudo aa-status \
    > ~/assessment-before/apparmor.txt 2>&1
```

For a Samba server also capture:

```bash
sudo testparm -s \
    > ~/assessment-before/samba.txt
```

From another authorized host, scan the server externally:

```bash
sudo nmap -sS -sV 10.10.10.20 -oN nmap-before.txt
```

For the file-server example:

```bash
sudo nmap -sS -sV 10.10.10.30 -oN nmap-before.txt
```

The local `ss` view shows what is listening; Nmap shows what is actually reachable from another host.

## 5. Patch the system

Inspect available upgrades:

```bash
apt list --upgradable
```

Refresh package metadata:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt full-upgrade
```

Reboot if required:

```bash
sudo reboot
```

Then check again:

```bash
apt list --upgradable
```

## 6. Audit local accounts

List accounts:

```bash
getent passwd
```

Highlight accounts with interactive shells:

```bash
awk -F: '$7 !~ /(nologin|false)$/ {print $1,$6,$7}' /etc/passwd
```

For an obsolete account such as `legacy` or `contractor`:

```bash
sudo passwd -l legacy
sudo usermod -s /usr/sbin/nologin legacy
sudo usermod --expiredate 1 legacy
```

Verify:

```bash
passwd -S legacy
getent passwd legacy
id legacy
```

`passwd -l` locks password authentication. The `nologin` shell and account expiration provide stronger disablement.

## 7. Review password-aging policy

```bash
sudo chage -l operator
```

A lab policy example is:

```bash
sudo chage -M 90 -m 1 -W 14 operator
```

Interpret this as an organizational example, not a universal security requirement.

## 8. Audit sudo privileges

```bash
getent group sudo
sudo -l
sudo grep -R "NOPASSWD" \
    /etc/sudoers \
    /etc/sudoers.d \
    2>/dev/null
```

A rule such as:

```text
sysadmin ALL=(ALL:ALL) NOPASSWD: ALL
```

grants unrestricted passwordless sudo. As a minimum lab remediation, require sudo authentication:

```text
sysadmin ALL=(ALL:ALL) ALL
```

Edit sudo configuration only with:

```bash
sudo visudo
```

or:

```bash
sudo visudo -f /etc/sudoers.d/<file>
```

Then verify again with:

```bash
sudo -l
```

## 9. Prepare SSH key authentication

On the administrative workstation:

```bash
ssh-keygen -t ed25519
ssh-copy-id sysadmin@10.10.10.20
ssh sysadmin@10.10.10.20
```

For the file server:

```bash
ssh-copy-id operator@10.10.10.30
ssh operator@10.10.10.30
```

Do not disable password authentication until key authentication has been tested successfully.

## 10. Harden the SSH server

Inspect the effective configuration:

```bash
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication'
```

Inspect existing drop-ins:

```bash
ls -la /etc/ssh/sshd_config.d/
```

Create a hardening drop-in, for example:

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

Suggested content:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no

X11Forwarding no
AllowTcpForwarding no
GatewayPorts no

MaxAuthTries 3
LoginGraceTime 30
AllowUsers sysadmin
```

For the file server, replace `sysadmin` with `operator` if appropriate.

Validate syntax:

```bash
sudo sshd -t
```

Inspect effective values:

```bash
sudo sshd -T
```

Reload only after validation succeeds:

```bash
sudo systemctl reload ssh
```

Positive test:

```bash
ssh sysadmin@10.10.10.20
```

Negative tests:

```bash
ssh root@10.10.10.20
ssh -o PubkeyAuthentication=no sysadmin@10.10.10.20
```

Root login and password-only access should fail.

## 11. Minimize the service attack surface

Re-enumerate:

```bash
ss -lntup
systemctl --type=service --state=running
systemctl list-unit-files --type=service
```

For every service ask whether the role actually requires it.

### Remove FTP if unnecessary

```bash
sudo systemctl disable --now vsftpd
sudo apt purge vsftpd
```

Verify locally:

```bash
ss -lntup | grep ':21 '
```

Verify remotely:

```bash
nmap -p21 10.10.10.20
```

### Remove rpcbind if unnecessary

```bash
sudo systemctl disable --now rpcbind
sudo systemctl disable --now rpcbind.socket
sudo apt purge rpcbind
```

Verify:

```bash
ss -lntup | grep ':111 '
```

## 12. Harden nginx if it is required

If nginx is unnecessary:

```bash
sudo systemctl disable --now nginx
sudo apt purge nginx nginx-common
```

If it is required, inspect the complete configuration:

```bash
sudo nginx -T
```

Remove unintended directory indexing such as:

```nginx
autoindex on;
```

Reduce banner disclosure:

```nginx
server_tokens off;
```

Move sensitive backups outside the web root:

```bash
sudo mkdir -p /srv/private-backups
sudo mv /var/www/html/backups/* /srv/private-backups/
```

Or remove obsolete material carefully:

```bash
sudo rm -rf /var/www/html/backups
```

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Test:

```bash
curl -I http://10.10.10.20/
curl http://10.10.10.20/backups/
```

The sensitive path should return a denial or not-found response rather than a listing.

## 13. Harden Apache if it is required

If Apache is unnecessary:

```bash
sudo systemctl disable --now apache2
sudo apt purge apache2 apache2-bin apache2-data apache2-utils
```

If it is required:

```bash
sudo apache2ctl -S
sudo grep -R "Options .*Indexes" /etc/apache2 2>/dev/null
```

Disable directory listing in the relevant configuration:

```apache
Options -Indexes
```

Reduce banner disclosure:

```apache
ServerTokens Prod
ServerSignature Off
```

Validate and reload:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Test:

```bash
curl -I http://10.10.10.30/
curl http://10.10.10.30/archive/
```

Use HTTPS/TLS for real applications carrying credentials or sensitive data.

## 14. Harden Samba for the file-server role

Inspect configuration and guest visibility:

```bash
sudo testparm -s
smbclient -L localhost -N
```

Create a dedicated collaboration group:

```bash
sudo groupadd -f fileshare
sudo usermod -aG fileshare operator
id operator
```

Add Samba credentials:

```bash
sudo smbpasswd -a operator
```

Fix ownership:

```bash
sudo chown -R root:fileshare /srv/public-share
```

Set directory permissions safely:

```bash
sudo find /srv/public-share \
    -type d \
    -exec chmod 2770 {} +
```

Set file permissions separately:

```bash
sudo find /srv/public-share \
    -type f \
    -exec chmod 0660 {} +
```

Example share configuration:

```ini
[public-lab]
    path = /srv/public-share
    browseable = yes
    guest ok = no
    read only = no
    valid users = @fileshare
    force group = fileshare
    create mask = 0660
    directory mask = 2770
```

Validate and reload:

```bash
sudo testparm -s
sudo systemctl reload smbd
```

Positive test:

```bash
smbclient //localhost/public-lab -U operator
```

Negative guest test:

```bash
smbclient //localhost/public-lab -N
```

Guest access should fail.

## 15. Audit filesystem permissions

Find world-writable directories:

```bash
sudo find / -xdev -type d -perm -0002 2>/dev/null
```

Find world-writable files:

```bash
sudo find / -xdev -type f -perm -0002 2>/dev/null
```

Do not blindly modify every result. `/tmp` and `/var/tmp`, for example, are expected to be writable and normally rely on the sticky bit.

Inspect a directory and its contents:

```bash
ls -ld /srv/company-share
ls -l /srv/company-share
stat /srv/company-share
```

For a generic protected share:

```bash
sudo chown -R root:sysadmin /srv/company-share
sudo find /srv/company-share -type d -exec chmod 0750 {} +
sudo find /srv/company-share -type f -exec chmod 0640 {} +
```

## 16. Audit privileged cron jobs

Inspect the script and job definition:

```bash
ls -l /opt/labapp/bin/maintenance.sh
sudo cat /etc/cron.d/lab-maintenance
```

If root executes a world-writable script, an ordinary user may be able to modify code that root later runs.

Fix ownership and permissions:

```bash
sudo chown root:root /opt/labapp/bin/maintenance.sh
sudo chmod 0750 /opt/labapp/bin/maintenance.sh
```

Also inspect every parent directory:

```bash
namei -l /opt/labapp/bin/maintenance.sh
```

If the scheduled job is unnecessary:

```bash
sudo rm -f /etc/cron.d/lab-maintenance
```

Audit scheduled jobs generally:

```bash
sudo ls -la /etc/cron.d
sudo ls -la /etc/cron.daily
sudo systemctl status cron
```

## 17. Search for misplaced secrets

A simple educational search is:

```bash
sudo grep -RniE \
'password|token|secret|key' \
/home /opt/labapp /srv/public-share \
2>/dev/null
```

This is not a complete secret scanner and can produce false positives.

Protect a private directory:

```bash
sudo install -d -m 0700 -o sysadmin -g sysadmin \
    /home/sysadmin/private
```

Move and restrict a note:

```bash
sudo mv /home/sysadmin/notes.txt /home/sysadmin/private/
sudo chmod 0600 /home/sysadmin/private/notes.txt
```

If a real password or token was exposed, deleting the file is not enough: revoke or rotate the credential as well.

## 18. Configure nftables

Inspect the current state:

```bash
sudo nft list ruleset
systemctl status nftables
```

Edit:

```bash
sudo nano /etc/nftables.conf
```

Example simple-server ruleset:

```nft
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
        policy drop;

        iifname "lo" accept
        ct state established,related accept
        ct state invalid drop

        ip protocol icmp accept
        ip6 nexthdr ipv6-icmp accept

        tcp dport 22 accept

        # Keep only if HTTP is required
        tcp dport 80 accept
    }

    chain forward {
        type filter hook forward priority 0;
        policy drop;
    }

    chain output {
        type filter hook output priority 0;
        policy accept;
    }
}
```

Where possible, restrict SSH to the management subnet:

```nft
ip saddr 192.168.1.0/24 tcp dport 22 accept
```

For Samba, restrict SMB to the authorized LAN:

```nft
ip saddr 10.10.10.0/24 tcp dport 445 accept
```

Validate syntax before loading:

```bash
sudo nft -c -f /etc/nftables.conf
```

Keep the current SSH session open and, preferably, console access available. Apply:

```bash
sudo nft -f /etc/nftables.conf
```

Immediately test a new SSH connection. If it succeeds, enable persistence:

```bash
sudo systemctl enable --now nftables
```

Verify:

```bash
sudo nft list ruleset
systemctl status nftables
```

## 19. Enable and verify AppArmor

```bash
systemctl status apparmor
sudo aa-status
```

If masked:

```bash
sudo systemctl unmask apparmor
```

Enable and start:

```bash
sudo systemctl enable --now apparmor
```

Verify:

```bash
systemctl is-enabled apparmor
systemctl is-active apparmor
sudo aa-status
```

If a relevant profile is in complain mode and has been tested adequately:

```bash
sudo aa-enforce /etc/apparmor.d/<profile>
```

After each policy change:

```bash
systemctl status <service>
journalctl -u <service>
```

Do not blindly enforce every profile.

## 20. Harden selected sysctl parameters

Inspect current values:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.default.accept_redirects
sysctl net.ipv4.conf.all.send_redirects
sysctl net.ipv4.conf.default.send_redirects
sysctl net.ipv4.conf.all.accept_source_route
sysctl net.ipv4.conf.default.accept_source_route
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
sysctl net.ipv4.tcp_syncookies
sysctl kernel.dmesg_restrict
sysctl kernel.kptr_restrict
sysctl fs.protected_hardlinks
sysctl fs.protected_symlinks
```

For a simple, single-homed, non-routing server, create:

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Suggested contents:

```text
net.ipv4.ip_forward = 0

net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

net.ipv4.tcp_syncookies = 1
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

Apply:

```bash
sudo sysctl --system
```

Strict `rp_filter=1` is role-dependent and may be inappropriate for multihomed hosts, asymmetric routing, VPNs, or policy routing.

## 21. Make journald persistent

Inspect existing journal state:

```bash
journalctl --list-boots
systemd-analyze cat-config systemd/journald.conf
```

Remove the deliberately insecure lab override if present:

```bash
sudo rm -f /etc/systemd/journald.conf.d/90-lab-insecure.conf
sudo rm -f /etc/systemd/journald.conf.d/90-lab-insecure-fileserver.conf
```

Create:

```bash
sudo nano /etc/systemd/journald.conf.d/90-persistent.conf
```

Contents:

```ini
[Journal]
Storage=persistent
Compress=yes
```

Create the persistent storage directory and restart journald:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

Verify:

```bash
journalctl --disk-usage
journalctl --list-boots
```

After a controlled reboot, verify that previous boots remain visible.

Inspect relevant services:

```bash
journalctl -u ssh
journalctl -u nftables
journalctl -u apparmor
```

On the file server also:

```bash
journalctl -u smbd
journalctl -u apache2
journalctl -u cron
```

## 22. Configure automatic updates

Check package and timers:

```bash
dpkg -l unattended-upgrades
systemctl status apt-daily.timer
systemctl status apt-daily-upgrade.timer
```

Install if necessary:

```bash
sudo apt install unattended-upgrades
```

Unmask timers if necessary:

```bash
sudo systemctl unmask apt-daily.timer 2>/dev/null || true
sudo systemctl unmask apt-daily-upgrade.timer 2>/dev/null || true
```

Enable them:

```bash
sudo systemctl enable --now apt-daily.timer
sudo systemctl enable --now apt-daily-upgrade.timer
```

Configure `/etc/apt/apt.conf.d/20auto-upgrades`:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Verify:

```bash
systemctl list-timers | grep apt
sudo unattended-upgrade --dry-run --debug
```

## 23. Perform final verification

Check listeners and services:

```bash
ss -lntup
systemctl --type=service --state=running
```

Validate SSH:

```bash
sudo sshd -t
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|x11forwarding|allowtcpforwarding|gatewayports'
```

Validate Samba where applicable:

```bash
sudo testparm -s
smbclient //localhost/public-lab -U operator
smbclient //localhost/public-lab -N
```

Validate web server where applicable:

```bash
sudo nginx -t
sudo apache2ctl configtest
```

Validate firewall:

```bash
sudo nft list ruleset
```

Validate AppArmor:

```bash
sudo aa-status
```

Validate sysctl values:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv4.conf.all.accept_redirects
sysctl net.ipv4.conf.all.accept_source_route
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.tcp_syncookies
sysctl kernel.dmesg_restrict
sysctl kernel.kptr_restrict
sysctl fs.protected_hardlinks
sysctl fs.protected_symlinks
```

## 24. Run the final external scan

```bash
sudo nmap -sS -sV 10.10.10.20 -oN nmap-after.txt
```

or:

```bash
sudo nmap -sS -sV 10.10.10.30 -oN nmap-after.txt
```

Compare the before and after scans. The intended result is that only services justified by the machine's role remain reachable.

## 25. Save the after-state

```bash
mkdir -p ~/assessment-after

ss -lntup > ~/assessment-after/ss.txt
systemctl --type=service --state=running \
    > ~/assessment-after/services.txt
sudo nft list ruleset \
    > ~/assessment-after/nftables.txt
sudo sshd -T \
    > ~/assessment-after/sshd-effective.txt
sudo aa-status \
    > ~/assessment-after/apparmor.txt 2>&1
```

For Samba:

```bash
sudo testparm -s \
    > ~/assessment-after/samba.txt
```

The before/after evidence is part of the hardening process: it shows what changed and makes the work auditable.

## Final principle

For every component:

```text
Needed?
  |
  +-- No  → remove or disable
  |
  +-- Yes → minimize privileges
            minimize exposure
            use restrictive policy
            log relevant events
            verify the result
```

Hardening is not one setting. It is a layered reduction of unnecessary trust, privilege, and exposure.
