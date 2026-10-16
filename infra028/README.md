# INFRA-028: Incident Drill: One Node's Clock Drifts (Ceph Clock Skew)

**Priority:** P2 (Plan v3, Day 4: troubleshooting-first)
**Component:** Ceph monitors, Linux time sync (chrony). Tools: `ceph health detail`, `ceph time-sync-status`, `systemctl`, `timedatectl`, `chronyc`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). MicroCeph with 3 monitors (one per node). chrony on all nodes.
**Linked tickets:** INFRA-011 (Ceph quorum), INFRA-023 (clocks ruled out during the power-loss incident), INFRA-027 (OSD down). Sets up INFRA-029 (LXD)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

Ceph's 3 **monitors** (one per node) keep the cluster's "map": which disks exist, which are up, where data lives. They vote to agree on every change, and they need their clocks to match closely. Ceph allows only **0.05 seconds** of difference by default.

Today we stop chrony (the time-sync service) on `mc2` and push its clock **1 second** ahead. Then we read Ceph's warning, find the cause on `mc2`, fix it with chrony, and watch the warning clear.

## Why

Clock problems are quiet. Nothing crashes at first, but monitors, logs, security tickets and certificates all depend on time. Ceph warns early. The skill is following the warning back to its root cause: a time-sync service that isn't running.

---

## Runbook

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check

Power on `mc1`, `mc2`, `mc3`. On **each node**: `sudo chronyc makestep`. Then on `mc1`:
```bash
sudo microceph.ceph -s | head -8
lxc start --all
```
**Expected:** `HEALTH_OK`, `mon: 3 daemons, quorum mc1,mc3,mc2` (or similar order).

**Real result:** _pending_

### Step 2: Record the healthy state

```bash
sudo microceph.ceph time-sync-status
```
**On `mc2`:**
```bash
systemctl is-active chrony
timedatectl | grep -E "synchronized|NTP service"
chronyc tracking | grep -E "System time|Leap status"
```
**What each shows:**
- `time-sync-status`: how far each monitor's clock is from the leader's, as Ceph measures it.
- `systemctl is-active chrony`: is the time-sync service running?
- `timedatectl`: does Linux think the clock is synchronised?
- `chronyc tracking`: how far off chrony thinks the clock is right now.

**Expected:** every monitor with `"skew"` near `0` and `"health": "HEALTH_OK"`. On `mc2`: `active`, `System clock synchronized: yes`, `NTP service: active`, `System time` a tiny fraction of a second, `Leap status: Normal`.

**Real result:** _pending_

### Step 3: Set `noout`, then break it

On `mc1`:
```bash
sudo microceph.ceph osd set noout
```
**On `mc2`:**
```bash
sudo systemctl stop chrony
sudo date -s "+1 second"
date +%T.%N
```
- Stopping chrony first matters: otherwise it would correct the clock straight back.
- `date -s "+1 second"` sets the clock to "now plus 1 second".

**Expected:** `noout is set`, then the new time printed.

From here on, pretend you don't know what changed.

**Real result:** _pending_

### Step 4: Symptom

```bash
sudo microceph.ceph health detail
```
**Expected, roughly:**
```
HEALTH_WARN noout flag(s) set; clock skew detected on mon.mc2
[WRN] MON_CLOCK_SKEW: clock skew detected on mon.mc2
    mon.mc2 clock skew 1.00s > max 0.05s (latency ...)
```

> **If there's no clock warning after about 1 minute:** monitors only compare clocks every few minutes. Restarting `mc2`'s monitor forces a fresh vote and a fresh clock check. **On `mc2`:** `sudo snap restart microceph.mon`, wait 20 seconds, then run Step 4 again.

**Real result:** _pending_

### Step 5: Diagnose

**5a. What does Ceph measure?** On `mc1`:
```bash
sudo microceph.ceph time-sync-status
```
**Expected:** `mc2` shows a `"skew"` of about `1.0` and `"health": "HEALTH_WARN"`; the others stay near `0`.

**5b. Why is `mc2`'s clock wrong?** On `mc2`:
```bash
systemctl is-active chrony
timedatectl | grep -E "synchronized|NTP service"
chronyc tracking
```
**Expected:** `inactive`; `System clock synchronized: no`, `NTP service: inactive`; and `chronyc tracking` fails with `506 Cannot talk to daemon`. That last one is the giveaway: the time-sync service isn't running at all.

**Real result:** _pending_

### Step 6: Fix and prove it

**On `mc2`:**
```bash
sudo systemctl start chrony
sleep 5
sudo chronyc makestep
chronyc tracking | grep -E "System time|Leap status"
```
**Expected:** `200 OK`, then `System time` back to a tiny fraction of a second, `Leap status: Normal`.

**On `mc1`** (run the first line every 30 seconds until the warning is gone):
```bash
sudo microceph.ceph health detail
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s | head -5
```
**Expected:** the clock-skew warning disappears (often within about 30 seconds), then `HEALTH_OK`.

**Real result:** _pending_

### Step 7: Shut down

On `mc1`: `lxc stop --all`. Then `sudo shutdown -h now` on `mc3`, `mc2`, then `mc1`.

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

- [ ] Healthy state recorded (Ceph clock view and chrony on `mc2`)
- [ ] `noout` set; chrony stopped and clock moved on `mc2`
- [ ] `MON_CLOCK_SKEW` warning seen and the exact text recorded
- [ ] Cause traced to chrony not running on `mc2`
- [ ] chrony restarted; warning cleared; `noout` unset; `HEALTH_OK`
- [ ] Lab shut down cleanly

## Retention Check

Ceph flagged `mon.mc2`, but the real problem wasn't Ceph at all. What was it, and which single command on `mc2` gave it away fastest?