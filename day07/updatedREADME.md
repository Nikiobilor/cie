# Building a 3-node LXD cluster on VMware Workstation Pro

This guide walks through building three Ubuntu VMs on VMware Workstation Pro and joining them into a real, fault-tolerant LXD cluster — one that can actually survive losing a node, not just look like it can.

If you're new to some of these tools: LXD is a system for running lightweight virtual machines and containers, and it can run across multiple physical or virtual servers as a "cluster" so the workload survives if one server goes down. Getting that survival property to actually work is what most of this guide is about.

## What you'll need

- VMware Workstation Pro installed on Windows (this guide assumes Pro — Player's cloning and networking tools are more limited)
- An Ubuntu Server ISO (this was built on 26.04 LTS)
- Around 60GB of free disk space (20GB per node, three nodes)
- A terminal you're comfortable typing commands into

## The shape of what you're building, before you start

A few decisions are baked into this guide. Knowing why up front will save you from second-guessing the steps later.

**One golden image, cloned three times.** Rather than installing Ubuntu from scratch three times, you install it once, configure it properly, and clone that clean image for each node. This is the same idea as baking an AMI in AWS or a VM template in vSphere.

**Two network adapters per node.** Each node gets a NAT adapter (for internet access, so it can install packages) and a host-only adapter (a private network that only exists between your VMs, used for node-to-node cluster traffic). Keeping these separate matters — you don't want cluster traffic mixed in with general internet traffic.

**Three nodes, not two.** This is the single most important thing to understand before you start. LXD's clustering is built on a consensus algorithm (similar to Raft, if you've encountered that term) where a majority of nodes has to agree before any change is considered official. With only two nodes, the math doesn't work in your favor — losing either one immediately breaks the majority, so LXD doesn't even bother making the second node a full voting member. It just sits idle as a standby, permanently. Three nodes is the minimum where a majority (2 out of 3) can survive losing one member. That's the whole point of clustering, so this guide targets three nodes from the start rather than two.

---

## Part 1: Build the golden image

**1.1 — Create the VM**

In Workstation Pro: **File → New Virtual Machine → Custom (advanced)**. Choose Linux → Ubuntu 64-bit, point it at your Ubuntu ISO, and give it a 20GB disk. Leave the network adapter on NAT for now — you'll add the second adapter per-node later, not on this base image.

**1.2 — Install Ubuntu**

Run through the installer normally. When you reach the SSH setup screen, **tick "Install OpenSSH server."** This matters — a minimal/unattended install can skip it silently, and you'd only discover that the hard way later when you try to SSH in and can't.

**1.3 — Install VMware's guest tools**

Once installed and logged in:
```bash
sudo apt update
sudo apt install -y open-vm-tools
```
This is VMware's equivalent of VirtualBox's Guest Additions — without it, things like copy-paste between host and guest, clean shutdowns, and proper network adapter behavior don't work right.

**1.4 — Confirm SSH is actually running**

```bash
sudo systemctl status ssh
```
You want to see `active (running)`. Don't skip this check just because you ticked the box in the installer.

**1.5 — Snapshot it as your template**

Shut the VM down cleanly (`sudo shutdown now`), then in Workstation Pro take a snapshot and name it something like `golden-clean`. This snapshot is what you'll clone from — never boot and modify this VM directly again, or your "clean" template stops being clean.

---

## Part 2: Set up networking (do this before cloning)

