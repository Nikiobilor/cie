# INFRA-027: Incident Drill: A Storage Disk Goes Down (Ceph OSD)

**Priority:** P2 (Plan v3, Day 3: troubleshooting-first)
**Status:** Resolved (2026-10-15)
**Component:** Storage: MicroCeph OSDs and placement groups. Tools: `ceph -s`, `ceph health detail`, `ceph osd tree`, `ceph pg stat`, `snap`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). MicroCeph, 3 OSDs × 10 GB (one per node), `HEALTH_OK` at start.
**Linked tickets:** INFRA-011 (Ceph quorum), INFRA-023 (power-loss incident), INFRA-026 (front door). Sets up INFRA-028 (Ceph clock skew)

> Rewritten after the session with the real outputs.

---

## Summary

Ceph keeps **3 copies** of every piece of data, one on each node's disk. Each disk is run by a small service called an **OSD** (Object Storage Daemon). The OSD on `mc2` (`osd.3`) was stopped on purpose, as if its disk had died.

**Result:** Ceph reported exactly one copy of all data missing (33.333%), but **every placement group stayed `active`**: the web app and `k8s1` kept working the whole time. After the OSD was started again, Ceph caught up by itself in about a minute and returned to `HEALTH_OK`.

**`noout` first:** normally, an OSD that stays down for 10 minutes is marked "out", and Ceph starts copying its data elsewhere. For planned work that's wasted effort, so `noout` ("don't give up on it, it's coming back") was set before anything was stopped.

## Why

A failed disk is one of the most common storage tickets. The skill is reading Ceph's health output calmly: what is broken, is data still safe and usable, and what will Ceph do next. It's also why the lab has 3 nodes: losing one still leaves 2 copies.

---

## Runbook (with real results)

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check, and restore the front door

```bash
sudo chronyc makestep                     # on each node
sudo microceph.ceph -s | head -5
lxc start --all
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31986
curl -s --max-time 3 http://192.168.82.35 | head -4
```
**Real result:** `<title>Welcome to nginx!</title>`. The front door rule removed after INFRA-026 is back. (The `ceph -s` output wasn't captured.)

### Step 2: Record the healthy state

```bash
sudo microceph.ceph osd tree
sudo microceph.ceph pg stat
sudo microceph.ceph osd pool ls detail | grep -o "^pool [0-9]* '[^']*'.* size [0-9]* min_size [0-9]*"
```
**Real result:**
```
ID  CLASS  WEIGHT   TYPE NAME      STATUS
-2         0.00980      host mc1
 1    hdd  0.00980          osd.1      up
-4         0.00980      host mc2
 3    hdd  0.00980          osd.3      up
-3         0.00980      host mc3
 2    hdd  0.00980          osd.2      up

65 pgs: 65 active+clean; 1.1 GiB data, 3.5 GiB used, 26 GiB / 30 GiB avail

pool 1 '.mgr' replicated size 3 min_size 2
pool 2 'remote' replicated size 3 min_size 2
pool 3 'microk8s-rbd0' replicated size 3 min_size 2
```
- One OSD per node. `mc2`'s is **`osd.3`** (there is no `osd.0`; the numbers come from the cluster's history).
- **Placement groups (PGs)** are the buckets Ceph splits data into. All 65 are `active+clean`.
- Every pool keeps **3 copies** (`size 3`) and keeps working as long as **2** are available (`min_size 2`). Pools: `.mgr` (Ceph's own), `remote` (LXD's storage), `microk8s-rbd0` (Kubernetes).

### Step 3: Set `noout`

```bash
sudo microceph.ceph osd set noout
sudo microceph.ceph -s | head -8
```
**Real result:**
```
noout is set
    health: HEALTH_WARN
            noout flag(s) set
    mon: 3 daemons, quorum mc1,mc3,mc2 (age 60m)
```
The warning is expected: Ceph reminds you the flag is on, so you don't forget to remove it.

### Step 4: Break it: stop the OSD on `mc2`

**On `mc2`:**
```bash
snap services microceph
sudo snap stop microceph.osd
```
**Real result:** completed; output not captured.

### Step 5: Diagnose

```bash
sudo microceph.ceph -s
sudo microceph.ceph health detail
sudo microceph.ceph osd tree
```
**Real result (shortened):**
```
    health: HEALTH_WARN
            noout flag(s) set
            1 osds down
            1 host (1 osds) down
            Degraded data redundancy: 324/972 objects degraded (33.333%), 38 pgs degraded

    osd: 3 osds: 2 up (since 34s), 3 in (since 2d)
         flags noout

    pgs:     324/972 objects degraded (33.333%)
             38 active+undersized+degraded
             27 active+undersized

[WRN] OSD_DOWN: 1 osds down
    osd.3 (root=default,host=mc2) is down
[WRN] PG_DEGRADED: Degraded data redundancy: ...
    pg 1.0 is active+undersized+degraded, acting [2,1]
    pg 2.0 is active+undersized+degraded, acting [1,2]
    ...

 3    hdd  0.00980          osd.3    down
```

