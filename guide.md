---
layout: default
title: Complete Hardening Procedure
---

# Complete Debian 13 Hardening Procedure for `portal.fav.it`

This is the main operational procedure for the final assignment. Follow it in order. Commands that only inspect the system come before commands that modify it.

> **Rule for placeholders:** never guess a path, IP address, subnet, account, or service name. Discover the real value first, record it, and only then substitute it into a remediation command.

## 1. Preserve recoverability and start collecting evidence

Before touching SSH, firewall rules, authentication, filesystem ownership or kernel parameters:

- create a VM snapshot if available;
- keep the current administrative SSH session open;
- keep console access available if possible.

Create evidence directories:

```bash
mkdir -p ~/hardening-evidence/{before,after}
script -a ~/hardening-evidence/hardening-session.log
```

`script` records the terminal session. Exit the recording at the end with:

```bash
exit
```

## 2. Confirm system identity and basic networking

```bash
cat /etc/os-release
uname -a
hostnamectl
hostname
hostname -f
ip -br -4 addr
ip route
getent ahostsv4 portal.fav.it
```

The intended FQDN is:

```text
portal.fav.it
```

If the configured hostname is wrong:

```bash
sudo hostnamectl set-hostname portal.fav.it
```

Inspect local name resolution before modifying it:

```bash
cat /etc/hosts
```

Do not assume that an address returned for `portal.fav.it` is necessarily the local interface address. Compare DNS/name-resolution output with `ip -br -4 addr`; NAT, a reverse proxy, or a load balancer can make them differ.

## 3. Capture the original baseline before remediation

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

The local `ss` view shows listening sockets. The external Nmap view shows what another host can actually reach.

## 4. Discover and record the real environment values

This step supplies the values used later in the guide.

### 4.1 Server IP, interface, gateway and locally connected subnet

```bash
ip -br -4 addr
ip -4 route
```

Typical output might resemble:

```text
ens33            UP             10.10.10.30/24

default via 10.10.10.1 dev ens33
10.10.10.0/24 dev ens33 proto kernel scope link src 10.10.10.30
```

From that example only:

```text
SERVER_IP     = 10.10.10.30
SERVER_SUBNET = 10.10.10.0/24
GATEWAY       = 10.10.10.1
INTERFACE     = ens33
```

Use your actual output, not these example values.

Verify the portal name separately:

```bash
getent ahostsv4 portal.fav.it
```

### 4.2 Current administrator client IP

If you are connected over SSH:

```bash
printf '%s\n' "$SSH_CONNECTION"
who
w
```

`SSH_CONNECTION` normally contains:

```text
client_IP client_source_port server_IP server_port
```

Extract the current SSH client address for convenience:

```bash
ADMIN_IP=$(printf '%s\n' "$SSH_CONNECTION" | awk '{print $1}')
printf 'ADMIN_IP=%s\n' "$ADMIN_IP"
```

If you are working from the local console, `SSH_CONNECTION` can be empty; do not treat an empty value as an error.

### 4.3 Administrator subnet

Do **not** infer a subnet such as `/24` merely from one administrator IP. Inspect routing information:

```bash
ip -4 route show table all
```

If `ADMIN_IP` is known, inspect the route the server would use to reach it:

```bash
ip route get "$ADMIN_IP"
```

An explicit route such as:

```text
192.168.50.0/24 via 10.10.10.1 dev ens33
```

supports using `192.168.50.0/24` as that routed network. If the routing table only shows a default route, obtain the administrator subnet from the lab/network design, router, DHCP configuration, or other authoritative network documentation instead of inventing it.

### 4.4 Active web server and current web ports

```bash
systemctl --type=service --state=running \
  | grep -E 'apache2|nginx'

sudo ss -lntp | grep -E ':(80|443)\b'
ps -ef | grep -E '[n]ginx|[a]pache2'
```

Follow only the nginx or Apache branch that actually applies.

### 4.5 Portal DocumentRoot and TLS certificate/key paths

For nginx:

```bash
sudo nginx -T 2>&1 \
  | grep -nE 'server_name|root |listen .*80|listen .*443|ssl_certificate|ssl_certificate_key'
```

For Apache:

```bash
sudo apache2ctl -S
sudo grep -RniE \
  'DocumentRoot|VirtualHost|SSLEngine|SSLCertificateFile|SSLCertificateKeyFile' \
  /etc/apache2/sites-enabled /etc/apache2/sites-available
```

From the applicable output identify the real values. Then set shell variables for this session, replacing the examples below:

