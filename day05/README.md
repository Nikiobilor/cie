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

**1. Clone, start, and connect**
```
cd C:\vbox-labs
VBoxManage clonevm "base-vm" --name "day5-vm" --register
VBoxManage startvm "day5-vm" --type headless
VBoxManage controlvm "day5-vm" natpf1 "day5ssh,tcp,,2205,,22"
```
```
ssh labuser@127.0.0.1 -p 2205
```

**2. Install LXD**
```
sudo snap install lxd
sudo lxd init --auto
```

**3. Launch two containers**
```
lxc launch ubuntu:24.04 app
lxc launch ubuntu:24.04 web
```
`app` will be your backend. `web` will front it.

**4. Give `app` something to serve**
```
lxc exec app -- bash -c "apt-get update -y && apt-get install -y python3"
lxc exec app -- bash -c "echo 'Hello from the app container' > /root/index.html"
lxc exec app -- bash -c "cd /root && nohup python3 -m http.server 8000 &> /var/log/http.log & disown"

lxc exec app -- bash -c "cd /root && nohup python3 -m http.server 8000 < /dev/null &> /var/log/http.log & disown"
```

**5. Install nginx on `web` and point it at `app` by name**
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

## Up Next — Week 2, Day 6

What clustering and quorum actually mean, in plain English, before you join your first two nodes together.