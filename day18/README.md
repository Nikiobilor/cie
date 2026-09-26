# INFRA-016: Does a Container's Network Identity Survive a Move?

**Priority:** P2 (Week 4, Day 18)
**Component:** Networking × migration (OVN logical ports vs. flat bridge, across cluster members)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), MicroOVN clustered, `UPLINK` on dedicated NIC `ens38`, OVN networks `ovntest` and `ovntest2`, flat bridge `flattest`, Ceph pool `remote`
**Linked tickets:** INFRA-013 (stop → move → start on Ceph; CRIU live migration disabled), INFRA-015 (OVN vs. flat reachability). Sets up INFRA-017 (Day 19: density comparison)

---

## Summary

INFRA-013 proved a container can move between nodes without copying its disk, because Ceph already holds the data on every node. That was the storage half. Today is the network half: when a container moves to a different machine, does it keep its **identity**, meaning its MAC address, its IP address, and who it can reach?

This ticket tests it in both network models from INFRA-015. It moves an OVN container and a flat-bridge container, records what changed and what didn't, and measures how long the OVN container was unreachable during the move.

---

## Acceptance Criteria

- [ ] Before-and-after record for an OVN container (`ovn-a`): node, MAC address, IPv4 address
- [ ] Reachability from `ovn-c` to `ovn-a` measured continuously during the move, with the outage window recorded
- [ ] Evidence from OVN itself that the container's logical port moved to the new node
- [ ] Before-and-after record for a flat-bridge container (`flat-a`): node, MAC, IPv4, and reachability to `flat-b`
- [ ] Plain-English write-up: which parts of network identity survived a move in each model, and why
- [ ] Lab-vs-reality gap documented: cold move (stop, move, start) vs. true live migration

---

## Comments Thread

**Reviewer:** In INFRA-013 you moved a container and the data came with it. Here's what an application owner will actually ask you: "after the move, can my clients still find it at the same address?"

**NwaChi:** The data didn't move, only the running process did. So the IP should stay the same too?

**Reviewer:** Maybe, but work out why before you assume it. Where does the container's IP address come from in each model?

**NwaChi:** In OVN, from OVN's DHCP, and that's one service for the whole network. On the flat bridge, from each node's own DHCP server.

**Reviewer:** Right. So in one model, the thing handing out addresses doesn't care which node you're on. In the other, the container arrives on a node whose DHCP server has never seen it. Also think about the MAC address. Is it a property of the machine, or of the container?

**NwaChi:** LXD generates it for the container, so it probably travels with the container.

**Reviewer:** Probably. Check it, don't assume it. And think about INFRA-015: `flat-a` couldn't reach `flat-b` because they were on different nodes' bridges. What happens if `flat-a` moves onto `flat-b`'s node?

**NwaChi:** Then they'd share a bridge... so a move could change who it can reach?

**Reviewer:** That's the question. In one model, moving a workload shouldn't change its network at all. In the other, moving it might quietly change everything. Measure both.

**NwaChi:** And the outage? INFRA-013 showed true live migration is off for containers.

**Reviewer:** So this is a cold move: stop, move, start. There *will* be downtime. Measure how long, and don't treat it as what production live migration looks like. That's your lab-vs-reality note.

---

## Implementation Runbook

> **Standing rule:** record the real output at every step. The "Record" lines are questions to answer from your terminal, not predictions.

### Step 1: Inventory the starting state (`mc1`)

Plain English: before moving anything, write down exactly what each container looks like now, so "before" and "after" can be compared line by line.

```bash
lxc list
```

If `flat-a` / `flat-b` were deleted after INFRA-015, relaunch them:
```bash
lxc launch ubuntu:24.04 flat-a --network flattest --storage remote --target mc1
lxc launch ubuntu:24.04 flat-b --network flattest --storage remote --target mc2
```

Then, for `ovn-a` and `flat-a`:
```bash
lxc config get ovn-a volatile.eth0.hwaddr
lxc exec ovn-a -- ip -4 addr show eth0
lxc config get flat-a volatile.eth0.hwaddr
lxc exec flat-a -- ip -4 addr show eth0
```

