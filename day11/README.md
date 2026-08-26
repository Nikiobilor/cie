# INFRA-011: Ceph Storage Resilience — Surviving an OSD/Node Failure

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-011 |
| **Priority** | High |
| **Component** | MicroCloud / Ceph Storage Resilience |
| **Environment** | VMware Workstation Pro (Windows host) — 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`) from INFRA-010 |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 3 — MicroCloud Compute/Storage (Day 11) |
| **Linked tickets** | INFRA-010 (compute + storage bootstrap) — this ticket tests what INFRA-010 built. INFRA-007 (LXD quorum test) — this ticket is the storage-layer equivalent. |

---

## Summary

INFRA-010 proved Ceph-backed storage works when every node is healthy. It hasn't been tested against failure yet. This ticket does that directly — take a node down, observe what Ceph reports and whether data stays accessible, take a second node down, observe where it actually breaks, then recover.

This mirrors INFRA-007's two-phase LXD quorum test almost exactly, but the failure mechanism underneath is different. LXD/dqlite uses a majority vote (quorum). Ceph uses a configured replica floor (`min_size`). Both happen to tolerate exactly one node failure in a 3-node setup — but for different reasons, which is worth understanding precisely rather than assuming "clustering just works like that."

---

## Acceptance Criteria

- [ ] Can explain, in plain English, what `size` and `min_size` mean for a Ceph pool
- [ ] Confirmed the pool's actual replication settings from real output, not assumed defaults
- [ ] One node taken offline; cluster health and data accessibility confirmed directly, not assumed from status text alone
- [ ] A second node taken offline; the point where I/O actually blocks confirmed directly
- [ ] Both nodes recovered; cluster confirmed to return to full health
- [ ] Can explain why LXD's quorum and Ceph's min_size are mechanically different, despite both tolerating exactly one failure here
- [ ] Retention check answered

---

## Comments (conceptual explanation thread)

**NwaChi:**
INFRA-010 got real Ceph storage working, but I've only ever seen it healthy. Want to actually break it on purpose and watch what happens, the same way INFRA-007 did for the LXD cluster.

**Senior Infra Engineer (review):**
Good instinct, and worth being precise about the concept before you pull anything offline. Ceph doesn't do a leader election or a majority vote like dqlite does. Every piece of data in a Ceph pool is stored as multiple copies, replicas, across different nodes. Two settings control this:

- `size` — how many copies of the data Ceph *wants* to maintain. Default is 3.
- `min_size` — the minimum number of copies that must be available before Ceph will accept reads or writes at all. Default is 2.

So with 3 nodes and `size=3, min_size=2`: lose one node, you're down to 2 copies. That's still ≥ `min_size`, so the cluster stays writable — just "degraded" instead of fully healthy. Lose a second node, you're down to 1 copy. That's below `min_size` 2, so Ceph blocks I/O outright to protect against data loss, rather than risk writing to a copy that might vanish next.

Compare that to dqlite: there's no "how many copies of the data exist" concept at all. It's a vote — do more than half the members agree on the current state. With 3 members, that's 2 of 3. Losing 1 still has a majority; losing 2 doesn't.

Both land on "tolerates exactly 1 of 3" here, but that's a coincidence of the numbers you chose (3 nodes), not the same underlying logic. Change the node count and they'd diverge — a 5-node dqlite cluster tolerates 2 failures (needs 3 of 5), while a Ceph pool's tolerance depends entirely on what `size`/`min_size` you configured, not the node count directly.

**NwaChi:**
So testing this properly means watching for two different things: does the cluster stay *writable* when I pull one node, and does it *actually stop* — not just warn — when I pull a second.

**Senior Infra Engineer:**
Exactly. And check real output for both, the same habit from every prior ticket — don't assume `HEALTH_WARN` versus `HEALTH_ERR` from memory, read what the cluster actually reports at each step.

---

## Implementation Runbook

### Phase 0 — Baseline health check

From any node:
```bash
sudo microceph status
sudo microceph.ceph status
```
Confirm all three nodes show `osd` in services, `Disks: 1`, and overall health is `HEALTH_OK`. This is your known-good state to compare against later.

---

### Phase 1 — Confirm real replication settings (don't assume the defaults)

```bash
sudo microceph.ceph osd pool ls detail
```
Find the pool backing your `remote` LXD storage pool (likely named `remote` or similar) and read its actual `size` and `min_size` values directly from the output — confirm they match `size 3 min_size 2` rather than assuming it from documentation.

---

### Phase 2 — Take one node offline, confirm degraded-but-writable

Power off `mc3` in VMware Workstation (not gracefully inside the VM — pull it, the same way `node2`/`node3` were pulled for the LXD quorum test).

From `mc1`:
```bash
sudo microceph.ceph status
```
Expect `HEALTH_WARN`, not `HEALTH_ERR` — and look specifically for PGs reported as `active+degraded` rather than `inactive`. `active` means still serving I/O; `degraded` means running below the target replica count. That distinction is the entire point of `min_size` — it's still working, just not at full redundancy.

Prove it's actually writable, not just reporting a hopeful status:
```bash
lxc launch ubuntu:24.04 test-degraded-write --storage remote
lxc list
```
If this succeeds with only 2 of 3 nodes up, that's `min_size` doing exactly what it's designed to do.

---

### Phase 3 — Take a second node offline, confirm I/O actually blocks

Power off `mc2` as well, leaving only `mc1` up.

```bash
sudo microceph.ceph status
```
Expect `HEALTH_ERR` this time, and PGs reported as `inactive` rather than `active+degraded` — the meaningful difference from Phase 2.

Try a write:
```bash
lxc launch ubuntu:24.04 test-blocked-write --storage remote
```
Expect this to hang or fail outright — same "reads/writes stall below the floor" behavior as the quorum-loss hang from INFRA-008, but for a completely different reason underneath. If it hangs, `Ctrl+C` after confirming the behavior rather than waiting indefinitely.

---

### Phase 4 — Recover both nodes

Power `mc2` and `mc3` back on, wait for boot, confirm each is reachable and the OSD is back:
```bash
sudo microceph status
```
From `mc1`, watch health return to normal:
```bash
sudo microceph.ceph status
```
You should see Ceph actively re-syncing (`recovering` or `backfilling` in the output) before settling back to `HEALTH_OK` — that resync is Ceph catching the previously-offline OSDs back up to the current data, not an instant flip.

Clean up the test containers once healthy:
```bash
lxc delete test-degraded-write --force
lxc delete test-blocked-write --force 2>/dev/null || true
```

---

### Phase 5 — Compare the two failure mechanisms directly

Write down, in your own words, the actual difference observed:
- **LXD/dqlite (INFRA-007)**: failure is about *agreement* — a vote needs a majority to trust the current state.
- **Ceph (this ticket)**: failure is about *copies* — I/O needs a minimum number of surviving replicas to be considered safe.

Both gave you "1 of 3 survivable" here only because of the specific numbers chosen (3 nodes, `min_size=2`). That's worth remembering the next time you size a cluster — the failure tolerance isn't a fixed property of "clustering," it's a direct consequence of the specific configuration you choose.

---

## Definition of Done

- [ ] Real pool replication settings confirmed from actual output
- [ ] One-node failure tested: `HEALTH_WARN`, PGs `active+degraded`, write succeeded
- [ ] Two-node failure tested: `HEALTH_ERR`, PGs `inactive`, write blocked/hung
- [ ] Both nodes recovered, cluster confirmed back to `HEALTH_OK` with resync observed
- [ ] Test containers cleaned up
- [ ] Written comparison of dqlite quorum vs. Ceph min_size completed
- [ ] Retention check answered

---

## Retention check

LXD's cluster and this Ceph pool both tolerated exactly one node failure out of three. Are they protected by the same underlying mechanism, or two different ones that just happen to produce the same number at this cluster size? What would happen to each one's failure tolerance if you scaled from 3 nodes to 5?