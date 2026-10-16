# INFRA-029: Incident Drill: A Container Won't Start (Bad Device in an LXD Profile)

**Priority:** P2 (Plan v3, Day 5: troubleshooting-first)
**Status:** Resolved (2026-10-16)
**Component:** LXD instances, profiles and devices. Tools: `lxc start`, `lxc info --show-log`, `lxc config show --expanded`, `lxc profile show`, `lxc monitor`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). A throwaway test container `t2` on `mc1`. **`k8s1` was not touched.**
**Linked tickets:** INFRA-016 (LXD containers), INFRA-021 (the `microk8s` profile), INFRA-024 (`k8s1` created with `--storage`). Sets up INFRA-030 (Kubernetes CrashLoopBackOff)

> Rewritten after the session with the real outputs.

---

## Summary

An LXD container's settings come from two places:
- **Its own config:** settings set directly on that container.
- **Profiles:** shared bundles of settings that many containers can use (like `default` and `microk8s`).

LXD merges the two at start. The merged view is the **expanded** config.

A test container `t2` was given a profile, `drill`, that shares a host folder into it. The folder was then deleted, like a disk that isn't mounted after a reboot. `t2` refused to start. The cause wasn't in `t2`'s own config or its log. It only showed up in the **expanded** config, which traced it to the `drill` profile. It was fixed in the profile.

**Two unplanned findings on the way:**
1. **The `default` profile has no root disk** (and, from the missing IP address, no network card either). Creating a container without `--storage` fails.
2. **LXD refuses a bad path when the profile is attached**, so a typo is caught early. It only bites when the folder disappears **later**.

## Why

"My container won't start" is a classic LXD ticket. The trap: the container's own config looks fine, because the bad setting is inherited from a profile. In production, one bad change to a shared profile can stop every container that uses it from starting. That's why a throwaway profile was used, not `default`.

---

## Runbook (with real results)

All commands run on **`mc1`**.

### Step 1: Start-up check

The lab was already running. `k8s1` `RUNNING` on `mc3`.

### Step 2: Create a healthy test container

**Attempt 1:**
```bash
lxc launch images:alpine/3.20 t2 --target mc1
```
```
Error: Failed instance creation: Failed getting remote image info: Failed getting image: The requested image couldn't be found for fingerprint "alpine/3.20"
```
The image server no longer offers this version (likely retired). Switched to `ubuntu:24.04`, the image `k8s1` uses.

**Attempt 2:**
```bash
lxc launch ubuntu:24.04 t2 --target mc1
```
```
Error: Failed instance creation: Failed creating instance record: Failed initialising instance: Failed getting root disk: No root device could be found
```

> **Finding 1: the `default` profile has no root disk.** Every container needs a root disk (the disk it boots from), from its own config or a profile. `default` doesn't provide one. This was never hit before because `k8s1` was always created with `--storage ...`, which adds the root disk directly. `t2` also came up with **no IP address**, so `default` likely has no network card either.

**Attempt 3:**
```bash
lxc launch ubuntu:24.04 t2 --target mc1 --storage local
lxc list
```
```
| k8s1 | RUNNING | 10.10.10.2 (eth0)           | ... | mc3 |
|      |         | 10.1.166.192 (vxlan.calico) |     |     |
| t2   | RUNNING |                             | ... | mc1 |
```
`t2` is running on `mc1`, with no IP (see Finding 1).

### Step 3: Break it

**First try: a profile pointing at a folder that doesn't exist.**
```bash
lxc stop t2
lxc profile create drill
lxc profile device add drill data disk source=/srv/does-not-exist path=/data
lxc profile add t2 drill
```
```
Profile drill created
Device data added to drill
Error: Failed add validation for device "data": Missing source path "/srv/does-not-exist" for disk "data"
```

> **Finding 2: LXD checks the path when the profile is attached.** A bad path is refused up front, so a typo can't stop a running setup. (Adding the device to an unused profile was accepted; the check happens when the profile is applied to a container.)

**Second try: the folder exists at first, then disappears.** This is how it really happens, e.g. a disk not mounted after a reboot.
```bash
lxc profile device set drill data source=/srv/drill-data
sudo mkdir -p /srv/drill-data
lxc profile add t2 drill
sudo rmdir /srv/drill-data
```
```
Profile drill added to t2
```

### Step 4: Symptom

```bash
lxc start t2
```
```
Error: Failed start validation for device "data": Missing source path "/srv/drill-data" for disk "data"
Try `lxc info --show-log t2` for more info
```

### Step 5: Diagnose

