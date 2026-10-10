# INFRA-023: Incident: Power Loss Corrupted Ceph and OVN State Across the Cluster

**Priority:** P1 (incident; interrupted Week 5, Day 24)
**Status:** **Resolved** (2026-10-09)
**Component:** Storage (MicroCeph monitors), networking (MicroOVN manager on `mc2`), LXD `remote` pool
**Environment:** VMware Workstation Pro on Windows (laptop), 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), MicroCeph 19.2.3 (`squid/stable`, rev 1895), MicroOVN (rev 1088), LXD storage pool `remote`, OVN network `ovntest`
**Linked tickets:** INFRA-011 (Ceph quorum), INFRA-015 (OVN uplink), INFRA-017 (slow-op alert), INFRA-022 (paused by this incident), INFRA-024 (workload rebuild)

> Written after the incident, from the real commands, outputs and decisions of the session.

---

## Summary

While the lab was unattended, the laptop running all three VMs **slept, then lost power**. All three machines stopped at the same instant, without shutting down. Files being written at that moment were left half-written.

On power-up, three separate things were broken:

| Layer | What was broken | Effect |
|---|---|---|
| **Ceph monitors** (all 3) | `mc1`: core history inconsistent (`ceph_assert(is_consistent())`). `mc2`, `mc3`: log bookkeeping record unreadable (`LogMonitor::log_external_backlog` → `std::invalid_argument`) | No monitor quorum; no storage at all; no container could start |
| **LXD ↔ Ceph** (after rebuild) | LXD's ownership marker `lxd_remote` didn't exist in the new pool | "Storage pool is unavailable" on every machine |
| **MicroOVN manager on `mc2`** | Three state files corrupted (zero bytes, a bad certificate, an unparseable member list) | `mc2` out of the OVN cluster; brief "connection refused" on OVN's database |

**Resolution:**
- **Ceph was rebuilt from scratch** (the OSD-based recovery was abandoned safely: no repair tools in the snap, and Ubuntu only offered a mismatched major version).
- **LXD's marker was recreated**, which restored the `remote` pool.
- **`mc2`'s MicroOVN state files were replaced** with healthy copies from `mc3`.
- **Lost:** `k8s1`, `web1`, and one cached image. Both containers are rebuildable from INFRA-018/020/021.
- **Proven:** a test container (`t1`) running on the rebuilt storage, with an `ovntest` address and internet access. All OVN databases are `OK` on all three machines.
- **Workloads:** `k8s1` and its `hello` app were rebuilt afterwards from INFRA-020/021; see **INFRA-024**.

---

## Timeline (UTC)

| When | What happened |
|---|---|
| 2026-10-06 ~17:21 | Last session ends. |
| 2026-10-06 **20:48** | Last write time on `mc2`'s MicroOVN trust-store files: most likely the moment of the power loss. |
| 2026-10-09 09:09 | `mc1`, `mc3` powered on. Ceph monitors crash and restart repeatedly; systemd gives up. |
| 09:17 | `mc2` powered on. Its monitor crashes too; its OVN manager and northbound database fail. |
| 09:20–09:30 | Diagnosis: clocks ruled out; monitors inactive on all three; crash reasons found. |
| ~09:40 | Monitor databases backed up on all three machines. |
| ~09:45 | OSD-based recovery ruled out. Decision: rebuild Ceph. |
| 10:08–10:16 | MicroCeph removed, reinstalled, re-bootstrapped; disks wiped and added; `remote` pool recreated. `HEALTH_OK`. |
| 10:18–10:20 | LXD reconnected to Ceph; stale instances and image removed. |
| 10:26–10:31 | Launch fails: "Storage pool is unavailable". Cause: missing `lxd_remote` marker. |
| ~10:35 | Marker recreated. Launch now creates `t1`'s disk, then fails on OVN ("connection refused"). |
| 10:42–10:43 | `t1` started on two healthy OVN members; internet confirmed. |
| 10:45–11:07 | `mc2`'s MicroOVN repaired, one corrupted file at a time. |
| ~11:08 | `microovn status`: Northbound and Southbound `OK`. **Resolved.** |

---

## Acceptance Criteria

