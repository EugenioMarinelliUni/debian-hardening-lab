---
layout: default
title: Command Reference
---

# Command Reference

This page explains the main commands used by the hardening procedure.

## System identification

### `cat /etc/os-release`
Prints the distribution identification file. Use it to confirm the OS and release before applying distribution-specific procedures.

### `uname -a`
Shows kernel, architecture, hostname, and related system information.

### `hostnamectl`
Shows hostname and system metadata managed through systemd.

### `ip addr`
Shows network interfaces and assigned IP addresses.

### `ip route`
Shows the kernel routing table, including directly connected networks and the default gateway.

## Files and directories

### `mkdir -p <path>`
Creates a directory. `-p` creates missing parents and does not fail merely because the directory already exists.

### `ls -l`
Long listing including permissions, owner, group, size, and timestamps.

### `ls -ld <directory>`
Shows metadata about the directory itself rather than listing its contents.

### `stat <path>`
Shows detailed filesystem metadata.

### `find`
Searches directory trees using predicates such as object type and permissions.

Example:

```bash
sudo find / -xdev -type f -perm -0002 2>/dev/null
```

means: search `/`, stay on the same filesystem, select regular files that are world-writable, and discard error messages.

### `chown`
Changes file or directory ownership.

```bash
sudo chown -R root:fileshare /srv/public-share
```

sets owner `root`, group `fileshare`, recursively.

### `chmod`
Changes Unix permission bits.

Common modes in the lab:

- `0700` — owner `rwx`, nobody else any permission
- `0600` — owner `rw`, nobody else any permission
- `0750` — owner `rwx`, group `r-x`, others none
- `0640` — owner `rw`, group `r`, others none
- `2770` — owner/group `rwx`, others none, SGID on directory
- `0660` — owner/group `rw`, others none

## Redirection and shell operators

### `>`
Redirects standard output to a file, replacing that file.

### `2>/dev/null`
Redirects standard error to `/dev/null`, effectively hiding it.

### `2>&1`
Redirects standard error to the same destination as standard output.

### `|`
Pipes the standard output of one command into the standard input of another.

### `|| true`
If the command on the left fails, run `true`, which exits successfully. Used when a failure is intentionally non-fatal.

## Processes, services, and sockets

### `ss -lntup`
Shows listening TCP and UDP sockets.

Options:

- `-l` listening
- `-n` numeric addresses and ports
- `-t` TCP
- `-u` UDP
- `-p` associated process

### `systemctl --type=service --state=running`
Lists systemd services currently in the running state.

### `systemctl list-unit-files --type=service`
Lists service unit definitions and their enablement state.

### `systemctl status <unit>`
Shows current unit state and recent status information.

### `systemctl enable --now <unit>`
Enables the unit for future startup and starts it immediately.

### `systemctl disable --now <unit>`
Disables automatic startup and stops the unit immediately.

### `systemctl reload <unit>`
Asks a running service to reload configuration without a full restart, if supported.

### `systemctl restart <unit>`
Stops and starts the service again.

### `systemctl mask <unit>` / `unmask`
A mask prevents a unit from being started through normal systemd dependency or manual operations. `unmask` removes that block.

## Package management

### `apt list --upgradable`
Shows installed packages for which APT currently knows about an upgrade.

### `sudo apt update`
Refreshes repository/package metadata. It does not itself upgrade installed packages.

### `sudo apt full-upgrade`
Upgrades packages and is allowed to resolve dependency transitions that may install or remove packages.

### `sudo apt purge <package>`
Removes a package and its package-managed configuration files.

### `dpkg -l <package>`
Queries Debian's installed-package database.

## User and group management

### `getent passwd`
Queries the system account database.

### `getent group <group>`
Queries a group from the group database.

### `id <user>`
Shows UID, primary GID, and supplementary group memberships.

### `passwd -l <user>`
Locks the password for the account. This should not be interpreted as disabling every possible authentication method by itself.

### `passwd -S <user>`
Shows password/account password-state information.

### `usermod -s /usr/sbin/nologin <user>`
Sets the login shell to `nologin`, preventing ordinary interactive shell login.

### `usermod --expiredate 1 <user>`
Expires the account.

### `groupadd -f <group>`
Creates a group; `-f` makes an already-existing group non-fatal.