`volatile.eth0.hwaddr` is the MAC address LXD generated for the container's `eth0`. "Volatile" keys are values LXD records for itself rather than settings you choose.

**Record:** node, MAC, and IPv4 for `ovn-a` and `flat-a`.

### Step 2: See where OVN thinks `ovn-a` lives (`mc1`)

Plain English: OVN keeps a "southbound" database that records which physical node (a "chassis") each logical port is bound to. This is OVN's own view of where the container is.

```bash
sudo microovn.ovn-sbctl show
```

**Record:** which chassis (node) lists the port belonging to `ovn-a`.

### Step 3: Start a continuous ping from `ovn-c` (second terminal)

Plain English: to measure the outage, keep pinging `ovn-a` the whole time it's moving.

```bash
lxc exec ovn-c -- ping -D -O <ovn-a IPv4>
```

- `-D` prints a timestamp on each line.
- `-O` prints a line for every ping that got no reply, so the gap is visible rather than silent.

Leave it running.

### Step 4: Move `ovn-a` from `mc1` to `mc2` (first terminal)

```bash
lxc stop ovn-a
lxc move ovn-a --target mc2
lxc start ovn-a
```

This is the same cold move as INFRA-013. Because the disk is on Ceph, `lxc move` only updates which node owns the container.

Once pings resume in the second terminal, stop the ping with `Ctrl+C`.

**Record:** the timestamp of the last reply before the gap, the first reply after it, and the length of the gap in seconds.

### Step 5: Compare `ovn-a` after the move

```bash
lxc list ovn-a
lxc config get ovn-a volatile.eth0.hwaddr
lxc exec ovn-a -- ip -4 addr show eth0
lxc exec ovn-a -- ping -c 3 8.8.8.8
sudo microovn.ovn-sbctl show
```

**Record:**
- Did the node change to `mc2`?
- Is the MAC the same as in Step 1?
- Is the IPv4 the same as in Step 1?
- Does it still reach the internet?
- Which chassis does OVN now bind its port to?

### Step 6: Repeat for the flat container

First, confirm the INFRA-015 baseline still holds:
```bash
lxc exec flat-a -- ping -c 3 <flat-b IPv4>
```

Then move `flat-a` onto `flat-b`'s node:
```bash
lxc stop flat-a
lxc move flat-a --target mc2
lxc start flat-a
```

Then compare:
```bash
lxc list flat
lxc config get flat-a volatile.eth0.hwaddr
lxc exec flat-a -- ip -4 addr show eth0
lxc exec flat-a -- ping -c 3 <flat-b IPv4>
```

**Record:**
- Is the MAC the same?
- Is the IPv4 the same? If not, why would a different DHCP server give a different answer?
- Can `flat-a` now reach `flat-b`? What changed, if the container itself didn't?

### Step 7: Write up the comparison

With real results from Steps 1 to 6, write a short paragraph for each model:
- Which parts of network identity survived the move (MAC, IP, reachability)?
- What mechanism explains each one? For example: where the MAC is stored, which DHCP server answered, and which bridge or logical switch the container landed on.
- For OVN: what did the chassis binding in Step 5 show OVN doing during the move?

**Lab-vs-reality gap:** this was a cold move, so the container was stopped, and any open connections to it were dropped. Production live migration moves a *running* workload with only a brief pause, keeping its connections open. In LXD, that's supported for virtual machines on shared storage (with `migration.stateful` enabled), not reliably for containers, as INFRA-013 showed. Keeping the same IP and MAC is what makes live migration *possible*, because clients never need to learn a new address. The outage measured today is the cost of the cold move, not of the network.

---

## Definition of Done

- [ ] Starting state recorded for `ovn-a` and `flat-a` (node, MAC, IPv4)
- [ ] OVN chassis binding recorded before and after the move
- [ ] Continuous ping captured, with the outage window recorded
- [ ] `ovn-a` after-state recorded, including an internet check
- [ ] `flat-a` moved, after-state recorded, and reachability to `flat-b` retested
- [ ] Write-up comparing both models, plus the lab-vs-reality note

---

## Retention Check

After the move, which model kept the container's network identity independent of the machine it runs on, and what single design choice explains the difference?ssh