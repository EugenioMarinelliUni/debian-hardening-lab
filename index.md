---
layout: default
title: Debian 13 Hardening Lab
---

# Debian 13 Hardening Lab

This site presents a practical, bottom-up hardening workflow for Debian 13 servers. It combines two lab scenarios: a general server exposing SSH, nginx, FTP and rpcbind, and a file/collaboration server exposing SSH, Apache and Samba.

The central method is:

```text
TARGET / ROLE
    ↓
CHECK / EVIDENCE
    ↓
RISK
    ↓
CORRECTIVE ACTION
    ↓
VERIFICATION
```

The goal is not to apply every possible hardening control indiscriminately. For each component, first ask whether it is needed. If it is not needed, remove or disable it. If it is needed, reduce privileges, reduce exposure, apply restrictive policy, log relevant events, and verify the result.

## Start here

- [Complete hardening procedure](guide.md)
- [Command reference](command-reference.md)
- [Final verification checklist](checklist.md)
- [Corrections, caveats, and improvements](corrections-and-notes.md)

## Defense in depth

A hardened server should rely on multiple complementary controls:

```text
network firewall / segmentation
        +
host nftables
        +
SSH hardening
        +
account and sudo security
        +
service-specific authorization
        +
filesystem permissions
        +
AppArmor
        +
sysctl hardening
        +
persistent logging
        +
patch management
```

No single control is sufficient on its own.

## Safety note

Run these procedures only on systems you administer or are authorized to modify. Commands affecting SSH, nftables, packages, filesystems, authentication, or kernel parameters can break access or services if applied incorrectly. Keep console or VM snapshot access available and adapt all addresses, usernames, paths, and service choices to your own environment.
