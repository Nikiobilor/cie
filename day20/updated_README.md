# INFRA-018: Capstone: Run a Real Service on Your Private Cloud, and Keep It Up During Maintenance

**Priority:** P1 (Week 4, Day 20: Project 1 capstone, "Build Your Own Private Cloud")
**Component:** Full stack: compute (LXD), storage (Ceph), networking (OVN)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), Ceph pool `remote`, OVN network `ovntest` on `UPLINK` (`ens38`)
**Linked tickets:** Pulls together INFRA-010 to INFRA-017

> **Note:** this ticket was rewritten after the session to record what actually happened. The plan was interrupted by a real problem: **containers on OVN couldn't download anything.** Finding and fixing it became the most valuable part of the day, and it's documented in full in Part C.

---

## Summary

For four weeks, each piece of the private cloud was tested on its own. Today they worked together: a small website (`web1`) ran on the cloud, was opened to the outside through a fixed address, and stayed at that address while its machine was taken down for maintenance.

**Results:**

| What | Result |
|---|---|
| Website reachable from outside the cloud | **Yes**: from `mc1` and from a Windows browser, at `192.168.82.33` |
| Downtime when its machine was evacuated | **~17 seconds** |
| Downtime when the machine was restored | **~18 seconds** |
| Data copied during the moves | **None**, because Ceph holds it on every machine |
| Front-door address during the moves | **Unchanged** |

**Unplanned finding:** containers on `ovntest` could ping the internet but couldn't download. The cause was **packet size**. The tunnel between machines made `ovntest`'s packet limit 1442 bytes, but downloads arrived in 1460-byte pieces, which were dropped. The fix was to give the network between the machines room for the tunnel (MTU 1600), then set `ovntest` back to the normal 1500.

---

## Acceptance Criteria

- [x] All three layers (cluster, storage, networking) checked before starting
- [x] Leftover test containers, networks and old images removed (see Part B note)
- [x] `web1` running a web server on `ovntest`, with a fixed internal address (`10.10.10.50`)
- [x] `web1` reachable from outside the cloud through a network forward
- [x] The machine running `web1` evacuated, with downtime measured (~17 s)
- [x] The machine restored, with `web1` back on it and still reachable (~18 s)
- [x] Plain-English write-up of how all four weeks come together
- [x] **Added:** download failure on OVN diagnosed and fixed

---

## Comments Thread

**Reviewer:** A client asks one question: "if you take a machine down for maintenance, what happens to my service?"

**NwaChi:** The data is already on every machine because of Ceph, and OVN keeps the container's address when it moves. So it should come back on another machine with the same address.

**Reviewer:** Clients don't reach containers by their inside address, though. How does outside traffic find `web1`?

**NwaChi:** Through a network forward: an outside address that passes traffic to `web1`'s inside address. So I pin that inside address, so the forward always points to the right place.

**Reviewer:** Good. Go build it.

**NwaChi:** Problem. `apt-get update` inside `web1` has been running for 20 minutes. Small files come through, big ones stall.

**Reviewer:** Don't guess. Is it the internet, or the cloud?

**NwaChi:** I downloaded the same file from the host and from `web1`. The host got about 800 KB/s. `web1` got zero.

**Reviewer:** So it's inside the cloud. What's different about `web1`'s path?

**NwaChi:** Its traffic crosses the tunnel to the gateway machine. But moving `web1` next to the gateway didn't help, and neither did turning off packet-merging on the uplink card. So I captured the packets. They arrive in 1460-byte pieces, and `web1` can only take about 1402. The server keeps resending the same pieces, because they never arrive.

**Reviewer:** That's the classic overlay-network trap. The tunnel adds its own wrapper, so the network inside it is smaller than normal. Something along the way isn't respecting that. Real clouds solve it by making the network *under* the tunnel bigger.

**NwaChi:** Then that's the fix: MTU 1600 between the machines, and 1500 inside OVN.

**Reviewer:** And after that, the maintenance test. Use evacuate, not manual moves, and measure the gap with a loop.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Part A: Check every layer (`mc1`)

Plain English: before building anything, make sure the foundations are healthy.

```bash
lxc cluster list          # compute: all 3 machines ONLINE
sudo microceph.ceph -s    # storage: HEALTH_OK, or only known warnings
sudo microovn status      # networking: all 3 machines, databases OK
```

### Part B: Tidy up leftovers (`mc1`)

```bash
lxc list
lxc delete ovn-a ovn-b ovn-c flat-a flat-b --force
lxc network delete flattest
lxc network delete ovntest2
lxc image list
lxc image delete <fingerprint of each unused image>
```

Keep `ovntest` and `UPLINK`. Only delete an image once no container depends on it.

