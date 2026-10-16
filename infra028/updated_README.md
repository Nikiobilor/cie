# INFRA-028: Incident Drill: One Node's Clock Drifts (Ceph Clock Skew)

**Priority:** P2 (Plan v3, Day 4: troubleshooting-first)
**Status:** Resolved (2026-10-16)
**Component:** Ceph monitors, Linux time sync (chrony). Tools: `ceph health detail`, `ceph time-sync-status`, `systemctl`, `timedatectl`, `chronyc`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). MicroCeph with 3 monitors (one per node). chrony on all nodes.
**Linked tickets:** INFRA-011 (Ceph quorum), INFRA-023 (clocks ruled out during the power-loss incident), INFRA-027 (OSD down). Sets up INFRA-029 (LXD)

> Rewritten after the session with the real outputs.

---

## Summary

Ceph's 3 **monitors** (one per node) keep the cluster's "map" and vote on every change. They need their clocks to match closely: Ceph allows **0.05 seconds** of difference by default.

chrony (the time-sync service) was stopped on `mc2` and its clock pushed **1 second** ahead.

**Result:**
- Ceph said **nothing for several minutes**: monitors compare clocks only every 5 minutes by default.
- After `mc2`'s monitor was restarted, Ceph raised `MON_CLOCK_SKEW` (0.98 s).
- On `mc2`, `chronyc tracking` returned `506 Cannot talk to daemon`: the time-sync service wasn't running.
- chrony was restarted. Its first `makestep` reported `200 OK` **without actually fixing anything yet**. Once chrony reached a time server, the clock was corrected and the warning cleared: `HEALTH_OK`.

## Why

Clock problems are quiet. Nothing crashes at first, but monitors, logs, security tickets and certificates all depend on time. Ceph warns early. The skill is following the warning back to its root cause: a time-sync service that isn't running.

---

## Runbook (with real results)

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check

```bash
sudo chronyc makestep                     # on each node
sudo microceph.ceph -s | head -8
lxc start --all
```
**Real result:** completed; output not captured.

### Step 2: Record the healthy state

```bash
sudo microceph.ceph time-sync-status
```
On `mc2`: `systemctl is-active chrony`, `timedatectl | grep -E "synchronized|NTP service"`, `chronyc tracking | grep -E "System time|Leap status"`

**Real result:** completed; output not captured.

### Step 3: Set `noout`, then break it

```bash
sudo microceph.ceph osd set noout          # on mc1
sudo systemctl stop chrony                 # on mc2
sudo date -s "+1 second"                   # on mc2
```
- chrony is stopped first, otherwise it would correct the clock straight back.
- `date -s "+1 second"` sets the clock to "now plus 1 second".

**Real result:** completed; output not captured.

### Step 4: Symptom (or the lack of one)

```bash
sudo microceph.ceph health detail
```
**Real result, first check:**
```
HEALTH_WARN noout flag(s) set
[WRN] OSDMAP_FLAGS: noout flag(s) set
```
**No clock warning yet.** Monitors compare clocks only every **5 minutes** by default, so a clock can be wrong for minutes before Ceph says anything.

**Forcing a check:** restarting `mc2`'s monitor makes the monitors vote again, and a clock check runs straight after the vote.
```bash
sudo snap restart microceph.mon            # on mc2
```
**Real result:** `Restarted.`

### Step 5: Diagnose

**5a. Why is `mc2`'s clock wrong?** On `mc2`:
```bash
timedatectl | grep -E "synchronized|NTP service"
chronyc tracking
```
**Real result:**
```
System clock synchronized: no
              NTP service: inactive
506 Cannot talk to daemon
```
`506 Cannot talk to daemon` is the giveaway: **the time-sync service isn't running at all.**

**5b. What does Ceph measure?** On `mc1`:
```bash
sudo microceph.ceph time-sync-status
sudo microceph.ceph health detail
```
**Real result:**
```
"mc1": { "skew": 0,                    "health": "HEALTH_OK" },
"mc3": { "skew": -0.0076927841279296,  "health": "HEALTH_OK" },
"mc2": { "skew": 0.9834273148704834,   "health": "HEALTH_WARN",
         "details": "clock skew 0.983427s > max 0.05s" }

HEALTH_WARN clock skew detected on mon.mc2; noout flag(s) set
[WRN] MON_CLOCK_SKEW: clock skew detected on mon.mc2
    mon.mc2 clock skew 0.982821s > max 0.05s (latency 0.00515554s)
```
- Skew is measured against the **leader** monitor (`mc1`, skew `0`).
- `mc2` is about 1 second ahead, nearly 20 times the 0.05 s limit.
- `mc3` is at -0.008 s: normal, well under the limit.

