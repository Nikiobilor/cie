# Day 4 — LXD Storage Pools: How a Container's Disk Actually Works

**Time:** ~45–60 min
**You'll need:** Day 0's `base-vm` golden image, internet access inside the VM

---

## Why This Matters

Every container you've launched this week had to store its filesystem somewhere. You never picked where or how, LXD decided for you when you ran `lxd init`. Today you find out what it actually chose, because that choice quietly decides whether "make a backup of this container" is instant and nearly free, or a slow full copy. That's not a small detail. It's the same question distributed storage systems like Ceph answer at cluster scale in Week 3, just sitting right in front of you on one machine first.

## The Concept

LXD keeps every container's filesystem inside something called a **storage pool**, a chunk of disk space managed by a specific backend driver. The driver matters more than the disk itself.

The simplest driver, **dir**, treats your storage pool like a plain filing cabinet. It works, and it's what a fresh install often defaults to. But making a copy, a snapshot, means physically duplicating every file, byte for byte.

Drivers like **ZFS** or **Btrfs** work differently. They're copy-on-write filesystems, meaning a snapshot doesn't copy anything at first. It just marks "this is the state right now" and only starts using extra space once something actually changes afterward. Think of it as the difference between photocopying an entire filing cabinet versus just writing today's date on a sticky note and leaving the files exactly where they are, only pulling a real copy of a page if someone actually edits it later.

## Hands-On Lab

**1. Clone, start, and connect**
```
cd C:\vbox-labs
VBoxManage clonevm "base-vm" --name "day4-vm" --register
VBoxManage startvm "day4-vm" --type headless
VBoxManage controlvm "day4-vm" natpf1 "day4ssh,tcp,,2204,,22"
```
```
ssh labuser@127.0.0.1 -p 2204
```

**2. Install LXD (same as every day this week — clean image each time)**
```
sudo snap install lxd
sudo lxd init --auto
```

**3. Find out which backend you actually got**
```
lxc storage list
```
Look at the `DRIVER` column for the `default` pool. On a fresh Ubuntu VM this is very often `dir`, the plain filing cabinet, not because it's the best choice, but because it needs nothing extra to work. Today you'll feel exactly what that trade-off costs.

**4. Launch a container and check its baseline disk usage**
```
lxc launch ubuntu:24.04 storage-test
lxc storage info default
```
Note the `Used` figure under the pool's usage stats.

**5. Take a snapshot**
```
lxc snapshot storage-test before-change
lxc storage info default
```
Check the `Used` figure again. If your pool is running on `dir`, you should see it jump by roughly the size of the container itself, because a snapshot on this driver is a real, full copy. On ZFS or Btrfs, this same command would barely move the number at all.

**6. Prove the snapshot actually captured a real state**

Make a change, then restore:
```
lxc exec storage-test -- bash -c "echo 'this should disappear' > /root/temp.txt"
lxc restore storage-test before-change
lxc exec storage-test -- ls /root/
```
`temp.txt` is gone. The snapshot rolled the container back to exactly the state it captured, regardless of which backend did the work underneath.

**7. Clean up**
```
lxc delete storage-test --force
exit
```
```
# from PowerShell
VBoxManage controlvm "day4-vm" poweroff
VBoxManage unregistervm "day4-vm" --delete
```

## Recap

- LXD chose a storage backend for you the same way it chose a network bridge on Day 3, a default you inherited without asking for it.
- On a `dir` backend, a snapshot is a full copy: reliable, but it costs real disk space and time, every single time.
- Restoring worked regardless of backend, but *how much it costs* to get there is exactly the same trade-off Ceph will force you to make deliberately in Week 3, at a much bigger scale.

## Retention Check

If a plain directory backend copies the entire container to make a snapshot, but ZFS or Btrfs only track what's changed, what would you expect to happen to snapshot speed and disk usage as a container gets bigger, under each approach? You don't need the exact numbers, just the direction each one moves.

## Up Next — Day 5

Recap and build day: run multiple containers on one LXD host at once, and put this week's four concepts, kernels, bridges, profiles, and storage, to work together in one small deployment.