> The cleanup output wasn't captured in this session. On a repeat run, record `lxc list` and `sudo microceph.ceph df` before and after.

### Part C: Run a small website

#### C1. Launch `web1` on `mc3`

```bash
lxc launch ubuntu:24.04 web1 --network ovntest --storage remote --target mc3
```

#### C2. Pin its inside address

Plain English: tell OVN "this container always gets this address," so the front door always points to the right place.

The first attempt failed:
```
$ lxc config device override web1 eth0 ipv4.address=10.10.10.50
Error: The device already exists
```

**Why:** `override` is for a network card a container *inherits* from a shared profile. Launching with `--network ovntest` gives the container its **own** `eth0`, so you change it directly instead:

```bash
lxc stop web1
lxc config device set web1 eth0 ipv4.address=10.10.10.50
lxc start web1
lxc list web1
```

#### C3. Install a web server, and the problem that appeared

```bash
lxc exec web1 -- apt-get update
```

**Real result:** after about 20 minutes, it still hadn't finished. Small files came through (`Hit`, and small `Get` lines), but bigger ones stalled (`Ign`, `Connection timed out`, `0% [Waiting for headers]`).

Press `Ctrl+C` to stop it. That's safe for `apt`.

#### C3a. Is it the internet, or the cloud?

Download the same file from the host and from inside `web1`:
```bash
curl -o /dev/null -s -w "host: %{speed_download} bytes/sec\n" --max-time 30 \
  http://archive.ubuntu.com/ubuntu/dists/noble/main/binary-amd64/Packages.gz
lxc exec web1 -- curl -o /dev/null -s -w "web1: %{speed_download} bytes/sec\n" --max-time 30 \
  http://archive.ubuntu.com/ubuntu/dists/noble/main/binary-amd64/Packages.gz
```

**Real result:** host **~800 KB/s**, `web1` **0**. The internet is fine, so the problem is inside the cloud.

#### C3b. Suspect 1: packets too big going out? Ruled out

```bash
lxc exec web1 -- ip link show eth0                  # mtu 1442
lxc exec web1 -- ping -c 3 -M do -s 1200 8.8.8.8
lxc exec web1 -- ping -c 3 -M do -s 1414 8.8.8.8    # 1414 + 28 = 1442
```

`-M do` forbids splitting a packet into smaller pieces. **Real result:** both sizes worked, in both directions. Outgoing size isn't the problem.

#### C3c. Suspect 2: the tunnel between machines? Ruled out

Plain English: `web1`'s traffic first travels through the tunnel to whichever machine holds the network's exit (the gateway).

```bash
sudo microovn.ovn-sbctl show     # look for cr-lxd-net8-lr-lrp-ext
```

**Real result:** the gateway was on `mc2`, and `web1` was on `mc3`. Moving `web1` next to the gateway removes the tunnel from the path:

```bash
lxc stop web1 && lxc move web1 --target mc2 && lxc start web1
```

**Real result:** still **35 bytes/sec**, which is effectively zero. The tunnel isn't the cause.

#### C3d. Suspect 3: packet merging on the uplink card? Ruled out

Plain English: GRO (Generic Receive Offload) lets a network card glue many small incoming packets into one big one, and that can break overlay networks.

On the gateway node (`mc2`):
```bash
ethtool -k ens38 | grep -E "generic-receive|large-receive"   # GRO on, LRO off [fixed]
sudo ethtool -K ens38 gro off
ethtool -k ens38 | grep generic-receive                       # confirm: off
```

**Real result:** still zero. GRO isn't the cause.

> `ethtool -K` changes don't survive a reboot, so GRO on `mc2`'s `ens38` returns to its default at the next boot. That's fine, because it wasn't the problem.

#### C3e. Look at the actual packets: found it

On `mc2`:
```bash
sudo tcpdump -ni ens38 -c 20 'tcp and src port 80'
```
Meanwhile, on `mc1`, start the download from `web1`.

**Real result (trimmed):**
```
11:35:58.743862 ... Flags [.], seq 1:1461,    ... length 1460: HTTP: HTTP/1.1 200 OK
11:35:58.743863 ... Flags [.], seq 1461:2921, ... length 1460: HTTP
...
11:35:58.835452 ... Flags [.], seq 1:1461,    ... length 1460: HTTP: HTTP/1.1 200 OK   <- sent again
```

**What it shows:**
- Download data arrives in **1460-byte** pieces.
- `web1` can only accept about **1402** bytes of data per piece (its 1442 limit, minus headers). So every full-size piece is dropped.
- The server **resends the same pieces** (`seq 1:1461` twice), because `web1` never confirms receiving them.
- Pings worked only because we chose their size ourselves.

