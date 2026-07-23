# Day 1 — What Is Virtualization, and Why Does LXD Do It Differently?

**Time:** ~45–60 min
**You'll need:** Day 0's `base-vm` already built

---

## Why This Matters

Every cloud platform — AWS, VMware, your future MicroCloud cluster — starts from the same problem: how do you run many isolated things on one physical machine without them crashing into each other? Get this wrong and you either waste hardware or break isolation. Today you build the mental model everything else this challenge sits on top of.

## The Concept

A **VM** is like giving each tenant a fully furnished apartment — its own plumbing, its own everything. Safe, but expensive to build and slow to spin up, because it's booting an entire second operating system (its own kernel) on top of yours.

A **container** (what LXD uses) is like giving each tenant a locked room in a shared house — they share the building's plumbing (your host's kernel) but can't see into each other's rooms. Less duplicated infrastructure per tenant, which is exactly why LXD-based systems (like MicroCloud) pack tighter than VMware — that's the "tripled density" from the TNTECH case study.

Today you'll only touch the VM side, so you can *feel* the weight of it before Day 2 shows you the lighter alternative.

## Hands-On Lab

**1. Clone and start today's VM from your Day 0 base image**
```
cd C:\vbox-labs
VBoxManage clonevm "base-vm" --name "day1-vm" --register
VBoxManage startvm "day1-vm" --type headless
```
Notice the boot itself takes seconds — that's the payoff of Day 0's one-time setup.

**2. Give it its own forwarded port**

Clones inherit `base-vm`'s NAT rule, which is already named `"ssh"` — adding another rule with that same name fails even on a different port, because VirtualBox matches rules by name, not port number. Give each day's rule its own name instead:
```
VBoxManage controlvm "day1-vm" natpf1 "day1ssh,tcp,,2201,,22"
```

**3. SSH in**
```
ssh labuser@127.0.0.1 -p 2201
```

**4. Confirm it has its own kernel**
```
uname -r
```
This is running a separate kernel from your host machine — proof it's a real, isolated OS, not a shared process.

**5. Check its footprint**
```
free -h
df -h
```
Note the disk usage. Write this number down — Day 2's LXD container will do a similar job at a fraction of this size, and that gap is the whole point of hyper-dense virtualization.

**6. Clean up**
```
exit
VBoxManage controlvm "day1-vm" poweroff
VBoxManage unregistervm "day1-vm" --delete
```
Deleting the clone (not the base) keeps every future day starting from the same clean state.

## Recap

- You cloned a VM in seconds → it still booted its own kernel → that's the overhead a container skips.
- The disk footprint you noted is the "cost" of the full-apartment model.
- This is the exact overhead LXD containers avoid — which is why TNTECH tripled their workload density switching from VMware VMs to LXD.

## Retention Check

If a friend asked, "why not just use containers for everything, then?" — write your answer in one or two sentences before moving on. (Hint: think about what a container *can't* isolate that a VM can.)

## Up Next — Day 2

Install LXD on a cloned VM, launch a container doing the same basic task, compare boot time and disk footprint side-by-side against today's numbers.