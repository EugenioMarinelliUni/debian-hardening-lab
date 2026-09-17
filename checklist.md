---
layout: default
title: Final Verification Checklist
---

# Final Verification Checklist — `portal.fav.it`

Use this checklist only after the remediation steps have been applied and validated.

## System identity

- [ ] Hostname/FQDN is `portal.fav.it`
- [ ] Debian 13 confirmed
- [ ] Expected IP addressing and routing confirmed
- [ ] `portal.fav.it` resolves to the intended address

Useful commands:

```bash
cat /etc/os-release
hostnamectl
hostname -f
ip -br addr
ip route
getent hosts portal.fav.it
```

## Evidence and rollback

- [ ] Initial snapshot/rollback point exists where applicable
- [ ] Existing administrative session kept open during SSH/firewall changes
- [ ] `~/hardening-evidence/before` captured
- [ ] `~/hardening-evidence/after` captured
- [ ] External `nmap-before.txt` captured
- [ ] External `nmap-after.txt` captured
- [ ] Each finding documents CHECK, significant OUTPUT, REMEDIATION and VERIFY

## Previous-administrator review

- [ ] Files/directories named after the previous administrator searched
- [ ] Configuration contents searched for the previous administrator's name
- [ ] Former-administrator account checked
- [ ] Former-administrator sudo rights checked
- [ ] SSH `authorized_keys` reviewed for unknown/old keys
- [ ] Custom systemd units reviewed
- [ ] Cron jobs reviewed
- [ ] Timers reviewed
- [ ] No suspicious artifact was deleted merely because of its name; each was evaluated first

## Accounts and credentials

- [ ] `sysadmin` password changed from the delivered known password
- [ ] `sysadmin` public-key SSH tested successfully before password auth was disabled
- [ ] Root retains a valid password for local-console access
- [ ] Root local-console login verified
- [ ] Root SSH login denied
- [ ] Obsolete accounts locked/expired and stripped of unnecessary group membership
- [ ] `webmaster` exists as the dedicated developer/SFTP account
- [ ] `webmaster` does not receive an ordinary interactive shell

## sudo

- [ ] `getent group sudo` reviewed
- [ ] `sudo -l` reviewed
- [ ] Unnecessary `NOPASSWD: ALL` rules removed
- [ ] `sudoers` edited only with `visudo`
- [ ] Remaining privileges are justified by role

## SSH administration

- [ ] `sudo sshd -t` succeeds
- [ ] `PermitRootLogin no`
- [ ] `PubkeyAuthentication yes`
- [ ] `PasswordAuthentication no` after key testing
- [ ] `KbdInteractiveAuthentication no`
- [ ] `PermitEmptyPasswords no`
- [ ] `MaxAuthTries` reduced appropriately
- [ ] `LoginGraceTime` reduced appropriately
- [ ] `AllowUsers` includes both `sysadmin` and `webmaster`
- [ ] `sysadmin` key-based SSH succeeds in a new session
- [ ] `ssh root@portal.fav.it` fails
- [ ] Password-only `sysadmin` authentication fails

## SFTP for `webmaster`

- [ ] `webmaster` authenticates by SSH key
- [ ] `ForceCommand internal-sftp` applied to `webmaster`
- [ ] `PermitTTY no` applied to `webmaster`
- [ ] X11 forwarding disabled for `webmaster`
- [ ] TCP forwarding disabled for `webmaster`
- [ ] Agent forwarding disabled for `webmaster`
- [ ] Password authentication disabled for `webmaster`
- [ ] `sftp webmaster@portal.fav.it` succeeds
- [ ] Upload/list/delete of a harmless test file succeeds
- [ ] `ssh webmaster@portal.fav.it` does not provide a normal shell
- [ ] Password-only SFTP test fails
- [ ] If chroot is used, chroot root ownership and permissions satisfy OpenSSH requirements

## Portal filesystem permissions

- [ ] Actual portal DocumentRoot identified before changing ownership
- [ ] Dedicated group such as `webcontent` created if used
- [ ] `webmaster` has only the write access required to maintain portal content
- [ ] Web-server account can read/traverse required portal content
- [ ] Web-server account is not given unnecessary write access to the entire application tree
- [ ] Directories and files use separate permission policies
- [ ] Required runtime-writable directories are handled explicitly
- [ ] Portal still functions after ownership/group changes

## HTTPS certificate

- [ ] Existing certificate path identified from the active web-server configuration
- [ ] Certificate subject/issuer/dates inspected
- [ ] SHA-256 fingerprint captured before changes
- [ ] Same certificate fingerprint captured after changes
- [ ] `diff` confirms the required self-signed certificate was not replaced
- [ ] Live service on TCP/443 presents the expected certificate

## HTTP / HTTPS behavior