```bash
PORTAL_ROOT='/actual/portal/document-root'
TLS_CERT='/actual/path/to/certificate.crt'
TLS_KEY='/actual/path/to/private-key.key'
```

Confirm that the paths really exist before using them:

```bash
sudo stat "$PORTAL_ROOT"
sudo stat "$TLS_CERT"
sudo stat "$TLS_KEY"
```

Record them:

```bash
printf 'PORTAL_ROOT=%s\nTLS_CERT=%s\nTLS_KEY=%s\n' \
  "$PORTAL_ROOT" "$TLS_CERT" "$TLS_KEY" \
  | tee ~/hardening-evidence/discovered-paths.txt
```

The variables exist only in the current shell session. If you reconnect, set them again from the recorded values.

### 4.6 Web-server worker account

For nginx:

```bash
ps -eo user,pid,cmd | grep '[n]ginx'
sudo nginx -T 2>&1 | grep -E '^\s*user\s'
```

For Apache:

```bash
ps -eo user,pid,cmd | grep '[a]pache2'
grep -E '^APACHE_RUN_USER=' /etc/apache2/envvars
```

The worker user is commonly `www-data` on Debian, but verify it. Then set the real value:

```bash
WEB_USER='actual-web-worker-user'
id "$WEB_USER"
```

### 4.7 Existing `webmaster` account and home directory

```bash
getent passwd webmaster
id webmaster 2>/dev/null || true
```

If `webmaster` already exists, discover its home directory:

```bash
WEBMASTER_HOME=$(getent passwd webmaster | cut -d: -f6)
printf 'WEBMASTER_HOME=%s\n' "$WEBMASTER_HOME"
```

If the account does not yet exist, create it later in Step 16 and then run these commands again.

## 5. Investigate artifacts from the previous administrator

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

## 6. Audit accounts and SSH authorization keys

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
```

If and only if it is a member of `sudo`:

```bash
sudo gpasswd -d asdrubale sudo
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

## 7. Change the known `sysadmin` password

The delivered password must be considered compromised.

```bash
sudo passwd sysadmin
sudo passwd -S sysadmin
sudo chage -l sysadmin
```

Do not put the new password directly in shell history.

## 8. Keep root local-console recovery, but block root over SSH

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

## 9. Patch the server

```bash
apt list --upgradable
sudo apt update
sudo apt full-upgrade
```

Reboot if required:

```bash
sudo reboot
```

After reconnecting, remember to restore any shell variables you need, for example:

```bash
PORTAL_ROOT='/actual/portal/document-root'
TLS_CERT='/actual/path/to/certificate.crt'
TLS_KEY='/actual/path/to/private-key.key'
WEB_USER='actual-web-worker-user'
```

Then verify updates again:

```bash
apt list --upgradable
```

## 10. Audit sudo privileges

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

## 11. Prepare administrative SSH key access before disabling passwords

On the administrator workstation:

```bash
ssh-keygen -t ed25519
ssh-copy-id sysadmin@portal.fav.it
ssh sysadmin@portal.fav.it
```

Do not disable password authentication until this succeeds in a **new** session.

On the server, confirm the current administrator source address again if needed:

```bash
printf '%s\n' "$SSH_CONNECTION"
```

## 12. Reconfirm the active web-server configuration

The read-only discovery commands from Step 4 can be repeated immediately before editing anything.

For nginx:

```bash
sudo nginx -T 2>&1 \
  | grep -nE 'server_name|root |listen .*80|listen .*443|ssl_certificate|ssl_certificate_key'
```

For Apache:

```bash
sudo apache2ctl -S
sudo grep -RniE \
  'DocumentRoot|VirtualHost|SSLEngine|SSLCertificateFile|SSLCertificateKeyFile' \
  /etc/apache2/sites-enabled /etc/apache2/sites-available
```

Before continuing, verify your variables still point to the discovered files:

```bash
printf 'PORTAL_ROOT=%s\nTLS_CERT=%s\nTLS_KEY=%s\nWEB_USER=%s\n' \
  "$PORTAL_ROOT" "$TLS_CERT" "$TLS_KEY" "$WEB_USER"

sudo stat "$PORTAL_ROOT" "$TLS_CERT" "$TLS_KEY"
id "$WEB_USER"
```

## 13. Preserve and inspect the existing self-signed certificate

Use the discovered certificate path:

```bash
sudo openssl x509 \
  -in "$TLS_CERT" \
  -noout -subject -issuer -serial -dates -fingerprint -sha256
```

Save its fingerprint before web-server changes:

```bash
sudo openssl x509 \
  -in "$TLS_CERT" \
  -noout -fingerprint -sha256 \
  | tee ~/hardening-evidence/before/certificate-fingerprint.txt
```

