# INFRA-014 — What Did `microcloud init` Actually Give You for Networking?

**Priority:** P2 — Week 4 kickoff
**Component:** Networking (MicroOVN / LXD network layer)
**Environment:** VMware Workstation Pro (Windows), 3-node MicroCloud cluster — `mc1`, `mc2`, `mc3`, each running LXD + MicroCeph + dqlite, static IPs bound by MAC in netplan, Ceph-backed LXD storage pool `remote`
**Linked tickets:** Opens Week 4 (MicroCloud networking/density). Builds on INFRA-010 (`microcloud init` bootstrap — networking was configured then too, just never inspected). Sets up INFRA-015 (Day 17: topology choices) and INFRA-016 (Day 18: live migration, this time asking whether network identity travels with a container the way storage already proved it could)

---

## Summary

Three weeks in, you've decoupled two things from any single physical node: compute (a container can run on any member) and storage (Ceph doesn't care which node reads or writes it). One thing hadn't been questioned yet — networking. This ticket set out to inspect whatever `microcloud init` had configured for networking back on Day 10. The real finding: nothing had been configured at all — MicroOVN was never installed on this cluster. What started as an inspection day turned into a full build day: installing MicroOVN from scratch, wiring it to LXD, and standing up a working OVN network, hitting (and fixing) several real integration gaps along the way.

---

## Acceptance Criteria

- [x] Every network LXD currently knows about was listed, with its type identified — result: no managed networks existed at all, only unmanaged physical NICs (`ens33`, `ens37`)
- [x] Whether MicroOVN is installed and clustered across `mc1`/`mc2`/`mc3` was confirmed with real command output — result: not installed anywhere on the cluster
- [x] MicroOVN installed and bootstrapped across all three nodes; northbound and southbound databases confirmed healthy via `microovn status`
- [x] A real `type=ovn` network (`ovntest`) was built end-to-end, backed by a physical uplink (`ens37`)
- [x] A test container was launched on `ovntest`, its IP traced to the logical switch, and outbound NAT connectivity confirmed with a live ping
- [x] Plain-English explanation of northbound (desired state — logical switches and ports) vs. southbound (actual state — chassis bindings) grounded in the real network built today
- [x] Day 18's open question written down: will this container's network identity survive a migration the way its storage already did?
- [ ] Lab-vs-reality gap write-up: home-lab OVN vs. real multi-tenant SDN at scale (still open — see Definition of Done)

---

## Comments Thread

**Reviewer:** Quick one before you touch anything today. You've spent three weeks decoupling things from specific physical machines. Compute — done, INFRA-010 onward. Storage — done, that was the entire point of Week 3. What's the one thing left that might still be quietly tied to whichever physical node a container happens to be on?

**NwaChi:** ...networking? Like, the actual IP and interface stuff?

**Reviewer:** Exactly. Right now a container gets an IP somehow, reaches the internet somehow, and you've never had to ask how. Before we can even talk about whether that survives a migration, we need to know what's actually there. Don't guess — go look.

**NwaChi:** *(runs `lxc network list`)* — there's nothing here. Just the raw NICs, both unmanaged.

**Reviewer:** That's a real answer, not a failed one. `microcloud init` doesn't always set up OVN — plenty of real deployments run on plain bridged networking, or nothing at all, because nobody opted in during setup. You've just confirmed which kind of cluster you actually have. Which means today isn't an inspection day anymore. It's a build day.

**NwaChi:** So I install MicroOVN and it just... works with LXD?

**Reviewer:** Not quite, and this is the part that trips people up. LXD's OVN support assumes it can find the OVN control plane at a predictable local path. MicroOVN puts its databases somewhere else, and encrypts the connection by default. LXD has to be told, explicitly, where MicroOVN's databases are and handed a certificate to prove it's allowed to talk to them. That's a real, separate step from "install the snap."

**NwaChi:** Got tripped up trying to hand LXD the uplink interface too — it refused because the NIC already had an IP.

