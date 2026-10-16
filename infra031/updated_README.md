# INFRA-031: Incident Drill: A Service That Fails Silently (systemd, journalctl, strace)

**Priority:** P2 (Plan v3, Day 7: troubleshooting-first)
**Status:** Resolved (2026-10-16)
**Component:** Linux userspace: systemd services, file permissions. Tools: `systemctl status`, `journalctl`, `strace`, `namei`
**Environment:** `mc1` only (Ubuntu Server). A small throwaway service `notes-svc`. **No cluster changes.**
**Linked tickets:** INFRA-021 (a systemd start-up service for MicroK8s), INFRA-028 (systemd and chrony). Sets up INFRA-032 (pdb and git)

> Rewritten after the session with the real outputs.

---

## Summary

A tiny service, `notes-svc`, was created and managed by **systemd** (the part of Linux that starts and watches services). It reads a config file, then writes a heartbeat line every 10 seconds.

It ran as a low-privilege user (`nobody`) that **couldn't read its config file**, and the script threw away its own error messages. So it failed in **29 milliseconds**, and **nothing said why**:
- `systemctl status`: `status=1/FAILURE`.
- `journalctl`: only systemd's own lines.

**`strace`** showed the reason. It lists every request a program makes to the Linux kernel (**system calls**):
```
openat(AT_FDCWD, "/etc/notes-svc/notes.conf", O_RDONLY) = -1 EACCES (Permission denied)
```
`namei -l` confirmed the file was `root`-only. It was fixed with **least privilege**, letting only the service's group read the file. The service then ran and wrote heartbeats every 10 seconds.

**Bonus finding:** systemd itself warned `Special user nobody configured, this is not safe!`

## Why

"The service won't start and the logs say nothing" is a classic hard ticket. When an app gives no useful error, the kernel still knows exactly what it refused, and `strace` is how you ask it. It's a standard tool for senior Linux support and escalation roles.

---

## Runbook (with real results)

All commands ran on **`mc1`**.

### Step 1: Check the tool is there

```bash
which strace || sudo apt-get install -y strace
```
```
/usr/bin/strace
```
Already installed.

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
```
DATA_DIR=/var/lib/notes-svc
```
- **The break:** `chmod 600` means only `root` can read the config.
- **Why it's silent:** `exec 2>/dev/null` throws away every error message the script would print.
- `User=nobody` runs it as a low-privilege user; `StateDirectory=notes-svc` has systemd create `/var/lib/notes-svc` owned by that user.

### Step 3: Symptom

```bash
sudo systemctl start notes-svc
sleep 2
systemctl status notes-svc --no-pager
```
```
× notes-svc.service - Notes heartbeat service (INFRA-031 drill)
     Active: failed (Result: exit-code) since Fri 2026-10-16 12:59:06 UTC; 10s ago
   Duration: 29ms
    Process: 29269 ExecStart=/usr/local/bin/notes-svc.sh (code=exited, status=1/FAILURE)

systemd[1]: Started notes-svc.service - Notes heartbeat service (INFRA-031 drill).
systemd[1]: notes-svc.service: Main process exited, code=exited, status=1/FAILURE
systemd[1]: notes-svc.service: Failed with result 'exit-code'.
```
- `start` printed **nothing**: systemd launched it, and the failure happened afterwards. Always check `status` after a start.
- `Duration: 29ms`: it died almost instantly. Like INFRA-030's crash, that points to something missing or blocked at start-up.
- `status=1/FAILURE`: the script chose to exit with code 1. That's a generic label, not a reason.

### Step 4: Diagnose, part 1: the logs

```bash
journalctl -u notes-svc -n 15 --no-pager
```
```
systemd[1]: /etc/systemd/system/notes-svc.service:5: Special user nobody configured, this is not safe!
systemd[1]: Started notes-svc.service - Notes heartbeat service (INFRA-031 drill).
systemd[1]: notes-svc.service: Main process exited, code=exited, status=1/FAILURE
systemd[1]: notes-svc.service: Failed with result 'exit-code'.
```
- **Every line comes from `systemd[1]`, none from the app.** The logs are a dead end for the cause.
- **Bonus finding:** systemd warns about `User=nobody`. `nobody` is shared by many things on a system, so a service running as `nobody` can reach anything else that runs as `nobody`. Better options are a dedicated user, or `DynamicUser=yes` (systemd creates a temporary user just for this service).

### Step 5: Diagnose, part 2: strace