Inspect what the live HTTPS service presents:

```bash
openssl s_client \
  -connect portal.fav.it:443 \
  -servername portal.fav.it \
  </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -fingerprint -sha256
```

Do not generate a replacement certificate for this exercise.

## 14. Enforce HTTP-to-HTTPS redirection

Test current behavior:

```bash
curl -I http://portal.fav.it/
curl -kI https://portal.fav.it/
```

Because the certificate is intentionally self-signed, `curl -k` is used only for this lab verification.

### nginx branch

Locate the active server block from `nginx -T` before editing its source file. The port-80 server block should only redirect:

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

### Apache branch

Use `apache2ctl -S` to identify the actual port-80 virtual-host configuration file. A port-80 virtual host can use:

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

### Verify the redirect

```bash
curl -sSI http://portal.fav.it/
curl -kIL http://portal.fav.it/
curl -sSI http://portal.fav.it/example/path
```

Expected logic:

```text
HTTP/80 → 301/308 → HTTPS/443 → portal response
```

## 15. Verify the certificate was not replaced

```bash
sudo openssl x509 \
  -in "$TLS_CERT" \
  -noout -fingerprint -sha256 \
  | tee ~/hardening-evidence/after/certificate-fingerprint.txt

diff \
  ~/hardening-evidence/before/certificate-fingerprint.txt \
  ~/hardening-evidence/after/certificate-fingerprint.txt
```

No `diff` output means the saved certificate fingerprint is unchanged.

## 16. Prepare the dedicated `webmaster` SFTP account

Check whether it exists:

```bash
getent passwd webmaster
id webmaster 2>/dev/null || true
```

If absent:

```bash
sudo useradd -m -s /usr/sbin/nologin webmaster
```

Now discover and store the real home directory:

```bash
WEBMASTER_HOME=$(getent passwd webmaster | cut -d: -f6)
printf 'WEBMASTER_HOME=%s\n' "$WEBMASTER_HOME"
sudo stat "$WEBMASTER_HOME"
```

Each developer should ideally use an individual SSH key rather than share a private key.

On the developer workstation:

```bash
ssh-keygen -t ed25519
ssh-copy-id webmaster@portal.fav.it
```

Back on the server, check key paths and permissions using the discovered home directory:

```bash
sudo stat "$WEBMASTER_HOME" \
  "$WEBMASTER_HOME/.ssh" \
  "$WEBMASTER_HOME/.ssh/authorized_keys"

sudo chown -R webmaster:webmaster "$WEBMASTER_HOME/.ssh"
sudo chmod 0700 "$WEBMASTER_HOME/.ssh"
sudo chmod 0600 "$WEBMASTER_HOME/.ssh/authorized_keys"
```

## 17. Restrict `webmaster` to SFTP only

First inspect where SSH settings are currently defined:

```bash
grep -n '^Include' /etc/ssh/sshd_config
ls -la /etc/ssh/sshd_config.d/

sudo grep -RniE \
  'PermitRootLogin|PasswordAuthentication|AllowUsers|Match|ForceCommand|ChrootDirectory' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d \
  2>/dev/null
```

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

Positive SFTP test from a developer workstation:

```bash
sftp webmaster@portal.fav.it
```

Negative shell tests:

```bash
ssh webmaster@portal.fav.it
ssh webmaster@portal.fav.it id
```

The account should not receive a normal shell.

Negative password-only SFTP test:

```bash
sftp -o PubkeyAuthentication=no webmaster@portal.fav.it
```

## 18. Discover the developer client IP and network before source-restricting SSH/SFTP

After a developer makes a successful SFTP connection, inspect the SSH journal:

```bash
sudo journalctl -u ssh --since '15 minutes ago' \
  | grep -i webmaster
```

While the connection is active, you can also inspect TCP/22 sessions:

```bash
sudo ss -tnp | grep ':22'
```

A log entry such as:

```text
Accepted publickey for webmaster from 192.168.60.44 port 50120
```

identifies the client as `192.168.60.44`. Record your real value:

```bash
DEVELOPER_IP='actual-developer-client-ip'
printf 'DEVELOPER_IP=%s\n' "$DEVELOPER_IP"
```

Inspect the route toward that client:

```bash
ip route get "$DEVELOPER_IP"
ip -4 route show table all
```

As with the administrator network, do not infer a CIDR prefix from a single client IP. Use an explicit route or authoritative network configuration to determine the real developer subnet.

## 19. Optional stronger SFTP confinement with chroot

