# Day 3 — LXD Networking: Bridges and Profiles

**Time:** ~45–60 min
**You'll need:** Day 0's `base-vm` golden image, internet access inside the VM

---

## Why This Matters

Every container you launched on Day 2 could reach the internet and get an IP address, and you never configured any of that. Today you find out what was doing that work quietly in the background, because understanding it now is what makes MicroOVN's software-defined networking (coming in Week 4) click instead of feeling like magic.

## The Concept

When you ran `lxd init --auto` on Day 2, LXD created something called a **bridge** — think of it as a virtual network switch living inside your VM. Every container plugs into it automatically, the same way multiple laptops plug into the same physical switch in an office. The bridge then does NAT to get traffic out to the real internet through your VM's own network connection. No hypervisor virtual NIC involved, just a Linux bridge and some routing rules, because containers don't need a hypervisor in the first place.

The second piece is a **profile** — a named, reusable bundle of settings (which network the container joins, what resource limits apply, and more) that gets stamped onto every new container automatically unless you say otherwise. Every container you've launched so far quietly inherited LXD's `default` profile. Today you actually look at what's in it.

## Hands-On Lab

**1. Clone, start, and connect**
```
cd C:\vbox-labs
VBoxManage clonevm "base-vm" --name "day3-vm" --register
VBoxManage startvm "day3-vm" --type headless
VBoxManage controlvm "day3-vm" natpf1 "day3ssh,tcp,,2203,,22"
```
```
ssh labuser@127.0.0.1 -p 2203
```

**2. Install LXD (same as Day 2 — each day starts from a clean image)**
```
sudo snap install lxd
sudo lxd init --auto
```

**3. Find the bridge LXD created for you**
```
lxc network list
```
You should see `lxdbr0` in the output. That's the virtual switch every container will plug into.

**4. See what the default profile actually contains**
```
lxc profile show default
```
Look for the `eth0` device entry, it names `lxdbr0` as the network every new container connects to. This is the "unless you say otherwise" part of a profile: you never typed this, it was applied for you.

**5. Launch a container and check what it was handed**
```
lxc launch ubuntu:24.04 net-test
lxc list
```
The `IPV4` column shows an address the bridge assigned automatically, this came from the same profile you just inspected, not from anything you configured by hand.

**6. Confirm it can actually reach the outside world**
```
lxc exec net-test -- curl -I http://archive.ubuntu.com
```
A `200 OK` confirms the bridge's NAT is doing its job, traffic went container → bridge → your VM's network → internet.

**7. Launch a second container and check if they can find each other**
```
lxc launch ubuntu:24.04 net-test-2
lxc exec net-test -- ping -c 2 net-test-2
```
Notice you pinged it by **name**, not IP. LXD runs a small built-in DNS resolver on the bridge so containers can find each other without you tracking IPs manually, a preview of what "service discovery" means once you're doing this at cluster scale.

**8. Clean up**
```
lxc delete net-test --force
lxc delete net-test-2 --force
exit
```
```
# from PowerShell
VBoxManage controlvm "day3-vm" poweroff
VBoxManage unregistervm "day3-vm" --delete
```

## Recap

- Every container joins `lxdbr0` automatically because of the `default` profile — you never configured networking by hand and it worked anyway.
- The bridge does NAT and DNS for every container on it, the same job a physical office switch plus a small router would do.
- Profiles are *why* this is repeatable: change one setting in a profile, and every future container inherits it — you're not reconfiguring machines one at a time.

## Retention Check

If a profile applies its settings to every new container automatically, what's the actual advantage of changing a setting in the profile instead of just fixing it manually on each container you already have running? Sit with that — it's the same reasoning behind Day 0's golden image, just one layer up.

## Up Next — Day 4

LXD storage pools: how a container's disk actually works, and why the backend you pick (dir vs. ZFS/btrfs) changes what you can do with it later — like snapshots.

