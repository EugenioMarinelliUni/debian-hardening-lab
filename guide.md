---
layout: default
title: Complete Hardening Procedure
---

# Complete Debian 13 Hardening Procedure for `portal.fav.it`

This procedure is tailored to the final assignment: a Debian 13 server hosting the company portal `portal.fav.it`.

## 1. Preserve recoverability

Before touching SSH, firewall rules, authentication, filesystem ownership or kernel parameters:

- create a VM snapshot if available;
- keep the current administrative SSH session open;
- keep console access available if possible.

Create evidence directories:

```bash
mkdir -p ~/hardening-evidence/{before,after}
script -a ~/hardening-evidence/hardening-session.log
```

Exit `script` recording later with:

```bash
exit
```

## 2. Confirm identity and networking

```bash
cat /etc/os-release
uname -a
hostnamectl
hostname
hostname -f
ip -br addr
ip route
getent hosts portal.fav.it
```

The intended FQDN is:

```text
portal.fav.it
```

If the configured hostname is wrong:

```bash
sudo hostnamectl set-hostname portal.fav.it
```

Inspect `/etc/hosts` before modifying local name resolution:

```bash
cat /etc/hosts
```

## 3. Capture the baseline

```bash
ss -lntup | tee ~/hardening-evidence/before/listening-sockets.txt
systemctl --type=service --state=running \
  | tee ~/hardening-evidence/before/running-services.txt
sudo nft list ruleset \
  | tee ~/hardening-evidence/before/nftables.txt
sudo sshd -T \
  | tee ~/hardening-evidence/before/sshd-effective.txt
sudo aa-status 2>&1 \
  | tee ~/hardening-evidence/before/apparmor.txt
```

From another authorized machine:

```bash
sudo nmap -sS -sV -p- portal.fav.it \
  -oN nmap-before.txt
```

The local `ss` view shows listening sockets. The external Nmap view shows what another host can reach.

## 4. Investigate artifacts from the previous administrator

Search filenames:

```bash
sudo find /etc /usr/local /opt \
  -xdev \
  -iname '*asdrubale*' \
  -ls \
  2>/dev/null
```

Search configuration contents:

```bash
sudo grep -RniI 'asdrubale' \
  /etc /usr/local /opt \
  2>/dev/null
```

Check systemd, cron, sudo and SSH locations:

```bash
systemctl list-unit-files --all | grep -i asdrubale
sudo grep -RniI 'asdrubale' /etc/cron* /var/spool/cron 2>/dev/null
sudo grep -RniI 'asdrubale' /etc/sudoers /etc/sudoers.d 2>/dev/null
sudo grep -RniI 'asdrubale' /etc/ssh /root/.ssh /home/*/.ssh 2>/dev/null
```

Do not delete matches automatically. Inspect suspicious files and units:

```bash
sudo stat /path/to/file
sudo less /path/to/file
sudo systemctl cat suspicious.service
sudo systemctl status suspicious.service
```

Disable only what is confirmed unnecessary:

```bash
sudo systemctl disable --now suspicious.service
```

## 5. Audit accounts and keys

List accounts and likely interactive shells:

```bash
getent passwd
awk -F: '$7 !~ /(nologin|false)$/ {print $1,$6,$7}' /etc/passwd
```

Look for a former-administrator account:

```bash
getent passwd | grep -i asdrubale
getent group | grep -i asdrubale
```

If it exists:

```bash
id asdrubale
sudo -l -U asdrubale
```

If confirmed obsolete:

```bash
sudo passwd -l asdrubale
sudo usermod -s /usr/sbin/nologin asdrubale
sudo usermod --expiredate 1 asdrubale
sudo gpasswd -d asdrubale sudo   # only if actually a member
```

Verify:

```bash
passwd -S asdrubale
getent passwd asdrubale
id asdrubale
```

Enumerate SSH authorization files:

```bash
sudo find /root /home \
  -path '*/.ssh/authorized_keys' \
  -type f \
  -print \
  2>/dev/null
```

Review key contents:

```bash
sudo find /root /home \
  -path '*/.ssh/authorized_keys' \
  -type f \
  -exec sh -c 'echo "===== $1 ====="; nl -ba "$1"' _ {} \; \
  2>/dev/null
```

Remove only keys that are demonstrably unauthorized.

## 6. Change the known `sysadmin` password

The delivered password must be considered compromised.

```bash
sudo passwd sysadmin
sudo passwd -S sysadmin
sudo chage -l sysadmin
```

Do not put the new password directly in shell history.

## 7. Keep root local-console recovery, but block root over SSH

Check root password state:

```bash
sudo passwd -S root
```

Do **not** lock root for this exercise.

Test local-console root access from the actual console:

```bash
whoami
tty
id
```

A local console normally shows a `/dev/tty*` terminal. Remote SSH root access will be disabled separately with:

```text
PermitRootLogin no
```

## 8. Patch the server

```bash
apt list --upgradable
sudo apt update
sudo apt full-upgrade
```

Reboot if required:

```bash
sudo reboot
```

Then verify again:

```bash
apt list --upgradable
```

## 9. Audit sudo privileges

```bash
getent group sudo
sudo -l
sudo grep -R "NOPASSWD" /etc/sudoers /etc/sudoers.d 2>/dev/null
```

Edit sudo configuration only with:

```bash
sudo visudo
```

or:

```bash
sudo visudo -f /etc/sudoers.d/<file>
```

At minimum, remove unnecessary unrestricted `NOPASSWD: ALL` rules. Prefer command-specific least privilege where practical.

## 10. Prepare administrative SSH key access before disabling passwords

On the administrator workstation:

```bash
ssh-keygen -t ed25519
ssh-copy-id sysadmin@portal.fav.it
ssh sysadmin@portal.fav.it
```

Do not disable password authentication until this succeeds in a **new** session.

## 11. Identify the active web server

```bash
systemctl --type=service --state=running \
  | grep -E 'apache2|nginx'

sudo ss -lntp | grep -E ':(80|443)\b'
ps -ef | grep -E '[n]ginx|[a]pache2'
```

Determine the portal document root and TLS configuration.

For nginx:

```bash
sudo nginx -T 2>&1 \
  | grep -nE 'server_name|root |listen .*80|listen .*443|ssl_certificate'
```

For Apache:

```bash
sudo apache2ctl -S
sudo grep -RniE \
  'DocumentRoot|VirtualHost|SSLEngine|SSLCertificate' \
  /etc/apache2/sites-enabled /etc/apache2/sites-available
```

## 12. Preserve and inspect the existing self-signed certificate

Once the certificate path is known:

```bash
sudo openssl x509 \
  -in /path/to/existing-certificate.crt \
  -noout -subject -issuer -serial -dates -fingerprint -sha256
```

Save its fingerprint before changes:

```bash
sudo openssl x509 \
  -in /path/to/existing-certificate.crt \
  -noout -fingerprint -sha256 \
  | tee ~/hardening-evidence/before/certificate-fingerprint.txt
```

Inspect what the live service presents:

```bash
openssl s_client \
  -connect portal.fav.it:443 \
  -servername portal.fav.it \
  </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -fingerprint -sha256
```

Do not generate a replacement certificate for this exercise.

## 13. Enforce HTTP-to-HTTPS redirection

Test current behavior:

```bash
curl -I http://portal.fav.it/
curl -kI https://portal.fav.it/
```

Because the certificate is intentionally self-signed, `curl -k` is used only for this lab verification.

### nginx

The HTTP server block should only redirect:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name portal.fav.it;
    return 301 https://$host$request_uri;
}
```

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Apache

A port-80 virtual host can use:

```apache
<VirtualHost *:80>
    ServerName portal.fav.it
    Redirect permanent / https://portal.fav.it/
</VirtualHost>
```

Validate and reload:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Verify redirect behavior:

```bash
curl -sSI http://portal.fav.it/
curl -kIL http://portal.fav.it/
curl -sSI http://portal.fav.it/example/path
```

Expected logic:

```text
HTTP/80 → 301/308 → HTTPS/443 → portal response
```

## 14. Verify the certificate was not replaced

After web changes:

```bash
sudo openssl x509 \
  -in /path/to/existing-certificate.crt \
  -noout -fingerprint -sha256 \
  | tee ~/hardening-evidence/after/certificate-fingerprint.txt

