---
layout: default
title: Debian 13 Hardening Lab — portal.fav.it
---

# Debian 13 Hardening Lab — `portal.fav.it`

This site documents a production-oriented hardening exercise for a Debian 13 server hosting the company portal `portal.fav.it`.

## Required services

| Port | Service | Final purpose |
|---|---|---|
| 22/tcp | SSH / SFTP | SSH administration for `sysadmin`; SFTP-only access for `webmaster` |
| 80/tcp | HTTP | Redirect only to HTTPS |
| 443/tcp | HTTPS | Company portal |

Everything else should be justified by the server role or removed/disabled.

## Identity and access requirements

- `sysadmin`: administrative SSH access; known password must be changed; public-key SSH preferred/required after keys are tested.
- `webmaster`: dedicated developer account; SFTP only; no ordinary interactive shell.
- `root`: valid password retained for local-console recovery; direct SSH login disabled.
- Previous-administrator artifacts: investigate unknown accounts, SSH keys, cron jobs, systemd units, sudo rules and configuration fragments before trusting them.

## Web requirements

- Preserve the already installed self-signed certificate for this exercise.
- HTTPS must work on 443.
- HTTP on 80 must return only a redirect to HTTPS.
- Do not expose backup/archive directories through the document root.

## Recommended reading order

1. [Complete procedure](guide.md)
2. [Command reference](command-reference.md)
3. [Final verification checklist](checklist.md)
4. [Evidence/report template](evidence-template.md)
5. [Corrections and operational notes](corrections-and-notes.md)

## Core methodology

```text
CHECK / EVIDENCE
       ↓
RISK / ROLE DECISION
       ↓
REMEDIATION
       ↓
VALIDATION
       ↓
FUNCTIONAL + SECURITY VERIFICATION
```

A successful hardening action is not merely a changed configuration file. It must also demonstrate that the required service still functions and that the prohibited behavior now fails.
