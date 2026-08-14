# 🎫 INFRA-007: Join node1 and node2 into an active LXD cluster

**Type:** Task
**Priority:** P2 — Blocks INFRA-008
**Component:** Platform / Virtualization / Networking
**Sprint:** Week 2, Day 2
**Status:** To Do → In Progress
**Reporter:** Platform Lead
**Assignee:** You
**Depends on:** INFRA-006 (`node1` provisioned, quorum math documented)

---

## Description

INFRA-006 documented what quorum means and provisioned `node1` standalone. This ticket forms an actual two-member LXD cluster and validates the quorum math against real behavior, not just a table.

**Scope note added during grooming:** the original plan assumed nodes could be joined before inter-node networking was set up. That's backwards, VirtualBox's default NAT isolates every VM from every other VM, each one can reach the internet, none of them can reach each other. A cluster join needs direct reachability between members. So this ticket now includes standing up that networking as a blocking prerequisite, rather than leaving a gap that would've made INFRA-007 impossible to actually complete as originally scoped.

---

## Acceptance Criteria

- [ ] `node1` and `node2` can reach each other directly over a private network, not just out to the internet
- [ ] `node2` provisioned from `base-vm`
- [ ] LXD clustering enabled on `node1` as the founding member (not the `--auto` default from Week 1)
- [ ] `node2` joined to the cluster using a join token
- [ ] `lxc cluster list` shows both members, status ONLINE
- [ ] Quorum validated for real: `node2` powered off, `node1` observed losing the ability to make cluster changes alone
- [ ] Both nodes returned to a known state, documented, for INFRA-008

---

## Comments

**Platform Lead:**
Two things worth knowing before you start.

First, the networking. VirtualBox's NAT gives every VM its own private, isolated network by default, that's exactly what let each of last week's containers reach the internet without you configuring anything, but it also means `node1` and `node2` currently can't see each other at all. Fixing that means giving both VMs a second network adapter, a Host-only adapter, which VirtualBox uses specifically for VM-to-VM and VM-to-host traffic on your machine, separate from the NAT adapter that still handles their internet access.

Second, the quorum test. You calculated in INFRA-006 that a 2-member cluster has quorum of 2 and tolerates exactly 0 failures. Before you run the test below, say out loud what you expect to happen the moment one of the two nodes goes offline. Then go verify it. If your prediction and the real result don't match, that's more valuable than if they do, it means the math didn't actually stick yet.

---

## Implementation Notes / Runbook

Exact `lxd init` prompt wording can shift slightly between versions. Read what's actually on screen rather than blindly pasting answers.

**1. Set up a Host-only network on the host, if one doesn't exist yet**
```
VBoxManage list hostonlyifs
```
If that returns nothing, create one:
```
VBoxManage hostonlyif create
```

**2. Give `node1` a second adapter on that network**
```
VBoxManage modifyvm "node1" --nic2 hostonly --hostonlyadapter2 "VirtualBox Host-Only Ethernet Adapter"
```

**3. Provision `node2` the same way**
```
VBoxManage clonevm "base-vm" --name "node2" --register
VBoxManage modifyvm "node2" --nic2 hostonly --hostonlyadapter2 "VirtualBox Host-Only Ethernet Adapter"
```

**4. Start both, connect**
```
VBoxManage startvm "node1" --type headless
VBoxManage startvm "node2" --type headless
VBoxManage controlvm "node2" natpf1 "node2ssh,tcp,,2207,,22"
```
`node1`'s existing SSH rule from INFRA-006 (port 2206) survives a poweroff/start, no need to recreate it.
```
ssh labuser@127.0.0.1 -p 2206
```

**5. Find each node's host-only IP**
```
ip a
```
Look for the second interface (commonly `enp0s8`), it'll be on the `192.168.56.0/24` range VirtualBox uses by default. Note it down. Repeat on `node2` via port 2207.

**6. Enable clustering on `node1` — the founding member**
```
sudo lxd init
```
Answer yes to clustering, provide `node1`'s host-only IP as the address other members will use to reach it, and answer no to "are you joining an existing cluster" since this one creates the cluster.

Generate a join token for `node2`:
```
lxc cluster add node2
```
Copy the token it prints out.

**7. Join `node2` to the cluster**

On `node2`:
```
sudo snap install lxd
sudo lxd init
```
Answer yes to clustering, yes to joining an existing cluster, paste the token from step 6 when asked, and provide `node2`'s own host-only IP as its address.

**8. Confirm the cluster is real**
```
lxc cluster list
```
Both `node1` and `node2` should show, status ONLINE.

**9. Validate quorum against real behavior**

State your prediction first (see the comment above). Then:
```
VBoxManage controlvm "node2" poweroff
```
On `node1`, attempt a cluster write operation:
```
lxc launch ubuntu:24.04 quorum-test
```
With only one of two members reachable, quorum can't be met, expect this to hang or fail outright rather than succeed. That's not a bug, it's the exact behavior the table in INFRA-006 predicted.

**10. Bring it back and confirm recovery**
```
VBoxManage startvm "node2" --type headless
```
Wait for it to rejoin, then re-run `lxc cluster list` and retry the launch. Both should succeed once quorum is restored. Clean up the test instance if it succeeded on retry:
```
lxc delete quorum-test --force
```

**11. Leave both nodes running for INFRA-008**

Unlike Week 1, don't power these off. INFRA-008 continues directly on this same two-node cluster.

---

## Definition of Done

The cluster exists, shows two ONLINE members, and you've watched it actually refuse to act with only one node reachable, not just read that it would. You can explain, without checking notes, why a 2-member cluster has zero fault tolerance even though it technically has "two of everything."

**Next ticket:** INFRA-008 — build on the live two-node cluster.