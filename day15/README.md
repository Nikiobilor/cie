# INFRA-015 — Why Not Just Put Everything on One Flat Network?

**Priority:** P2 — Week 4, Day 17
**Component:** Networking (OVN multi-network topology vs. flat bridge)
**Environment:** VMware Workstation Pro (Windows), 3-node MicroCloud cluster — `mc1`, `mc2`, `mc3`, MicroOVN installed and clustered (INFRA-014), OVN network `ovntest` (10.10.10.0/24) already live on top of the `UPLINK` physical network (`ens37`)
**Linked tickets:** Builds directly on INFRA-014 (MicroOVN, `UPLINK`, `ovntest` all already exist). Sets up INFRA-016 (Day 18: live migration, network-identity question)

---

## Summary

INFRA-014 proved OVN works. It didn't prove OVN is *necessary* — a flat bridge network would also let a container reach the internet. Today's job is to find the actual reason real infrastructure teams bother with logical network segmentation instead of putting everything on one shared network: build a second isolated OVN network, empirically test whether it can reach the first one, then build a genuinely flat bridge network and test what isolation looks like there. Don't assume either result — test both, because even LXD's own documentation on this is more nuanced than a flat "isolated vs. not."

---

## Acceptance Criteria

- [ ] A second OVN network (`ovntest2`) created — its own logical switch, its own subnet, its own NAT to the same `UPLINK`
- [ ] A container launched on `ovntest2`, alongside the existing `nettest` container on `ovntest`
- [ ] Real, tested answer (not assumed) to: can a container on `ovntest` reach a container on `ovntest2` by private IP, with no peering configured?
- [ ] A genuinely flat bridge network (`flattest`, `type=bridge`) created, with two containers launched onto it
- [ ] Real, tested answer to: can those two containers on the same flat bridge reach each other?
- [ ] Plain-English write-up comparing what's true by default in each model, and what deliberate action it would take to flip that default (peering two OVN networks together vs. VLAN-tagging or ACL'ing apart two hosts on a flat bridge)
- [ ] Lab-vs-reality gap documented: a 3-node lab can't show what a flat network looks like at real scale (VLAN ID exhaustion at 4094, broadcast domain blast radius, switch config sprawl) — explain the mechanism even though it can't be demonstrated directly here

---

## Comments Thread

**Reviewer:** Yesterday you proved OVN can build a working, NAT'd, internet-reaching network from nothing. Fair question someone will ask you in an interview: why go through all that — MicroOVN, uplinks, certificates — when a single bridge network does the same job with way less setup?

**NwaChi:** Because... isolation? Like, keeping things separate?

**Reviewer:** That's the right instinct, but don't just assert it — I went looking at LXD's own docs before handing you this ticket, and they say something more specific than "OVN networks are isolated." They say traffic between two OVN networks *can* route through the uplink, it's just an inefficient path compared to a direct peering connection. That's not the same as a hard wall. So today, don't take my word for it or the docs' word for it — build two separate OVN networks and actually try to ping one from the other. Then build a flat bridge and do the same test there. Let the actual behavior tell you what's true.

**NwaChi:** And if it turns out they *can* reach each other by default?

**Reviewer:** Then that's a more interesting and more honest finding than "OVN = isolated" — and it changes what you'd tell someone in an interview. The real distinction isn't "possible vs. impossible." It's "what's the default, and what does it cost to change it." On a flat network, isolating two hosts from each other is work you do *after the fact* — VLANs, ACLs, physical switch config. With OVN, connecting two networks together is the thing you do on purpose, explicitly, via peering — nothing happens between them until you ask for it. Whichever direction today's test comes out, that's the lesson to write down.

**NwaChi:** Feels like a small difference for a 3-container lab.

**Reviewer:** It is, at this scale. That's worth saying honestly in your write-up, not glossing over. The difference stops being small once you're not managing 3 containers but 3,000 — VLANs top out at 4094 IDs total across an entire physical network, and every new segment means touching physical switch configuration on every switch in the path. A logical switch in OVN, by contrast, is just a database record. That gap is the real reason this exists — it's not really visible today, but it's real.

---

## Implementation Runbook

### Step 1 — Build a second, separate OVN network

Plain English: same pattern as `ovntest` from INFRA-014, just a different name and subnet, so it's a genuinely separate logical switch.

```
lxc network create ovntest2 --type=ovn network=UPLINK \
  ipv4.address=10.10.20.1/24 \
  ipv4.nat=true
```

### Step 2 — Launch a container on each network

`nettest` already exists on `ovntest` from INFRA-014. Launch a second container on the new network:

```
lxc launch ubuntu:24.04 nettest2 --network ovntest2 -s remote
lxc list
```

Note both IPs — `nettest` should be in `10.10.10.0/24`, `nettest2` in `10.10.20.0/24`.

### Step 3 — Test cross-network reachability, for real

```
lxc exec nettest -- ping -c 3 <nettest2's 10.10.20.x address>
```

Record exactly what happens — full replies, timeouts, or something in between. Whatever the actual result is, that's your data point, not an assumption.

### Step 4 — Build a genuinely flat network for contrast

Plain English: this is a plain Linux bridge, no OVN involved — the closest thing in this lab to "just put everything on one network."

```
lxc network create flattest --type=bridge
```

Launch two containers directly onto it:

```
lxc launch ubuntu:24.04 flat1 --network flattest -s remote
lxc launch ubuntu:24.04 flat2 --network flattest -s remote
lxc list
```

### Step 5 — Test reachability on the flat network

```
lxc exec flat1 -- ping -c 3 <flat2's address>
```

### Step 6 — Write up the comparison

With both real results in hand, write a short plain-English paragraph: what was true by default in each case, and — regardless of what Step 3 actually showed — what deliberate action would be required to flip that default in each direction (peering to connect two OVN networks; VLANs or ACLs to separate two hosts on a flat bridge).

---

## Definition of Done

- [ ] `ovntest2` created and confirmed as a distinct logical switch from `ovntest`
- [ ] Cross-network ping test between `nettest` and `nettest2` run and its real result recorded
- [ ] `flattest` bridge network created with two containers on it
- [ ] Same-network ping test between `flat1` and `flat2` run and its real result recorded
- [ ] Write-up comparing default behavior and the deliberate action needed to flip it in each model
- [ ] Lab-vs-reality gap on VLAN/broadcast-domain scale documented

---

## Retention Check

Based on what you actually observed today — not what you expected going in — which model makes isolation the default, and which model makes connectivity the default?