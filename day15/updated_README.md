# INFRA-015: Why Not Just Put Everything on One Flat Network?

**Priority:** P2 (Week 4, Day 17)
**Component:** Networking (OVN multi-network topology vs. flat bridge), plus uplink redesign
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), MicroOVN installed and clustered (INFRA-014), Ceph storage pool `remote`
**Linked tickets:** Builds on INFRA-014 (MicroOVN, `UPLINK`, `ovntest`). Sets up INFRA-016 (Day 18: live migration, does network identity survive a move?)

> **Note on this ticket:** it was rewritten after the session to reflect what actually happened, not the original plan. The original plan hit a real blocker (the hosts lost internet access), and the fix changed the lab's network design. Both are documented here so you can avoid the same trap.

---

## Summary

INFRA-014 proved OVN works. It didn't prove OVN is *necessary*, because a plain bridge network also lets a container reach the internet. This ticket tests what actually happens by default in each model: two separate OVN networks, and one flat bridge, spread across three machines.

Before the tests could run, a blocker appeared: the cluster nodes had lost their own internet access. Root cause: in INFRA-014, the node's only internet-facing NIC (`ens37`) had been handed to OVN as its uplink. The fix was to give OVN a **dedicated third NIC** (`ens38`) and return `ens37` to the host. This mirrors real deployments, which keep the OVN uplink on its own interface.

**Key findings (all verified against real output):**

| Test | Result | Why |
|---|---|---|
| Same OVN network, different machines | Reachable | OVN stretches one network across nodes using Geneve tunnels |
| Different OVN networks, by internal address | Unreachable | No route between them. Not a firewall. Traffic leaves via the uplink and dies |
| Different OVN networks, by uplink address | Reachable | Both OVN routers are neighbours on the shared uplink |
| Flat bridge, containers on different machines | Unreachable | Each node has its own separate bridge. Same subnet on paper, different wires |

---

## Acceptance Criteria

- [ ] Host internet access restored on all three nodes, with `ens37` back on DHCP
- [ ] A dedicated uplink NIC (`ens38`) added to each node and claimed in netplan with no IP address
- [ ] `UPLINK` rebuilt on `ens38`, with `ens37` confirmed absent from Open vSwitch
- [ ] Two OVN networks (`ovntest`, `ovntest2`) created on the new uplink
- [ ] Real, tested answer: can a container on `ovntest` reach a container on `ovntest2`, by internal address and by uplink address?
- [ ] Real, tested answer: can two containers on the same OVN network, on different machines, reach each other?
- [ ] A flat bridge network (`flattest`) created, with containers on two different machines
- [ ] Real, tested answer: can those two flat-bridge containers reach each other?
- [ ] Plain-English write-up: what's true by default in each model, and what it takes to change that default
- [ ] Lab-vs-reality gap documented (flat networks at scale)

---

## Comments Thread

**Reviewer:** Yesterday you proved OVN can build a working, internet-reaching network from nothing. Someone will ask you in an interview: why go through all that when a single bridge network does the same job with less setup?

**NwaChi:** Isolation? Keeping things separate?

**Reviewer:** Right instinct, but don't just assert it. LXD's own docs are more nuanced than "OVN networks are isolated." Build two OVN networks and a flat bridge, and let the real behaviour tell you what's true.

**NwaChi:** Before I start: `mc1` can't pull the Ubuntu image. DNS fails and there's no internet at all.

**Reviewer:** Don't guess. What does `ip route show` say?

**NwaChi:** No default route at all. Only the management subnet on `ens33`.

**Reviewer:** So where was the default route coming from before?

**NwaChi:** From `ens37`'s DHCP lease. And in INFRA-014 I gave `ens37` to LXD as the OVN uplink and removed its IP. So I cut off the host's only way out.

