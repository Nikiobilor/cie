# INFRA-031: Incident Drill: A Service That Fails Silently (systemd, journalctl, strace)

**Priority:** P2 (Plan v3, Day 7: troubleshooting-first)
**Component:** Linux userspace: systemd services, file permissions. Tools: `systemctl status`, `journalctl`, `strace`, `namei`
**Environment:** `mc1` only (Ubuntu Server). A small throwaway service `notes-svc`. **No cluster changes.**
**Linked tickets:** INFRA-021 (a systemd start-up service for MicroK8s), INFRA-028 (systemd and chrony). Sets up INFRA-032 (pdb and git)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

We create a tiny service, `notes-svc`, managed by **systemd** (the part of Linux that starts and watches services). It reads a config file, then writes a heartbeat line every 10 seconds.

The broken part: it runs as a low-privilege user (`nobody`) that **isn't allowed to read its config file**. And the script throws away its own error messages. So it fails, and **nothing says why**.

`journalctl` (systemd's log) shows only that it failed. **`strace`** shows the reason. It lists every request a program makes to the Linux kernel (these requests are called **system calls**): open this file, read that one. One of those requests is refused with `Permission denied`.

## Why

"The service won't start and the logs say nothing" is a classic hard ticket. When an app gives no useful error, the kernel still knows exactly what it refused. `strace` is how you ask it. It's a standard tool for senior Linux support and escalation roles.

---

## Runbook

All commands run on **`mc1`**.

### Step 1: Check the tool is there

```bash
which strace || sudo apt-get install -y strace
```
**Expected:** `/usr/bin/strace` (or a short install).

**Real result:** _pending_

### Step 2: Create the service (already broken)

```bash
sudo mkdir -p /etc/notes-svc
echo 'DATA_DIR=/var/lib/notes-svc' | sudo tee /etc/notes-svc/notes.conf
sudo chmod 600 /etc/notes-svc/notes.conf

sudo tee /usr/local/bin/notes-svc.sh > /dev/null <<'EOF'
#!/bin/bash
exec 2>/dev/null
source /etc/notes-svc/notes.conf || exit 1
mkdir -p "$DATA_DIR"
while true; do date >> "$DATA_DIR/heartbeat.log"; sleep 10; done
EOF
sudo chmod 755 /usr/local/bin/notes-svc.sh

sudo tee /etc/systemd/system/notes-svc.service > /dev/null <<'EOF'
[Unit]
Description=Notes heartbeat service (INFRA-031 drill)

[Service]
User=nobody
StateDirectory=notes-svc
ExecStart=/usr/local/bin/notes-svc.sh
EOF
sudo systemctl daemon-reload
```
**What each part does:**
- **The config file** holds one setting, where to write data. `chmod 600` means only its owner (`root`) can read it. **This is the break.**
- **The script:**
  - `exec 2>/dev/null` throws away all error messages. **This is why it's silent.**
  - It loads the config, or exits with code 1.
  - Then it writes the date to `heartbeat.log` every 10 seconds.
- **The unit file** tells systemd how to run it:
  - `User=nobody` runs it as a low-privilege user.
  - `StateDirectory=notes-svc` has systemd create `/var/lib/notes-svc`, owned by that user.
- `daemon-reload` makes systemd read the new unit file.

**Expected:** `DATA_DIR=/var/lib/notes-svc` printed once; nothing else.

From here on, pretend you don't know what's wrong.

**Real result:** _pending_

### Step 3: Symptom

```bash
sudo systemctl start notes-svc
sleep 2
systemctl status notes-svc --no-pager
```
**Expected:** `Active: failed (Result: exit-code)` and `status=1/FAILURE`. No explanation.

**Real result:** _pending_

### Step 4: Diagnose, part 1: the logs

```bash
journalctl -u notes-svc -n 15 --no-pager
```
**Expected:** only systemd's own lines: `Started ...`, `Main process exited, code=exited, status=1/FAILURE`, `Failed with result 'exit-code'`. **Nothing from the app itself.** The logs are a dead end.

**Real result:** _pending_

### Step 5: Diagnose, part 2: strace

Run the script **exactly as the service does** (same user), with `strace` watching:
```bash
sudo strace -f -u nobody -e trace=file -o /tmp/notes.trace /usr/local/bin/notes-svc.sh
echo "exit code: $?"
grep notes /tmp/notes.trace
```
**What the flags mean:**
- `-u nobody`: run it as user `nobody`, like the service.
- `-f`: also follow any programs it starts.
- `-e trace=file`: show only file-related system calls (open, stat and so on). That keeps the output short.
- `-o /tmp/notes.trace`: save everything to a file.

`grep notes` then picks out the lines about our files.

**Expected:** `exit code: 1`, and a line like:
```
openat(AT_FDCWD, "/etc/notes-svc/notes.conf", O_RDONLY) = -1 EACCES (Permission denied)
```
That's the kernel refusing to let `nobody` open the config file. `EACCES` is the kernel's code for "permission denied".

**Real result:** _pending_

### Step 6: Confirm the permissions

```bash
namei -l /etc/notes-svc/notes.conf
```
`namei -l` shows the owner and permissions of **every folder along the path**, not just the file. A missing permission on any of them can block access.

**Expected:** the folders are readable by everyone (`drwxr-xr-x`), but the file is `-rw------- root root`: only `root` can read it.

**Real result:** _pending_

### Step 7: Fix and prove it

Let the service's group read the file, and nobody else (least privilege):
```bash
sudo chgrp nogroup /etc/notes-svc/notes.conf
sudo chmod 640 /etc/notes-svc/notes.conf
sudo systemctl restart notes-svc
sleep 12
systemctl status notes-svc --no-pager | head -5
cat /var/lib/notes-svc/heartbeat.log
```
- `nogroup` is the group of the `nobody` user.
- `640` means: owner can read and write, group can read, everyone else gets nothing.

**Expected:** `Active: active (running)`, and 2 heartbeat lines with the date.

**Real result:** _pending_

### Step 8: Clean up

```bash
sudo systemctl stop notes-svc
sudo rm /etc/systemd/system/notes-svc.service /usr/local/bin/notes-svc.sh /tmp/notes.trace
sudo rm -r /etc/notes-svc /var/lib/notes-svc
sudo systemctl daemon-reload
systemctl status notes-svc --no-pager 2>&1 | head -2
```
**Expected:** `Unit notes-svc.service could not be found.`

**Real result:** _pending_

---

## Incident write-up (fill in after the session)

| | |
|---|---|
| **Symptom** | |
| **What I checked** | |
| **Root cause** | |
| **Fix** | |
| **Prevention** | |

---

## Definition of Done

- [ ] Service created; failure seen in `systemctl status`
- [ ] `journalctl` shown to have no app-level reason
- [ ] `strace` showed the exact refused system call (`EACCES` on the config file)
- [ ] Permissions confirmed with `namei -l`
- [ ] Fixed with least privilege (group read only); service running and writing heartbeats
- [ ] Everything cleaned up

## Retention Check

The service's logs said only `status=1/FAILURE`. Why were they empty, and what did `strace` see that the logs couldn't?