# INFRA-013 — Migrate a Running Container Across the MicroCloud Cluster

**Priority:** P2 — Week 3 capstone / demo day
**Component:** Compute + Storage (MicroCloud: LXD, MicroCeph)
**Environment:** VMware Workstation Pro (Windows), 3-node MicroCloud cluster — `mc1`, `mc2`, `mc3`, each running LXD + MicroCeph + dqlite, static IPs bound by MAC in netplan, Ceph-backed LXD storage pool `remote`
**Linked tickets:** Closes out Week 3. Builds on INFRA-010 (pool created + bridged to LXD), INFRA-011 (replication floor: `size 3`, `min_size 2`), INFRA-012 (CRUSH failure domain = `host`)

---

## Summary

All of Week 3 has been proving one claim in pieces: once storage lives on Ceph instead of on any single node's disk, that storage stops caring which node is running the workload. INFRA-010 built the pool. INFRA-011 proved the pool survives a node dying. INFRA-012 proved *why* — replicas are spread across hosts, not just across disks.

INFRA-013 is where that claim gets used. We take a container that's actually running, actually holding data, and move it from `mc1` to `mc2` — while it's live, or as close to live as LXD containers actually get. Along the way we'll hit a very real, very common wall (CRIU) and use that wall to learn what "live migration" actually means for containers vs. what marketing slides imply it means.

---

## Acceptance Criteria

- [ ] A running container (`migrationtest`) exists on `mc1`, backed by the `remote` Ceph pool, holding a known test file
- [ ] A naive live-move attempt (`lxc move` on a running container, CRIU untouched) is run and its real output is captured and explained
- [ ] Container is relocated to `mc2` using the realistic pattern: stop → move → start
- [ ] Post-migration, the test file is confirmed byte-for-byte present on `mc2` (proof no data was lost or freshly copied)
- [ ] The stop-to-start downtime window is measured and recorded
- [ ] Ceph pool utilization is confirmed unchanged before/after (proof no bulk data copy occurred — only the "who's allowed to run this" pointer moved)
- [ ] *(Stretch, optional)* CRIU enabled and a true live migration attempted against a minimal, non-systemd, network-device-free container
- [ ] Recap explicitly ties the result back to INFRA-010/011/012 — why this was possible
- [ ] Lab-vs-reality gap documented: LXD's actual level of container live-migration support vs. the "live migration" language people throw around casually

---

## Comments Thread

**Reviewer:** Before you touch `mc2` — what do you *expect* to happen if you run `lxc move` on a container that's currently running?

**NwaChi:** It should just move over, no? That's the whole point of putting it on shared storage instead of local disk.

**Reviewer:** That's true for VMs. LXD supports proper zero-downtime live migration for virtual machines. Containers are a different story — LXD can only live-migrate a *running* container using something called CRIU (Checkpoint/Restore In Userspace), it's shipped but disabled by default, and even when you turn it on, Canonical's own docs say it reliably works only for very basic containers — no systemd, no network device. A normal Ubuntu container running an actual service doesn't qualify. Go try it anyway. Run the naive move on a running container and read exactly what LXD tells you.

**NwaChi:** *(runs it)* — Got an error. Ok, so it refused outright.

**Reviewer:** Good. That error is doing you a favor — it's the same wall every team hits the first time someone says "just live-migrate it" without checking. Here's the reframe: the interesting thing today was never going to be zero-downtime motion. It's that the container's *data* was never welded to `mc1` in the first place. Stop it, move it, start it, and time the whole thing.

**NwaChi:** *(does it)* — That was fast. Way faster than I expected for something that's supposedly "moving."

**Reviewer:** Because nothing moved. Nothing had to be copied over the network. The `remote` pool — the one INFRA-010 built — is already visible from every node in the cluster. "Moving" the container is really just LXD updating cluster metadata: *mc2 is now the one allowed to run this instance*, then starting a process on `mc2` that opens the exact same Ceph RBD volume `mc1` was using a second ago. Compare that to what would've happened if `migrationtest` had been sitting on `mc1`'s local disk instead — LXD would've had to copy the entire filesystem across the network before anything could start on `mc2`. Seconds vs. potentially minutes, depending on data size.

**NwaChi:** So does INFRA-011's `size 3, min_size 2` matter here at all, or was that just about node failure?

**Reviewer:** It matters more than you'd think. While `migrationtest` was mid-move, its data was still sitting on Ceph, replicated across all three hosts per INFRA-012's failure domain. If a node had died in that exact window, the pool would've stayed writable per INFRA-011. The migration wasn't a moment of fragility — the data was exactly as protected during the move as it was before and after it. That's the actual payoff of the week: decoupling storage from a node doesn't just survive failures, it makes routine operations like this boring instead of risky.

---

## Implementation Runbook

### Step 1 — Confirm the cluster is healthy before you start

Plain English: don't debug a migration on top of a cluster you haven't confirmed is actually fine right now.

```
lxc cluster list
sudo microceph.ceph status
```