**Reviewer:** Exactly. An OVN uplink needs the whole interface. Open vSwitch takes it over, and the host can't use it anymore. That's why real deployments dedicate a separate physical NIC to the uplink. You'll do the same thing here: give `ens37` back to the host and add a third NIC just for OVN.

**NwaChi:** Once that's done and the tests run, what if the two OVN networks *can* reach each other?

**Reviewer:** Then that's a more honest finding than "OVN = isolated." The real question isn't "possible or impossible." It's "what's the default, and what does it cost to change it?" On a flat network, separating hosts is work you do afterwards: VLANs, ACLs (Access Control Lists), switch config. With OVN, connecting networks is something you do on purpose, via peering.

**NwaChi:** It feels like a small difference with three containers.

**Reviewer:** At this scale, it is. Say that honestly. It stops being small at 3,000 workloads. VLAN IDs top out at 4094 across a whole physical network, and each new segment typically means touching switch config along the path. An OVN logical switch is just a database record.

---

## Implementation Runbook

**Lab reference: NIC map.** Your MAC addresses will differ, so capture your own with `ip link`.

| Node | `ens33` (management, static) | `ens37` (host internet, DHCP) | `ens38` (OVN uplink, no IP) |
|---|---|---|---|
| mc1 | 00:0c:29:57:a7:08, 192.168.20.140 | 00:0c:29:57:a7:12 | 00:0c:29:57:a7:1c |
| mc2 | 00:0c:29:47:0a:85, 192.168.20.141 | 00:0c:29:47:0a:8f | 00:0c:29:47:0a:99 |
| mc3 | 00:0c:29:2d:e7:3d, 192.168.20.142 | 00:0c:29:2d:e7:47 | 00:0c:29:2d:e7:51 |

VMware NAT network for `ens37`/`ens38`: the `192.168.82.x` subnet, gateway `192.168.82.2`.

> **If you're starting fresh** and your uplink is already on a dedicated NIC, skip Part A and start at Part B.

---

### Part A: Fix the uplink design

#### Step A1: Inspect before touching anything (`mc1`)

Plain English: before deleting anything, look at what exists and what depends on what. Deleting a network that something is still attached to will fail.

```bash
lxc list
lxc network list
ip route show
sudo microovn.ovs-vsctl show
sudo cat /etc/netplan/*.yaml
```

What to look for:
- **`lxc network list`**, "USED BY" column: `UPLINK` was used by both OVN networks, and `ovntest` by the test container.
- **`ip route show`**: in this lab, only `192.168.20.0/24 dev ens33`. **No `default` line**, which confirms the root cause.
- **`ovs-vsctl show`**: Open vSwitch (OVS) is the software switch OVN programs on each node. Here, `ens37` appeared as a port inside a bridge called `lxdovn2`. That's the concrete reason the host lost internet: OVS owned the NIC, not the host.

#### Step A2: Tear down, dependents first (`mc1`)

Plain English: the container depends on `ovntest`, and both OVN networks depend on `UPLINK`, so delete in that order.

```bash
lxc delete nettest --force
lxc network delete ovntest
lxc network delete ovntest2
lxc network delete UPLINK
```

`UPLINK` is cluster-wide, so deleting it on `mc1` removes it from all nodes.

#### Step A3: Confirm `ens37` was released (all three nodes)

```bash
sudo microovn.ovs-vsctl show
```

Expected: only `Bridge br-int` with the Geneve tunnels to the other two nodes (e.g. `ovn-mc2-0`, `ovn-mc3-0`). No `lxdovn` bridge, no patch ports, and **no `ens37`**.

Side note: `bfd_status` also disappears from the tunnels. BFD (Bidirectional Forwarding Detection) is a tunnel health check that OVN only runs when there's a gateway to monitor. It comes back in Part B.

> **Don't continue if `ens37` still appears under an OVS bridge.** DHCP won't work on it until it's released.

#### Step A4: Return `ens37` to DHCP (all three nodes)

