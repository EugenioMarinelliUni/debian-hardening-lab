---
layout: default
title: Command Reference
---

# Command Reference for `portal.fav.it`

This page explains the main commands used by the revised hardening procedure.

## Evidence and shell recording

### `mkdir -p ~/hardening-evidence/{before,after}`
Creates the evidence directory tree. `-p` creates missing parent directories and avoids failing if they already exist. Brace expansion creates both `before` and `after` subdirectories.

### `script -a FILE`
Records an interactive terminal session to a file. `-a` appends rather than overwriting an existing log.

### `tee FILE`
Copies standard input both to the terminal and to a file, useful when an assessment command must remain visible while also being captured as evidence.

## System identity and networking

### `cat /etc/os-release`
Prints Debian distribution/version metadata.

### `uname -a`
Displays kernel, architecture and related system information.

### `hostnamectl`
Shows systemd-managed hostname information and can change the persistent hostname.

### `hostname -f`
Attempts to display the fully qualified domain name (FQDN).

### `ip -br addr`
Shows interface/address information in compact form. `-br` means brief.

### `ip route`
Displays the kernel routing table and default gateway.

### `getent hosts portal.fav.it`
Queries the system name-service configuration to show how the hostname resolves.

## Processes, services and sockets

### `ss -lntup`
Shows listening network sockets.

- `-l`: listening
- `-n`: numeric addresses/ports
- `-t`: TCP
- `-u`: UDP
- `-p`: associated process

### `systemctl --type=service --state=running`
Lists running systemd services.

### `systemctl list-unit-files --state=enabled`
Shows service/unit definitions configured to be enabled at boot or through dependencies.

### `systemctl list-timers --all`
Lists systemd timers, including inactive ones, and their schedules.

### `systemctl disable --now SERVICE`
Stops a service now and disables normal automatic startup. It does not remove the package.

### `systemctl enable --now SERVICE`
Starts a unit now and enables its configured startup behavior.

### `systemctl reload SERVICE`
Asks a running service to reread its configuration where supported without a full stop/start.

### `systemctl restart SERVICE`
Stops and starts a service, creating new processes with updated environment/group membership.

## Searching suspicious artifacts

### `find PATHS -iname '*asdrubale*' -ls`
Searches names case-insensitively for files/directories whose names contain the previous administrator's name. `-ls` prints metadata.

### `grep -RniI PATTERN PATHS`
Recursively searches text files.

- `-R`: recurse
- `-n`: line numbers
- `-i`: case-insensitive
- `-I`: skip binary files

### `systemctl cat UNIT`
Shows the complete systemd unit definition and drop-ins used for a service.

### `stat PATH`
Displays detailed filesystem metadata including permissions, owner, timestamps and inode information.

### `less FILE`
Safely views a text file interactively without editing it.

## Users, accounts and groups

### `getent passwd`
Queries the configured user database.

### `awk -F: '$7 !~ /(nologin|false)$/ {print $1,$6,$7}' /etc/passwd`
Uses `:` as field separator and prints username, home directory and shell for accounts whose shell does not end in `nologin` or `false`.

### `id USER`
Shows UID, primary GID and supplementary groups for a user.

### `passwd -S USER`
Shows password-account status.

### `passwd -l USER`
Locks password authentication for an account. It does not necessarily disable every possible authentication mechanism by itself.

### `usermod -s /usr/sbin/nologin USER`
Changes the login shell to `nologin`, preventing ordinary interactive shell login.

### `usermod --expiredate 1 USER`
Sets an account-expiry date in the past, providing stronger account disablement for an obsolete account.

### `gpasswd -d USER GROUP`
Removes a user from a group, for example removing an obsolete account from `sudo`.

### `chage -l USER`
Displays password-age and account-expiry information.

### `useradd -m -s /usr/sbin/nologin webmaster`
Creates the dedicated SFTP account, creates its home directory (`-m`), and assigns `nologin` as its ordinary shell.

### `usermod -aG GROUP USER`
Adds a user to a supplementary group. `-a` is important because it appends rather than replacing all existing supplementary memberships.

## sudo

### `sudo -l`
Lists sudo permissions for the current user.

### `sudo -l -U USER`
Lists sudo privileges associated with another user when permitted.