- [x] Root cause identified with evidence
- [x] Monitor databases backed up before any change
- [x] Recovery options compared, decision recorded with reasons
- [x] Ceph rebuilt: 3 monitors in quorum, 3 OSDs, `HEALTH_OK`
- [x] LXD reconnected; `remote` pool working
- [x] Lost instances and image cleaned out of LXD's records
- [x] Test container running on `remote`, reachable, with internet access
- [x] `mc2`'s MicroOVN manager repaired; all OVN databases `OK`
- [ ] Prevention actions adopted (see below)

---

## Comments Thread

**NwaChi:** I only powered on `mc1` and `mc3`. I thought we only needed 2 for quorum?

**Reviewer:** You do. That's why it was worth checking further: 2 of 3 *should* have worked. But `ceph -s` hangs, so something else is wrong.

**NwaChi:** The monitor service is inactive on all three, with two different crash reasons. And the laptop slept and lost power.

**Reviewer:** That's the shared cause: three servers losing power at the same instant. Quorum protects you against **one** failure, not all three dying mid-write together.

**NwaChi:** Which option would a cloud infrastructure engineer pick?

**Reviewer:** That depends on what's at stake. Here, everything on Ceph can be rebuilt from the tickets, so rebuilding is operationally correct. In production with valuable data and no backups, you'd recover from the OSDs, usually with vendor support. With backups, you'd restore.

**NwaChi:** The recovery tools aren't in the snap, and Ubuntu only has Ceph 20.2. We run 19.2.3.

**Reviewer:** Then stop. A newer major version's tools can rewrite a damaged store in a format the old version can't read. Fall back to the rebuild.

**NwaChi:** Ceph is healthy and LXD can see the pool, but launching says it's unavailable.

**Reviewer:** `storage info` only asks Ceph for usage. Mounting the pool does more checks. What does LXD's own log say?

**NwaChi:** "Placeholder volume does not exist." The `lxd_remote` marker.

**Reviewer:** Right. A brand-new Ceph pool doesn't know it belongs to LXD. Then the launch hit OVN. Why only `mc2`?

**NwaChi:** Its manager had three corrupted files, one after another. The first one had zeros; the second one had no zeros but a certificate that wouldn't decode.

**Reviewer:** That's the lesson to remember: **"no zero bytes" doesn't mean "intact".** Compare fingerprints against a known-good copy.

---

## Investigation and Recovery Runbook (what was run, and what it showed)

> **Standing rule:** check the real output at every step. ⚠️ marks steps that destroy data.

### Part 1: Ceph monitors

#### 1.1 Symptoms

```bash
lxc list                                     # k8s1, web1: STOPPED
sudo microceph.ceph -s --connect-timeout 15  # "timed out"
sudo microceph status                        # lists configured services only, not running state
```

#### 1.2 Rule out clocks

```bash
date -u        # on each machine
```
**Result:** all within seconds. Ruled out.

#### 1.3 Are the monitors running? Why did they stop?

```bash
sudo snap services microceph | grep -E "mon|osd"
systemctl status snap.microceph.mon --no-pager | head -8
sudo journalctl -u snap.microceph.mon -b --no-pager | grep -m3 -iE "assert|FAILED"
sudo journalctl -u snap.microceph.mon -b --no-pager | grep -v "systemd\[" | tail -25
```