Plain English: netplan matches each NIC by MAC address, not by name. That keeps the config tied to the physical NIC even if interface names shift, but it also means the MAC in the file must be correct.

Edit the file:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Set the `ens37` block to the following, using your own MAC:
```yaml
    ens37:
      match:
        macaddress: 00:0c:29:57:a7:12
      set-name: ens37
      dhcp4: yes
      dhcp6: no
      dhcp-identifier: mac
```

> **Why `dhcp-identifier: mac`?** In the original run this line was *not* added at first. After a reboot, **all three nodes received the same DHCP address** (`192.168.82.144`). The lease file on `mc1` showed a client ID starting `ff...0002 0000ab11`: a DUID (DHCP Unique Identifier) of type DUID-EN, systemd's format, derived from `/etc/machine-id`. The VMs were most likely cloned with identical machine IDs, so the DHCP server saw one client asking three times. This was confirmed on `mc1`; the other two machine IDs weren't captured. Adding `dhcp-identifier: mac` makes each node identify itself by its MAC instead. Immediately after the change, each node got a unique lease back (`.139`, `.138`, `.137`). Add it now to avoid the problem.

Apply it safely:
```bash
sudo netplan try
```
`netplan try` applies the change and rolls back automatically after 120 seconds unless you press **Enter**. If a change breaks your SSH session, it undoes itself. You're connected over `ens33`, so the session should survive this change.

#### Step A5: Verify host connectivity (all three nodes)

```bash
ip -4 addr show ens37
ip route show
resolvectl query cloud-images.ubuntu.com
ping -c 3 8.8.8.8
```

Expected:
- `ens37` has a `192.168.82.x` address marked `dynamic`, and it's **different on each node**.
- `default via 192.168.82.2 dev ens37 proto dhcp`. The `proto dhcp` part proves the route came from the lease.
- The hostname resolves via `link: ens37`, and the ping gets replies.

#### Step A6: Pre-maintenance health check and `noout` (`mc1`)

Plain English: shutting down all three nodes is safe, but Ceph needs one precaution. When an OSD (Object Storage Daemon, the process managing each disk) goes offline for about 10 minutes, Ceph marks it "out" and starts copying its data elsewhere. For a planned shutdown that's wasted work. The `noout` flag tells Ceph "this downtime is planned; don't rebuild."

```bash
lxc cluster list
sudo microceph.ceph -s
sudo microceph.ceph osd set noout
```

Expected: all members ONLINE, Ceph healthy, then a `noout flag(s) set` warning after setting the flag.

#### Step A7: Shut down and add the third NIC

On each node:
```bash
sudo shutdown -h now
```

In VMware, for each VM (powered off):
1. Open **VM → Settings** and check which network the **second** adapter (`ens37`) uses: NAT, or Custom with a specific VMnet.
2. Go to **Add… → Network Adapter → Finish**.
3. Set the new adapter to **exactly the same network** as the second adapter.
4. Make sure **Connect at power on** is ticked.

Then power on all three VMs within about a minute of each other. The cluster databases and Ceph monitors each need a majority (2 of 3) to form quorum, so booting them close together gets there quickly.

#### Step A8: Identify the new NIC (all three nodes)

Don't guess the new interface's name or MAC. Read them:
```bash
ip link
```

In this lab it appeared as **`ens38`** on all three nodes. Record its `link/ether` MAC for each node.

You'll also see `ovs-system`, `genev_sys_6081` and `br-int` showing `DOWN`. These are OVS's kernel-side devices (6081 is the Geneve port number), and `DOWN` is normal for them.

#### Step A9: Post-boot health check (`mc1`)

```bash
lxc cluster list
sudo microceph.ceph -s
sudo microovn status
```

Real result in this lab: everything healthy except `clock skew detected on mon.mc3`. Ceph monitors tolerate only about 0.05 seconds of clock difference between them. `mc3` had booted without a working path to a time server. **It resolved on its own** once host internet was restored. `timedatectl` showed `System clock synchronized: yes`, and the warning was gone at the next check.