- [ ] Port 443 serves the portal over HTTPS
- [ ] `curl -kI https://portal.fav.it/` returns a legitimate portal response
- [ ] Port 80 does not directly serve portal content
- [ ] `curl -sSI http://portal.fav.it/` returns 301/308 redirect
- [ ] Redirect `Location` points to `https://portal.fav.it/...`
- [ ] Redirect behavior verified for a non-root path as well
- [ ] nginx `nginx -t` or Apache `apache2ctl configtest` succeeds, depending on active web server
- [ ] Directory listing disabled unless explicitly required
- [ ] Backup/archive material removed from public document root or otherwise protected
- [ ] Unnecessary server-version disclosure reduced

## Obsolete file-transfer services

- [ ] FTP/TFTP listeners checked
- [ ] FTP/TFTP packages checked
- [ ] FTP/TFTP systemd services checked
- [ ] Unnecessary FTP/TFTP daemons disabled and removed
- [ ] SFTP remains functional on TCP/22

Useful checks:

```bash
sudo ss -lntup | grep -E ':(20|21|69)\b'
dpkg -l | grep -Ei 'vsftpd|proftpd|pure-ftpd|tftpd'
```

## General attack surface

- [ ] `ss -lntup` reviewed
- [ ] Running services reviewed
- [ ] Enabled services reviewed
- [ ] systemd timers reviewed
- [ ] Custom systemd services reviewed
- [ ] Cron jobs reviewed
- [ ] Every remaining service has an explicit role justification

## Filesystem and privilege review

- [ ] World-writable directories reviewed
- [ ] World-writable files reviewed
- [ ] `/tmp` and `/var/tmp` not blindly modified
- [ ] SUID/SGID executables reviewed
- [ ] Linux file capabilities reviewed
- [ ] Suspicious privileged scripts have safe ownership/modes
- [ ] Parent directories of privileged scripts reviewed where relevant
- [ ] Obvious plaintext secrets searched in authorized scope
- [ ] Any real leaked credential was rotated/revoked, not merely deleted from a file

## nftables

- [ ] nftables enabled
- [ ] Input default policy is drop
- [ ] Forward default policy is drop for this non-router role
- [ ] Loopback permitted
- [ ] Established/related traffic permitted
- [ ] Invalid tracked traffic dropped
- [ ] ICMP/ICMPv6 handled appropriately
- [ ] TCP/22 permitted for required administrator/developer source networks
- [ ] TCP/80 permitted for redirect service
- [ ] TCP/443 permitted for HTTPS portal
- [ ] No unnecessary application ports permitted
- [ ] `sudo nft -c -f /etc/nftables.conf` succeeds before load
- [ ] New SSH session tested after firewall application
- [ ] New SFTP session tested after firewall application

## AppArmor

- [ ] AppArmor active
- [ ] AppArmor enabled at boot
- [ ] Relevant profiles reviewed
- [ ] Relevant tested profiles in enforce mode where appropriate
- [ ] Services tested after policy changes
- [ ] AppArmor denials reviewed in the journal

## sysctl

- [ ] `ip_forward` disabled because the host is not a router
- [ ] unnecessary redirects disabled
- [ ] source routing disabled
- [ ] reverse-path policy chosen for actual topology rather than blindly copied
- [ ] TCP SYN cookies enabled
- [ ] kernel message disclosure restricted
- [ ] kernel pointer disclosure restricted
- [ ] hardlink/symlink protections enabled where appropriate
- [ ] persistent sysctl file loaded successfully

## Logging

- [ ] journald persistent storage enabled
- [ ] `/var/log/journal` exists
- [ ] previous boot visible after controlled reboot
- [ ] journal disk usage checked
- [ ] SSH events available
- [ ] firewall events/service state available
- [ ] AppArmor events available
- [ ] active web-server events available

## Updates

- [ ] Package index refreshed
- [ ] Installed packages upgraded appropriately
- [ ] `unattended-upgrades` installed/configured if required
- [ ] APT timers enabled
- [ ] `20auto-upgrades` reviewed
- [ ] `50unattended-upgrades` reviewed
- [ ] unattended-upgrades dry run completed successfully

## Final external state

Run from an authorized assessment host:

```bash
sudo nmap -sS -sV -p- portal.fav.it -oN nmap-after.txt
```

Expected required TCP exposure:

```text
22/tcp   SSH / SFTP
80/tcp   HTTP redirect only
443/tcp  HTTPS portal
```

- [ ] Any additional exposed port has a documented business/technical justification
- [ ] HTTP redirect checked independently with `curl`
- [ ] HTTPS response checked independently with `curl`
- [ ] Required SFTP functionality tested independently
- [ ] Required administrative SSH functionality tested independently
- [ ] Before/after evidence compared and included in the final report