Use the discovered `PORTAL_ROOT`; do not assume `/var/www/portal`.

Inspect the path hierarchy first:

```bash
namei -l "$PORTAL_ROOT"
```

A convenient candidate parent can be displayed with:

```bash
CHROOT_ROOT=$(dirname "$PORTAL_ROOT")
SFTP_START="/$(basename "$PORTAL_ROOT")"
printf 'CHROOT_ROOT=%s\nSFTP_START=%s\n' "$CHROOT_ROOT" "$SFTP_START"
```

Do **not** use `CHROOT_ROOT` automatically. An OpenSSH `ChrootDirectory` and all path components leading to it must satisfy OpenSSH ownership/permission requirements; in particular, the chroot root must not be writable by `webmaster`.

Inspect before changing anything:

```bash
namei -l "$CHROOT_ROOT"
sudo stat "$CHROOT_ROOT"
```

If the chosen chroot root is correct for the actual layout, the corresponding SSH configuration would use the literal discovered paths, for example:

```text
Match User webmaster
    ChrootDirectory /actual/chroot/root
    ForceCommand internal-sftp -d /actual-sftp-start-directory
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

Retest SFTP immediately.

## 20. Create a controlled portal-content permission model

Create a dedicated group:

```bash
sudo groupadd -f webcontent
sudo usermod -aG webcontent webmaster
sudo usermod -aG webcontent "$WEB_USER"
```

Verify:

```bash
id webmaster
id "$WEB_USER"
sudo stat "$PORTAL_ROOT"
```

Apply permissions to the **discovered** portal root:

```bash
sudo chown -R webmaster:webcontent "$PORTAL_ROOT"
sudo find "$PORTAL_ROOT" -type d -exec chmod 2750 {} +
sudo find "$PORTAL_ROOT" -type f -exec chmod 0640 {} +
```

If the application needs writable runtime directories, grant write access only to those specific directories rather than to the whole application tree.

If the web-server worker was newly added to `webcontent`, restart only the active web server so new workers receive the supplementary group:

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

## 21. Harden the global SSH policy

Inspect current values:

```bash
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|allowusers'
```

Inspect drop-ins and existing definitions:

```bash
ls -la /etc/ssh/sshd_config.d/

sudo grep -RniE \
  'PermitRootLogin|PasswordAuthentication|KbdInteractiveAuthentication|PubkeyAuthentication|AllowUsers|Match' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d \
  2>/dev/null
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

Remember that OpenSSH option precedence and `Match` blocks matter; verify effective values instead of assuming that a later-looking filename overrides another setting.

Validate syntax:

```bash
sudo sshd -t
```

Inspect effective global values:

```bash
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

For user-specific effective SSH configuration, use the discovered client IPs.

For `webmaster`:

```bash
sudo sshd -T \
  -C user=webmaster,host=portal.fav.it,addr="$DEVELOPER_IP" \
  | grep -E \
  'forcecommand|chrootdirectory|passwordauthentication|pubkeyauthentication|x11forwarding|allowtcpforwarding|permittty'
```

For `sysadmin`, first restore/discover `ADMIN_IP` if needed:

```bash
ADMIN_IP=$(printf '%s\n' "$SSH_CONNECTION" | awk '{print $1}')
printf 'ADMIN_IP=%s\n' "$ADMIN_IP"

sudo sshd -T \
  -C user=sysadmin,host=portal.fav.it,addr="$ADMIN_IP"
```

## 22. Remove obsolete file-transfer services

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

## 23. Audit other unnecessary services and persistence mechanisms

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

## 24. Audit dangerous filesystem permissions and privilege mechanisms

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

Linux file capabilities:

```bash
sudo getcap -r / 2>/dev/null
```

Investigate unexpected/custom entries instead of stripping privileges indiscriminately.

## 25. Search for obvious exposed secrets

```bash
sudo grep -RniE \
  'password|token|secret|key' \
  /home /opt "$PORTAL_ROOT" \
  2>/dev/null
```

This is a rough educational check and can produce false positives. If a real credential has been exposed, remove the exposed copy **and rotate/revoke the credential**.

## 26. Discover/confirm source networks before writing nftables rules

Display all relevant IPv4 routes again:

```bash
ip -4 route show table all
```

If the current admin and developer IP variables are set:

```bash
printf 'ADMIN_IP=%s\nDEVELOPER_IP=%s\n' "$ADMIN_IP" "$DEVELOPER_IP"
ip route get "$ADMIN_IP"
ip route get "$DEVELOPER_IP"
```

Determine `ADMIN_SUBNET` and `DEVELOPER_SUBNET` only from an explicit network route/configuration or authoritative network documentation.

Record the confirmed values manually, for example:

```bash
ADMIN_SUBNET='actual-admin-cidr'
DEVELOPER_SUBNET='actual-developer-cidr'
printf 'ADMIN_SUBNET=%s\nDEVELOPER_SUBNET=%s\n' \
  "$ADMIN_SUBNET" "$DEVELOPER_SUBNET" \
  | tee ~/hardening-evidence/discovered-networks.txt
