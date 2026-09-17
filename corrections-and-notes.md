---
layout: default
title: Corrections and Notes
---

# Corrections, Caveats, and Improvements

This page records important clarifications added while turning the original lab notes into a safer procedure.

## Account locking is not complete account disablement

`passwd -l user` locks password authentication, but should not be interpreted as disabling every possible authentication path. For a genuinely obsolete human account, also consider a non-login shell, account expiration, removal of privileged group memberships, and review of SSH authorized keys.

Example:

```bash
sudo passwd -l legacy
sudo usermod -s /usr/sbin/nologin legacy
sudo usermod --expiredate 1 legacy
id legacy
```

## SSH drop-in order must be verified

Do not assume that a filename such as `90-hardening.conf` automatically overrides every earlier SSH setting. OpenSSH processes configuration according to its own precedence rules. Always confirm the effective result with:

```bash
sudo sshd -T
```

A deliberately early file such as `00-hardening.conf` may be appropriate for global defaults, but the effective configuration is the authority.

## Always validate SSH before reload

Use:

```bash
sudo sshd -t
```

before:

```bash
sudo systemctl reload ssh
```

Keep the existing administrative session open and test a second session before closing the first.

## Nmap SYN scans generally require privileges

The original lab uses `nmap -sS`. On Linux, SYN scanning generally requires raw-packet privileges. Use:

```bash
sudo nmap -sS -sV <host>
```

or use a normal TCP connect scan when appropriate.

## Separate file and directory permission changes

A command such as:

```bash
chmod 0660 /srv/public-share/*
```

can be unsafe if the wildcard includes directories, because directories require execute/traverse permission. Prefer separate operations:

```bash
sudo find /srv/public-share -type d -exec chmod 2770 {} +
sudo find /srv/public-share -type f -exec chmod 0660 {} +
```

Likewise for a read-mostly generic share:

```bash
sudo find /srv/company-share -type d -exec chmod 0750 {} +
sudo find /srv/company-share -type f -exec chmod 0640 {} +
```

## Protect the whole path of privileged scripts

If root executes `/opt/labapp/bin/maintenance.sh`, protecting only the script file is not sufficient if a parent directory is writable by an untrusted user. Inspect the full path:

```bash
namei -l /opt/labapp/bin/maintenance.sh
```

## Firewall syntax validation does not prevent lockout

This command:

```bash
sudo nft -c -f /etc/nftables.conf
```

checks syntax but cannot determine whether the resulting policy will accidentally block your administrative network. Keep console/snapshot access available, retain the current SSH session, verify source networks, load the rules, and test a new SSH session immediately.

## `rp_filter=1` is topology-dependent

Strict reverse-path filtering is suitable for many simple single-homed servers but can break legitimate traffic on hosts using asymmetric routing, multiple interfaces, VPNs, or policy routing. Treat it as a role-dependent setting rather than a universal requirement.

## Password expiration policy is organization-dependent

The example:

```bash
sudo chage -M 90 -m 1 -W 14 operator
```

is useful for demonstrating password-age controls, but a fixed 90-day rotation rule should not be treated as universally optimal. Follow current organizational, regulatory, and authentication-policy requirements.

## Deleting a secret does not invalidate it

If a file contained an actual password, token, API key, or other credential, removing the file only removes one copy. Also revoke or rotate the credential and investigate whether additional copies exist in backups, logs, shares, or snapshots.

## Samba security has multiple layers

A successful Samba design depends on several distinct layers:

```text
Unix user/group identity
        ↓
Samba authentication
        ↓
share-level authorization (`valid users`)
        ↓
Unix filesystem permissions
```

`smbpasswd` does not replace Unix filesystem security.

For modern deployments, also review whether SMB signing and SMB encryption are appropriate for the environment and clients.

## Web services should use TLS when sensitive data is involved

The original lab focuses mainly on reducing directory listing and banner disclosure. For real applications carrying credentials, personal information, or sensitive data, add HTTPS/TLS and consider redirecting cleartext HTTP to HTTPS.

## Persistent logs need retention management

`Storage=persistent` improves troubleshooting and incident analysis, but persistent logs consume disk space. Define appropriate retention and disk-usage limits, and consider off-host/centralized logging for important systems.

## Automatic updates require policy review

Enabling APT timers is not the whole policy. Also inspect:

```text
/etc/apt/apt.conf.d/50unattended-upgrades
```

so you understand which repositories/origins and packages are actually eligible for unattended installation.

## Useful additional audits

The original labs can be extended with:

- SUID/SGID file review
- Linux file-capability review
- ACL review with `getfacl`
- `auditd` for security auditing
- time synchronization verification
- configuration-integrity tooling such as AIDE
- `systemd-analyze security` for service sandboxing review
- service-specific systemd restrictions such as `NoNewPrivileges`, `ProtectSystem`, and `PrivateTmp` where compatible
- tested backup and restore procedures
- remote/off-host logging
- multifactor or hardware-backed SSH authentication where appropriate

These are additions, not replacements for the role-based methodology used in the lab.