```bash
sudo strace -f -u nobody -e trace=file -o /tmp/notes.trace /usr/local/bin/notes-svc.sh
echo "exit code: $?"
grep notes /tmp/notes.trace
```
```
exit code: 1
29353 execve("/usr/local/bin/notes-svc.sh", ["/usr/local/bin/notes-svc.sh"], ...) = 0
29353 openat(AT_FDCWD, "/usr/local/bin/notes-svc.sh", O_RDONLY) = 3
29353 newfstatat(AT_FDCWD, "/usr/local/bin/notes-svc.sh", {st_mode=S_IFREG|0755, st_size=158, ...}, 0) = 0
29353 openat(AT_FDCWD, "/etc/notes-svc/notes.conf", O_RDONLY) = -1 EACCES (Permission denied)
```
**Reading it, line by line:**
1. `execve(...) = 0`: the script started.
2. `openat(... notes-svc.sh ...) = 3`: bash opened the script to read it. `3` is the "handle" the kernel gave back, meaning success.
3. `newfstatat(... 0755 ...) = 0`: bash checked the script file's details. Success.
4. `openat(... notes.conf ...) = -1 EACCES (Permission denied)`: **the kernel refused to let `nobody` open the config.** `-1` means failure, and `EACCES` is the kernel's code for "permission denied".

Same user and same exit code as the service, so the cause is confirmed. Nothing in the logs could have shown this, because the script had thrown its error message away.

### Step 6: Confirm the permissions

```bash
namei -l /etc/notes-svc/notes.conf
```
```
f: /etc/notes-svc/notes.conf
drwxr-xr-x root root /
drwxr-xr-x root root etc
drwxr-xr-x root root notes-svc
-rw------- root root notes.conf
```
- Every folder on the path is open to everyone (`r-x` at the end), so the folders aren't the problem.
- The file is `-rw-------`: read and write for `root` only, nothing for anyone else.

### Step 7: Fix and prove it

```bash
sudo chgrp nogroup /etc/notes-svc/notes.conf
sudo chmod 640 /etc/notes-svc/notes.conf
sudo systemctl restart notes-svc
sleep 12
systemctl status notes-svc --no-pager | head -5
cat /var/lib/notes-svc/heartbeat.log
```
```
● notes-svc.service - Notes heartbeat service (INFRA-031 drill)
     Active: active (running) since Fri 2026-10-16 13:01:43 UTC; 18s ago
   Main PID: 29465 (notes-svc.sh)

Fri Oct 16 01:01:43 PM UTC 2026
Fri Oct 16 01:01:53 PM UTC 2026
Fri Oct 16 01:02:03 PM UTC 2026
```
- `nogroup` is the group of the `nobody` user. `640` means: owner reads and writes, group reads, everyone else gets nothing.
- **Least privilege:** the file wasn't opened up to everyone (`644`). Only the service's group got read access.
- Running, with a heartbeat exactly every 10 seconds.

### Step 8: Clean up

```bash
sudo systemctl stop notes-svc
sudo rm /etc/systemd/system/notes-svc.service /usr/local/bin/notes-svc.sh /tmp/notes.trace
sudo rm -r /etc/notes-svc /var/lib/notes-svc
sudo systemctl daemon-reload
systemctl status notes-svc --no-pager 2>&1 | head -2
```
```
Unit notes-svc.service could not be found.
```

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `notes-svc` failed 29 ms after start: `status=1/FAILURE`, no error message anywhere |
| **What I checked** | `systemctl status` and `journalctl`: only systemd's lines, no app message. `strace` (same user as the service): `openat(".../notes.conf") = -1 EACCES (Permission denied)`. `namei -l`: file `-rw------- root root` |
| **Root cause** | The config file was readable only by `root`; the service runs as `nobody`. The script also discarded its own errors (`exec 2>/dev/null`), so nothing was logged |
| **Fix** | `chgrp nogroup` + `chmod 640` on the config: only the service's group can read it. Service running, heartbeat every 10 s |
| **Prevention** | Don't throw away stderr: systemd sends it to the journal automatically, which would have shown `Permission denied`. Run services as a dedicated user or with `DynamicUser=yes`, not `nobody`. After changing a file a service needs, check access as that user (`sudo -u <user> cat <file>`) |

**Lessons:**
1. **When the logs are silent, ask the kernel.** `strace` shows every refused request, whatever the app logs.
2. **Reproduce as the same user.** `strace -u nobody` made the test match the service exactly. As `root`, it would have worked and hidden the problem.
3. **`exec 2>/dev/null` is how silent failures are made.** The error existed; it was thrown away.
4. **Read warnings even when they aren't the bug.** systemd flagged the risky `nobody` user before anything failed.

**AWS comparison:** this is like an EC2 app that fails because its **IAM role can't read an S3 object** while the app swallows the error. There, **CloudTrail** records the `AccessDenied`, the way `strace` recorded `EACCES` here: proof of what was refused and for whom.

---

## Definition of Done

- [x] Service created; failure seen in `systemctl status`
- [x] `journalctl` shown to have no app-level reason
- [x] `strace` showed the exact refused system call (`EACCES` on the config file)
- [x] Permissions confirmed with `namei -l`
- [x] Fixed with least privilege (group read only); service running and writing heartbeats
- [x] Everything cleaned up

## Retention Check

The service's logs said only `status=1/FAILURE`. Why were they empty, and what did `strace` see that the logs couldn't?

**Answer:** the script threw away its own error messages (`exec 2>/dev/null`), so nothing reached the journal except systemd's own lines. `strace` watches the program's requests to the kernel directly, so it saw the refused one: opening `/etc/notes-svc/notes.conf` returned `EACCES (Permission denied)` for user `nobody`.