**Reviewer:** Right, that's the other gotcha. LXD wants exclusive control of whatever interface becomes the OVN uplink — no IP of its own. Worth noting *why* that NIC had an address in the first place, though: it was never actually configured by you or by netplan. It was falling back to a generic default because nothing had explicitly claimed it. That's worth remembering well past today — services can appear to be "working" for reasons nobody actually decided, and it holds up fine until the day you need that thing to behave differently.

**NwaChi:** And once all of that was sorted?

**Reviewer:** Then it's the same lesson INFRA-010 and INFRA-013 already taught you, just for a third resource. Compute, storage, now networking — none of them have to live on one specific machine. The container that just pinged 8.8.8.8 doesn't know or care that its virtual switch is running across three physical machines behind the scenes. That's the whole point of software-defined anything.

---

## Implementation Runbook

### Step 1 — See what networks LXD already knows about

```
lxc network list
```

Result on this cluster: no managed networks at all. `ens33` and `ens37` showed up as unmanaged physical interfaces, `USED BY: 0`, no type indicating a bridge or OVN network on top of either.

### Step 2 — Check whether MicroOVN is installed and clustered

```
snap list microovn
sudo microovn status
```

Result: `snap list microovn` returned "no matching snaps installed," and `sudo microovn status` returned `command not found` on all three nodes. MicroOVN had never been installed on this cluster — confirmed finding, not an assumption.

### Step 3 — Install and bootstrap MicroOVN across all three nodes

Plain English: same bootstrap → add → join pattern MicroCeph used back in INFRA-010, just for the networking piece.

On `mc1`:

```
sudo snap install microovn --channel=24.03/stable
sudo microovn cluster bootstrap
sudo microovn cluster add mc2
```

That prints a join token. On `mc2`:

```
sudo snap install microovn --channel=24.03/stable
sudo microovn cluster join <token>
```

Back on `mc1`, generate a second token for `mc3`, then repeat the install/join on `mc3`.

Confirmed all three joined:

```
sudo microovn status
```

Result:

```
MicroOVN deployment summary:
- mc1 (192.168.20.140): central, chassis, switch
- mc2 (192.168.20.141): central, chassis, switch
- mc3 (192.168.20.142): central, chassis, switch
OVN Northbound: OK (7.3.0)
OVN Southbound: OK (20.33.0)
```

All three nodes running all three services — a symmetric, Raft-replicated setup, same resilience pattern as MicroCeph's mon quorum.

### Step 4 — Identify the uplink NIC and confirm it's safe to hand over

Before touching either physical NIC, confirmed which one carried the cluster's own management traffic, since handing that one to LXD would have killed the ability to manage the cluster at all.

```
ip a
```

Found: `ens33` held each node's static, MAC-bound management IP (`.20.140`/`.141`/`.142`, `valid_lft forever`) — this one stays alone. `ens37` held a `dynamic` DHCP-leased address on a separate `192.168.82.0/24` subnet (a VMware NAT adapter) on all three nodes — this one became the OVN uplink candidate.

### Step 5 — Build the physical uplink network

```
lxc network create UPLINK --type=physical parent=ens37 --target=mc1
lxc network create UPLINK --type=physical parent=ens37 --target=mc2
lxc network create UPLINK --type=physical parent=ens37 --target=mc3
lxc network create UPLINK --type=physical \
  ipv4.gateway=192.168.82.2/24 \
  ipv4.ovn.ranges=192.168.82.10-192.168.82.20 \
  dns.nameservers=8.8.8.8
```

Gateway (`192.168.82.2`) was confirmed via `ip route show dev ens37`, not assumed. The OVN range was chosen clear of the DHCP leases already observed (`.137`-`.139`) to avoid collisions.

### Step 6 — Connect LXD to MicroOVN's database, and troubleshoot the real gaps

First attempt at creating an OVN network failed:

```
Error: Failed to run: ovn-nbctl ... database connection failed (No such file or directory)
```

Root cause: LXD didn't know where MicroOVN's northbound database lived, and MicroOVN encrypts that connection by default. Fixed with:

```
lxc config set network.ovn.northbound_connection=ssl:192.168.20.140:6641,ssl:192.168.20.141:6641,ssl:192.168.20.142:6641
lxc config set network.ovn.ca_cert="$(sudo cat /var/snap/microovn/common/data/pki/cacert.pem)"
lxc config set network.ovn.client_cert="$(sudo cat /var/snap/microovn/common/data/pki/client-cert.pem)"
lxc config set network.ovn.client_key="$(sudo cat /var/snap/microovn/common/data/pki/client-privkey.pem)"
```

Note: an earlier attempt to also set `network.ovs.connection` failed with `Unknown key` — that setting belongs to Incus (a separate project forked from LXD), not LXD itself. LXD doesn't need it: MicroOVN's own `cluster bootstrap`/`join` already wires each node's local Open vSwitch to its southbound database.

A leftover half-created network from the first failed attempt then blocked the retry (`Network is not in pending state`) — cleared with `lxc network delete ovntest` before retrying.

The retry then failed again:

```
Error: Failed starting network: Cannot start network as uplink network interface "ens37" has one or more IP addresses configured on it
```

Investigated with `networkctl status ens37` and found the real cause: `ens37` was never claimed by netplan at all (only `ens33` was in `/etc/netplan/*.yaml`) — it was falling back to systemd-networkd's generic `zzzz-dracut-default.network`, which DHCPs any unclaimed interface automatically. Fixed by explicitly claiming `ens37` by MAC address in netplan on all three nodes, with DHCP turned off:

```yaml
    ens37:
      match:
        macaddress: <node-specific MAC>
      set-name: ens37
      dhcp4: no
      dhcp6: no
```

Applied with `sudo netplan apply` on each node, confirmed `ens37` came up with no `inet` line.

### Step 7 — Create the OVN network and prove it works

```
lxc network create ovntest --type=ovn network=UPLINK \
  ipv4.address=10.10.10.1/24 \
  ipv4.nat=true
```

Result: `Network ovntest created`.

Confirmed the logical switch exists at the OVN layer:

```
sudo microovn.ovn-nbctl show
```

Launched a test container — first attempt failed with `No root device could be found` (the default profile has no storage pool configured; every prior launch in this challenge had explicitly passed `-s remote`, and this was the first one that didn't):

```
lxc launch ubuntu:24.04 nettest --network ovntest -s remote
```

Confirmed the full chain works:

```
lxc list nettest
```
→ `nettest` got `10.10.10.2` on `ovntest`.

```
lxc exec nettest -- ping -c 3 8.8.8.8
```
→ 3 packets transmitted, 3 received, 0% packet loss.

---

## Definition of Done

- [x] Confirmed MicroOVN was not installed anywhere on the cluster
- [x] MicroOVN installed, bootstrapped, and confirmed healthy across `mc1`/`mc2`/`mc3`
- [x] Uplink NIC identified and confirmed safe to hand over (distinct from the management NIC)
- [x] `UPLINK` physical network and `ovntest` OVN network both built successfully
- [x] LXD-to-MicroOVN database connection wired (northbound connection + certs)
- [x] Real integration gaps hit and resolved: wrong config key (`network.ovs.connection` is Incus-only), stale pending network record, unclaimed `ens37` falling back to default DHCP, missing storage pool on launch
- [x] Test container launched on `ovntest`, got a real IP, and proved outbound NAT connectivity with a live ping
- [x] Day 18's open question written down explicitly
- [ ] Lab-vs-reality gap write-up: home-lab OVN vs. real multi-tenant SDN at scale

---

## Retention Check

In your own words: what's the difference between what the northbound database tracks and what the southbound database tracks — and which one would you expect to change first if a container moved to a different node?
the northbound is where ovntest's logical switch and its ports got recorded the moment you ran lxc network create. The southbound is what would tell you, later, which physical chassis (mc1, right now) is actually running nettest's port — the thing you'd check on Day 18 if this container ever moved.