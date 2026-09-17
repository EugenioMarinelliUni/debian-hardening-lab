# Debian 13 Hardening Lab — `portal.fav.it`

A practical, bottom-up hardening guide for the Debian 13 server used to host the company portal `portal.fav.it`.

The revised lab is role-driven: the server should expose **only** the services required for the portal and its administration.

## Required target state

- SSH administration remains available for `sysadmin`.
- The portal is published over **HTTPS (TCP/443)**.
- **HTTP (TCP/80)** is used only to redirect clients to HTTPS.
- Developers manage portal files through **SFTP over SSH (TCP/22)** with the dedicated account `webmaster`.
- Obsolete file-transfer services such as FTP/TFTP are removed when not required.
- The existing self-signed TLS certificate is preserved for the exercise.
- The known `sysadmin` password is changed.
- `root` keeps a valid password for **local console** access, while remote root SSH login is disabled.
- Configurations, keys, services, timers, cron jobs or other artifacts associated with the previous administrator are treated as suspicious until reviewed.
- Every remediation is documented with evidence of the original finding and of the final verified state.

## Workflow

```text
Understand the server role
        ↓
Capture a baseline
        ↓
Identify inappropriate accounts/services/configuration
        ↓
Decide whether each component is necessary
        ↓
Remove unnecessary components
        ↓
Harden required components using least privilege
        ↓
Validate configuration before reload/restart
        ↓
Test required behavior
        ↓
Test forbidden behavior
        ↓
Capture final evidence and compare
```

## Repository contents

- [`index.md`](index.md) — GitHub Pages landing page
- [`guide.md`](guide.md) — complete step-by-step procedure for `portal.fav.it`
- [`command-reference.md`](command-reference.md) — explanation of the important commands and options
- [`checklist.md`](checklist.md) — final verification checklist
- [`evidence-template.md`](evidence-template.md) — reusable assessment/remediation/verification report template
- [`corrections-and-notes.md`](corrections-and-notes.md) — caveats and improvements to avoid common hardening mistakes

## Safety

This material is for systems you administer or are explicitly authorized to modify. Several commands can remove packages, change authentication, alter file ownership, or cut off remote network access. Keep a working administrative session open while changing SSH or the firewall, and maintain console/snapshot access whenever possible.