> On this system `systemd-timesyncd` wasn't installed, so `systemctl restart systemd-timesyncd` failed. A different NTP (Network Time Protocol) client is in use. If you need to force a resync, check which client you have first.

Once only the `noout` warning remains:
```bash
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s
```
Expected: `HEALTH_OK`.

#### Step A10: Claim `ens38` with no IP address (all three nodes)

Plain English: the OVN router brings its own address onto this wire. If the host also had an address on `ens38`, it would have two paths into the same subnet, and could send its own traffic into a NIC that OVS has taken over. So `ens38` stays up with a stable name, but has no address and no routes.

Add this under `ethernets:`, using each node's own `ens38` MAC:
```yaml
    ens38:
      match:
        macaddress: 00:0c:29:57:a7:1c
      set-name: ens38
      dhcp4: no
      dhcp6: no
```

```bash
sudo netplan try
ip -4 addr show ens38
ip route show
```

Expected: **no `inet` line** on `ens38`, and a routing table unchanged from Step A5.

---

### Part B: Rebuild the uplink and OVN networks

#### Step B1: Create `UPLINK` on `ens38` (`mc1`)

Plain English: it's a two-stage cluster create. First, tell each node which of its NICs is the uplink. Then set the shared settings once.

```bash
lxc network create UPLINK --type=physical parent=ens38 --target=mc1
lxc network create UPLINK --type=physical parent=ens38 --target=mc2
lxc network create UPLINK --type=physical parent=ens38 --target=mc3

lxc network create UPLINK --type=physical \
  ipv4.gateway=192.168.82.2/24 \
  ipv4.ovn.ranges=192.168.82.10-192.168.82.20 \
  dns.nameservers=8.8.8.8
```

`ipv4.ovn.ranges` is the pool OVN routers take their external addresses from. In this lab, host leases came from the upper part of the subnet, so `.10` to `.20` doesn't collide with them.

Verify:
```bash
lxc network show UPLINK
lxc network show UPLINK --target mc2
ip route show
```

Expected:
- `status: Created`, with all three nodes under `locations`.
- `parent: ens38`. It only appears with `--target`, because it's per-node config.
- `volatile.last_state.created: "false"`, which is LXD's record that it didn't create `ens38`, so it won't delete it later.
- The host's default route is still via `ens37`.

#### Step B2: Create two OVN networks (`mc1`)

```bash
lxc network create ovntest --type=ovn network=UPLINK \
  ipv4.address=10.10.10.1/24 ipv4.nat=true

lxc network create ovntest2 --type=ovn network=UPLINK \
  ipv4.address=10.10.20.1/24 ipv4.nat=true
```

No `--target` steps are needed. An OVN network is a logical object in OVN's shared database, not a per-node interface.

#### Step B3: Confirm the plumbing (`mc1`)

```bash
sudo microovn.ovs-vsctl show
```

Real result:
- A new `Bridge lxdovn7` containing **`Port ens38`**. `ens37` doesn't appear anywhere.
- Two patch ports (`lxd-net8`, `lxd-net9`) linking `br-int` to `lxdovn7`, one per OVN network.
- `bfd_status: state=up` back on the Geneve tunnels.

#### Step B4: Ping each OVN router's external address (`mc1`)

```bash
lxc network get ovntest volatile.network.ipv4.address
lxc network get ovntest2 volatile.network.ipv4.address
ip route get <address>
ping -c 3 <address>
```

Real result: `.10` for `ovntest` and `.11` for `ovntest2`. `ip route get` showed `dev ens37` with no `via` (a direct neighbour), and the pings replied with `ttl=254` in 1 to 4 ms.

