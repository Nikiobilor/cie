# INFRA-010: Bootstrap a Real MicroCloud Cluster — Compute + Storage

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-010 |
| **Priority** | High |
| **Component** | MicroCloud / Compute + Storage |
| **Environment** | VMware Workstation Pro (Windows host) — 3 new VMs (`mc1`, `mc2`, `mc3`), separate from the existing `node1`–`node3` LXD cluster |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 3 — MicroCloud Compute/Storage (Day 10) |
| **Linked tickets** | INFRA-009 (MicroCloud overview) — this ticket executes the bootstrap deferred there. INFRA-008 — the netplan lesson from that ticket was re-applied here, and re-broke in a new way (see incident log). |
| **Status** | Closed. Compute and storage genuinely clustered, verified with a real container launch on Ceph-backed storage. Two unplanned real-world corrections along the way — see below. |

---

## Summary

The plan was straightforward: clone three VMs, install the MicroCloud tooling, run `microcloud init`, get LXD and Ceph clustered together in one guided flow.

What actually happened surfaced two genuine gaps in that plan — neither was hypothetical, both were hit live and had to be diagnosed and fixed mid-session. Both are documented in detail below because they're the kind of thing no tutorial mentions and every real deployment eventually runs into.

---

## What Actually Happened (incident log)

**1. Cloned `mc1`, `mc2`, `mc3` from `base-vm-vmware`.**
All three initially showed the clone's default hostname. Renamed properly in two places — the VMware library label (cosmetic only) and the actual Linux hostname via `sudo hostnamectl set-hostname mcN` on each VM, since only the second one affects SSH prompts and cluster output.

**2. Added one 8 GB disk per VM, per the original plan — this later proved insufficient (see incident 6).**

**3. LXD wasn't pre-installed on the new VMs.**
`sudo snap list lxd` returned `no matching snaps installed` on `mc1`. Unlike an assumption baked into the original plan, the golden image doesn't ship with LXD — it was installed manually back in Week 2 on `node1`–`node3`, and had to be installed fresh here too, on all three new nodes.

**4. Netplan static IP setup hit a real, unplanned bug.**
The existing netplan file on `mc2` (`00-installer-config.yaml`) contained a MAC address — `00:0c:29:43:c8:4e` — that matched **neither** of the machine's two actual interfaces. It was stale config, most likely carried over from the clone before VMware assigned this VM's adapters their own unique MACs. The file also had `dhcp4`/`dhcp6` set `true` right next to a static `addresses:` line — self-contradictory.

Checking `ip a` to find the real host-only adapter surfaced a second finding worth keeping: on `mc2`, `ens33` was the host-only adapter and `ens37` was NAT — the **opposite** of what `node2` had back in Week 2. Interface enumeration order isn't just inconsistent across power cycles (INFRA-008's finding) — it's inconsistent across different VMs entirely. MAC-based netplan matching isn't a nice-to-have here, it's the only approach that actually works.

**5. `sudo netplan apply` appeared to hang.**
Root cause: the SSH session was connected over the very interface being re-addressed, so the terminal lost its own connection mid-command and looked stuck. Confirmed by opening a fresh connection to the new IP, which worked immediately — the apply had actually succeeded.

**6. `microcloud init` silently consumed the only disk for local storage, leaving none for Ceph.**
The init flow asked "Would you like to set up local storage?", defaulted to yes, and used the single 8 GB disk on each node for a **local** (non-replicated) storage pool. It then printed: `Warning: No disks available for distributed storage. Skipping configuration.` Local and distributed storage are two separate resource pools competing for the same disks — with only one disk per node, answering yes to the first question forecloses the second. `microceph status` confirmed it directly: `Disks: 0` on all three nodes, despite `init` completing "successfully."

**7. Fix: added a second 10 GB disk per VM, then manually added it to Ceph.**
```bash
sudo microceph disk add /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
```
Run on each node individually. `microceph status` afterward showed `osd` in the services list and `Disks: 1` on all three — real distributed storage, this time.

**8. `lxc storage list` was still empty — even with Ceph fully healthy.**
This is the second real gap: MicroCeph and LXD are separate systems with separate configuration. Telling Ceph about a disk doesn't tell LXD anything. The bridge that `microcloud init` normally builds automatically has to be built by hand when storage was skipped there:
```bash
lxc storage create remote ceph --target mc1
lxc storage create remote ceph --target mc2
lxc storage create remote ceph --target mc3
lxc storage create remote ceph
```
The first three register the pool per cluster member; the last one (no `--target`) activates it cluster-wide. `lxc storage list` then showed `remote`, driver `ceph`, state `CREATED`.

