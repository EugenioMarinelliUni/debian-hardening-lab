---
layout: default
title: Corrections and Notes
---

# Corrections and Operational Notes

This page records caveats that matter when translating a hardening checklist into an executable procedure for `portal.fav.it`.

## 1. `passwd -l` does not necessarily disable an account completely

`passwd -l USER` locks password authentication, but other authentication paths can still exist. For a genuinely obsolete human account, combine password locking with an expired account, a `nologin` shell and removal of unnecessary privileged group membership.

## 2. Do not lock root in this assignment

The final requirements explicitly need a valid root password for local-console recovery. Therefore the correct model is:

```text
root local-console password → remains valid
root SSH login              → disabled
```

Use `PermitRootLogin no` for the network restriction rather than `passwd -l root`.

## 3. `AllowUsers sysadmin` is no longer sufficient

SFTP is a subsystem of SSH and normally uses the same TCP/22 daemon. Because developers must connect as `webmaster`, a global allowlist must account for both identities, for example:

```text
AllowUsers sysadmin webmaster
```

Then restrict `webmaster` further with a `Match User webmaster` block and `ForceCommand internal-sftp`.

## 4. Test SSH key access before disabling passwords

Always establish and verify key-based `sysadmin` access in a **new** session before setting `PasswordAuthentication no`. Keep the existing administrative session open during SSH changes.

The same principle applies to firewall changes: validate rules first, load them while a known-good session remains open, and test a second admin connection before closing the original session.

## 5. OpenSSH drop-in ordering can be counterintuitive

Do not assume a filename such as `90-hardening.conf` always overrides earlier files. OpenSSH uses first-obtained values for many scalar options and `Match` blocks can make effective values context-dependent.

Always verify with:

```bash
sudo sshd -T
```

and for user-specific rules:

```bash
sudo sshd -T -C user=webmaster,host=portal.fav.it,addr=<CLIENT_IP>
```

## 6. The self-signed certificate is deliberately retained

For this exercise the existing self-signed certificate is considered acceptable and must not be replaced. Capture its SHA-256 fingerprint before web-server changes and compare it afterward.

`curl -k` should be understood as a **lab-specific verification convenience** here. In normal production use, clients should validate a certificate chain against an appropriate trust anchor rather than disabling verification.

## 7. Port 80 being open does not prove compliance

The assignment permits HTTP only for redirection. Nmap can show that TCP/80 is open, but it cannot prove the application-level policy.

Verify separately with:

```bash
curl -sSI http://portal.fav.it/
```

and confirm a `301` or `308` response pointing to the HTTPS URL.

## 8. Port 443 must be added to the firewall model

The older generic example allowed SSH and optionally HTTP. The portal role requires HTTPS, therefore TCP/443 must be permitted. TCP/80 remains permitted only because a redirect service is explicitly required.

## 9. Restricting port 22 by source must account for developers too

If nftables limits TCP/22 to a management subnet, remember that SFTP also uses TCP/22. Both administrator and developer source networks must be considered, otherwise the firewall may break required SFTP access even though the SSH server configuration is correct.

## 10. Do not assume Apache or nginx

The assignment requires HTTPS behavior, not a particular web server. Discover the active implementation first:

```bash
systemctl --type=service --state=running | grep -E 'apache2|nginx'
sudo ss -lntp | grep -E ':(80|443)\b'
```

Then use the correct syntax and validation command:

```text
nginx  → nginx -t
Apache → apache2ctl configtest
```

## 11. Discover the real DocumentRoot before changing permissions

Do not blindly assume `/var/www/html` or `/var/www/portal`. Inspect nginx/Apache configuration first, then construct the `webmaster` and web-server access policy around the actual application layout.

## 12. Separate file modes from directory modes

A command such as:

```bash
chmod 0640 /some/tree/*
```

is unsafe if the wildcard includes directories because directories need execute/traverse permission.

Prefer separate operations:

```bash
find /some/tree -type d -exec chmod 2750 {} +
find /some/tree -type f -exec chmod 0640 {} +
```

with modes adjusted to the actual application requirements.

## 13. Be careful with SFTP chroot ownership

If `ChrootDirectory` is used, OpenSSH requires strict ownership/permission conditions on the chroot root and its path components. A common safe design is that the chroot root is owned by root and not writable by `webmaster`, while a child directory contains writable portal content.

Do not turn the whole chroot root over to `webmaster` just to make uploads convenient.