**Why the sender doesn't use smaller pieces:** normally `web1` announces its maximum size and the sender obeys. Something in between isn't respecting that. The most likely candidate is VMware's NAT, which sits in the middle of every connection. That fits the evidence, but it wasn't proven: `web1`'s own announcement wasn't captured.

#### C3f. The fix: make the network under the tunnel bigger

Plain English: the tunnel wraps each packet in about 58 extra bytes. If the network *between* the machines (`ens33`) carries packets up to 1600 bytes, a normal 1500-byte packet plus its wrapping fits. Then `ovntest` can use the normal 1500. This is how production clouds run OVN.

**1. Test whether VMware allows bigger packets.** This is temporary and undone by a reboot:
```bash
sudo ip link set ens33 mtu 1600                 # on mc2, then on mc1
ping -c 3 -M do -s 1572 192.168.20.141          # from mc1: 1572 + 28 = 1600
```
**Real result:** replies in under 1 ms. VMware allows it.

> If there's no reply, undo immediately with `sudo ip link set ens33 mtu 1500`, because `ens33` carries all the cluster's own traffic.

**2. Make it permanent: add `mtu: 1600` to each node's `ens33` block in netplan.**

> **Warning: what went wrong here.** Running `sudo netplan try` printed:
> ```
> ['ens38', 'lxdovn7']
> Cannot find unique matching interface for ens38
> ```
> The OVN bridge `lxdovn7` had copied `ens38`'s MAC address. Netplan matches `ens38` by MAC, so it found two interfaces and couldn't choose between them.
>
> On `mc3`, the network then stopped working entirely. SSH gave `No route to host`, and the console was flooded with:
> ```
> e1000 0000:02:01.0 ens33: Detected Tx Unit Hang
> ```
> That means `mc3`'s virtual network card (VMware's `e1000` type) got stuck and couldn't send. It happened right after the live network change. The exact trigger isn't proven: `mc1` ran at 1600 on the same setup without problems.
>
> **Recovery:**
> 1. On `mc1`: `sudo microceph.ceph osd set noout`, so storage doesn't start rebuilding.
> 2. In VMware, reset `mc3` (**VM → Power → Reset**).
> 3. It booted with `mtu 1600` and a working network.
> 4. From `mc1`, both `ping 192.168.20.142` and a 1600-byte ping succeeded.
>
> If the console floods and you can't type, keystrokes still register. `sudo dmesg -n 1` stops kernel messages from appearing on the console.

**3. Fix the MAC confusion permanently: match `ens38` by name *and* MAC.** On each node:
```yaml
    ens38:
      match:
        macaddress: 00:0c:29:57:a7:1c   # each node's own ens38 MAC
        name: ens38
      set-name: ens38
      dhcp4: no
      dhcp6: no
```
And the `ens33` block, on each node:
```yaml
    ens33:
      match:
        macaddress: 00:0c:29:57:a7:08   # each node's own ens33 MAC
      set-name: ens33
      addresses: [192.168.20.140/24]    # each node's own address
      mtu: 1600
```

**Don't apply it live again.** All nodes were already running at 1600. Just check the files are valid:
```bash
sudo netplan generate      # no output means valid; applies cleanly at next boot
```
Then bring storage back to normal:
```bash
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s
```

**4. Raise `ovntest` to the normal 1500.** The first attempt failed:
```
$ lxc network set ovntest bridge.mtu=1500
Error: Uplink address key "volatile.network.ipv6.address" cannot be empty when network address key "ipv6.address" is populated
```
**Why:** LXD checks the whole network config on any change. `ovntest` handed out IPv6 addresses, but the uplink had no IPv6, so its router had no outside IPv6 address. Nothing in the lab uses IPv6, so turn it off in the same command:
```bash
lxc network set ovntest ipv6.address=none bridge.mtu=1500
lxc restart web1
lxc exec web1 -- ip link show eth0       # mtu 1500
```

**5. Retest.**

| `web1` location | Download speed |
|---|---|
| Before the fix | **0** |
| After the fix, on `mc2` (next to the gateway) | **~750 KB/s** |
| After the fix, on `mc3` (through the tunnel) | **~573 KB/s** |

It works both with and without the tunnel. `web1` was moved back to `mc3` for the rest of the test.

#### C3g. Install the web server

```bash
lxc exec web1 -- apt-get update
lxc exec web1 -- apt-get install -y nginx
lxc exec web1 -- sh -c 'echo "Hello from web1 on $(hostname)" > /var/www/html/index.html'
lxc exec web1 -- curl -s http://localhost
```

### Part D: Open it to the outside (`mc1`)

Plain English: `10.10.10.50` only exists inside the cloud. Give `ovntest` a small block of outside addresses it may use, then create a forward: "traffic to this outside address, port 80, goes to `web1`, port 80."

