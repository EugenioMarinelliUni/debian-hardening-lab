---
layout: default
title: Evidence Template
---

# Evidence Template for the Final Hardening Report

Use one copy of the following structure for every service, account or configuration that you identify as inappropriate or insecure.

## Finding title

Example:

```text
FTP service unnecessarily exposed on TCP/21
```

## 1. CHECK

Write the command used to discover the problem:

```bash
<command>
```

## 2. OUTPUT

Include only the significant evidence:

```text
<relevant output>
```

Do not paste hundreds of unrelated lines if a few lines prove the finding.

## 3. Why this is a problem

State briefly:

- whether the component is unnecessary for the role of `portal.fav.it`;
- what attack surface or privilege it introduces;
- why it conflicts with the required final architecture.

## 4. REMEDIATION

Describe the chosen correction and show the command/configuration used:

```bash
<remediation command>
```

or:

```text
<configuration fragment>
```

If the action may affect remote administration, state what rollback/recovery mechanism was kept available.

## 5. VALIDATION BEFORE APPLYING/RELOADING

Where a service provides a configuration checker, include it.

Examples:

```bash
sudo sshd -t
sudo nginx -t
sudo apache2ctl configtest
sudo nft -c -f /etc/nftables.conf
```

Record the significant output.

## 6. VERIFY

Show that the desired security state now exists:

```bash
<verification command>
```

```text
<verification output>
```

## 7. SERVICE CONTINUITY TEST

When the remediated component is still required, prove it still works.

Examples:

```bash
ssh sysadmin@portal.fav.it
sftp webmaster@portal.fav.it
curl -kI https://portal.fav.it/
```

## 8. NEGATIVE TEST

Where meaningful, prove the prohibited behavior fails.

Examples:

```bash
ssh root@portal.fav.it
ssh -o PubkeyAuthentication=no sysadmin@portal.fav.it
ssh webmaster@portal.fav.it
sftp -o PubkeyAuthentication=no webmaster@portal.fav.it
curl -sSI http://portal.fav.it/
```

For the HTTP test, the expected result is not failure but a redirect rather than direct portal content.

---

# Example: HTTP must redirect to HTTPS

## CHECK

```bash
curl -sSI http://portal.fav.it/
```

## OUTPUT

```text
HTTP/1.1 200 OK
...
```

## Why this is a problem

The assignment permits TCP/80 only as a redirect mechanism. Serving portal content directly over HTTP allows an unencrypted path that conflicts with the required design.

## REMEDIATION

nginx example:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name portal.fav.it;
    return 301 https://$host$request_uri;
}
```

## VALIDATION

```bash
sudo nginx -t
```

## VERIFY

```bash
curl -sSI http://portal.fav.it/
```

Expected significant output:

```text
HTTP/1.1 301 Moved Permanently
Location: https://portal.fav.it/
```

## SERVICE CONTINUITY

```bash
curl -kI https://portal.fav.it/
```

Expected: the portal responds successfully over HTTPS using the certificate intentionally retained for the exercise.

---

# Example: `webmaster` must be SFTP-only

## CHECK

```bash
getent passwd webmaster
sudo sshd -T -C user=webmaster,host=portal.fav.it,addr=<DEVELOPER_IP> \
  | grep -E 'forcecommand|passwordauthentication|allowtcpforwarding|permittty'
```

## Why this is a problem

The developer role requires file management through SFTP, not a general-purpose interactive shell or SSH tunnel capabilities.

## REMEDIATION

```text
AllowUsers sysadmin webmaster

Match User webmaster
    ForceCommand internal-sftp
    PermitTTY no
    X11Forwarding no
    AllowTcpForwarding no
    AllowAgentForwarding no
    GatewayPorts no
    PasswordAuthentication no
```

## VALIDATION

```bash
sudo sshd -t
```

## POSITIVE VERIFY

```bash
sftp webmaster@portal.fav.it
```

## NEGATIVE VERIFY

```bash
ssh webmaster@portal.fav.it
sftp -o PubkeyAuthentication=no webmaster@portal.fav.it
```

Expected: SFTP with the authorized key works, while a normal shell and password-only authentication do not.

---

# Suggested final report order

1. System identity and initial baseline
2. Previous-administrator artifacts
3. Account and credential remediation
4. sudo remediation
5. SSH administrative hardening
6. `webmaster` SFTP-only configuration
7. HTTPS certificate evidence
8. HTTP→HTTPS redirect
9. Portal filesystem permissions
10. Obsolete FTP/TFTP removal
11. General service minimization
12. nftables
13. AppArmor
14. sysctl
15. persistent logging
16. automatic updates
17. final external Nmap comparison
18. final functional tests and negative tests
