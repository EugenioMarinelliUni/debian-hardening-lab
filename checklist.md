---
layout: default
title: Final Verification Checklist
---

# Final Verification Checklist

Use this page after remediation to confirm that the intended controls are actually in effect.

## System and baseline

- [ ] Correct Debian host identified
- [ ] Network addresses and routes documented
- [ ] `assessment-before` saved
- [ ] Initial external Nmap scan saved
- [ ] Rollback/snapshot available

## Patch management

- [ ] `sudo apt update` completed
- [ ] `sudo apt full-upgrade` completed
- [ ] Reboot performed if required
- [ ] `apt list --upgradable` reviewed afterward

## Accounts and sudo

- [ ] Interactive accounts reviewed
- [ ] Obsolete accounts locked/expired or removed according to policy
- [ ] Obsolete accounts no longer have an interactive shell
- [ ] Privileged group memberships reviewed
- [ ] `sudo -l` reviewed
- [ ] Unnecessary `NOPASSWD: ALL` rules removed
- [ ] Sudoers changes made with `visudo`

## SSH

- [ ] Public-key login tested before disabling passwords
- [ ] Current administrative session kept open during changes
- [ ] `PermitRootLogin no`
- [ ] `PubkeyAuthentication yes`
- [ ] `PasswordAuthentication no`
- [ ] `KbdInteractiveAuthentication no`
- [ ] `PermitEmptyPasswords no`
- [ ] `X11Forwarding no` if unused
- [ ] `AllowTcpForwarding no` if unused
- [ ] `GatewayPorts no`
- [ ] `MaxAuthTries` reduced appropriately
- [ ] `AllowUsers` or equivalent access restriction configured where appropriate
- [ ] `sudo sshd -t` succeeds
- [ ] `sudo sshd -T` confirms intended effective values
- [ ] Normal administrative key login succeeds
- [ ] Root SSH login fails
- [ ] Password-only SSH login fails

## Service minimization

- [ ] `ss -lntup` reviewed
- [ ] Running services reviewed
- [ ] Installed service units reviewed
- [ ] FTP removed if unnecessary
- [ ] rpcbind removed if unnecessary
- [ ] Web server removed if unnecessary
- [ ] Only services justified by the server role remain

## Web server

- [ ] nginx/Apache retained only if required
- [ ] Directory indexes disabled unless explicitly required
- [ ] Version/banner disclosure reduced where practical
- [ ] Backups/archives removed from document root
- [ ] Configuration validation succeeds
- [ ] Sensitive paths return `403`/`404` rather than directory listings
- [ ] HTTPS/TLS considered for any real application carrying credentials or sensitive data

## Samba file server

- [ ] `testparm -s` succeeds
- [ ] Guest access disabled
- [ ] Dedicated `fileshare` group exists
- [ ] Authorized users are group members
- [ ] Samba credentials exist only for required accounts
- [ ] `valid users = @fileshare` configured
- [ ] Share directory is not world-writable
- [ ] Directories use appropriate group/traverse permissions
- [ ] Files use appropriate group read/write permissions
- [ ] Authenticated SMB access succeeds
- [ ] Anonymous/guest SMB access fails
- [ ] TCP/445 restricted to the authorized LAN where appropriate
- [ ] TCP/139 is not exposed unless compatibility requires it

## Filesystem and privileged scripts

- [ ] World-writable directories reviewed rather than blindly changed
- [ ] World-writable regular files reviewed
- [ ] Shared directories have intentional owner/group assignments
- [ ] Directory and file modes are applied separately
- [ ] Privileged cron jobs reviewed
- [ ] Scripts executed by root are not modifiable by untrusted users
- [ ] Parent directories of privileged scripts reviewed (`namei -l`)
- [ ] Unnecessary privileged scheduled jobs removed
- [ ] Misplaced secrets removed or protected
- [ ] Exposed real credentials rotated/revoked, not merely deleted from disk

## nftables

- [ ] Current rules reviewed before replacement
- [ ] `input` default policy is `drop`
- [ ] `forward` default policy is `drop` for a non-router
- [ ] Loopback traffic allowed
- [ ] `established,related` allowed
- [ ] Invalid states dropped
- [ ] ICMP/ICMPv6 handled appropriately
- [ ] Only required listening services allowed
- [ ] SSH restricted to the management network where possible
- [ ] SMB restricted to the authorized LAN where applicable
- [ ] `sudo nft -c -f /etc/nftables.conf` succeeds before applying
- [ ] Existing SSH/console recovery path kept available while applying firewall rules
- [ ] New SSH session tested after firewall load
- [ ] nftables enabled for boot persistence

## AppArmor

- [ ] AppArmor service active
- [ ] AppArmor enabled at boot
- [ ] `aa-status` reviewed
- [ ] Relevant profiles loaded
- [ ] Relevant profiles in enforce mode where tested and appropriate
- [ ] Services function correctly after policy enforcement
- [ ] Denials/errors reviewed in the journal

## sysctl

- [ ] `ip_forward=0` for a host that is not a router
- [ ] ICMP redirect settings reviewed
- [ ] Source routing disabled where appropriate
- [ ] `rp_filter` chosen according to actual routing topology
- [ ] SYN cookies enabled
- [ ] `dmesg_restrict` enabled
- [ ] `kptr_restrict` set appropriately
- [ ] Protected hardlinks enabled
- [ ] Protected symlinks enabled
- [ ] Settings persisted under `/etc/sysctl.d/`
- [ ] `sudo sysctl --system` succeeds
- [ ] Runtime values verified after loading

## Logging

- [ ] journald configured for persistent storage
- [ ] `/var/log/journal` exists
- [ ] Journal disk usage reviewed
- [ ] Previous boot logs remain available after reboot
- [ ] SSH events are available
- [ ] Firewall/AppArmor events are available
- [ ] Samba/Apache/cron events are available where relevant

## Automatic updates

- [ ] `unattended-upgrades` installed if required by policy
- [ ] APT timers active
- [ ] `/etc/apt/apt.conf.d/20auto-upgrades` configured
- [ ] `/etc/apt/apt.conf.d/50unattended-upgrades` reviewed for actual policy
- [ ] `systemctl list-timers | grep apt` reviewed
- [ ] `unattended-upgrade --dry-run --debug` succeeds

## Final evidence

- [ ] `assessment-after` saved
- [ ] Final external Nmap scan saved
- [ ] Before/after listeners compared
- [ ] Before/after services compared
- [ ] Before/after firewall rules compared
- [ ] Before/after SSH effective configuration compared
- [ ] Before/after AppArmor state compared
- [ ] Before/after Samba configuration compared where applicable
- [ ] Final externally reachable services match the documented server role