**9. First container launch attempt failed with a transient database error.**
```
Error: Failed instance creation: Fetch project database object: Failed to fetch from "projects" table: ... sql: transaction has already been committed or rolled back
```
A dqlite-level race condition, most likely from four `lxc storage create` commands run in quick succession right before. Retried the identical command with no changes — succeeded. Confirmed as transient, not a real misconfiguration, by checking `lxc storage show remote` (healthy) before retrying rather than guessing.

**10. Container verified running on the Ceph-backed pool, then deleted cleanly** — closing out the ticket with a genuine end-to-end proof, not just a status check.

---

## Acceptance Criteria

- [x] Existing `node1`–`node3` cluster confirmed untouched throughout
- [x] Three new VMs cloned, static IPs set via MAC-based netplan matching (after diagnosing a real stale-MAC bug, not the clean path originally planned)
- [x] `lxd`, `microceph`, and `microcloud` snaps installed on all three (MicroOVN deliberately excluded); `lxd` required a fresh install, contrary to assumption
- [x] `microcloud init` run successfully; mDNS discovery observed live
- [x] Ceph storage bootstrapped — required a second disk per node after the first was silently consumed by local storage
- [x] LXD storage pool manually bridged to Ceph after `init` skipped that step
- [x] A container actually launched using the new Ceph-backed storage pool (after one transient retry) and deleted cleanly
- [x] Comparison note: effort here vs. what Week 2 took to build LXD clustering alone
- [x] Retention check answered (MicroOVN addition in Week 4 — see below)

---

## Comments (conceptual explanation thread)

**NwaChi:**
Ready to actually run `microcloud init` now instead of just reading about it. Want to do it on clean nodes so I'm not risking the Week 2 cluster.

**Senior Infra Engineer (review):**
Right call. Use MAC-address matching in netplan this time, not interface-name matching — `node2` drifted to DHCP after a power cycle specifically because its static config was bound to a name, not a MAC. Skip MicroOVN deliberately this week too, so you see `init` skip the network prompts live.

**NwaChi (mid-lab):**
Two things came up that weren't in the plan. First, the netplan file I inherited on `mc2` had a MAC that matched neither real interface — turned out `mc2`'s host-only adapter is `ens33`, not `ens37` like `node2` had. Second, `init` only gave Ceph zero disks even though I answered yes to the storage question — it used my one disk for something called "local storage" instead.

**Senior Infra Engineer:**
Both are worth sitting with. The interface-naming issue confirms this isn't a one-off quirk of `node2` — it's a property of how VMware enumerates virtual NICs, full stop. Treat MAC-based matching as the default from now on, not a special case.

The storage split is more subtle and more important: local storage (one disk, one node, no replication) and distributed storage (Ceph, needs disks across multiple nodes) are genuinely different products competing for the same hardware. `init` doesn't know your intent — it just does what you told it, which was "yes, set up local storage," using the only disk available. Nothing was misconfigured. You got exactly what you asked for; it just wasn't what you wanted. That distinction — system did what you said vs. system did what you meant — is worth remembering far beyond MicroCloud.

**NwaChi:**
And then even after Ceph had the disk, LXD didn't know the pool existed until I told LXD directly. Two separate systems, two separate configs — same lesson as the first one, really.

---

## Implementation (as actually run)

### Phase 0 — Clone and rename

```bash
# In VMware: clone base-vm-vmware three times → mc1, mc2, mc3
```
Then, on each VM:
```bash
sudo hostnamectl set-hostname mc1   # mc2, mc3 respectively
```
(Also renamed in VMware's library pane — cosmetic only, doesn't affect the VM itself.)

### Phase 1 — First disk (later found insufficient — see Phase 5)

```
VM Settings → Add → Hard Disk → 8 GB → Store as a single file
```
Added to all three VMs before first boot.

### Phase 2 — Install LXD (not pre-installed, contrary to assumption)

```bash
sudo snap install lxd --channel=5.21/stable --cohort="+"
```
Run on `mc1`, `mc2`, `mc3` after discovering `snap list lxd` returned nothing.

### Phase 3 — Static IPs, MAC-based, after diagnosing a real stale-config bug

Found the real host-only interface and its MAC via:
```bash
ip a
```
(confirmed `ens33` on `mc2`, not `ens37` — opposite of `node2`). Corrected netplan:
```yaml
network:
  version: 2
  ethernets:
    ens33:
      match:
        macaddress: "00:0c:29:47:0a:85"
      set-name: ens33
      addresses: [192.168.20.141/24]
```
Removed the conflicting `dhcp4`/`dhcp6` lines from the inherited config. Applied:
```bash
sudo netplan apply
```
(Appeared to hang over the SSH session using the same interface — confirmed successful via a fresh connection, not a real failure.)