Check that all three nodes show as `ONLINE` and Ceph shows `HEALTH_OK` (or at least not `HEALTH_ERR`). If MicroCeph shows anything else, fix that before continuing — don't migrate on top of a degraded pool.

### Step 2 — Launch a test container on the shared `remote` pool

Plain English: we need something real to move — a running container that's actually reading/writing to Ceph, not local disk, so the migration means something.

```
lxc launch ubuntu:24.04 migrationtest --target mc1 -s remote
```

`--target mc1` pins it to a specific node so we know exactly where it's starting from. `-s remote` is explicit about the storage pool — don't rely on it being the default, confirm it.

Wait for it to report `RUNNING`:

```
lxc list migrationtest
```

Now write a known file inside it — this is our proof that data survives the move untouched:

```
lxc exec migrationtest -- bash -c 'echo "written on mc1 at $(date)" > /root/proof.txt'
lxc exec migrationtest -- cat /root/proof.txt
```

Confirm the second command prints back what you just wrote. Note the exact timestamp — you'll compare it after the move.
Now capture a storage baseline, before anything moves:
sudo microceph.ceph df

Note the USED value for the remote pool. This is your "before" number — you'll compare Step 6's result against it to prove the migration didn't copy any data.
### Step 3 — Attempt the naive move (expect this to fail — that's the point)

Plain English: try the "obvious" command on a running container and read whatever LXD actually tells you, rather than assuming.

```
lxc move migrationtest --target mc2
```

CRIU ships with the LXD snap but is disabled by default (`criu.enable=false`), so this will almost certainly refuse to move a running instance. The exact wording can differ across LXD versions — read what you actually got. If it doesn't clearly say something about the instance needing to be stopped, paste the output back before continuing so we can adjust.

### Step 4 — Do the realistic migration: stop → move → start

Plain English: this is the pattern every real team actually uses for container migration. It's not zero-downtime, but because the disk is already shared, the downtime is measured in seconds, not minutes.

```
lxc stop migrationtest
time lxc move migrationtest --target mc2
lxc start migrationtest
```

The `time` prefix on the move command gives you a real number for how long the storage-pointer handoff itself took. Note it — this is the number you'll compare against a local-disk migration in your writeup.

### Step 5 — Verify the container actually landed on `mc2`, with its data intact

```
lxc list migrationtest
```

Confirm the location column now shows `mc2`.

```
lxc exec migrationtest -- cat /root/proof.txt
```

Confirm it prints the exact same timestamp from Step 2 — not a fresh file, the *same* file. That's your proof this was a relocation, not a rebuild.

### Step 6 — Confirm no bulk data copy actually happened

Plain English: if the migration had needed to copy the container's disk, Ceph's usage numbers would show a spike. They shouldn't have moved at all here.

```
sudo microceph.ceph df
```

Compare the `remote` pool's used space against what you'd expect from before the move (roughly unchanged, aside from normal container activity). No new full copy of `migrationtest`'s disk was created — the same Ceph RBD volume just got a new node reading and writing it.

### Step 7 (Stretch, optional) — Try genuine CRIU live migration on a minimal container

Plain English: this is the "ideal case" LXD's docs describe — it's real, it's just narrow. Worth seeing once, not required for this ticket to be done.

Enable CRIU on both `mc1` and `mc2`:

```
sudo snap set lxd criu.enable=true
sudo systemctl reload snap.lxd.daemon
```

Launch a minimal container and strip its network device, since Canonical's own docs say reliable CRIU support is limited to non-systemd containers without a network device:

```
lxc launch images:alpine/edge migrationtest2 --target mc1 -s remote
lxc config device remove migrationtest2 eth0
```

Attempt a move while it's running:

```
lxc move migrationtest2 --target mc2
```
This may succeed, or it may fail with a CRIU-specific error (kernel/CRIU version mismatches are common and well-documented in LXD's own community forum). Either outcome is a valid, useful field note — record whichever one you get.


To clean up run the following commands
lxc delete migrationtest --force
lxc delete migrationtest2 --force

Confirm both are gone
lxc list
---

## Definition of Done

- [ ] `migrationtest` relocated from `mc1` to `mc2` via stop → move → start, backed throughout by the `remote` Ceph pool
- [ ] Naive live-move attempt on a running container documented with its real output
- [ ] `proof.txt` confirmed identical (same timestamp) before and after the move
- [ ] Downtime window from `time lxc move` recorded
- [ ] Ceph pool utilization confirmed unchanged pre/post migration
- [ ] Stretch CRIU attempt documented (success or failure, either is fine) — optional
- [ ] Recap written connecting this result back to INFRA-010 (the pool), INFRA-011 (the replication floor), INFRA-012 (the host failure domain)
- [ ] Lab-vs-reality gap written up: real LXD container live-migration support vs. the loose way "live migration" gets used casually
- [ ] Week 3 formally closed

---

## Retention Check

If `migrationtest`'s storage had been sitting on `mc1`'s **local** disk instead of the shared `remote` Ceph pool, what would this migration have required instead of stop → move → start, and roughly how much longer would you expect it to take compared to what you actually measured?