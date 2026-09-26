# INFRA-016: Does a Container's Network Identity Survive a Move?

**Priority:** P2 (Week 4, Day 18)
**Component:** Networking × migration (OVN logical ports vs. flat bridge, across cluster members)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), MicroOVN clustered, `UPLINK` on dedicated NIC `ens38`, OVN networks `ovntest` and `ovntest2`, flat bridge `flattest`, Ceph pool `remote`
**Linked tickets:** INFRA-013 (stop → move → start on Ceph; CRIU live migration disabled), INFRA-015 (OVN vs. flat reachability). Sets up INFRA-017 (Day 19: density comparison)

> **Note:** this ticket was rewritten after the session to record the real results.

---

## Summary

INFRA-013 proved a container can move between nodes without copying its disk, because Ceph already holds the data on every node. That was the storage half. This ticket covers the network half: when a container moves to a different machine, does it keep its **identity**, meaning its MAC address, its IP address, and who it can reach?

**Result:** both containers kept their MAC and IP address, but only the OVN container kept its network. The flat-bridge container landed on a different machine's local bridge and could suddenly reach a neighbour it couldn't reach before.

| | OVN (`ovn-a`, mc1 → mc2) | Flat bridge (`flat-a`, mc1 → mc2) |
|---|---|---|
| MAC | Same (`00:16:3e:98:62:f0`) | Same (`00:16:3e:38:9c:1d`) |
| IPv4 | Same (`10.10.10.2`) | Same (`10.10.30.23`), by luck, not by design |
| Reachability | Unchanged | **Changed**: `flat-b` became reachable |
| Outage (cold move) | ~43 seconds | Not measured |

**Public cloud parallel:** after an EC2 stop and start, the instance usually lands on a new underlying host, but its attached network interface, and with it the private IP, persists. The auto-assigned public IPv4 is released unless an Elastic IP is used. OVN behaves like the network interface: identity belongs to the network, not the host.

---

## Acceptance Criteria

- [x] Before-and-after record for an OVN container (`ovn-a`): node, MAC address, IPv4 address
- [x] Reachability from `ovn-c` to `ovn-a` measured continuously during the move, with the outage window recorded
- [x] Evidence from OVN itself that the container's logical port moved to the new node
- [x] Before-and-after record for a flat-bridge container (`flat-a`): node, MAC, IPv4, and reachability to `flat-b`
- [x] Plain-English write-up: which parts of network identity survived a move in each model, and why
- [x] Lab-vs-reality gap documented: cold move vs. true live migration

---

## Comments Thread

**Reviewer:** In INFRA-013 you moved a container and the data came with it. Here's what an application owner will ask you: "after the move, can my clients still find it at the same address?"

**NwaChi:** The data didn't move, only the running process did. So the IP should stay the same too?

**Reviewer:** Maybe, but work out why first. Where does the container's IP come from in each model?

**NwaChi:** In OVN, from OVN's DHCP, one service for the whole network. On the flat bridge, from each node's own DHCP server.

**Reviewer:** So in one model, the thing handing out addresses doesn't care which node you're on. In the other, the container arrives on a node whose DHCP server has never seen it. And the MAC: is it a property of the machine, or of the container?

**NwaChi:** LXD generates it for the container, so it probably travels with it.

**Reviewer:** Check it. Also remember INFRA-015: `flat-a` couldn't reach `flat-b` because they were on different nodes' bridges. What happens if `flat-a` moves onto `flat-b`'s node?

**NwaChi:** They'd share a bridge. So a move could change who it can reach?

**Reviewer:** That's the question. In one model, moving a workload shouldn't change its network at all. In the other, it might quietly change everything. And remember this is a cold move, so measure the downtime, but don't treat it as production live migration.

---

## Implementation Runbook

> **Standing rule:** record the real output at every step.

### Step 1: Inventory the starting state (`mc1`)

Plain English: write down exactly what each container looks like before moving anything, so before and after can be compared line by line.

```bash
lxc list
lxc config get ovn-a volatile.eth0.hwaddr
lxc config get flat-a volatile.eth0.hwaddr
```

`volatile.eth0.hwaddr` is the MAC address LXD generated for the container's `eth0`. "Volatile" keys are values LXD records for itself rather than settings you choose.

If `flat-a`/`flat-b` don't exist, relaunch them:
```bash
lxc launch ubuntu:24.04 flat-a --network flattest --storage remote --target mc1
lxc launch ubuntu:24.04 flat-b --network flattest --storage remote --target mc2
```