> **Order note:** in the session, Step 6 (start chrony) was run on `mc2` before the monitor restart. Ceph still measured 0.98 s afterwards, which is how the `makestep` gotcha below was found.

### Step 6: Fix and prove it

**On `mc2`:**
```bash
sudo systemctl start chrony
sleep 5
sudo chronyc makestep
chronyc tracking | grep -E "System time|Leap status"
timedatectl | grep -E "synchronized|NTP service"
```
**Real result:**
```
200 OK
System time     : 0.000000000 seconds slow of NTP time
Leap status     : Normal
System clock synchronized: no
              NTP service: active
```

> **Gotcha: `200 OK` doesn't mean fixed.** chrony had been running for only 5 seconds and hadn't heard from a time server yet, so it had nothing to correct against. The clues:
> - `System time` exactly `0.000000000`: no measurement yet, not a perfect clock.
> - `System clock synchronized: no`.
>
> Ceph confirmed it: it still measured `mc2` at 0.98 s ahead. Once chrony reached a time server, the clock was corrected and the warning cleared by itself. **`200 OK` means "command accepted", not "problem solved".**

**On `mc1`:**
```bash
sudo microceph.ceph health detail
sudo microceph.ceph time-sync-status | grep -A2 '"mc2"'
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s | head -5
```
**Real result:**
```
HEALTH_WARN noout flag(s) set
[WRN] OSDMAP_FLAGS: noout flag(s) set

        "mc2": {
            "skew": -0.032343343762573244,

noout is unset
    health: HEALTH_OK
```
`mc2` is now at -0.032 s, under the 0.05 s limit. (Ceph's measurement includes network delay, so "under the limit", not exactly 0, is healthy.)

### Step 7: Shut down

On `mc1`: `lxc stop --all`. Then `sudo shutdown -h now` on `mc3`, `mc2`, then `mc1`.

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `MON_CLOCK_SKEW: clock skew detected on mon.mc2` (0.98 s > 0.05 s). Appeared only after a monitor vote; nothing for several minutes before that |
| **What I checked** | `ceph time-sync-status`: only `mc2` off, about 1 s ahead of the leader. On `mc2`: `timedatectl` said `synchronized: no`, `NTP service: inactive`; `chronyc tracking` said `506 Cannot talk to daemon` |
| **Root cause** | chrony was stopped on `mc2`, so nothing kept its clock correct (it was set 1 s ahead for the drill) |
| **Fix** | Started chrony. Waited for it to actually sync (`synchronized: yes`), then confirmed the skew was under 0.05 s in Ceph and unset `noout` |
| **Prevention** | Keep chrony enabled at boot (`systemctl is-enabled chrony`) and monitored. After starting chrony, check `timedatectl` shows `synchronized: yes` instead of trusting `200 OK`. Optional: run an internal time server on `mc1` so all nodes sync to the same source |

**Lessons:**
1. **The warning names the victim, not the cause.** Ceph flagged `mon.mc2`; the real problem was a stopped Linux service.
2. **Ceph checks clocks only every few minutes.** No warning doesn't mean no problem. A monitor restart forces a fresh check.
3. **`200 OK` means "command accepted", not "problem solved".** Check the result (`synchronized: yes`) before moving on.
4. **One line, root cause:** `506 Cannot talk to daemon` = the service isn't running.

**AWS comparison:** EC2 instances sync to the **Amazon Time Sync Service** by default, through chrony on Amazon Linux. Stop chrony on an EC2 instance and the same drift happens. AWS keeps the time source running, but not your chrony service.

---

## Definition of Done

- [x] `noout` set; chrony stopped and clock moved on `mc2`
- [x] `MON_CLOCK_SKEW` warning seen and the exact text recorded
- [x] Cause traced to chrony not running on `mc2` (`506 Cannot talk to daemon`)
- [x] chrony restarted; warning cleared; `noout` unset; `HEALTH_OK`
- [ ] Lab shut down cleanly
- [ ] Steps 1–3 output captured (on a repeat run)

## Retention Check

Ceph flagged `mon.mc2`, but the real problem wasn't Ceph at all. What was it, and which single command on `mc2` gave it away fastest?

**Answer:** chrony, the time-sync service, was stopped on `mc2`, so its clock drifted 1 second ahead. Ceph's skew warning showed the symptom. On `mc2`, `chronyc tracking` returning `506 Cannot talk to daemon` showed the cause in one line.