> **False-positive warning.** In the original run, the *internal* gateway `10.10.10.1` was pinged from the host first, and it **replied**. It wasn't OVN. The clues: `ttl=128` (the Windows default; Linux uses 64) and ~50 ms latency. `ip route get 10.10.10.1` showed `via 192.168.82.2 dev ens37`: the host has no route to OVN's internal subnets, so the ping went out through VMware NAT, and some device on the physical network happened to own that address. **A ping reply isn't proof until you know where it came from.**

---

### Part C: The reachability tests

#### Step C1: Launch one container per OVN network

```bash
lxc launch ubuntu:24.04 ovn-a --network ovntest --storage remote --target mc1
lxc launch ubuntu:24.04 ovn-b --network ovntest2 --storage remote --target mc2
lxc list
```

Real result: `ovn-a` got `10.10.10.2` and `ovn-b` got `10.10.20.2`. `ovn-b`'s IPv4 address took about 30 seconds to appear. Its IPv6 address showed first because the container builds that itself from router announcements, while IPv4 waits for a DHCP exchange.

This `lxc launch` is also the image pull that originally failed, so its success confirms the blocker is fixed.

#### Step C2: Baseline (each container reaches the internet)

```bash
lxc exec ovn-a -- ping -c 3 8.8.8.8
lxc exec ovn-b -- ping -c 3 8.8.8.8
```

Both succeeded. This matters: if a cross-network test fails later, the failure means something, because neither container is broken.

#### Step C3: Different OVN networks, internal address

```bash
lxc exec ovn-a -- ping -c 3 10.10.20.2
lxc exec ovn-b -- ping -c 3 10.10.10.2
```

**Real result: 100% packet loss, both directions.**

#### Step C4: Find out why

```bash
lxc exec ovn-a -- tracepath -n -m 5 10.10.20.2
```

Real result:
```
 1:  10.10.10.1      (ovn-a's own OVN router)
 2:  192.168.82.2    (VMware's NAT gateway, so the packet has left OVN)
 3:  no reply
```

Each OVN network has its own virtual router, and `ovn-a`'s router has no route to `10.10.20.0/24`. So it treats the packet like internet traffic, NATs it out through the uplink, and it dies outside the lab. **Separation by missing route, not by firewall.**

`tracepath` also reported `pmtu 1442`. The largest packet on this path is 1442 bytes rather than the usual 1500, because Geneve adds an extra header to carry each packet between nodes.

#### Step C5: Different OVN networks, uplink address

```bash
lxc exec ovn-a -- ping -c 3 192.168.82.11
```

**Real result: success.** Both OVN routers sit on the same uplink subnet as neighbours, so anything reachable on the uplink side is reachable from both. `ovn-b` itself stays hidden behind its router's NAT for now. But a port forward on `.11` would expose it to `ovn-a`. **"OVN networks are isolated by default" is too simple.**

#### Step C6: Same OVN network, different machines

```bash
lxc launch ubuntu:24.04 ovn-c --network ovntest --storage remote --target mc3
lxc list ovn
lxc exec ovn-a -- ping -c 3 <ovn-c address>
```

**Real result: success.** `ovn-c` got `10.10.10.3` (no collision, because OVN runs one DHCP service for the whole network), and the pings replied in about 1.5 ms with `ttl=64`. A TTL of 64 means no router hop: to the containers, it's one shared wire. In reality, the Geneve tunnel `ovn-mc3-0` carried the traffic between `mc1` and `mc3`.

#### Step C7: Create a flat bridge network (`mc1`)

```bash
lxc network create flattest --type=bridge --target=mc1
lxc network create flattest --type=bridge --target=mc2
lxc network create flattest --type=bridge --target=mc3
lxc network create flattest --type=bridge \
  ipv4.address=10.10.30.1/24 ipv4.nat=true ipv6.address=none
```

Plain English: in a cluster, a bridge network doesn't span the nodes. Each node gets its **own separate** local bridge called `flattest`, all using the same `10.10.30.1/24`, and each runs its own DHCP server.

#### Step C8: Two flat containers on different machines

