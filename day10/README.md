# INFRA-010: Bootstrap a Real MicroCloud Cluster — Compute + Storage

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-010 |
| **Priority** | High |
| **Component** | MicroCloud / Compute + Storage |
| **Environment** | VMware Workstation Pro (Windows host) — 3 new VMs (`mc1`, `mc2`, `mc3`), separate from the existing `node1`–`node3` LXD cluster |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 3 — MicroCloud Compute/Storage (Day 10) |
| **Linked tickets** | INFRA-009 (MicroCloud overview) — this ticket executes the bootstrap that was deliberately deferred there. INFRA-008 — the netplan lesson from that ticket is applied here properly. |

---

## Summary

INFRA-009 installed the MicroCloud tooling and explained what `microcloud init` would do, but stopped short of running it, to avoid disturbing the working `node1`–`node3` LXD cluster from Week 2.

This ticket does the real bootstrap: three fresh VMs, a genuine `microcloud init` run, and a working LXD + Ceph cluster at the end — the compute and storage layers only. Networking (MicroOVN) is intentionally left out this week; it's Week 4's topic, and MicroCloud's own init process skips the network prompts entirely when MicroOVN isn't installed. That's a real, documented behavior, not a workaround.

---

## Acceptance Criteria

- [ ] Existing `node1`–`node3` cluster confirmed untouched throughout
- [ ] Three new VMs cloned, each with a static host-only IP that survives a full power cycle (MAC-based netplan match, not interface-name based — applying the INFRA-008 lesson)
- [ ] `lxd`, `microceph`, and `microcloud` snaps installed on all three (MicroOVN deliberately excluded)
- [ ] `microcloud init` run successfully; mDNS discovery observed live, not just described
- [ ] Ceph storage bootstrapped using a real second disk on each VM, not a placeholder
- [ ] A container actually launched using the new Ceph-backed storage pool, proving it works rather than just looks configured
- [ ] Comparison note: effort here vs. what Week 2 took to build LXD clustering alone
- [ ] Retention check answered

---

## Comments (conceptual explanation thread)

**NwaChi:**
Ready to actually run `microcloud init` now instead of just reading about it. Want to do it on clean nodes so I'm not risking the Week 2 cluster.

**Senior Infra Engineer (review):**
Right call, and one more thing worth doing properly this time: when you set static IPs on these new VMs, use MAC-address matching in netplan, not interface-name matching. `node2` drifted to a DHCP address after a real power-off/power-on cycle specifically because its static config was bound to an interface name that didn't survive re-enumeration. The MAC address doesn't change. Fix it going forward rather than repeating it on three more machines.

On MicroOVN — deliberately skip it this week. You already learned from INFRA-009's docs that `microcloud init` conditionally skips the network prompts when MicroOVN isn't present. Installing it later, once you're specifically studying networking in Week 4, means you'll see the *difference* in the init flow firsthand, rather than just being told it exists.

**NwaChi:**
Makes sense — same instinct as inspecting real JSON output before writing a script instead of assuming. Let the tool show me what changes when a component is added, rather than reading about it.

---

## Implementation Runbook

### Phase 0 — Clone three new VMs

From `base-vm-vmware` (the same golden image used for `node1`–`node3`), create three full clones:

```
mc1, mc2, mc3
```

Each needs the same two adapters as the original nodes: NAT (internet) and a host-only adapter on `VMnet1` (node-to-node).

---

### Phase 1 — Add a second virtual disk to each VM (for Ceph)

Ceph needs a raw, unused disk to claim as an OSD, it can't share the OS disk. With each VM powered off, in VMware Workstation:

`VM Settings → Add → Hard Disk → 8 GB → Store as a single file`

Do this for all three VMs before booting them. This disk should show up as `/dev/sdb` (or similar) once booted, and stay completely unpartitioned, don't format it, MicroCloud's init will claim it directly.

---

### Phase 2 — Static IPs, done properly this time (MAC-based)

Boot each VM. Find the host-only adapter's MAC address:

```bash
ip link show
```

Note the MAC for the host-only interface (whatever it currently enumerates as that's exactly the part we're no longer depending on).

Edit netplan (path may be `/etc/netplan/50-cloud-init.yaml` or similar — check with `ls /etc/netplan/`):

