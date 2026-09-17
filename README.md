# Debian 13 Hardening Lab

A practical, bottom-up hardening guide for Debian 13 servers, derived from two laboratory procedures covering a general-purpose server and an internal file/collaboration server.

The repository follows a repeatable workflow:

```text
Understand the server role
        ↓
Capture a baseline
        ↓
Identify unnecessary exposure
        ↓
Remove what is not needed
        ↓
Harden what must remain
        ↓
Validate configuration
        ↓
Test allowed and denied behavior
        ↓
Compare before and after
```

## Contents

- [`index.md`](index.md) — GitHub Pages landing page
- [`guide.md`](guide.md) — complete step-by-step hardening procedure
- [`command-reference.md`](command-reference.md) — explanation of the main commands and options
- [`checklist.md`](checklist.md) — final verification checklist
- [`corrections-and-notes.md`](corrections-and-notes.md) — important caveats and improvements to the original lab material

## Scope

This repository is intended for systems you administer or laboratory systems you are authorized to modify. Some commands can interrupt network access, remove packages, delete files, or change authentication. Keep console or snapshot access available, read each explanation before running a command, and adapt usernames, IP addresses, services and paths to your environment.