### Phase 4 — Install remaining snaps (MicroOVN deliberately excluded)

```bash
sudo snap install microceph --channel=squid/stable --cohort="+"
sudo snap install microcloud --channel=2/stable --cohort="+"
```

### Phase 5 — `microcloud init`, real output

```bash
sudo microcloud init
```
```
Verify the fingerprint 4209e1f01605 is displayed on joining systems.
Selected mc1 at 192.168.20.140
Selected mc2 at 192.168.20.141
Selected mc3 at 192.168.20.142
Would you like to set up local storage? (yes/no) [default=yes]:
Using /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0 on mc1 for local storage pool
Using /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0 on mc2 for local storage pool
Using /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0 on mc3 for local storage pool
! Warning: No disks available for distributed storage. Skipping configuration
Local MicroCloud is ready
Local MicroCeph is ready
Local LXD is ready
```
Confirmed with `sudo microceph status`: `Disks: 0` on all three, matching the warning exactly.

### Phase 6 — Add real Ceph storage (second disk, manual add)

Added a second 10 GB disk per VM, then on each node:
```bash
sudo microceph disk add /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:2:0
```
`microceph status` afterward: `osd` present in services, `Disks: 1` on all three.

### Phase 7 — Bridge Ceph to LXD (the step `init` normally handles, done by hand)

```bash
lxc storage create remote ceph --target mc1
lxc storage create remote ceph --target mc2
lxc storage create remote ceph --target mc3
lxc storage create remote ceph
```
`lxc storage list` then showed `remote / ceph / CREATED`.

### Phase 8 — Prove it with a real container

```bash
lxc launch ubuntu:24.04 test-ceph-container --storage remote
```
First attempt failed with a transient dqlite error; retried with no changes and succeeded. Verified:
```bash
lxc list
```
then cleaned up:
```bash
lxc delete test-ceph-container --force
```

### Phase 9 — Compare against Week 2

Week 2 took multiple tickets (INFRA-006 through 008) to get LXD clustering, quorum, and failure detection working for compute alone. This ticket got compute and storage clustered in one guided `init` run — but "one guided run" still needed two manual corrections when the defaults didn't match intent. Worth remembering going into Week 4: MicroCloud reduces the *amount* of manual work, not the need to understand what each layer is actually doing.

---

## Lab-vs-reality gap

Two gaps surfaced this session that don't show up in the "why MicroCloud" pitch:

1. **Local vs. distributed storage are competing claims on the same disks**, and `init`'s default answer can silently pick one for you. Planning disk count per node matters before you start, not after, one disk per node gets you local storage or nothing; three-plus nodes each need at least one disk specifically earmarked for Ceph if distributed storage is the actual goal.
2. **MicroCloud's automation has a boundary.** When a component's setup gets skipped (here, distributed storage, because no disk was available), the pieces it would have wired together don't get wired, you inherit that integration work manually, with no warning beyond the one-line message at init time. "MicroCloud bundles these for you" is true right up until something gets skipped, and then you're back to doing exactly what Week 2 already taught: configuring each system and connecting them yourself.

---

## Follow-ups (still open)

1. Netplan MAC-address fix for `node1`–`node3` (flagged in INFRA-008, not yet actioned).
2. INFRA-007 housekeeping: leftover test cluster groups, retiring old VirtualBox VMs.
3. **New**: when planning future MicroCloud labs, provision two disks per node up front if distributed storage is the goal — or explicitly answer "no" to the local storage prompt so the single disk goes to Ceph instead.

---

## Definition of Done

- [x] `mc1`–`mc3` cloned, static IPs set via MAC-matched netplan, confirmed to survive reboot
- [x] Required snaps installed (LXD required an unplanned fresh install)
- [x] `microcloud init` completed, live mDNS discovery observed
- [x] Real Ceph storage achieved after diagnosing and fixing the local-storage disk conflict
- [x] LXD storage pool manually bridged to Ceph
- [x] Test container launched (after one transient retry) and deleted successfully
- [x] Week 2 vs. Week 3 effort comparison written, including the manual corrections needed
- [x] Retention check answered

---

## Retention check

**Question:** We deliberately did not install MicroOVN this week, and `microcloud init` skipped the networking prompts as a result. What specifically do you expect to be different about the `init` flow once MicroOVN is added in Week 4 — and why does MicroCloud check for its presence rather than just always asking those questions?