```yaml
network:
  version: 2
  ethernets:
    hostonly:
      match:
        macaddress: "AA:BB:CC:DD:EE:FF"
      set-name: hostonly
      addresses: [192.168.20.140/24]
```
00:0c:29:2d:e7:3d
Use the real MAC address from `ip link show`, and use `.140`, `.141`, `.142` for `mc1`, `mc2`, `mc3` respectively — deliberately outside the `.128`–`.131` range already used by `node1`–`node3`, so the two clusters can't collide.

Apply and verify:

```bash
sudo netplan apply  
ip a
```

Confirm the IP is correct and **not** marked `dynamic`. Repeat on all three VMs, then confirm they can reach each other:

```bash
ping 192.168.20.141   # from mc1, adjust per machine
```

---

### Phase 3 — Confirm the existing cluster is untouched

Before installing anything else, a quick sanity check from `node1` (not `mc1`):

```bash
lxc cluster list
```

Should still show `node1`, `node2`, `node3` exactly as INFRA-008 left them. This is the check that proves Phase 0–2 didn't accidentally touch the wrong VMs.

---

### Phase 4 — Install the required snaps (on `mc1`, `mc2`, `mc3` — not the OVN one)

```bash
sudo snap list lxd
```

If missing, install it same as the original nodes. Then, on all three:

```bash
sudo snap install microceph --channel=squid/stable --cohort="+"
sudo snap install microcloud --channel=2/stable --cohort="+"
```

Deliberately **not** installing `microovn` here — see the Comments thread for why.

---

### Phase 5 — Run `microcloud init`

On `mc1` (the initiator):

```bash
sudo microcloud init
```

Walk through the prompts, and actually read them rather than accepting defaults blindly:

1. It selects an address for MicroCloud's internal traffic — confirm it's picked the `192.168.20.140` address, not the NAT one.
2. It asks to limit the mDNS search to that `/24` — say yes, since `mc2`/`mc3` are on the same host-only segment.
3. It scans and lists the other machines it found via multicast — confirm both `mc2` and `mc3` show up. This is the live version of the discovery mechanism from INFRA-009's Q&A, not a description of it.
4. It prompts for local storage — select the unpartitioned 8 GB disk on each machine for Ceph.
5. Notice what it does **not** ask you: no networking/uplink prompt. That's MicroOVN's absence, confirmed live.

---

### Phase 6 — Verify all three layers, not just LXD

```bash
microcloud cluster list
microceph cluster list
lxc cluster list
microceph status
lxc storage list
```

The last command should show a Ceph-backed storage pool now available to LXD — that's the actual proof compute and storage are integrated, not just co-located.

---

### Phase 7 — Prove it with a real write, not just status output

Same principle as INFRA-008's Phase 6 — status text isn't proof:

```bash
lxc launch ubuntu:24.04 test-ceph-container --storage <ceph-pool-name>
lxc list
lxc delete test-ceph-container --force
```

If the container launches and deletes cleanly using the Ceph-backed pool, storage is genuinely working, not just registered.

---

### Phase 8 — Compare the effort against Week 2

Worth writing down while it's fresh: Week 2 took multiple tickets (INFRA-006 through 008) to get LXD clustering, quorum understanding, and failure detection working for compute alone. This ticket got compute *and* storage clustered, in one guided `init` run. That gap is the entire argument for MicroCloud's existence — and you now have first-hand evidence of it, not just the claim from INFRA-009.

---

## Definition of Done

- [ ] `node1`–`node3` cluster confirmed unchanged
- [ ] `mc1`–`mc3` cloned, static IPs set via MAC-matched netplan, confirmed to survive a reboot (not marked `dynamic`)
- [ ] Required snaps installed, MicroOVN deliberately absent
- [ ] `microcloud init` completed successfully with live mDNS discovery observed
- [ ] `lxc storage list` shows a working Ceph-backed pool
- [ ] Test container launched and deleted successfully using that pool
- [ ] Week 2 vs. Week 3 effort comparison written
- [ ] Retention check answered

---

## Retention check

We deliberately did not install MicroOVN this week, and `microcloud init` skipped the networking prompts as a result. What specifically do you expect to be different about the `init` flow once MicroOVN is added in Week 4 — and why does MicroCloud check for its presence rather than just always asking those questions?