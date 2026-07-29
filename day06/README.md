# 🎫 INFRA-006: Document clustering fundamentals and quorum model ahead of Week 2 rollout

**Type:** Task
**Priority:** P3 — Planned
**Component:** Platform / Virtualization
**Sprint:** Week 2, Day 1
**Status:** To Do → In Progress
**Reporter:** Platform Lead
**Assignee:** You

---

## Description

Before the team stands up a two-node LXD cluster later this sprint (see INFRA-007, INFRA-010), we need a documented, verified understanding of how clustering and quorum actually behave, plus one provisioned node ready to be joined once clustering work begins.

This ticket does not stand up a cluster. It prepares the ground for one.

---

## Acceptance Criteria

- [ ] `node1` provisioned from the team's golden image (`base-vm`)
- [ ] LXD installed and confirmed to be running standalone (not clustered by default)
- [ ] Quorum size and fault tolerance documented for cluster sizes 1–5
- [ ] Quorum size and fault tolerance calculated, not looked up, for a 7-member cluster, to be verified against real behavior once clustering is live
- [ ] `node1` left powered off, not deleted, ready for INFRA-007

---

## Comments

**Platform Lead:**
Wanted to leave some context before you pick this up, since the "why" matters more than the commands here.

Everything built so far this challenge lived on one machine, so there was never a disagreement to resolve — one kernel, one bridge, one storage pool. The moment we're running more than one machine as a group, a new failure mode shows up that a single machine never has to deal with: if the network between members drops mid-operation and the group splits into two halves, what stops both halves from assuming they're the only ones left and continuing to accept writes independently? That's called split-brain, and it's the reason every real clustering system, LXD included, enforces a rule called quorum: before any member, or group of members, is allowed to act, a majority of the total cluster has to be reachable and in agreement. Not "some" — a majority. That guarantees only one side of any split can ever hold quorum at once, so only one side is ever allowed to keep working.

The math is simpler than it sounds: quorum size equals half the total membership, rounded down, plus one.

| Total members | Quorum needed | Failures tolerated |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |

Look at rows 3 and 4 closely before you file this ticket as done. A 4-member cluster costs an extra machine over a 3-member one, and tolerates exactly the same single failure. That's not a rounding artifact, it's the actual reason production clusters are so often sized at 3 or 5 rather than 4 or 6 — odd numbers give the best fault tolerance per machine you're paying for.

Work out the 7-member row yourself before closing this out. Don't look it up, we'll verify it against real cluster behavior once INFRA-007 has two nodes actually joined.

---

## Implementation Notes / Runbook

**Environment note:** `base-vm` is already sized at 20GB with SSH access baked in — no per-clone disk resize needed anymore now that the golden image itself was resized at the source.

**1. Provision `node1`**
```
VBoxManage clonevm "base-vm" --name "node1" --register
VBoxManage startvm "node1" --type headless
VBoxManage controlvm "node1" natpf1 "node1ssh,tcp,,2206,,22"
```
```
ssh labuser@127.0.0.1 -p 2206
```

**2. Install LXD, standalone — clustering is not on by default**
```
sudo snap install lxd
sudo lxd init --auto
```

**3. Confirm current state before assuming anything**
```
lxc cluster list
```
Expect an error confirming this server isn't part of a cluster. That's the correct, current state — log it as evidence for the acceptance criteria rather than skipping the step because you already know the answer.

**4. Do the math for the missing row**

Work out quorum size and failures tolerated for a 7-member cluster using the formula above. Write the answer down — it becomes the validation check on INFRA-007.

**5. Leave the environment in the correct state for the next ticket**
```
exit
```
```
VBoxManage controlvm "node1" poweroff
```
Do not delete `node1`. INFRA-007 depends on this exact instance still existing.

---

## Definition of Done

`node1` exists, is provisioned, and is powered off, not deleted. You can state from memory, without re-deriving it, what quorum means, why it prevents split-brain, and why odd-numbered clusters are the efficient choice. If you can't explain the "why" without looking back at the table, this ticket isn't actually done yet, even if every command above ran clean.

**Next ticket:** INFRA-007 — Join `node1` and a new second node into an active LXD cluster.