Open **Edit → Virtual Network Editor** in Workstation Pro (click **Change Settings** if it opens read-only — needs admin rights). Find your host-only network — usually `VMnet1` — and note its subnet (something like `192.168.20.0/24`; the exact number varies per machine, don't assume it matches this guide exactly).

Confirm `VMnet8` (NAT) is also present and enabled.

**A warning worth taking seriously:** Workstation Pro's network dropdown shows both a NAT option and multiple "Custom" options, and it's easy to pick the wrong one — for example `VMnet0` (bridged, onto your real network) instead of `VMnet1` (host-only, private to your VMs). They look similar in the dropdown but behave completely differently. When you configure each node's second adapter later, don't just trust the label — open **Advanced...** on each adapter and note its MAC address too, so you can double check inside the guest OS which physical adapter you're actually looking at.

---

## Part 3: Clone your three nodes

For each of `node1`, `node2`, and `node3`:

1. Right-click `golden-clean` (your snapshot) → **Manage → Clone…**
2. Choose **Create a full clone**, not a linked clone. A linked clone stays dependent on the original snapshot; a full clone is a completely independent copy, which is what you want for infrastructure that's going to run indefinitely.
3. Name it accordingly.
4. Open its Settings → **Add** → Network Adapter → set it to **Custom** → select the host-only network you identified in Part 2. Open **Advanced...** on both adapters and write down their MAC addresses — you'll need these in Part 5.

---

## Part 4: Give each clone its own identity

Power on each node and log in. This part matters because a full clone copies everything byte-for-byte — including things that are supposed to be unique per machine. Without fixing this, your three "different" servers are secretly identical twins as far as the network is concerned.

On each node, run:
```bash
sudo hostnamectl set-hostname node1    # use node2 / node3 as appropriate
sudo sed -i 's/base-vm/node1/' /etc/hosts

sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup

sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo systemctl restart ssh

sudo reboot
```

Machine ID is used by systemd and other services to identify the host — duplicated IDs across nodes can cause confusing network identity conflicts. SSH host keys are what your terminal checks to make sure it's really talking to the server it thinks it is — if two machines share a host key, that verification is meaningless.

(You'll likely get an SSH "host key has changed" warning the next time you connect to a node you've done this on — that's expected and correct, it means the fix worked. Clear it with the `ssh-keygen -R` command your terminal shows you, then reconnect.)

---

## Part 5: Confirm interfaces and set static IPs

After reboot, run:
```bash
ip a
```
You'll see two interfaces. **Don't assume names like `ens33`/`ens37` mean the same thing on every node** — clones can enumerate their NICs in a different order. Instead, compare the `link/ether` (MAC address) shown for each interface against the MACs you noted from Workstation's Advanced panel in Part 3. That tells you, with certainty, which interface is NAT and which is host-only, on this specific node.

DHCP will assign temporary addresses to both. This is worth replacing with a fixed address on the host-only side, because DHCP-assigned addresses can change on reboot — and when they do, the cluster loses track of where its members actually are. Once you know which interface name is host-only for this node:

```bash
sudo tee /etc/netplan/00-installer-config.yaml > /dev/null << 'EOF'
network:
  ethernets:
    <your-NAT-interface-name>:
      dhcp4: true
      dhcp6: true
    <your-hostonly-interface-name>:
      addresses: [192.168.20.128/24]
  version: 2
EOF
sudo netplan apply
```
Give each node a different static address on the same subnet (e.g. `.128`, `.130`, `.131`).

Then, on **every** node, add all three nodes to `/etc/hosts`:
```
192.168.20.128 node1
192.168.20.130 node2
192.168.20.131 node3
```

Confirm connectivity between every pair of nodes:
```bash
ping -c 3 node1
ping -c 3 node2
ping -c 3 node3
```
Don't move on until every node can reach every other node by name.

---

## Part 6: One preventative fix before installing anything

Freshly cloned VMs can hit a specific kernel issue where installing a snap package (which is how LXD gets installed) hangs forever, waiting on a signal that got silently dropped due to a kernel buffer overflow on the udev event socket. Rather than wait to hit this, apply the fix on every node now:

```bash
sudo tee /etc/sysctl.d/99-netlink-buffer.conf > /dev/null << 'EOF'
net.core.rmem_max=8388608
net.core.rmem_default=8388608
EOF
sudo sysctl --system
sudo systemctl restart systemd-udevd
```

---

## Part 7: Install LXD and build the cluster

**7.1 — Bootstrap `node1`**

```bash
sudo snap install lxd
sudo lxd init
```
Answer the prompts:
- Use LXD clustering? **yes**
- What address should be used to reach this server? Type your host-only IP **explicitly** (e.g. `192.168.20.128`) — don't accept whatever's defaulted, it may pick the wrong interface.
- Joining an existing cluster? **no** (this is the first node)
- Storage backend: type **`dir`** explicitly, not Enter. `dir` stores containers as plain directories with no extra loop-device layer — simpler and more predictable for a small setup like this than `zfs` or `btrfs`.
- Remote storage pool / MAAS / existing bridge? **no** to all
- Create a new Fan overlay network? **no** — this is a known-broken legacy LXD feature; you'll create a normal bridge manually in Part 9.
- Available over the network? **yes**

Confirm it worked:
```bash
lxc cluster list
```
You should see just `node1`, healthy.

**7.2 — Join `node2` and `node3`**

Generate a one-time join token from `node1`:
```bash
lxc cluster add node2
```
Copy the token it prints.

On `node2` — **do this from the Workstation console window, not over SSH.** A dropped SSH connection mid-install can leave the install in a broken half-finished state (this happened during testing and cost real time to untangle).
```bash
sudo snap install lxd
sudo lxd init
```
- Clustering: **yes**
- Joining an existing cluster: **yes**, paste the token
- Storage backend: **dir**
- Reachable address: type `node2`'s host-only IP explicitly

Repeat the whole process for `node3`.

Confirm:
```bash
lxc cluster list
```
All three should show `ONLINE`. `node2` and `node3` may briefly show `database-standby` — give it a minute or two. Once a third real member exists, both should promote to full `database` voters on their own. If either is still stuck on standby after a few minutes with no errors in the logs, that usually means something's off in the earlier networking steps, not the LXD config itself — worth re-checking Part 5.

---

## Part 8: Create the shared network bridge

Run all of these from a single node (any one) — the `--target` flag is what tells the cluster which member each command applies to, not which terminal you're typing into:

```bash
lxc network create lxdbr0 --target node1
lxc network create lxdbr0 --target node2
lxc network create lxdbr0 --target node3
lxc network create lxdbr0
lxc profile device add default eth0 nic network=lxdbr0
```
The first three stage each node's local piece of the config; the fourth (no `--target`) finalizes it cluster-wide, which only works once every member has staged its part.

Confirm:
```bash
lxc network list
```
You should see `lxdbr0` listed as `MANAGED: yes`.

---

## Part 9: Prove the cluster actually survives a failure

This is the step that validates everything above was worth doing. Rather than launching a full container (which needs to download an image from the internet — an unrelated dependency that can fail for reasons that have nothing to do with your cluster), test with a lightweight database write instead:

**With all 3 nodes up**, confirm a baseline write works:
```bash
lxc cluster group create baseline-test
lxc cluster group delete baseline-test
```

**Phase A — lose one node, majority holds.** Power off `node3` (from Workstation, or `sudo shutdown now` inside it). With `node1` and `node2` still up (2 of 3 — still a majority):
```bash
lxc cluster group create quorum-test-a
```
This should succeed. The cluster is designed to keep working with one member down.

**Phase B — lose a second node, majority breaks.** Also power off `node2`, leaving only `node1` (1 of 3 — a minority):
```bash
lxc cluster group create quorum-test-b
```
This should fail or hang. No majority means no writes — that failure is the actual proof the clustering is doing its job.

Power `node2` and `node3` back on, confirm `lxc cluster list` shows all three healthy again, and clean up:
```bash
lxc cluster group delete quorum-test-a
lxc cluster group delete quorum-test-b
```

---

## Troubleshooting quick reference

**`snap install lxd` hangs forever** — almost always the netlink buffer issue from Part 6. Check `sudo dmesg | grep -i netlink` for confirmation, apply the sysctl fix if you skipped it, then `sudo snap abort <change-id>` (find it with `sudo snap changes`) and retry.

**A node is stuck on `database-standby` forever** — normal and expected with only 2 nodes (see the explanation at the top of this guide). Add a third node.

**"No route to host" / "Connection timed out" over SSH** — usually one of: the VM is actually powered off, its IP changed since your last check, or (if you're SSHing from WSL specifically) WSL2's own network is isolated from VMware's — try connecting from plain PowerShell instead.

**SSH "host key has changed" warning** — expected right after Part 4's identity fixup. Clear it with the `ssh-keygen -R` command shown in the warning, then reconnect and accept the new key.

**A node ends up on the wrong subnet after cloning** — almost always a wrong VMnet selection (see the warning in Part 2), not a software issue. Double-check the Advanced panel's MAC address against what the guest OS actually shows.

**LXD daemon won't start / crash-loops on boot** — check `sudo journalctl -u snap.lxd.daemon -n 100`. A corrupted cluster recovery file is one known cause; if you see a `recovery tarball` error, removing `/var/snap/lxd/common/lxd/database/lxd_recovery_db.tar.gz` and `patch.global.sql` and restarting the daemon usually clears it.