diff \
  ~/hardening-evidence/before/certificate-fingerprint.txt \
  ~/hardening-evidence/after/certificate-fingerprint.txt
```

No `diff` output means the fingerprint is unchanged.

## 15. Prepare the dedicated `webmaster` SFTP account

Check whether it exists:

```bash
getent passwd webmaster
id webmaster
```

If absent:

```bash
sudo useradd -m -s /usr/sbin/nologin webmaster
```

Each developer should ideally use an individual SSH key rather than share a private key.

On the developer workstation:

```bash
ssh-keygen -t ed25519
ssh-copy-id webmaster@portal.fav.it
```

Check key permissions:

```bash
sudo stat /home/webmaster \
  /home/webmaster/.ssh \
  /home/webmaster/.ssh/authorized_keys

sudo chown -R webmaster:webmaster /home/webmaster/.ssh
sudo chmod 0700 /home/webmaster/.ssh
sudo chmod 0600 /home/webmaster/.ssh/authorized_keys
```

## 16. Restrict `webmaster` to SFTP only

The global SSH allowlist must include both required identities:

```text
AllowUsers sysadmin webmaster
```

Use a `Match` block for `webmaster`:

```text
Match User webmaster
    ForceCommand internal-sftp
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    AllowAgentForwarding no
    GatewayPorts no
    PasswordAuthentication no
```

Validate:

```bash
sudo sshd -t
```

Reload only if validation succeeds:

```bash
sudo systemctl reload ssh
```

Positive SFTP test:

```bash
sftp webmaster@portal.fav.it
```

Negative shell test:

```bash
ssh webmaster@portal.fav.it
ssh webmaster@portal.fav.it id
```

The account should not receive a normal shell.

Negative password-only SFTP test:

```bash
sftp -o PubkeyAuthentication=no webmaster@portal.fav.it
```

## 17. Optional stronger SFTP confinement with chroot

First determine the **actual** portal path. Suppose only as an example it is `/var/www/portal`.

The chroot root itself must not be writable by `webmaster`:

```bash
sudo chown root:root /var/www
sudo chmod 0755 /var/www
```

Example Match block:

```text
Match User webmaster
    ChrootDirectory /var/www
    ForceCommand internal-sftp -d /portal
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    AllowAgentForwarding no
    PasswordAuthentication no
```

Validate and reload:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Do not use this exact path unless it matches the discovered DocumentRoot layout.

## 18. Create a controlled portal-content permission model

Create a dedicated group:

```bash
sudo groupadd -f webcontent
sudo usermod -aG webcontent webmaster
sudo usermod -aG webcontent www-data
```

Verify:

```bash
id webmaster
id www-data
```

Set the real portal root first, for example:

```bash
PORTAL_ROOT=/var/www/portal
```

Then:

```bash
sudo chown -R webmaster:webcontent "$PORTAL_ROOT"
sudo find "$PORTAL_ROOT" -type d -exec chmod 2750 {} +
sudo find "$PORTAL_ROOT" -type f -exec chmod 0640 {} +
```

If the application needs writable runtime directories, grant write access only to those specific directories rather than to the whole application tree.

If `www-data` was newly added to a group, restart the active web server so new worker processes receive the new supplementary group:

```bash
sudo systemctl restart nginx
```

or:

```bash
sudo systemctl restart apache2
```

Verify the portal again:

```bash
curl -kI https://portal.fav.it/
```

## 19. Harden the global SSH policy

Inspect current values:

```bash
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication'
```

Inspect drop-ins:

```bash
ls -la /etc/ssh/sshd_config.d/
```

A suitable global policy is:

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
AllowUsers sysadmin webmaster
```

Remember that the `webmaster` Match block further restricts that account.

Validate:

```bash
sudo sshd -t
sudo sshd -T
```

Reload:

```bash
sudo systemctl reload ssh
```

Test required and prohibited behavior:

```bash
ssh sysadmin@portal.fav.it
ssh root@portal.fav.it
ssh -o PubkeyAuthentication=no sysadmin@portal.fav.it
sftp webmaster@portal.fav.it
ssh webmaster@portal.fav.it
```

Expected:

- `sysadmin` key-based SSH works;
- root SSH fails;
- password-only `sysadmin` authentication fails;
- `webmaster` SFTP works;
- `webmaster` normal shell does not.

For user-specific effective SSH configuration:

```bash
sudo sshd -T \
  -C user=webmaster,host=portal.fav.it,addr=<DEVELOPER_IP> \
  | grep -E \
  'forcecommand|chrootdirectory|passwordauthentication|pubkeyauthentication|x11forwarding|allowtcpforwarding|permittty'
```

And for `sysadmin`:

```bash
sudo sshd -T \
  -C user=sysadmin,host=portal.fav.it,addr=<ADMIN_IP>
```

## 20. Remove obsolete file-transfer services

Check sockets:

```bash
sudo ss -lntup | grep -E ':(20|21|69)\b'
```

Check packages:

```bash
dpkg -l | grep -Ei 'vsftpd|proftpd|pure-ftpd|tftpd'
```

Check running services:

```bash
systemctl --type=service --state=running \
  | grep -Ei 'vsftpd|proftpd|pure-ftpd|tftp'
```

If `vsftpd` is actually present and unnecessary:

```bash
sudo systemctl disable --now vsftpd
sudo apt purge vsftpd
```

Remove only services/packages you actually discover and confirm are unnecessary.

Verify:

```bash
sudo ss -lntup | grep -E ':(20|21|69)\b'
```

SFTP remains on TCP/22 through OpenSSH.

## 21. Audit other unnecessary services and persistence mechanisms

```bash
ss -lntup
systemctl --type=service --state=running
systemctl list-unit-files --state=enabled
systemctl list-timers --all
sudo crontab -l
sudo ls -la /etc/cron.d /etc/cron.daily /etc/cron.hourly /etc/cron.weekly /etc/cron.monthly
sudo find /etc/systemd/system -type f -ls
sudo grep -Rni 'ExecStart\|ExecStartPre\|ExecStartPost' /etc/systemd/system
```

For every component ask whether it is required by the portal role. Disable/remove only after establishing that it is unnecessary.

## 22. Audit dangerous filesystem permissions and privilege mechanisms

World-writable directories:

```bash
sudo find / -xdev -type d -perm -0002 2>/dev/null
```

World-writable files:

```bash
sudo find / -xdev -type f -perm -0002 2>/dev/null
```

Do not blindly change `/tmp` or `/var/tmp`; they are expected to be writable and normally protected by the sticky bit.

SUID/SGID files:

```bash
sudo find / \
  -xdev -type f \
  \( -perm -4000 -o -perm -2000 \) \
  -ls 2>/dev/null
```

Linux capabilities:

```bash
sudo getcap -r / 2>/dev/null
```

Investigate unexpected/custom entries instead of stripping privileges indiscriminately.

## 23. Search for obvious exposed secrets

```bash
sudo grep -RniE \
  'password|token|secret|key' \
  /home /opt /var/www \
  2>/dev/null
```

This is a rough educational check and can produce false positives. If a real credential has been exposed, remove the exposed copy **and rotate/revoke the credential**.

## 24. Configure nftables for the actual role

The final externally required TCP ports are:

```text
22  SSH + SFTP
80  HTTP redirect only
443 HTTPS portal
```

A simple baseline ruleset is:

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
        tcp dport 80 accept
        tcp dport 443 accept
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

Where topology permits, restrict TCP/22 by source. Remember that both administrators **and developers** need TCP/22:

```nft
ip saddr <ADMIN_SUBNET> tcp dport 22 accept
ip saddr <DEVELOPER_SUBNET> tcp dport 22 accept
```

Validate before applying:

```bash
sudo nft -c -f /etc/nftables.conf
```

