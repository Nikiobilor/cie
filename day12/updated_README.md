# INFRA-012: CRUSH Maps — Where Your Data Actually Lives

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-012 |
| **Priority** | Medium |
| **Component** | MicroCloud / Ceph CRUSH Placement |
| **Environment** | VMware Workstation Pro (Windows host) — 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`) |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 3 — MicroCloud Compute/Storage (Day 13) |
| **Linked tickets** | INFRA-010 (storage bootstrap), INFRA-011 (failure resilience) — this ticket explains the mechanism behind both. |

---

## Summary

INFRA-011 proved Ceph survives losing one node because it keeps three replicas of everything. What it didn't explain is *how Ceph decides which node gets which replica* — and specifically, how it guarantees two replicas never end up on the same node by accident. That decision is made by an algorithm called **CRUSH** (Controlled Replication Under Scalable Hashing). This ticket looks at it directly.

---

## Acceptance Criteria

- [ ] Can explain, in plain English, what CRUSH stands for and what problem it solves
- [ ] Viewed the cluster's real CRUSH hierarchy — not assumed it
- [ ] Identified the actual failure domain the cluster is using, from real output
- [ ] Traced where a specific piece of data would actually be placed
- [ ] Can explain why INFRA-011's one-node failure only ever cost exactly one replica, using CRUSH's failure domain as the reason
- [ ] Retention check answered

---

## Comments (conceptual explanation thread)

**NwaChi:**
INFRA-011 showed losing one node only cost one replica out of three, never two. That felt like it was working correctly, but I never actually confirmed *why* it worked, versus just gotten lucky.

**Senior Infra Engineer (review):**
Good thing to chase down, because it's not luck — it's a deliberate placement decision, made by CRUSH.

Here's the plain-English version. When Ceph needs to store a piece of data three times (per `size 3`), it needs to decide which three OSDs (disks) get a copy. The naive way to do that would be a lookup table — a big list mapping every piece of data to its exact location, stored somewhere central. That's fragile: the table itself becomes a single point of failure, and it has to be updated and distributed every time anything changes.

CRUSH avoids that entirely. Instead of looking anything up, every node *calculates* the answer independently, using the same formula and the same starting inputs — the object's name and the current cluster map. Feed the same inputs into the same formula anywhere, you get the same answer, with nothing to look up and nothing to keep in sync.

The part that actually protects you from correlated failures is the **failure domain**. CRUSH's hierarchy models your physical layout — by default: root, then host, then OSD. When placing three replicas, the default rule tells CRUSH: don't just pick three different OSDs, pick three OSDs from three *different hosts*. That's the actual guarantee behind what you saw in INFRA-011 — it's not that you got unlucky in a good way, it's that the rule made two-replicas-on-one-node impossible by construction.

**NwaChi:**
So the hierarchy and the rule are two separate things — the hierarchy describes what physically exists, the rule decides how many hops apart replicas have to be.

**Senior Infra Engineer:**
Exactly. Go look at both directly rather than taking that on faith.

---

## Implementation Runbook

### Phase 1 — View the real hierarchy

From any node:
```bash
sudo microceph.ceph osd tree
```
This lists every OSD, which host it belongs to, and its current status. You should see three hosts (`mc1`, `mc2`, `mc3`), each with exactly one OSD underneath it — matching what INFRA-010 built.

For the CRUSH-specific view of the same structure:
```bash
sudo microceph.ceph osd crush tree
```
Look at the shape: a `root` bucket (usually named `default`) at the top, `host` buckets underneath it for each node, and individual OSDs underneath their host. That nesting — root → host → osd — *is* the CRUSH hierarchy. Nothing abstract about it; it's just your real cluster, described as a tree.

---

### Phase 2 — Find the rule that actually uses this hierarchy

A hierarchy alone doesn't decide anything — a **rule** tells CRUSH how to use it. List the rules:
```bash
sudo microceph.ceph osd crush rule ls
```
Dump the one backing your storage pool (likely named `replicated_rule` or similar):
```bash
sudo microceph.ceph osd crush rule dump <rule-name>
```
In the output, look for a `"type"` field with value `"host"` — that's your confirmation, from real output, that this cluster's failure domain is indeed "host," not "osd" or "rack." If it said `"osd"` instead, two replicas could legally end up on the same physical machine — worth knowing that's a configurable choice, not a Ceph law of nature.

---

### Phase 3 — Trace where a specific piece of data actually goes

CRUSH placement can be checked directly, without needing real data to exist yet — the calculation only needs a pool name and an object name as input:
```bash
sudo microceph.ceph osd map remote test-crush-lookup
```
The output shows something like `up ([X,Y,Z], pZ)` — those numbers are the actual OSD IDs chosen for this object's three replicas. Cross-reference those IDs against Phase 1's `osd tree` output and confirm they land on three different hosts, not just three different OSD numbers.

Try it again with a different object name:
```bash
sudo microceph.ceph osd map remote another-test-object
```
The specific OSDs chosen will likely differ from the first lookup — different input, different calculated output — but the "three different hosts" guarantee holds every time, because that guarantee comes from the rule, not from the specific object.

---

### Phase 4 — Connect this back to INFRA-011

Re-read INFRA-011's result with this in mind: when one node went offline, Ceph reported exactly one replica missing per affected placement group, never two. That wasn't fortunate timing — every single object in the pool was already guaranteed, by the failure-domain rule confirmed in Phase 2, to have its three copies spread across three separate hosts. Losing one host could only ever cost one copy, for every object in the pool, by construction — not by chance.

---

## Definition of Done

- [ ] `osd tree` and `crush tree` output reviewed, hierarchy matches the real 3-host, 3-OSD cluster
- [ ] Crush rule dumped, failure domain confirmed as `host` from real output
- [ ] At least two different object names mapped with `osd map`, confirming three-different-hosts placement each time
- [ ] Written connection made between this ticket's findings and INFRA-011's observed behavior
- [ ] Retention check answered

---

## Retention check

If a fourth node (`mc4`) were added to this cluster tomorrow, would CRUSH automatically start placing some new replicas on it, or would it need to be told to? Base your answer on where in the hierarchy a new host would land, and where the `root` bucket sits relative to everything you looked at in Phase 1.