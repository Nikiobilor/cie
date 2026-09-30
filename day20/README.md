# INFRA-018: Capstone: Run a Real Service on Your Private Cloud, and Keep It Up During Maintenance

**Priority:** P1 (Week 4, Day 20: Project 1 capstone, "Build Your Own Private Cloud")
**Component:** Full stack: compute (LXD), storage (Ceph), networking (OVN)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), Ceph pool `remote`, OVN network `ovntest` on `UPLINK` (`ens38`)
**Linked tickets:** Pulls together INFRA-010 to INFRA-017

---

## Summary

For four weeks, each piece of the private cloud was tested on its own: the cluster, shared storage, virtual networks, and moving workloads. Today they all work together, the way a real client would use them.

The plan, in plain words:
1. **Check** that every layer is healthy.
2. **Tidy up** leftovers from earlier tickets.
3. **Run a small website** (`web1`) on the cloud.
4. **Open it to the outside** through a fixed address.
5. **Do maintenance on a machine** while the website keeps a fixed address, and measure how long visitors can't reach it.

That last step is the real test. In production, machines need updates and repairs all the time, and clients expect their services to survive it.

---

## Acceptance Criteria

- [ ] All three layers (cluster, storage, networking) confirmed healthy before starting
- [ ] Leftover test containers, networks and old images removed
- [ ] `web1` running a web server on `ovntest`, with a fixed internal address
- [ ] `web1` reachable from outside the cloud through a network forward
- [ ] The machine running `web1` put into maintenance mode (evacuated), with the downtime measured
- [ ] The machine brought back (restored), with `web1` still reachable afterwards
- [ ] Plain-English write-up of how all four weeks come together

---

## Comments Thread

**Reviewer:** You've tested each part on its own. A client doesn't care about parts. They ask one question: "if you take a machine down for maintenance, what happens to my service?"

**NwaChi:** From INFRA-013 and INFRA-016: the data is already on every machine because of Ceph, and OVN keeps the container's address when it moves. So the service should come back on another machine with the same address.

**Reviewer:** Good. But clients don't reach containers by their inside address. They come in from outside. How does outside traffic find `web1`?

**NwaChi:** Through a network forward: a public-facing address on the uplink that passes traffic to `web1`'s inside address.

**Reviewer:** Right. So that forward is the "front door." Now, what has to stay the same for the front door to keep working after a move?

**NwaChi:** `web1`'s inside address. So I should pin it, instead of hoping DHCP hands out the same one.

**Reviewer:** Exactly. And don't move it by hand this time. LXD has a maintenance mode for a whole machine, called evacuate: it moves everything off, and restore brings it back. That's what an operator actually uses.

**NwaChi:** Will visitors notice?

**Reviewer:** For containers, yes, briefly. INFRA-013 showed true live migration isn't available for them, so evacuate does a stop, move and start. Measure the gap with a loop that checks the site every second. Don't guess it.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Part A: Check every layer (`mc1`)

Plain English: before building anything, make sure the foundations are healthy. It's like checking the tyres before a road trip.

```bash
lxc cluster list          # compute: all 3 machines ONLINE
sudo microceph.ceph -s    # storage: HEALTH_OK (or only known, explained warnings)
sudo microovn status      # networking: all 3 machines listed, databases OK
```

### Part B: Tidy up leftovers (`mc1`)

Plain English: remove the test containers and networks from earlier tickets, so today's results aren't mixed up with old ones, and free up storage space.

```bash
lxc list
lxc delete ovn-a ovn-b ovn-c flat-a flat-b --force
lxc network delete flattest
lxc network delete ovntest2
lxc image list
lxc image delete <fingerprint of each unused image>
sudo microceph.ceph df
```

Keep `ovntest` and `UPLINK`, since today uses them. Only delete an image once no container depends on it.

### Part C: Run a small website (`mc1`)

**C1. Launch `web1` on `mc3`:**
```bash
lxc launch ubuntu:24.04 web1 --network ovntest --storage remote --target mc3
```

**C2. Pin its inside address.** Plain English: tell OVN "this container always gets this address," so the front door (Part D) always points to the right place.
```bash
lxc stop web1
lxc config device override web1 eth0 ipv4.address=10.10.10.50
lxc start web1
lxc list web1
```
Expected: `web1` shows `10.10.10.50`.