### `usermod -aG <group> <user>`
Appends the user to a supplementary group. `-a` is important because it preserves existing supplementary groups.

### `chage -l <user>`
Lists password-aging policy for the account.

## sudo

### `sudo -l`
Shows the sudo permissions available to the current user.

### `visudo`
Safely edits sudoers configuration and validates syntax.

```bash
sudo visudo -f /etc/sudoers.d/example
```

edits a specific sudoers fragment.

## SSH

### `ssh-keygen -t ed25519`
Creates an Ed25519 SSH public/private key pair.

### `ssh-copy-id user@host`
Installs the local public key into the remote user's SSH authorized-keys setup.

### `ssh user@host`
Starts an SSH connection.

### `sshd -T`
Prints the effective SSH server configuration after configuration processing.

### `sshd -t`
Validates SSH server configuration syntax without starting a new daemon.

### `ssh -o PubkeyAuthentication=no ...`
Overrides a client option for one invocation. In the lab it is used as a negative test to verify that password-only access no longer works.

## Text-processing commands

### `grep`
Searches text for matching lines.

Useful options:

- `-R` recursive
- `-n` print line numbers
- `-i` case-insensitive
- `-E` extended regular expressions

### `awk`
Processes structured text by fields.

Example:

```bash
awk -F: '$7 !~ /(nologin|false)$/ {print $1,$6,$7}' /etc/passwd
```

uses `:` as the field delimiter and prints username, home directory, and shell for accounts whose shell does not end in `nologin` or `false`.

## Network scanning and HTTP testing

### `nmap -sS -sV <host>`
Performs a SYN scan and service/version detection. A SYN scan generally requires appropriate raw-packet privileges, so the examples use `sudo`.

### `nmap -p21 <host>`
Tests a specific port.

### `curl -I URL`
Fetches HTTP headers only.

### `curl URL`
Fetches the response body and is useful for verifying whether a directory or resource is actually accessible.

## Web servers

### `nginx -T`
Tests and dumps the effective nginx configuration.

### `nginx -t`
Tests nginx configuration syntax.

### `apache2ctl -S`
Displays Apache virtual-host interpretation.

### `apache2ctl configtest`
Validates Apache configuration syntax.

## Samba

### `testparm -s`
Parses and validates Samba configuration and prints interpreted settings.

### `smbclient -L localhost -N`
Lists local SMB shares without supplying a password; useful for guest-access testing.

### `smbclient //server/share -U user`
Connects to an SMB share as the specified user.

### `smbpasswd -a <user>`
Adds/enables Samba credentials for an existing Unix account.

## nftables

### `nft list ruleset`
Shows the active nftables ruleset.

### `nft -c -f /etc/nftables.conf`
Checks a rules file for validity without loading it.

### `nft -f /etc/nftables.conf`
Loads the rules from the file into the active ruleset.

Important nftables concepts used in the lab:

- `policy drop` — deny packets that do not match an allow rule
- `iifname "lo" accept` — allow loopback traffic
- `ct state established,related accept` — allow packets belonging to accepted flows
- `ct state invalid drop` — discard invalid tracked traffic
- `tcp dport 22 accept` — allow TCP destination port 22
- `ip saddr 10.10.10.0/24 ...` — restrict a rule to a source subnet

## AppArmor

### `aa-status`
Shows AppArmor status, loaded profiles, enforcement state, and confinement information.

### `aa-enforce <profile>`
Switches a profile into enforcement mode.

## sysctl

### `sysctl <parameter>`
Reads a kernel runtime parameter.

### `sysctl --system`
Loads persistent sysctl configuration from the standard sysctl configuration locations.

## Logging

### `journalctl --list-boots`
Lists boot sessions retained in the journal.

### `journalctl -u <unit>`
Shows journal entries associated with a specific systemd unit.

### `journalctl --disk-usage`
Shows current journal storage usage.

### `systemd-analyze cat-config systemd/journald.conf`
Displays journald's merged/effective configuration, including drop-ins.

## Automatic updates

### `systemctl list-timers`
Lists systemd timers and their scheduling state.

### `unattended-upgrade --dry-run --debug`
Simulates unattended-upgrade decisions without installing packages and emits detailed diagnostic output.