**Real result:**

| Container | Node | IPv4 | MAC |
|---|---|---|---|
| `ovn-a` | mc1 | 10.10.10.2 | 00:16:3e:98:62:f0 |
| `ovn-c` | mc3 | 10.10.10.3 | (not needed) |
| `flat-a` | mc1 | 10.10.30.23 | 00:16:3e:38:9c:1d |
| `flat-b` | mc2 | 10.10.30.93 | (not needed) |

Two details worth knowing:
- `ovn-a`'s IPv6 address (`...216:3eff:fe98:62f0`) already contains its MAC. It's built from the MAC using a standard method called EUI-64, so the MAC can be read straight out of `lxc list`.
- `00:16:3e` is the vendor prefix LXD uses for the MACs it generates, which shows the MAC comes from LXD, not from VMware or the node.

### Step 2: See where OVN thinks `ovn-a` lives (`mc1`)

Plain English: OVN has two databases. The **northbound** database holds intent: LXD writes "there's a network, a router, a port for this container," with no mention of machines. A translator service (`ovn-northd`) turns that into forwarding rules in the **southbound** database, which also records physical reality: which nodes ("chassis") exist, and which node each port is currently bound to. A local agent on each node (`ovn-controller`) reads southbound and programs that node's Open vSwitch.

```bash
sudo microovn.ovn-sbctl show
```

**Real result (before the move):**
```
Chassis mc3
    Port_Binding cr-lxd-net9-lr-lrp-ext
    Port_Binding lxd-net8-instance-3cd29ee9-...-eth0      (ovn-c)
Chassis mc2
    Port_Binding lxd-net9-instance-dbf06e43-...-eth0      (ovn-b)
    Port_Binding cr-lxd-net8-lr-lrp-ext
Chassis mc1
    Port_Binding lxd-net8-instance-65097141-...-eth0      (ovn-a)
```

How to read it:
- Ports are named by network ID and instance ID, not container name. `net8` is `ovntest` and `net9` is `ovntest2`.
- `ovn-a` was identified as the `net8` port on `mc1` by elimination: it's the only `ovntest` container on that node. To confirm directly, check that `lxc config get ovn-a volatile.uuid` starts with `65097141`. (This check wasn't run in the original session.)
- `cr-...-lrp-ext` is each router's **external gateway port** ("cr" = chassis-redirect): the node where that network's traffic actually enters and leaves the uplink. `ovntest`'s gateway was active on `mc2`, so before the move, `ovn-a`'s internet traffic crossed a Geneve tunnel from `mc1` to `mc2` before leaving through `ens38`.

### Step 3: Start a continuous ping from `ovn-c` (second terminal)

```bash
lxc exec ovn-c -- ping -D -O 10.10.10.2
```
- `-D` prints a timestamp on each line.
- `-O` prints a line for every ping that got no reply, so the gap is visible.

### Step 4: Move `ovn-a` from `mc1` to `mc2` (first terminal)

```bash
lxc stop ovn-a
lxc move ovn-a --target mc2
lxc start ovn-a
```

Because the disk is on Ceph, `lxc move` only changes which node owns the container.

