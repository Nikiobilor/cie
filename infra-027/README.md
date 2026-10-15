# INFRA-027: Incident Drill: A Storage Disk Goes Down (Ceph OSD)

**Priority:** P2 (Plan v3, Day 3: troubleshooting-first)
**Component:** Storage: MicroCeph OSDs and placement groups. Tools: `ceph -s`, `ceph health detail`, `ceph osd tree`, `ceph pg stat`, `snap`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). MicroCeph, 3 OSDs × 10 GB (one per node), `HEALTH_OK` at start.
**Linked tickets:** INFRA-011 (Ceph quorum), INFRA-023 (power-loss incident), INFRA-026 (front door). Sets up INFRA-028 (Ceph clock skew)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

Ceph keeps **3 copies** of every piece of data, one on each node's disk. Each disk is run by a small service called an **OSD** (Object Storage Daemon). Today we stop the OSD on `mc2`, as if its disk had died. Then we read what Ceph reports, prove storage keeps working on 2 copies, and bring the OSD back.

**`noout` first:** normally, when an OSD stays down for 10 minutes, Ceph gives up on it and starts copying its data elsewhere. For planned work that's wasted effort, so we set the `noout` flag ("don't give up on it, it's coming back") before we stop anything.

## Why

A failed disk is one of the most common storage tickets. The skill is reading Ceph's health output calmly: what is broken, is data still safe and usable, and what Ceph will do next. This is also why we run 3 nodes: losing one still leaves 2 copies.

---

## Runbook

All commands run on **`mc1`** unless the step says otherwise. `ceph` here means `sudo microceph.ceph`.

### Step 1: Start-up check, and restore the front door

Power on `mc1`, `mc2`, `mc3`. On **each node**: `sudo chronyc makestep`. Then on `mc1`:
```bash
sudo microceph.ceph -s | head -5
lxc start --all
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31986
curl -s --max-time 3 http://192.168.82.35 | head -4
```
The third line puts back the front door rule removed after INFRA-026. If it says the rule already exists, that's fine.

**Expected:** `HEALTH_OK`, then the nginx page.

**Real result:** _pending_

### Step 2: Record the healthy state (the "before" picture)

```bash
sudo microceph.ceph osd tree
sudo microceph.ceph pg stat
sudo microceph.ceph osd pool ls detail | grep -o "^pool [0-9]* '[^']*'.* size [0-9]* min_size [0-9]*"
```
**What each shows:**
- `osd tree`: each node and its OSD, with status `up`.
- `pg stat`: the **placement groups** (PGs). Ceph splits data into these buckets; healthy ones say `active+clean`.
- `pool ls detail`: `size 3` means 3 copies are kept. `min_size 2` means Ceph keeps working as long as 2 copies are available.

**Expected:** 3 OSDs `up`; all PGs `active+clean`; pools with `size 3 min_size 2`. Note which OSD number belongs to `mc2`.

**Real result:** _pending_

### Step 3: Set `noout`

```bash
sudo microceph.ceph osd set noout
sudo microceph.ceph -s | head -8
```
**Expected:** `noout is set`, and health changes to `HEALTH_WARN` with `noout flag(s) set`. That warning is expected: Ceph reminds you the flag is on, so you don't forget it.

**Real result:** _pending_

### Step 4: Break it: stop the OSD on `mc2`

**On `mc2`:**
```bash
snap services microceph
sudo snap stop microceph.osd
```
The first line lists MicroCeph's services; `microceph.osd` runs the OSD on this node.

**Expected:** `microceph.osd` listed as `active`, then `Stopped.`

**Real result:** _pending_

### Step 5: Diagnose (back on `mc1`)

```bash
sudo microceph.ceph -s
sudo microceph.ceph health detail
sudo microceph.ceph osd tree
```
**Expected, roughly:**
- `ceph -s`: `HEALTH_WARN`, `osd: 3 osds: 2 up, 3 in`. (`in` = still counted as part of the cluster, thanks to `noout`.)
- `health detail`: lines like `1 osds down`, `1 host (1 osds) down`, `Degraded data redundancy`.
- `osd tree`: `mc2`'s OSD shows `down`.
- PG states change to `active+undersized+degraded`:
  - **active**: still serving reads and writes. Data is usable.
  - **undersized**: fewer copies than wanted (2 of 3).
  - **degraded**: some data has one copy missing.

**Real result:** _pending_

### Step 6: Prove storage still works

```bash
lxc exec k8s1 -- true && echo "k8s1 ok"
curl -s --max-time 3 http://192.168.82.35 | head -4
sudo microceph.ceph pg stat
```
**Expected:** `k8s1 ok`, the nginx page, and PGs still `active` (no `down`, `inactive` or `incomplete`). With 2 of 3 copies, Ceph is above `min_size`, so it keeps working.

**Real result:** _pending_

### Step 7: Recover

**On `mc2`:**
```bash
sudo snap start microceph.osd
```
**On `mc1`** (run the first line a few times, until all PGs are `active+clean`):
```bash
sudo microceph.ceph pg stat
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s | head -8
```
**Expected:** PGs return to `active+clean` within a minute or two; after `unset noout`, `HEALTH_OK`.

**Real result:** _pending_

### Step 8: Shut down

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

- [ ] Healthy state recorded (OSD tree, PG states, pool size and min_size)
- [ ] `noout` set before stopping anything
- [ ] OSD on `mc2` stopped; health output read and explained
- [ ] Storage proven still working with 2 of 3 copies
- [ ] OSD restarted; PGs back to `active+clean`
- [ ] `noout` unset; `HEALTH_OK`
- [ ] Lab shut down cleanly

## Retention Check

With one OSD down, the PGs said `active+undersized+degraded`. Which of those three words tells you users can still read and write data, and what would happen if a second OSD went down?