**Results:**
- `microceph.mon` **inactive on all three**; `microceph.osd` active.
- All three: `failed (Result: signal)`, `signal=ABRT`. The monitor stopped itself on purpose.
- **`mc1`:** `./src/mon/Paxos.cc: 91: FAILED ceph_assert(is_consistent())`.
- **`mc2`, `mc3`:** `LogMonitor::log_external_backlog()` → `std::__throw_invalid_argument`. A [Proxmox user reported the same crash](https://forum.proxmox.com/threads/ceph-bug-or-fixable-terminate-called-after-throwing-an-instance-of-std-invalid_argument.138870/) and concluded the monitor store was corrupt.

> Grep gotcha: on `mc2`/`mc3`, `grep "assert|FAILED"` matched systemd's own "Failed" lines first. Filtering out systemd lines (`grep -v "systemd\["`) showed the monitor's real message.

#### 1.4 Preserve the evidence

```bash
sudo ls /var/snap/microceph/common/data/mon/
sudo tar czf ~/mon-backup-$(hostname)-$(date +%F).tar.gz -C /var/snap/microceph/common/data mon
```
**Result:** `mon-backup-mc{1,2,3}-2026-10-09.tar.gz`, 1.6 MB each.

#### 1.5 Can the monitors be rebuilt from the OSDs?

```bash
find /snap/microceph/current -name "ceph-objectstore-tool" -o -name "ceph-monstore-tool"
apt-cache policy ceph-osd ceph-mon | grep -E "ceph-|Candidate"
```
**Result:** the tools aren't in the snap. Ubuntu offers **20.2.0** ("tentacle"); the cluster runs **19.2.3** ("squid"). **Not safe. Abandoned.**

#### 1.6 Record what to rebuild against

```bash
lxc storage show remote              # pool_name remote, pool_size 3, pg_num 32, user admin
snap connections lxd | grep -i ceph  # lxd:ceph-conf <-> microceph:ceph-conf
sudo microceph disk list             # OSD disk on each node: /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
snap list microceph                  # 19.2.3+snap897bcdd902, rev 1895, squid/stable
```

#### 1.7 Confirm the Ceph disk before erasing anything

```bash
readlink -f /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
```
**Result (same on all three):** `sdc`, 10 GB, nothing mounted. `sda` (20 GB) is the operating system. `sdb` (8 GB, partitions `sdb1`/`sdb9`) is the ZFS local-storage disk from `microcloud init`, and was left alone.

#### 1.8 ⚠️ Rebuild Ceph

On each machine:
```bash
sudo snap remove --purge microceph
sudo snap install microceph --channel=squid/stable
sudo snap refresh --hold microceph       # same version everywhere, no auto-updates
snap list microceph                      # 19.2.3, rev 1895, held
```

On `mc1`:
```bash
sudo microceph cluster bootstrap --public-network 192.168.20.0/24
sudo microceph cluster add mc2           # prints a join token
sudo microceph cluster add mc3
```
On `mc2` and `mc3`: `sudo microceph cluster join <token>`.

On each machine:
```bash
sudo microceph disk add /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0 --wipe
```

On `mc1`:
```bash
sudo microceph.ceph osd pool create remote 32
sudo microceph.ceph osd pool set remote size 3
sudo microceph.rbd pool init remote
sudo microceph.ceph -s
```
**Result:** new cluster id `3095efaa…` (old: `b5d092dd…`), 3 monitors in quorum, 3 OSDs, **2 pools, 33 PGs `active+clean`, `HEALTH_OK`**.

### Part 2: LXD and the new Ceph

#### 2.1 Reconnect, and clean up the lost records

On each machine:
```bash
sudo snap connect lxd:ceph-conf microceph:ceph-conf
sudo snap restart lxd
```
On `mc1`:
```bash
lxc storage info remote          # total 9.47 GiB; used_by still lists k8s1, web1, image
lxc delete web1 --force          # deleted cleanly, despite the disk being gone
lxc delete k8s1 --force
lxc image delete ee3016f85cc4
lxc storage info remote          # used by: {}
```
LXD accepted deleting instances whose disks no longer existed.

#### 2.2 "Storage pool is unavailable"

```bash
lxc launch ubuntu:24.04 t1 --storage remote --network ovntest --target mc2
# Error: Failed creating instance from image: Storage pool is unavailable on this server
```

`lxc storage info` worked, yet the launch failed on every machine. `storage info` only asks Ceph for usage figures; creating a container needs the pool **mounted**, which does more checks.

```bash
lxc warning list            # "Storage pool unavailable", HIGH, on mc1, mc2, mc3
sudo journalctl -u snap.lxd.daemon --since "1 hour ago" --no-pager | grep -iE "remote|pool|ceph" | tail -15
```
**Result, every minute:** `Failed mounting storage pool" err="Placeholder volume does not exist" pool=remote`.

**Cause:** when LXD first creates a Ceph pool, it writes an empty marker disk called `lxd_<pool>` (`lxd_remote`, visible in INFRA-017's `rbd du`). It checks for that marker every time it mounts the pool. A brand-new Ceph pool doesn't have it.

> Timestamp gotcha: the nodes run on UTC. `journalctl --since "11:10"` (local time) asked for the future and returned nothing. Use relative times (`"1 hour ago"`).

**Fix:** recreate the marker exactly as LXD makes it:
```bash
sudo microceph.rbd create --image-feature layering --size 0B remote/lxd_remote
```
LXD retries the mount every minute, and the pool became available.

### Part 3: OVN

#### 3.1 The launch reaches networking, and fails there

```
Failed to start device "eth0": Failed setting up OVN port: ...
ovn-nbctl ... database connection failed (Connection refused)
```
`t1` **had been created**: its disk was in the new Ceph pool. Only the network step failed.

```bash
sudo snap services microovn       # on each machine
sudo microovn status              # NB/SB: "mc2: Error. Failed to contact member"
sudo ss -tlnp | grep -E ":664[1-4]"   # on mc1: 6641-6644 all listening
```
**Result:** `mc1` and `mc3` were fully healthy (2 of 3 is enough). On `mc2`, `ovn-ovsdb-server-nb` and `microovn.daemon` were **inactive**. The "refused on all three" was most likely caught during a leader election.

#### 3.2 Prove the cloud works on two OVN members

```bash
sudo microceph.rbd ls -p remote       # lxd_remote, container_t1, image_...
lxc start t1
lxc exec t1 -- ping -c 3 8.8.8.8      # 3 of 3 replies
```

#### 3.3 `mc2`'s northbound database

```bash
sudo snap start microovn.ovn-ovsdb-server-nb
sudo journalctl -u snap.microovn.ovn-ovsdb-server-nb -b --no-pager | tail -20
```
**Result:** it joined, learned `mc3` was leader, and logged `rejecting append_request because previous entry 49,1892 not in local log`. That's normal Raft catch-up for a member that's behind, not damage.

#### 3.4 `mc2`'s MicroOVN manager: three corrupted files

`microovn status` talks to `microovn.daemon`, which was the real reason for "Failed to contact member". Each repair revealed the next broken file:

```bash
sudo journalctl -u snap.microovn.daemon --since "5 min ago" --no-pager | grep -iE "error" | tail -5
```

| # | Error | Damage | Evidence |
|---|---|---|---|
| 1 | `Unable to parse yaml for "mc1.yaml": yaml: control characters are not allowed` | Zero bytes (`^@`) from line 10 | `cat -A` showed `^@`; fingerprint `e67d…` vs healthy `9bada…` |
| 2 | `Unable to parse yaml for "mc2.yaml": Failed to decode certificate` | Bad certificate, **no** zero bytes | `grep -c '\^@'` = 0; fingerprint `0338…` vs healthy `c1e0…` |
| 3 | `Failed to join dqlite cluster open cluster.yaml node store: yaml: unmarshal errors` | Member list unparseable | Fixed by replacement |

`mc3.yaml` also differed (`bbd8…` vs `0915…`) and was replaced as a precaution. All three trust-store files on `mc2` were last written **Oct 6 at 20:48**; `mc3`'s were rewritten normally on Oct 9.

**Fix pattern, used for all three files:**

On `mc3` (healthy member):
```bash
sudo tar czf /tmp/truststore-mc3.tgz -C /var/snap/microovn/common/state truststore
sudo cp /var/snap/microovn/common/state/database/cluster.yaml /tmp/cluster-mc3.yaml
sudo chmod 644 /tmp/cluster-mc3.yaml
scp /tmp/truststore-mc3.tgz /tmp/cluster-mc3.yaml nicky@192.168.20.141:~/
```

On `mc2`:
```bash
D=/var/snap/microovn/common/state/truststore
DB=/var/snap/microovn/common/state/database
mkdir -p ~/ts-mc3 && tar xzf ~/truststore-mc3.tgz -C ~/ts-mc3
# compare fingerprints, damaged vs healthy
for f in mc1.yaml mc2.yaml mc3.yaml; do
  echo "$f: $(sudo sha256sum $D/$f | cut -c1-12)  vs mc3's copy: $(sha256sum ~/ts-mc3/truststore/$f | cut -c1-12)"
done
# keep the damaged files as evidence, then replace
sudo cp $D/mc1.yaml ~/mc1.yaml.corrupt-backup     # (and mc2/mc3.yaml, cluster.yaml likewise)
sudo cp ~/ts-mc3/truststore/mc1.yaml $D/mc1.yaml
sudo cp ~/ts-mc3/truststore/mc2.yaml $D/mc2.yaml
sudo cp ~/ts-mc3/truststore/mc3.yaml $D/mc3.yaml
sudo cp ~/cluster-mc3.yaml $DB/cluster.yaml       # ONLY cluster.yaml, never the database files
sudo snap restart microovn.daemon
```

> **Two gotchas:** (1) that folder is root-only, and `sudo cd` doesn't exist (`cd` is a shell built-in), so use full paths with `sudo` on each command. (2) Copy **only** `cluster.yaml` from the `database` folder; the rest is `mc2`'s own database content.

**Final result:**
```
OVN Northbound: OK (7.3.0)
OVN Southbound: OK (20.33.0)
```

### Part 4: Restoring workloads

The rebuild of `k8s1` and its `hello` app is recorded separately in **INFRA-024**, since it is about proving the earlier tickets work as rebuild instructions, not about the incident itself.

---

## Root Cause

All three machines lost power at the same instant (the laptop slept, then its battery ran out). Several services were writing state at that moment. On disk, those files ended up with the right size but wrong or empty contents. Quorum (2 of 3) protects against **one** member failing, not against all of them failing together, mid-write.

## Lessons

1. **Quorum isn't a backup.** It survives one failure; it can't survive everyone corrupting their own state at once.
2. **"No zero bytes" doesn't mean "intact".** Compare fingerprints against a known-good copy.
3. **Rebuilding storage under a platform means rebuilding the platform's own markers too** (here, `lxd_remote`). The platform has no other copy.
4. **Peers are a source of truth.** Corrupted per-member state (trust stores, member lists) can be restored from a healthy member, as long as you copy *only* the shared parts.
5. **Repair tools must match the major version.** Using a newer version's tools on damaged data risks making it unrecoverable.
6. **One fix can reveal the next fault.** Keep reading the newest error after every change.
7. **Documentation you can rebuild from is worth more than you think.** Everything lost could be rebuilt from the tickets (see INFRA-024).
8. **Three copies is not a backup.** Ceph's 3 replicas lived on 3 machines sharing one power source, so they failed together. There was no backup because none had been set up yet.

## Prevention

| Action | Why |
|---|---|
| Shut the lab down cleanly before leaving it: `lxc stop --all`, then `sudo shutdown -h now` on each node | Clean shutdowns finish their writes |
| Keep the laptop on mains power while the VMs run; disable sleep while the lab is up | Prevents the same event |
| Don't suspend the VMs for long periods; shut them down | Suspending freezes a cluster mid-operation |
| Back up Ceph data off-cluster (Week 9, Day 45) | A restore option, instead of rebuild or risky recovery |
| Record versions and channels (`snap list`) as part of the docs | A rebuild must match them |

**For production (and Premise2Cloud):** this is why servers get **two independent power feeds plus UPS**, why clusters span failure domains (racks, sites), and why backups live somewhere else.

---

## Definition of Done

- [x] Root cause found with evidence
- [x] Monitor databases backed up
- [x] OSD recovery attempted and stopped safely, with reasons
- [x] Ceph rebuilt, `HEALTH_OK`
- [x] LXD reconnected; marker recreated; stale records removed
- [x] `t1` running on `remote`, reachable, with internet access
- [x] `mc2`'s MicroOVN repaired; all OVN databases `OK`
- [ ] Prevention actions adopted

**Follow-ups:**
- Rebuild workloads from the tickets: done for `k8s1` in **INFRA-024**.
- Resume INFRA-022 (persistent storage, Ceph-backed volumes).

---

## Retention Check

Ceph needs only 2 of 3 monitors for quorum, yet the cluster was completely down even with all three machines running. Why didn't quorum protect it, and what would have?