**Real result (from `ovn-c`'s ping):**
```
[1790419896.799260] 64 bytes from 10.10.10.2: icmp_seq=42 ttl=64 time=0.609 ms
[1790419898.846650] no answer yet for icmp_seq=43
...
[1790419939.870655] no answer yet for icmp_seq=83
[1790419939.874461] 64 bytes from 10.10.10.2: icmp_seq=84 ttl=64 time=3.56 ms
```

**The outage was about 43 seconds**, with 41 pings unanswered. This covers the whole cold move: shutdown, ownership change, boot, and the network coming up. The share caused by networking alone wasn't measured. To split it, prefix each command with `time` on a repeat run.

### Step 5: Compare `ovn-a` after the move

```bash
lxc list ovn-a
lxc config get ovn-a volatile.eth0.hwaddr
lxc exec ovn-a -- ip -4 addr show eth0
lxc exec ovn-a -- ping -c 3 8.8.8.8
sudo microovn.ovn-sbctl show
```

**Real result:**
- **Node:** `mc2`
- **MAC:** `00:16:3e:98:62:f0` (unchanged)
- **IPv4:** `10.10.10.2` (unchanged), with `valid_lft 3488sec`. That's a nearly fresh lease, so the container made a new DHCP request on `mc2`, and OVN gave it the same address.
- **Internet:** 2 of 3 replies, with the first ping lost. This is possibly first-packet setup right after start, but it's unconfirmed; the check wasn't repeated.
- **Southbound:** port `65097141...` is now listed under **chassis `mc2`**, next to `cr-lxd-net8`. `mc2`'s local OVN agent claimed the port when the container started there.
- **From `ovn-c`:** replies resumed with `ttl=64` and ~1 ms latency, the same as before the move.

### Step 6: Move the flat container

Confirm the INFRA-015 baseline first:
```bash
lxc exec flat-a -- ping -c 3 10.10.30.93
```
**Real result:** `Destination Host Unreachable` (from `10.10.30.23`), as before.

Move `flat-a` onto `flat-b`'s node, then compare:
```bash
lxc stop flat-a
lxc move flat-a --target mc2
lxc start flat-a
lxc exec flat-a -- ping -c 3 10.10.30.93
lxc list flat
lxc config get flat-a volatile.eth0.hwaddr
lxc exec flat-a -- ip -4 addr show eth0
```

**Real result:**
- **Node:** `mc2`, the same node as `flat-b`
- **MAC:** `00:16:3e:38:9c:1d` (unchanged)
- **IPv4:** `10.10.30.23` (unchanged), `mtu 1500`
- **Ping to `flat-b`:** 3 of 3 replies, 0.1 to 0.4 ms. **Previously unreachable.**

Why the IP stayed the same, even though `mc2`'s DHCP server had never seen `flat-a`: there are two plausible explanations, and **neither is verified**.
1. The container remembered its old lease and asked for `.23`, and `mc2`'s server granted it because it was free.
2. LXD's DHCP server (dnsmasq) picks a starting address by hashing the client's MAC. Two independent servers with the same range and the same MAC would choose the same address.

Either way, it's luck, not a guarantee. If `.23` had been taken on `mc2`, the result would have been different.

Also notice `mtu 1500` here versus `1442` on `ovn-a`: the flat bridge has no Geneve overhead, because nothing is tunnelled.

---

## Write-up

**OVN:** everything survived: MAC, IP, and reachability. The MAC survived because LXD stores it in the container's own config, not on the node. The IP survived because OVN runs one DHCP service for the whole network and ties the address to the container's logical port. Reachability was unchanged because the logical switch spans every node. The southbound binding showed OVN re-homing the same port from chassis `mc1` to chassis `mc2`.

**Flat bridge:** MAC and IP survived, but reachability changed. The MAC survived for the same reason as with OVN. The IP was granted by a *different* DHCP server that happened to hand out the same address. Reachability changed because the container landed on `mc2`'s own bridge, the same wire as `flat-b`. The container didn't change; its network did.

**Bottom line:** in OVN, a workload's network belongs to the workload. On a flat bridge, it belongs to whichever machine the workload lands on. The single design choice behind this: OVN defines the network once, centrally, in a shared database that spans every node. A flat bridge is built separately on each host.

**Lab-vs-reality gap:** this was a cold move, so the container was stopped and any open connections to it were dropped. Production live migration moves a *running* workload with only a brief pause, keeping connections open. In LXD, that's supported for virtual machines on shared storage (with `migration.stateful` enabled), not reliably for containers (see INFRA-013). Keeping the same IP and MAC is what makes live migration *possible*, because clients never need to learn a new address. The ~43-second outage here is the cost of the cold move, not proven to be the cost of the network.

---

## Definition of Done

- [x] Starting state recorded for `ovn-a` and `flat-a` (node, MAC, IPv4)
- [x] OVN chassis binding recorded before and after the move
- [x] Continuous ping captured, with a ~43-second outage window recorded
- [x] `ovn-a` after-state recorded, including an internet check
- [x] `flat-a` moved, after-state recorded, and reachability to `flat-b` retested
- [x] Write-up comparing both models, plus the lab-vs-reality note

**Open follow-ups (optional):**
- Confirm the `ovn-a` UUID match with `lxc config get ovn-a volatile.uuid`.
- Repeat the internet check from `ovn-a` to rule out a persistent first-packet loss.
- Time each move command to separate boot time from network convergence.

---

## Retention Check

After the move, which model kept the container's network identity independent of the machine it runs on, and what single design choice explains the difference?