## 14. Avoid shared private keys

The assignment specifies one `webmaster` account, but the development team may contain multiple people. If that account must be shared, each developer should still have an individual public key in `authorized_keys` so access can be revoked per developer without redistributing a common private key.

Where organizational requirements permit, distinct named developer accounts are even better for accountability.

## 15. `nologin` and `ForceCommand internal-sftp` serve different purposes

`/usr/sbin/nologin` expresses that the account is not intended to receive a conventional shell. `ForceCommand internal-sftp` is the SSH-side control that forces the authenticated remote session into SFTP. Use and verify the SSH restriction rather than relying on the shell field alone.

## 16. Service removal is preferable to hardening an unnecessary service

If FTP/TFTP or another daemon is not required by the portal role, disable/remove it instead of investing effort in securing a service that should not exist on the machine.

Do not run every removal command blindly; first prove that the service/package is present and unnecessary.

## 17. `apt purge` does not guarantee every application-created file disappears

`apt purge PACKAGE` removes the package and package-managed configuration files, but service-created data, custom directories or administrator-created files can remain. Recheck sockets, processes, files and package state after removal.

## 18. `nmap -sS` commonly requires elevated raw-packet privileges

Use:

```bash
sudo nmap -sS -sV ...
```

when performing a SYN scan in the authorized lab. An unprivileged TCP connect scan uses `-sT` instead.

## 19. Nmap and `ss` answer different questions

`ss -lntup` is the host-local view of listeners. Nmap from another machine is the external reachability view. A service can listen locally while being blocked by a firewall, so both forms of evidence are useful.

## 20. The firewall syntax check cannot prove you will not lock yourself out

```bash
sudo nft -c -f /etc/nftables.conf
```

checks syntax, not operational correctness. A syntactically valid rule can still block the administrator. Keep console access and an existing SSH session, then test a new connection immediately after loading the rules.

## 21. Do not blindly enable strict `rp_filter`

`net.ipv4.conf.*.rp_filter = 1` can be suitable for a simple single-homed host, but can break legitimate traffic on multihomed, asymmetric-routing, VPN or policy-routing systems. Treat it as topology-dependent.

## 22. Password-age numbers are policy examples, not universal truths

A command such as:

```bash
chage -M 90 -m 1 -W 14 USER
```

should be understood as an organizational example. Hardening should prioritize strong unique credentials, compromise response, appropriate MFA where possible, and the policy/regulatory context rather than assuming forced periodic rotation is always beneficial.

## 23. Deleting a plaintext secret is not the same as revoking it

If a password/token/private key has been exposed:

```text
remove exposed copy
      ↓
revoke/rotate credential
      ↓
search for additional copies
      ↓
review logs/use where appropriate
```

Deleting the file alone does not make a known credential unknown again.

## 24. Audit persistence because the previous administrator is untrusted

The assignment makes prior-administrator artifacts particularly relevant. At minimum review:

- local accounts and groups;
- SSH authorized keys;
- sudoers fragments;
- custom systemd units and drop-ins;
- timers;
- root/system cron jobs;
- `/usr/local` and `/opt` custom executables/configuration.

Search results are clues, not proof of maliciousness. Inspect before removing.

## 25. SUID/SGID and file capabilities are not automatically vulnerabilities

Commands such as:

```bash
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -ls
getcap -r / 2>/dev/null
```

are discovery tools. Many legitimate OS binaries require these mechanisms. Focus on unexpected/custom entries and compare them with the server role and package ownership.

## 26. Logging should be bounded and, for higher assurance, exported

Persistent journald improves local troubleshooting and incident evidence, but also creates disk-consumption and local-tampering considerations. Define sensible retention limits and consider remote/off-host logs for stronger production designs.

## 27. Unattended-upgrades timers are not the whole policy

Review both the scheduling mechanism and the policy controlling which origins/packages are accepted. In particular, inspect `/etc/apt/apt.conf.d/50unattended-upgrades` rather than assuming enabled timers mean the desired security updates are automatically applied.

## 28. Verification must prove both security and continuity

A configuration change is incomplete until both sides are demonstrated. For example:

```text
sysadmin SSH works       + root SSH fails
webmaster SFTP works     + webmaster shell fails
HTTPS portal works       + HTTP only redirects
certificate unchanged    + TLS service still works
required ports reachable + unnecessary ports absent
```

That distinction is central to the final assignment.