**C3. Install a web server inside it.** This also proves `web1` can reach the internet.
```bash
lxc exec web1 -- apt-get update
lxc exec web1 -- apt-get install -y nginx
lxc exec web1 -- sh -c 'echo "Hello from web1 on $(hostname)" > /var/www/html/index.html'
lxc exec web1 -- curl -s http://localhost
```
Expected: `Hello from web1 on web1`.

### Part D: Open it to the outside (`mc1`)

Plain English: `10.10.10.50` only exists inside the cloud. To let outside visitors in, give `ovntest` a small block of outside addresses it's allowed to use, then create a forward: "traffic to this outside address, port 80, goes to `web1`, port 80."

**D1. Allow a small block of uplink addresses for forwards.** This uses `192.168.82.32/28` (`.32` to `.47`), which is outside both the OVN router range (`.10`–`.20`) and VMware's DHCP leases (`.128` and up).
```bash
lxc network set UPLINK ipv4.routes=192.168.82.32/28
```

**D2. Create the front door:**
```bash
lxc network forward create ovntest 192.168.82.33
lxc network forward port add ovntest 192.168.82.33 tcp 80 10.10.10.50 80
lxc network forward show ovntest 192.168.82.33
```

**D3. Test it from the host `mc1`:**
```bash
curl -s --max-time 3 http://192.168.82.33
```
Expected: `Hello from web1 on web1`.

> **If there's no answer, don't guess.** Newer OVN versions don't always answer "who has this address?" (ARP) for forward addresses. If the curl hangs, check `ip neigh show 192.168.82.33` on `mc1`. If it shows `FAILED` or `INCOMPLETE`, a known workaround is to route the forward address to the OVN router's own uplink address:
> ```bash
> sudo ip route add 192.168.82.33/32 via $(lxc network get ovntest volatile.network.ipv4.address)
> ```
> Record which case happened.

**Optional:** open `http://192.168.82.33` in a browser on your Windows PC. If Windows is attached to the same VMware NAT network, the page should load there too.

### Part E: Maintenance day (`mc1`)

**E1. Start a "visitor" loop in a second terminal on `mc1`.** It checks the site every second and prints the time and result.
```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s --max-time 1 http://192.168.82.33 || echo "DOWN"
  sleep 1
done
```
You should see a steady stream of `Hello from web1 on web1`.

**E2. Put `mc3` into maintenance mode (first terminal):**
```bash
lxc cluster evacuate mc3 --force
lxc list web1
```
Plain English: evacuate moves every workload off `mc3` so it can safely be updated or rebooted. `--force` skips the "are you sure?" prompt.

**Record:**
- Where did `web1` go?
- In the loop: when was the last reply, the first `DOWN`, and the first reply again? How many seconds was it down?

**E3. Bring `mc3` back:**
```bash
lxc cluster restore mc3 --force
lxc list web1
```
**Record:** did `web1` move back to `mc3`? Was there a second gap in the loop?

Stop the loop with `Ctrl+C`.

### Part F: Write-up

In plain words, from the real results:
- What each week contributed: the cluster (Weeks 1–2), shared storage (Week 3), and virtual networks (Week 4).
- Why `web1` came back on a different machine **with no data copied** (Ceph) and **at the same address** (OVN plus the pinned address).
- How long visitors were cut off, and why (a cold move, not live migration).
- **Lab-vs-reality gap:** production would run **two or more copies** of the website behind a load balancer. Then evacuating one machine causes no visible downtime at all, because the other copy keeps answering. One copy, like today, always means a short outage during maintenance.

---

## Definition of Done

- [ ] Health check recorded for all three layers
- [ ] Leftovers removed and storage freed
- [ ] `web1` serving a page at a pinned internal address
- [ ] The network forward working from outside, with the ARP result recorded
- [ ] Evacuation and restore done, with downtime measured from the loop
- [ ] Write-up completed

---

## Retention Check

When `mc3` was evacuated, `web1` came back on another machine with the same address and the same data, without copying anything. Which two parts of the private cloud made that possible, and what would you add to make the downtime zero?