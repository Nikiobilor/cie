# Day 2 — LXD Containers vs VMs: Boot Time, Disk Footprint, and Density

**Time:** ~45–60 min
**You'll need:** Day 0's `base-vm` golden image, internet access inside the VM (already confirmed working on Day 0)

---

## Why This Matters

Day 1 gave you the analogy: a VM is a fully furnished apartment, a container is a locked room in a shared house. Today you stop taking that on faith and actually watch it happen, same kernel, shared memory pool, and a boot time you can feel the difference in. This is the mechanism behind the density story from the TNTECH case study, not just a nice metaphor.

## The Concept

LXD doesn't create a new virtual computer the way VirtualBox does. It uses two features already built into the Linux kernel, namespaces (which give a process its own isolated view of the filesystem, network, and process list) and cgroups (which limit how much CPU and memory it can use). No second kernel gets booted. No hypervisor sits in between. The container is a regular process on your VM's existing kernel, just heavily fenced in.

That's why you can run LXD *inside* a VM with zero problems, unlike nesting a second VM inside a VM, which needs hardware virtualization support your setup doesn't have. Containers were never doing hardware virtualization in the first place.

## Hands-On Lab

**1. Clone and start today's VM**
```
cd C:\vbox-labs
VBoxManage clonevm "base-vm" --name "day2-vm" --register
VBoxManage startvm "day2-vm" --type headless
VBoxManage controlvm "day2-vm" natpf1 "day2ssh,tcp,,2202,,22"
```

**2. SSH in**
```
ssh labuser@127.0.0.1 -p 2202
```

**3. Install LXD**
```
sudo snap install lxd
sudo lxd init --auto
```
`--auto` accepts sensible defaults so you're not walking through an interactive wizard today,you can revisit `lxd init` manually later once you know what each prompt actually configures.

**4. Launch your first container, and actually time it**
```
time lxc launch ubuntu:24.04 my-first-container
```
Compare that number, mentally, against how long `startvm` took to boot a VM on Day 0 and Day 1. This is the boot-time half of the density story.

**5. Get a shell inside it**
```
lxc exec my-first-container -- bash
```

**6. Check the kernel — this is the reveal**
```
uname -r
```
Then exit the container (`exit`) and run the same command on the VM itself. They match. Identically. That's namespaces at work: the container isn't running a different kernel, it's borrowing yours.

**7. Check memory from inside the container**
```
lxc exec my-first-container -- free -h
```
Notice it reports figures close to the whole VM's memory, not some smaller reserved slice. Unlike a VM, a container doesn't get a hard-carved-out chunk of RAM by default, it draws from the same shared pool as everything else on the host, fenced by cgroup limits rather than a fixed allocation.

**8. Check disk footprint**
```
sudo du -sh /var/snap/lxd/common/lxd/storage-pools/default/containers/my-first-container
```
(Path may differ slightly if `lxd init --auto` picked a different storage backend on your system `lxc storage list` will show you which pool is in use if this path doesn't exist.) Compare this number against the VM disk usage you noted on Day 1.

**9. Clean up**
```
exit
lxc delete my-first-container --force
```
```
# from PowerShell
VBoxManage controlvm "day2-vm" poweroff
VBoxManage unregistervm "day2-vm" --delete
```

## Recap

- The container's kernel version was identical to the host's — proof there's no second kernel being booted.
- Memory wasn't pre-carved the way a VM's is — the container shares the pool and gets fenced by limits instead.
- Boot time and disk footprint should both have felt dramatically smaller than Day 0/Day 1's VM — that gap, multiplied across hundreds or thousands of instances, is exactly how MicroCloud tripled TNTECH's density on the same hardware.

## Retention Check

If a container shares the host's memory pool instead of getting a fixed reservation, what happens on a busy host if one container starts consuming way more than its share, and what would you want in place to stop it from starving the others? (You don't need the answer yet, just sit with the question. It's where cgroup limits and resource governance come in later.)

## Up Next — Day 3

LXD networking basics: bridges and profiles, and how a container actually reaches the outside world without a hypervisor's virtual NIC doing the work.