### `visudo`
Edits sudo configuration with syntax validation. Prefer it to a normal text editor for `/etc/sudoers` and files in `/etc/sudoers.d`.

## SSH keys and authorized keys

### `ssh-keygen -t ed25519`
Creates an Ed25519 public/private SSH key pair.

### `ssh-copy-id USER@HOST`
Copies the current user's public key into the target account's SSH authorization file.

### `authorized_keys`
Usually `~/.ssh/authorized_keys`; contains public keys allowed to authenticate as that account. Unknown keys inherited from a former administrator are a significant finding and should be investigated.

### `nl -ba FILE`
Prints a file with line numbers, including blank lines (`-b a`), useful when documenting exactly which authorized key is present.

## SSH server validation and policy

### `sshd -t`
Checks SSH server configuration syntax. Run it before reloading the daemon.

### `sshd -T`
Prints effective OpenSSH server configuration.

### `sshd -T -C user=...,host=...,addr=...`
Evaluates effective settings for a specific connection context and is particularly useful for `Match User webmaster` rules.

### `PermitRootLogin no`
Disables root login over SSH while still allowing a valid root password to remain for local-console access.

### `AllowUsers sysadmin webmaster`
Restricts SSH authentication to the required administration and SFTP identities. Omitting `webmaster` would also block SFTP because SFTP runs through SSH.

### `ForceCommand internal-sftp`
Forces the matched account into OpenSSH's internal SFTP subsystem rather than a normal shell.

### `PermitTTY no`
Prevents allocation of an interactive terminal for the matched account.

### `AllowTcpForwarding no`, `AllowAgentForwarding no`, `X11Forwarding no`, `GatewayPorts no`
Disable SSH capabilities not required for a restricted SFTP account.

### `ChrootDirectory PATH`
Confines a matched SSH/SFTP session to a subtree. The chroot root must satisfy OpenSSH ownership/permission requirements and must not be writable by the restricted user.

## SFTP

### `sftp webmaster@portal.fav.it`
Starts an SFTP session over SSH, normally using TCP/22.

Useful interactive commands include:

```text
pwd
ls
put FILE
get FILE
rm FILE
bye
```

### `sftp -o PubkeyAuthentication=no webmaster@portal.fav.it`
Negative test that disables the client's use of public-key authentication. After password authentication is disabled, this should fail.

### `ssh webmaster@portal.fav.it`
Negative test: if `webmaster` is correctly constrained to SFTP, it should not receive a normal interactive shell.

## Filesystem ownership and permissions

### `chown -R OWNER:GROUP PATH`
Recursively changes owner/group. Use with care and only after identifying the correct application layout.

### `chmod 0700 DIRECTORY`
Owner has read/write/execute; group and others have no permissions.

### `chmod 0600 FILE`
Owner has read/write; group and others have no permissions.

### `chmod 2750 DIRECTORY`
Owner `rwx`, group `r-x`, others none, plus SGID so new objects tend to inherit the directory group.

### `chmod 0640 FILE`
Owner read/write, group read, others none.

### `find PATH -type d -exec chmod 2750 {} +`
Applies a directory-specific mode recursively without mistakenly removing traversal bits from directories.

### `find PATH -type f -exec chmod 0640 {} +`
Applies a file-specific mode separately.

### `namei -l PATH`
Displays ownership and permissions for every component in a path, useful when assessing privileged scripts or chroot paths.

## Web-server discovery

### `nginx -T`
Tests and dumps nginx's interpreted configuration. Useful for locating `server_name`, `root`, `listen`, and TLS certificate directives.

### `apache2ctl -S`
Shows Apache virtual-host interpretation and configuration sources.

### `grep -RniE 'DocumentRoot|VirtualHost|SSLEngine|SSLCertificate' /etc/apache2/...`
Locates Apache document roots, virtual hosts and TLS configuration.

## TLS / HTTPS

### `openssl x509 -in CERT -noout ...`
Parses a certificate file without modifying it. Useful options include:

- `-subject`: subject identity
- `-issuer`: issuing identity
- `-serial`: certificate serial number
- `-dates`: validity window
- `-fingerprint -sha256`: stable SHA-256 fingerprint for before/after comparison

### `openssl s_client -connect portal.fav.it:443 -servername portal.fav.it`
Opens a TLS connection and uses SNI for `portal.fav.it`. Piping its output to `openssl x509` shows the certificate the live server actually presents.