**Reading it:**

| Output | Meaning |
|---|---|
| `2 up, 3 in` | One OSD is **down** (not running) but still **in** (still counted as part of the cluster). `noout` keeps it in. |
| `324/972 objects degraded (33.333%)` | 324 objects × 3 copies = 972 copies. 324 are missing: exactly **one copy of everything**, the copy on `osd.3`. |
| `active` | Still serving reads and writes. **Data is usable.** |
| `undersized` | Fewer copies than wanted: 2 of 3. |
| `degraded` | Some data in this PG is missing a copy. |
| `acting [2,1]` | This PG is currently served only by `osd.2` and `osd.1`. |

**Why 27 PGs are `undersized` but not `degraded`:** those PGs are **empty**. They are short one OSD, but no data is missing a copy. The count fits: `.mgr` 1 + `remote` 32 + 5 from `microk8s-rbd0` = 38 degraded; the other 27 are empty `microk8s-rbd0` PGs.

### Step 6: Prove storage still works

```bash
lxc exec k8s1 -- true && echo "k8s1 ok"
curl -s --max-time 3 http://192.168.82.35 | head -4
sudo microceph.ceph pg stat
```
**Real result:**
```
k8s1 ok
<title>Welcome to nginx!</title>
65 pgs: 27 active+undersized, 38 active+undersized+degraded; ...
```
Everything kept working. No PG was `down`, `inactive` or `incomplete`: 2 copies is still above `min_size 2`.

### Step 7: Recover

**On `mc2`:** `sudo snap start microceph.osd`

**On `mc1`:**
```bash
sudo microceph.ceph pg stat        # run a few times
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s | head -8
```
**Real result:**
```
65 pgs: 36 peering, 12 active+undersized, 17 active+undersized+degraded; ... 151/972 objects degraded (15.535%)
65 pgs: 65 active+clean; 1.1 GiB data, 3.5 GiB used, 26 GiB / 30 GiB avail
noout is unset
    health: HEALTH_OK
    osd: 3 osds: 3 up (since 48s), 3 in (since 2d)
```
- **`peering`**: the returning OSD and the other two compare notes on what changed while it was away.
- Degraded copies fell from 324 to 151 to 0 as `osd.3` caught up.
- Back to `active+clean` and `HEALTH_OK` in **under a minute** (`3 up (since 48s)`).

### Step 8: Shut down

On `mc1`: `lxc stop --all`. Then `sudo shutdown -h now` on `mc3`, `mc2`, then `mc1`.

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `HEALTH_WARN`: `1 osds down`, `1 host (1 osds) down`, `324/972 objects degraded (33.333%)` |
| **What I checked** | `ceph health detail` named the OSD (`osd.3` on `mc2`); `osd tree` showed it `down`; PG states were `active+undersized(+degraded)`; app and `k8s1` still worked |
| **Root cause** | The OSD service on `mc2` was stopped (planned drill), so one of the three copies was unavailable |
| **Fix** | Started the OSD again (`snap start microceph.osd`), waited for `active+clean`, then unset `noout` |
| **Prevention** | Set `noout` before planned maintenance, and unset it straight after. Work on **one** node at a time: with `min_size 2`, a second OSD down would stop I/O for the PGs it shares |

**Lessons:**
1. **`active` is the word that matters.** It means users can still read and write. `undersized` and `degraded` mean "less safe", not "down".
2. **33.333% degraded on a 3-copy cluster = exactly one node's copies missing.** Simple maths confirms the scope of the problem.
3. **`noout` is a planned-work tool.** Without it, after 10 minutes Ceph would try to copy data elsewhere. Here, with only 2 hosts left and copies required on different hosts, there was nowhere to copy to anyway.
4. **Recovery is automatic.** Ceph tracks what changed and catches the returning OSD up by itself.

**AWS comparison:** this is like one copy of an EBS volume's data failing inside AWS. AWS hides it and repairs it for you. With Ceph, you see it and manage it yourself.

---

## Definition of Done

- [x] Healthy state recorded (OSD tree, PG states, pool size and min_size)
- [x] `noout` set before stopping anything
- [x] OSD on `mc2` stopped; health output read and explained
- [x] Storage proven still working with 2 of 3 copies
- [x] OSD restarted; PGs back to `active+clean`
- [x] `noout` unset; `HEALTH_OK`
- [ ] Lab shut down cleanly
- [ ] Step 1 `ceph -s` and Step 4 outputs captured (on a repeat run)

## Retention Check

With one OSD down, the PGs said `active+undersized+degraded`. Which of those three words tells you users can still read and write data, and what would happen if a second OSD went down?