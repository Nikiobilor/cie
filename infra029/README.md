# INFRA-029: Incident Drill: A Container Won't Start (Bad Device in an LXD Profile)

**Priority:** P2 (Plan v3, Day 5: troubleshooting-first)
**Component:** LXD instances, profiles and devices. Tools: `lxc start`, `lxc info --show-log`, `lxc config show --expanded`, `lxc monitor`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). A throwaway test container `t2` on `mc1`. **`k8s1` is not touched.**
**Linked tickets:** INFRA-016 (LXD containers), INFRA-021 (the `microk8s` profile). Sets up INFRA-030 (Kubernetes CrashLoopBackOff)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

An LXD container's settings come from two places:
- **Its own config:** settings set directly on that container.
- **Profiles:** shared bundles of settings that many containers can use (like `default` and `microk8s`).

LXD merges the two when the container starts. That merged view is called the **expanded** config.

Today we create a small test container `t2` and give it a profile called `drill`. The profile asks for a folder from the host that **doesn't exist**. The container refuses to start. Then we find the cause, which is hidden in the profile, not in the container, and fix it.

## Why

"My container won't start" is a classic LXD ticket. The trap: the container's own config looks fine, because the bad setting is inherited from a profile. In production that's serious: one bad change to a shared profile can stop every container that uses it from starting. That's why we use a throwaway profile today, not `default`.

---

## Runbook

All commands run on **`mc1`**.

### Step 1: Start-up check

Power on `mc1`, `mc2`, `mc3`. On **each node**: `sudo chronyc makestep`. Then on `mc1`:
```bash
sudo microceph.ceph -s | head -5
lxc start --all
lxc list -c nsL
```
**Expected:** `HEALTH_OK`; `k8s1` `RUNNING` on `mc3`.

**Real result:** _pending_

### Step 2: Create a healthy test container

```bash
lxc launch images:alpine/3.20 t2 --target mc1
lxc list t2 -c nsL
```
Alpine is a tiny Linux (a few MB), so the download is quick.

**Expected:** `t2` `RUNNING` on `mc1`.

> If the `images:` download fails, use `ubuntu:24.04` instead (bigger, slower).

**Real result:** _pending_

### Step 3: Break it: a profile with a missing folder

```bash
lxc stop t2
lxc profile create drill
lxc profile device add drill data disk source=/srv/does-not-exist path=/data
lxc profile add t2 drill
```
- `drill` is a new, empty profile.
- The device says: "share the host folder `/srv/does-not-exist` into the container at `/data`". That folder doesn't exist on `mc1`.
- `lxc profile add` attaches the profile to `t2` only.

**Expected:** `Profile drill created`, `Device data added to drill`, `Profile drill added to t2`.

From here on, pretend you don't know what changed.

> If any of these commands is refused (LXD may check the folder earlier than expected), record the exact error. That's a valid finding: LXD caught the mistake before it could stop a container.

**Real result:** _pending_

### Step 4: Symptom

```bash
lxc start t2
```
**Expected:** an error, something like `Failed to start device "data": ... /srv/does-not-exist ...`. Write down the exact message.

**Real result:** _pending_

### Step 5: Diagnose

**5a. The container's log:**
```bash
lxc info t2 --show-log | tail -15
```
**Expected:** probably nothing useful. The failure happens **before** the container's own processes start, so its log may be empty or old. That's a lesson in itself: the log you'd check first isn't always where the answer is.

**5b. The container's own config vs the expanded config:**
```bash
lxc config show t2 | grep -A8 "^devices"
lxc config show t2 --expanded | grep -A8 "^devices"
```
**Expected:** the first shows **no** `data` device. The second (`--expanded`, with profiles merged in) shows `data` with `source: /srv/does-not-exist`.

**5c. Where does it come from?**
```bash
lxc config show t2 | grep -A4 "^profiles"
lxc profile show drill
```
**Expected:** `t2` uses `default` and `drill`; `drill` holds the bad device.

**5d. Watch LXD's own events while starting:**
```bash
timeout 10 lxc monitor --type=logging --pretty --loglevel=error &
sleep 2; lxc start t2; wait
```
**Expected:** an `ERROR` line from LXD with the same reason, as LXD itself reports it.

**Real result:** _pending_

### Step 6: Fix and prove it

Fix it where the problem lives, in the profile:
```bash
lxc profile device remove drill data
lxc start t2
lxc list t2 -c nsL
```
**Expected:** `Device data removed from drill`, then `t2` `RUNNING`.

**Real result:** _pending_

### Step 7: Clean up

```bash
lxc delete t2 --force
lxc profile delete drill
lxc list -c nsL
```
**Expected:** `t2` gone, `drill` gone, `k8s1` still `RUNNING`.

Then shut down: `lxc stop --all`, then `sudo shutdown -h now` on `mc3`, `mc2`, `mc1`.

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

- [ ] Healthy `t2` created and running
- [ ] `drill` profile with a bad disk device attached to `t2`
- [ ] Start failure recorded with the exact error
- [ ] Cause found with `--expanded` and traced to the `drill` profile
- [ ] Fixed in the profile; `t2` running again
- [ ] `t2` and `drill` cleaned up; `k8s1` untouched
- [ ] Lab shut down cleanly

## Retention Check

`lxc config show t2` showed no `data` device, but the container still failed because of it. Why, and which flag shows the full picture?