Keep the old SSH session open, then apply:

```bash
sudo nft -f /etc/nftables.conf
```

Immediately test a new admin session and SFTP session:

```bash
ssh sysadmin@portal.fav.it
sftp webmaster@portal.fav.it
```

Then enable persistence:

```bash
sudo systemctl enable --now nftables
```

Verify:

```bash
sudo nft list ruleset
systemctl status nftables
```

## 25. AppArmor

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

Move a tested relevant profile to enforce mode only when appropriate:

```bash
sudo aa-enforce /etc/apparmor.d/<profile>
```

After policy changes:

```bash
systemctl status <service>
journalctl -u <service>
```

## 26. Selected sysctl hardening

Inspect:

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

For a simple single-homed non-routing server, `/etc/sysctl.d/99-hardening.conf` can contain:

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

Do not blindly use strict `rp_filter=1` on multihomed, VPN, asymmetric-routing or policy-routing systems.

## 27. Persistent logging

Inspect:

```bash
journalctl --list-boots
systemd-analyze cat-config systemd/journald.conf
```

Configure persistent storage with a drop-in such as:

```ini
[Journal]
Storage=persistent
Compress=yes
```

Create storage and restart journald:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

Verify:

```bash
journalctl --disk-usage
journalctl --list-boots
journalctl -u ssh
journalctl -u nftables
journalctl -u apparmor
journalctl -u nginx
journalctl -u apache2
```

Check only the relevant web-server unit.

## 28. Automatic updates

```bash
dpkg -l unattended-upgrades
systemctl status apt-daily.timer
systemctl status apt-daily-upgrade.timer
```

Install if missing:

```bash
sudo apt install unattended-upgrades
```

Enable timers:

```bash
sudo systemctl enable --now apt-daily.timer
sudo systemctl enable --now apt-daily-upgrade.timer
```

Verify:

```bash
systemctl list-timers | grep apt
sudo unattended-upgrade --dry-run --debug
```

Review `/etc/apt/apt.conf.d/50unattended-upgrades` as well as `20auto-upgrades` so you know what origins/packages the unattended policy actually covers.

## 29. Final functional and security verification

Hostname:

```bash
hostname -f
```

Listeners:

```bash
sudo ss -lntup
```

SSH syntax:

```bash
sudo sshd -t
```

Firewall:

```bash
sudo nft list ruleset
```

HTTP redirect:

```bash
curl -sSI http://portal.fav.it/
```

HTTPS portal:

```bash
curl -kI https://portal.fav.it/
```

Admin SSH:

```bash
ssh sysadmin@portal.fav.it
```

SFTP:

```bash
sftp webmaster@portal.fav.it
```

Negative tests:

```bash
ssh root@portal.fav.it
ssh -o PubkeyAuthentication=no sysadmin@portal.fav.it
ssh webmaster@portal.fav.it
sftp -o PubkeyAuthentication=no webmaster@portal.fav.it
```

External scan:

```bash
sudo nmap -sS -sV -p- portal.fav.it -oN nmap-after.txt
```

Expected final externally required TCP exposure:

```text
22/tcp   SSH/SFTP
80/tcp   HTTP redirect only
443/tcp  HTTPS
```

Any additional listener needs an explicit role justification.

## 30. Capture the after-state

```bash
ss -lntup | tee ~/hardening-evidence/after/listening-sockets.txt
systemctl --type=service --state=running \
  | tee ~/hardening-evidence/after/running-services.txt
sudo nft list ruleset \
  | tee ~/hardening-evidence/after/nftables.txt
sudo sshd -T \
  | tee ~/hardening-evidence/after/sshd-effective.txt
sudo aa-status 2>&1 \
  | tee ~/hardening-evidence/after/apparmor.txt
```

Use the [evidence template](evidence-template.md) for each finding so the final report demonstrates:

```text
CHECK + OUTPUT
      ↓
WHY IT IS A RISK / WHY IT IS UNNECESSARY
      ↓
REMEDIATION
      ↓
VERIFICATION
      ↓
REQUIRED SERVICE STILL WORKS
```