```

Do not place the literal strings `actual-admin-cidr` or `actual-developer-cidr` into the firewall; replace them with confirmed CIDRs first.

## 27. Configure nftables for the actual role

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

Where the topology permits source restriction, replace the unrestricted TCP/22 rule with literal confirmed networks such as:

```nft
ip saddr <CONFIRMED_ADMIN_SUBNET> tcp dport 22 accept
ip saddr <CONFIRMED_DEVELOPER_SUBNET> tcp dport 22 accept
```

Remember that both administrators **and developers** need TCP/22 because SFTP is part of SSH.

Before applying any new firewall, inspect the interface and current SSH source one last time:

```bash
ip -br -4 addr
ip -4 route
printf '%s\n' "$SSH_CONNECTION"
```

Validate firewall syntax:

```bash
sudo nft -c -f /etc/nftables.conf
```

`nft -c` checks syntax; it does **not** prove that the policy will not lock you out.

Keep the old SSH session and console access available, then apply:

```bash
sudo nft -f /etc/nftables.conf
```

Immediately test a **new** administrative session and a new SFTP session:

```bash
ssh sysadmin@portal.fav.it
sftp webmaster@portal.fav.it
```

Only after both succeed, enable persistence:

```bash
sudo systemctl enable --now nftables
```

Verify:

```bash
sudo nft list ruleset
systemctl status nftables
```

## 28. AppArmor

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

Do not blindly enforce every available profile.

## 29. Selected sysctl hardening

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

## 30. Persistent logging

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
```

Check only the web-server unit that is actually active:

```bash
journalctl -u nginx
```

or:

```bash
journalctl -u apache2
```

## 31. Automatic updates

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

Review both policy files:

```bash
sudo cat /etc/apt/apt.conf.d/20auto-upgrades
sudo sed -n '1,240p' /etc/apt/apt.conf.d/50unattended-upgrades
```

This tells you not only whether timers run, but also what unattended-upgrade policy is actually configured.

## 32. Final functional and security verification

Hostname and name resolution:

```bash
hostname -f
getent ahostsv4 portal.fav.it
```

Interfaces/routes:

```bash
ip -br -4 addr
ip -4 route
```

Listeners:

```bash
sudo ss -lntup
```

SSH syntax and effective policy:

```bash
sudo sshd -t
sudo sshd -T | grep -E \
'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|allowusers'
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

Certificate fingerprint:

```bash
openssl s_client \
  -connect portal.fav.it:443 \
  -servername portal.fav.it \
  </dev/null 2>/dev/null \
  | openssl x509 -noout -fingerprint -sha256
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

From an authorized external assessment host:

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

## 33. Capture the final after-state

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

Compare selected before/after evidence:

```bash
diff -u \
  ~/hardening-evidence/before/listening-sockets.txt \
  ~/hardening-evidence/after/listening-sockets.txt

diff -u \
  ~/hardening-evidence/before/nftables.txt \
  ~/hardening-evidence/after/nftables.txt
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

## 34. Values you should have discovered by the end

Do not complete this table from assumptions; complete it from the commands above.

| Value | How it was discovered |
|---|---|
| `SERVER_IP` | `ip -br -4 addr` |
| `SERVER_SUBNET` | `ip -4 route` / interface prefix |
| `GATEWAY` | `ip -4 route` |
| `ADMIN_IP` | `SSH_CONNECTION`, `who`, `w` |
| `ADMIN_SUBNET` | explicit route or authoritative network configuration |
| `DEVELOPER_IP` | SSH journal / active TCP/22 session |
| `DEVELOPER_SUBNET` | explicit route or authoritative network configuration |
| `PORTAL_ROOT` | nginx/Apache active virtual-host configuration |
| `TLS_CERT` | nginx/Apache TLS configuration |
| `TLS_KEY` | nginx/Apache TLS configuration |
| `WEB_USER` | active worker processes/configuration |
| `WEBMASTER_HOME` | `getent passwd webmaster` |

The operating rule throughout the lab is:

```text
DISCOVER → RECORD → VALIDATE → MODIFY → VERIFY
```