**5a. The container's log:**
```bash
lxc info t2 --show-log | tail -15
```
```
Name: t2
Status: STOPPED
...
Log:

```
**Empty.** LXD refused to start the container, so the container never ran and wrote nothing. The log the error points to doesn't help here.

**5b. Own config vs expanded config:**
```bash
lxc config show t2 | grep -A8 "^devices"
lxc config show t2 --expanded | grep -A8 "^devices"
```
```
# own config
devices:
  root:
    path: /
    pool: local
    type: disk
profiles:
- default
- drill

# expanded (profiles merged in)
devices:
  data:
    path: /data
    source: /srv/drill-data
    type: disk
  root:
    path: /
    pool: local
    type: disk
```
**The `data` device isn't in `t2`'s own config, only in the expanded view.** So it comes from a profile.

**5c. Which profile?**
```bash
lxc profile show drill
```
```
devices:
  data:
    path: /data
    source: /srv/drill-data
    type: disk
used_by:
- /1.0/instances/t2
```
`drill` holds the device. `used_by` lists every container this profile affects: here only `t2`. On a shared profile, this is the list of everything at risk.

**5d. LXD's own event stream while starting:**
```bash
timeout 10 lxc monitor --type=logging --pretty --loglevel=error &
sleep 2; lxc start t2; wait
```
```
Error: Failed start validation for device "data": Missing source path "/srv/drill-data" for disk "data"
[1]+  Exit 124                   timeout 10 lxc monitor ...
```
`lxc monitor` printed **nothing** at error level. The refusal is a validation check returned straight to the command, not an LXD error event. **The command's own error message was the best evidence.** (`Exit 124` only means `timeout` stopped the monitor.)

### Step 6: Fix and prove it

Fixed where the problem lives, in the profile:
```bash
lxc profile device remove drill data
lxc start t2
lxc list t2
```
```
Device data removed from drill
| t2   | RUNNING |      |      | CONTAINER | 0 | mc1 |
```

### Step 7: Clean up

```bash
lxc delete t2 --force
lxc profile delete drill
lxc list -c nsL
```
```
Profile drill deleted
| k8s1 | RUNNING | mc3 |
```
`t2` and `drill` gone; `k8s1` untouched.

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `lxc start t2` → `Failed start validation for device "data": Missing source path "/srv/drill-data"` |
| **What I checked** | `lxc info --show-log`: empty (the container never ran). Own config: no `data` device. `--expanded`: `data` present, `source: /srv/drill-data`. `lxc profile show drill`: the device lives in `drill`, `used_by` only `t2`. `lxc monitor`: nothing at error level |
| **Root cause** | The `drill` profile shares a host folder that no longer existed on `mc1`, so LXD refused to start the container |
| **Fix** | Removed the device from the profile (`lxc profile device remove drill data`); `t2` started |
| **Prevention** | Check `used_by` before changing a shared profile. For host folders that may be missing (e.g. removable disks), use `required=false` on the disk device so the container still starts. Make sure disks that containers need are mounted at boot (`/etc/fstab`) |

**Lessons:**
1. **Always check `--expanded`.** A container's own config hides everything it inherits from profiles.
2. **Read the first error carefully.** The pointer to `--show-log` was a dead end; the error text itself named the device and the path.
3. **Fix the problem where it lives.** The bad setting was in the profile, so it was fixed there, not worked around on the container.
4. **`used_by` = blast radius.** It lists every container a profile change will affect.

**AWS comparison:** a profile is like an **EC2 launch template**: shared settings many instances start from. A bad change to a shared template breaks every instance launched from it. And a missing host folder is like an instance failing because a required EBS volume isn't attached.

**Follow-up:** decide whether `default` should get a root disk and a network card, so plain `lxc launch` works, or whether always passing `--storage` and `--network` is the house rule. Record whichever is chosen.

---

## Definition of Done

- [x] Healthy `t2` created and running (after two real creation errors)
- [x] `drill` profile with a disk device attached to `t2`
- [x] Start failure recorded with the exact error
- [x] Cause found with `--expanded` and traced to the `drill` profile
- [x] Fixed in the profile; `t2` running again
- [x] `t2` and `drill` cleaned up; `k8s1` untouched
- [ ] `lxc profile show default` output captured to confirm Finding 1
- [ ] Lab shut down cleanly

## Retention Check

`lxc config show t2` showed no `data` device, but the container still failed because of it. Why, and which flag shows the full picture?

**Answer:** the device came from the `drill` profile, not from `t2`'s own config, and plain `lxc config show` shows only a container's own settings. `--expanded` merges in the profiles and shows what LXD actually uses at start.