### `curl -I URL`
Requests response headers only.

### `curl -kI https://portal.fav.it/`
Tests HTTPS while ignoring certificate-chain trust errors. `-k` is appropriate here only because the exercise explicitly requires retaining the self-signed certificate.

### `curl -sSI http://portal.fav.it/`
Silently fetches HTTP response headers and is useful for proving that port 80 returns a redirect rather than portal content.

### `curl -kIL http://portal.fav.it/`
Follows redirects (`-L`) and allows the intentionally self-signed HTTPS endpoint (`-k`) so the complete HTTP→HTTPS flow can be demonstrated.

### `diff BEFORE AFTER`
Compares two text files. If the stored certificate fingerprint files are identical, `diff` normally prints nothing.

## nginx HTTP redirect

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name portal.fav.it;
    return 301 https://$host$request_uri;
}
```

This ensures port 80 redirects rather than serving application content.

### `nginx -t`
Validates nginx configuration syntax before reload.

## Apache HTTP redirect

```apache
<VirtualHost *:80>
    ServerName portal.fav.it
    Redirect permanent / https://portal.fav.it/
</VirtualHost>
```

### `apache2ctl configtest`
Validates Apache configuration before reload.

## Package management and obsolete protocols

### `apt update`
Refreshes local package metadata.

### `apt full-upgrade`
Upgrades installed packages while allowing necessary dependency transitions.

### `apt purge PACKAGE`
Removes a Debian package and package-managed configuration files. It does not guarantee removal of every data file the application may have created.

### `dpkg -l | grep -Ei 'vsftpd|proftpd|pure-ftpd|tftpd'`
Searches installed package metadata for common FTP/TFTP servers.

### `ss -lntup | grep -E ':(20|21|69)\b'`
Checks for listeners on common FTP/TFTP ports. Port 69 is normally UDP/TFTP.

## nftables

### `nft list ruleset`
Displays the active nftables ruleset.

### `nft -c -f /etc/nftables.conf`
Parses/checks the configuration without applying it.

### `nft -f /etc/nftables.conf`
Loads the configuration and immediately changes packet filtering.

For this portal, the required TCP services are normally:

```text
22  SSH + SFTP
80  HTTP redirect
443 HTTPS portal
```

If TCP/22 is source-restricted, both the administrator and developer source networks must be accounted for.

## External validation

### `nmap -sS -sV -p- portal.fav.it`
Performs an authorized TCP SYN scan of all TCP ports (`-p-`) and service/version detection (`-sV`). A SYN scan commonly needs elevated raw-packet privileges, hence `sudo` in the guide.

A focused final check can use:

```bash
sudo nmap -sS -sV -p22,80,443 portal.fav.it
```

## Privilege audits

### `find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls`
Lists SUID and SGID executables on the filesystem. These are not automatically vulnerabilities; investigate unexpected/custom entries.

### `getcap -r / 2>/dev/null`
Recursively displays Linux file capabilities. Capabilities can grant specific privileged operations without a full SUID-root executable.

## AppArmor

### `aa-status`
Displays AppArmor state, loaded profiles, enforcement/complain modes and confinement information.

### `aa-enforce PROFILE`
Moves a profile to enforcement mode. Do this only after confirming the profile is appropriate for the service.

## sysctl

### `sysctl NAME`
Reads a kernel runtime parameter.

### `sysctl --system`
Loads persistent sysctl configuration from system configuration locations such as `/etc/sysctl.d`.

Strict reverse-path filtering (`rp_filter=1`) can be inappropriate for multihomed, VPN, asymmetric-routing or policy-routing systems.

## Journald

### `journalctl --list-boots`
Lists boot sessions represented in the journal and helps demonstrate persistence across reboot.

### `journalctl -u UNIT`
Shows journal messages associated with a systemd unit.

### `journalctl --disk-usage`
Reports journal storage usage.

### `systemd-analyze cat-config systemd/journald.conf`
Shows the effective journald configuration including drop-ins.

## Automatic updates

### `dpkg -l unattended-upgrades`
Checks whether the package is installed.

### `systemctl list-timers | grep apt`
Shows APT-related systemd timers.

### `unattended-upgrade --dry-run --debug`
Simulates unattended-upgrade behavior while printing detailed decisions.