```bash
lxc network set UPLINK ipv4.routes=192.168.82.32/28
lxc network forward create ovntest 192.168.82.33
lxc network forward port add ovntest 192.168.82.33 tcp 80 10.10.10.50 80
curl -s --max-time 3 http://192.168.82.33
```

`192.168.82.32/28` (`.32` to `.47`) sits outside both the OVN router range (`.10`–`.20`) and VMware's DHCP leases (`.128` and up).

**Real result:** `Hello from web1 on web1`, **right away**. OVN answered for the forward address by itself, so no route workaround was needed. The page also loaded in a **browser on the Windows PC**.

> If it hangs on your setup: check `ip neigh show 192.168.82.33`. If it shows `FAILED`, route the address to the OVN router's own uplink address:
> `sudo ip route add 192.168.82.33/32 via $(lxc network get ovntest volatile.network.ipv4.address)`

### Part E: Maintenance day (`mc1`)

**E1. A "visitor" loop, in a second terminal:**
```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s --max-time 1 http://192.168.82.33 || echo "DOWN"
  sleep 1
done
```

**E2. Evacuate `mc3`:**
```bash
lxc cluster evacuate mc3 --force
lxc list web1
```

**Real result:** `web1` moved to **`mc1`**. The loop showed:
```
12:13:13 Hello from web1 on web1
12:13:14 DOWN
...
12:13:30 DOWN
12:13:31 Hello from web1 on web1
```
**Down for about 17 seconds.**

**E3. Restore `mc3`:**
```bash
lxc cluster restore mc3 --force
lxc list web1
```

**Real result:** `web1` moved back to **`mc3`**:
```
12:14:01 Hello from web1 on web1
12:14:02 DOWN
...
12:14:18 DOWN
12:14:20 Hello from web1 on web1
```
**Down for about 18 seconds.**

Notes:
- The loop printed every ~2 seconds while the site was down, because each failed `curl` waits its full 1-second timeout before the 1-second sleep.
- `lxc list` showed a blank IPv4 right after the move. That's only timing: the listing ran before the address appeared. The website answering proves `10.10.10.50` was back.

---

## Write-up

**How the four weeks come together:**
- **Weeks 1–2, the cluster:** three machines agree on one shared state, and keep working with one of them down.
- **Week 3, shared storage (Ceph):** a container's data is already on every machine, so moving it copies nothing.
- **Week 4, virtual networks (OVN):** a container's network belongs to the network itself, not to the machine it runs on, so its address survives a move. The front-door forward gives outside visitors one fixed address.

Today all three worked at once. `web1` left its machine, ran on another one, and came back, with its data and its address intact, while a visitor kept using the same web address.

**Downtime:** about **17–18 seconds** per move. That's the cost of a cold move: stop, move, start. INFRA-013 showed true live migration isn't available for containers. It's also far less than INFRA-016's 43 seconds, which supports the idea that the typing gaps between hand-entered commands inflated that number.

**The download problem, in one line:** a tunnel makes the network inside it smaller than normal, so the network underneath has to be bigger to compensate. We raised the underlay to 1600, which let the overlay use a normal 1500.

**Lab-vs-reality gaps:**
- **Zero downtime needs two copies.** Production would run two or more copies of the website behind a load balancer, on different machines. Evacuating one machine would then cause no visible outage.
- **Network card type:** VMware's `e1000` is an older emulated card, and it hung once during a live change. VMware's `vmxnet3` type is designed for virtual machines and is generally more robust. It's a worthwhile hardening step, but untested here.
- **MTU in production:** real clouds usually set the network under the tunnel to 1600 or larger (often 9000, called "jumbo frames") from day one, specifically to avoid this problem.

---

## Definition of Done

- [x] Health check run for all three layers
- [x] Leftovers removed (output not captured)
- [x] `web1` serving a page at a pinned internal address (`10.10.10.50`)
- [x] Network forward working from `mc1` and from a Windows browser; OVN answered ARP directly
- [x] Download failure diagnosed with evidence (three suspects ruled out, packet capture) and fixed (underlay MTU 1600, overlay 1500)
- [x] `mc3` network hang recovered, and the netplan MAC ambiguity fixed
- [x] Evacuation and restore done, with ~17 s and ~18 s downtime measured
- [x] Write-up completed

**Open follow-ups (optional):**
- Capture `web1`'s outgoing connection request to confirm which size it announced, to prove or rule out VMware's NAT as the part ignoring it.
- Consider switching the nodes' network cards from `e1000` to `vmxnet3`.
- Run a second copy of `web1` on another machine behind a load balancer, and repeat the evacuation test aiming for zero downtime.

---

## Retention Check

When `mc3` was evacuated, `web1` came back on another machine with the same address and the same data, without copying anything. Which two parts of the private cloud made that possible, and what would you add to make the downtime zero?