```bash
lxc launch ubuntu:24.04 flat-a --network flattest --storage remote --target mc1
lxc launch ubuntu:24.04 flat-b --network flattest --storage remote --target mc2
lxc list flat
```

Real result: `flat-a` got `10.10.30.23` and `flat-b` got `10.10.30.93`. They're different by luck, not coordination. Each node's DHCP server picks from the same range independently, so a duplicate is possible.

#### Step C9: Test flat reachability

```bash
lxc exec flat-a -- ping -c 3 10.10.30.93
lxc exec flat-a -- ip neigh
```

**Real result: `Destination Host Unreachable`, reported by `flat-a` itself (`From 10.10.30.23`).** The ARP table showed:
```
10.10.30.93 dev eth0 FAILED
10.10.30.1  dev eth0 lladdr 00:16:3e:55:82:94 STALE
```

`flat-a` saw `.93` in its own subnet, so it asked directly on its local wire, "who has `.93`?" That question is an ARP (Address Resolution Protocol) request. The wire is `mc1`'s bridge, and `flat-b` is on `mc2`'s separate bridge, so it never heard the question. The packet never left `mc1`. **Same subnet on paper, different wires in reality.**

(On a single host, two containers on the same bridge *would* reach each other. The failure here comes from the cluster: each node has its own bridge.)

---

### Part D: Write-up

**What decides reachability is network membership, not machine location.** `ovn-a` reached `ovn-c` on another machine, but not `ovn-b`, which was also on another machine.

**OVN:**
- One network spans every node, carried by Geneve tunnels, with one DHCP service.
- Separate networks don't know about each other: they have no route between them. That isn't a firewall rule.
- They share the uplink, so their uplink-side addresses reach each other.
- To connect two OVN networks directly, you add it deliberately, with LXD network peering (`lxc network peer`).
- To add real security boundaries, you add ACLs deliberately.
- **A missing route is not a security policy.**

**Flat bridge in a cluster:**
- Each node is its own island: a separate bridge, a separate DHCP server, and possible address collisions.
- Connecting islands across nodes requires external work: VLANs or trunking on physical switches, or an overlay.
- Separating hosts on a *shared* flat segment also requires external work: VLANs, ACLs, switch config.

**Lab-vs-reality gap:** with three containers, the difference feels small. At scale, flat segmentation hits hard limits. VLAN IDs are 12 bits, giving 4094 usable IDs across a whole physical network. Every broadcast (such as ARP) reaches every host in the segment, so one noisy or failing host affects all of them. And each new segment typically means changing configuration on the physical switches along the path. An OVN logical switch is a database record, created with one command. That gap is why SDN (Software-Defined Networking) exists, even though a three-node lab can't show it directly.

**Optional cleanup** (keep `ovntest`/`ovntest2` for INFRA-016):
```bash
lxc delete flat-a flat-b --force
lxc network delete flattest
```

---

## Definition of Done

- [x] Blocker root-caused (no default route, `ens37` owned by OVS) and fixed with a dedicated uplink NIC
- [x] `ens37` back on DHCP with unique leases on all nodes (`dhcp-identifier: mac`)
- [x] `ens38` claimed with no IP, and `UPLINK` rebuilt on it, with `ens37` absent from OVS
- [x] Ceph back to `HEALTH_OK`, `noout` cleared
- [x] `ovntest` and `ovntest2` rebuilt, with both routers answering on the uplink
- [x] Cross-network tests recorded: internal address fails (with `tracepath` evidence), uplink address succeeds
- [x] Same-network, cross-node test recorded: succeeds
- [x] Flat bridge cross-node test recorded: fails (with ARP `FAILED` evidence)
- [x] Write-up and lab-vs-reality gap documented

---

## Retention Check

`ovn-a` could reach `ovn-c` on a different machine, but not `ovn-b`. `flat-a` couldn't reach `flat-b` even though they shared a subnet. In one sentence: what actually decides whether two workloads can reach each other?