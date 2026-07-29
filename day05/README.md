# Day 5 — Recap and Build: A Two-Container Deployment on One LXD Host

**Time:** ~45–60 min
**You'll need:** Day 0's `base-vm` golden image, internet access inside the VM

---

## Why This Matters

Every day this week taught you one piece in isolation: kernels are shared, not duplicated (Day 1/2), containers find each other by name through a bridge (Day 3), snapshots cost different amounts depending on the backend (Day 4). Today you stop testing pieces and actually build something, two containers working together, talking to each other by name, with a safety net you can roll back to if you break it. This is the smallest possible version of what a real deployment looks like.

## The Concept

No new concept today, just the same four ideas from this week, used together instead of one at a time:

- Two containers sharing one kernel (Day 1/2), instead of two VMs each carrying their own.
- One talking to the other by container name, not a hardcoded IP (Day 3's bridge and built-in DNS).
- A snapshot taken *before* a risky change, so a mistake costs you a rollback instead of a rebuild (Day 4).

If this feels like less new information than earlier in the week, that's intentional. Recognizing when you already have what you need is its own skill.

## Hands-On Lab

**1. Clone, then resize the disk before you boot it**

Two containers, plus nginx and python3 installs, plus a `dir`-backend snapshot that needs room for a full extra copy of `app`, don't comfortably fit in the 8GB Day 0 built `base-vm` with. Resize the clone's disk now, while it's still powered off, so today's lab doesn't run out of space partway through:
```
C:\vbox-labs\
VBoxManage clonevm "base-vm" --name "day5-vm" --register
VBoxManage list hdds
```
Find the entry whose `Location` contains `day5-vm.vdi`, and copy its path or UUID.

```
VBoxManage modifymedium disk "<path or UUID you just copied>" --resize 20480
```

That grows the virtual disk to 20GB. The VM's OS doesn't know that happened yet, that comes next.

**2. Start, connect, and grow the filesystem into the new space**
```
VBoxManage startvm "day5-vm" --type headless
VBoxManage controlvm "day5-vm" natpf1 "day5ssh,tcp,,2205,,22"
```
```
ssh labuser@127.0.0.1 -p 2205
```
```
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
df -h
```
Resizing the `.vdi` only grew the container the disk lives in  `growpart` extends the partition to fill it, and `resize2fs` extends the filesystem to fill the partition. `df -h` should now show close to 20G available before you continue.

**3. Install LXD**
```
sudo snap install lxd
sudo lxd init --auto
```

**4. Launch two containers**
```
lxc launch ubuntu:24.04 app
lxc launch ubuntu:24.04 web
```
`app` will be your backend. `web` will front it.

**5. Give `app` something to serve — as a real systemd service, not a backgrounded shell command**
```
lxc exec app -- bash -c "apt-get update -y && apt-get install -y python3"
lxc exec app -- bash -c "echo 'Hello from the app container' > /root/index.html"
lxc exec app -- bash -c "cat > /etc/systemd/system/simplehttp.service << 'EOF'
[Unit]
Description=Simple HTTP server for the Day 5 lab
After=network.target

[Service]
WorkingDirectory=/root
ExecStart=/usr/bin/python3 -m http.server 8000
Restart=always

[Install]
WantedBy=multi-user.target
EOF"
lxc exec app -- systemctl daemon-reload
lxc exec app -- systemctl enable --now simplehttp
```
`nohup ... & disown` looks like it should detach a process from the terminal that launched it, but `lxc exec` tracks everything spawned during that call in its own cgroup and can tear it down when the exec session ends regardless. Even `setsid`, which puts a process in a brand new session, doesn't reliably survive that. The container already ships a real init system, systemd, as PID 1 — handing the process to it with a proper unit file sidesteps the problem entirely, because the process was never really "inside" the exec call to begin with.

`enable`, not just `start`, matters here too: it registers the service to come back on its own after a reboot. A transient unit made with `systemd-run` would run fine right now, but silently not exist anymore the next time this VM restarts, which is exactly the kind of gap that shows up two steps later and looks like a completely different bug.

**6. Install nginx on `web` and point it at `app` by name**
```
lxc exec web -- bash -c "apt-get update -y && apt-get install -y nginx"
lxc exec web -- bash -c "cat > /etc/nginx/sites-available/default << 'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:8000;
    }
}
EOF"
lxc exec web -- systemctl restart nginx
```
Notice `proxy_pass http://app:8000` uses the container's *name*, not an IP address. That only works because of Day 3's bridge DNS, no config file anywhere told `web` where `app` lives.

**6. Prove it works end to end**
```
lxc exec web -- curl http://localhost
```
You should see `Hello from the app container`, served by `app`, fetched through `web`.

**7. Snapshot `app` before touching it, this is Day 4, used on purpose this time**
```
lxc snapshot app before-recap-change
```

**8. Break it deliberately**
```
lxc exec app -- bash -c "echo 'oops, overwritten' > /root/index.html"
lxc exec web -- curl http://localhost
```
Now it shows the broken content, still served correctly through the proxy, just wrong.

**9. Roll it back**
```
lxc restore app before-recap-change
lxc exec web -- curl http://localhost
```
Back to the original response. No rebuild, no redeploying `web`, just a restore on `app` alone.

**10. Clean up**
```
lxc delete app --force
lxc delete web --force
exit
```
```
# from PowerShell
VBoxManage controlvm "day5-vm" poweroff
VBoxManage unregistervm "day5-vm" --delete
```

## Recap

- Two containers, one shared kernel underneath, talking to each other by name with zero manual network configuration.
- A snapshot taken *before* the risky step turned a mistake into a thirty-second rollback instead of a rebuild.
- This is a two-container toy version of the same shape a real deployment takes: services that find each other by name, and a safety net you set up before you need it, not after.

## Retention Check

This week, four things were configured for you without asking: which kernel a container uses, which network it joins, how it gets discovered by name, and how its storage is backed. If you'd built this same two-container setup on plain VMs instead of containers, which of those four would you have had to configure by hand yourself? Write down all four before moving on.

Let's compare the four items:

Configured automatically by LXD	If using plain VMs...
1. Kernel	You must install and boot a separate kernel for each VM. Each VM has its own operating system and kernel.
2. Network	You must create and configure the network yourself. For example, set up VirtualBox NAT, Bridged, or Host-Only networking, configure interfaces, IP addresses, etc.
3. Service discovery (DNS)	You must configure it yourself. VMs don't automatically resolve each other's names. You'd need to use static IPs, edit /etc/hosts, or run a DNS server.
4. Storage backend	You choose and manage it yourself. You decide the virtual disk format (VDI, VMDK, QCOW2, etc.) and how snapshots are implemented by the hypervisor.
So the answer is:

If I built this on plain VMs instead of LXD containers, I would have had to configure by hand:

A kernel for each VM (by installing an operating system).
Networking so the VMs could communicate.
Name resolution (DNS or /etc/hosts) so one VM could find the other by name.
The storage backend and snapshot mechanism used by the virtual disks.
The key lesson

LXD abstracts away much of the infrastructure work. When you launched:

lxc launch ubuntu:24.04 app
lxc launch ubuntu:24.04 web

LXD automatically:

Shared the host's kernel.
Connected both containers to the default bridge.
Registered their names (app and web) with its built-in DNS.
Placed their files on the default storage backend (dir in your lab).

With plain VMs, none of those conveniences exist by default. You have to make each of those decisions and configure them yourself, which is why containers are much lighter and faster to deploy.

## Up Next — Week 2, Day 6

What clustering and quorum actually mean, in plain English, before you join your first two nodes together.