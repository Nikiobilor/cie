# INFRA-019: Two Copies Behind a Load Balancer: Does Maintenance Still Cause Downtime?

**Priority:** P2 (Week 5, Day 21)
**Component:** High availability (LXD network load balancer on OVN)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), LXD 6.0.6, Ceph pool `remote`, OVN network `ovntest` (MTU 1500 on a 1600 underlay), `UPLINK` with `ipv4.routes=192.168.82.32/28`
**Linked tickets:** INFRA-018 (one copy: ~17 s of downtime per move). Sets up INFRA-020

---

## Summary

INFRA-018 proved that one copy of a website always goes dark for a while during maintenance: about 17 seconds per move. The usual fix is **two copies on different machines behind a load balancer**, so that while one is away, the other keeps answering.

Today we build exactly that, and test it honestly instead of assuming it works. We check two things:
1. **A planned move** (evacuating a machine): does the visitor notice anything?
2. **A sudden failure** (one copy just stops): does the load balancer notice, and stop sending visitors to it?

---

## Acceptance Criteria

- [ ] A second website copy (`web2`) running on a different machine from `web1`, with its own fixed address
- [ ] A load balancer on `ovntest` sending visitors to both copies, reachable at one outside address
- [ ] Visitor loop showing both copies answering
- [ ] Evacuation of `web1`'s machine done, with what the visitor saw recorded
- [ ] `web1` stopped outright, with what the visitor saw recorded
- [ ] Checked whether this LXD version supports health checks, with the real answer recorded
- [ ] Plain-English write-up: what two copies fixed, and what they didn't

---

## Comments Thread

**Reviewer:** Last time, one copy meant 17 seconds dark. What's the fix?

**NwaChi:** Two copies on different machines, and a load balancer in front. If one goes away, the other answers.

**Reviewer:** How does the load balancer know one went away?

**NwaChi:** It... checks?

**Reviewer:** Only if it's been told to. Checking is called a **health check**: the load balancer regularly asks each copy "are you alive?" and stops sending visitors to one that doesn't answer. Without health checks, a load balancer is just a traffic splitter. It keeps sending visitors to a dead copy.

**NwaChi:** Does LXD's load balancer do health checks?

**Reviewer:** Find out. Don't assume. LXD's newer versions have health-checked backend pools, but you're on the 6.0 long-term support release, and features differ between versions. Check what your version actually supports, then test what happens with and without.

**NwaChi:** So two copies might *not* mean zero downtime?

**Reviewer:** That's the question for today. Measure it.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Step 1: Check what's running (`mc1`)

```bash
lxc list
lxc network forward list ovntest
curl -s --max-time 3 http://192.168.82.33
```

Expected: `web1` running at `10.10.10.50`, the INFRA-018 front door on `192.168.82.33`, and the page answering.

### Step 2: Launch a second copy on a different machine

Plain English: the whole point is that the two copies don't share a machine. If they did, maintenance on that one machine would take both down.

```bash
lxc launch ubuntu:24.04 web2 --network ovntest --storage remote --target mc1
lxc stop web2
lxc config device set web2 eth0 ipv4.address=10.10.10.51
lxc start web2
lxc exec web2 -- apt-get update
lxc exec web2 -- apt-get install -y nginx
lxc exec web2 -- sh -c 'echo "Hello from web2 on $(hostname)" > /var/www/html/index.html'
lxc exec web2 -- curl -s http://localhost
lxc list web -c n4L
```

Expected: `web1` on `mc3` at `.50`, and `web2` on `mc1` at `.51`.

### Step 3: Check whether health checks exist in this LXD version

```bash
lxc version
lxc network load-balancer pool --help
```

Plain English: newer LXD versions group backends into **pools** that can have health checks. If the `pool` command doesn't exist here, this version's load balancer can't health-check.

**Record:** does `pool` exist?

### Step 4: Create the load balancer (`mc1`)

Plain English: a load balancer is a new front door. Visitors come to one outside address, and it spreads them across the copies. It needs its own address, because `192.168.82.33` is already used by the INFRA-018 forward.

```bash
lxc network load-balancer create ovntest 192.168.82.34
lxc network load-balancer backend add ovntest 192.168.82.34 web1 10.10.10.50 80
lxc network load-balancer backend add ovntest 192.168.82.34 web2 10.10.10.51 80
lxc network load-balancer port add ovntest 192.168.82.34 tcp 80 web1,web2
lxc network load-balancer show ovntest 192.168.82.34
```

Test it a few times:
```bash
for i in 1 2 3 4 5 6; do curl -s --max-time 2 http://192.168.82.34; done
```

Expected: a mix of `Hello from web1` and `Hello from web2`. It may not alternate neatly, because each new connection is assigned on its own.

### Step 5: The visitor loop (second terminal, `mc1`)

```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s --max-time 1 http://192.168.82.34 || echo "DOWN"
  sleep 1
done
```

### Step 6: Test 1: a planned move

```bash
lxc cluster evacuate mc3 --force
```

Watch the loop while `web1` moves. **Record:** did the visitor see any `DOWN` lines? How many, and for how long? Then:
```bash
lxc cluster restore mc3 --force
```

### Step 7: Test 2: a sudden failure

Plain English: this simulates `web1` crashing, not being moved. It stays down until we bring it back.

```bash
lxc stop web1 --force
```

Watch the loop for about 30 seconds. **Record:** does the visitor keep seeing `DOWN` lines for as long as `web1` is stopped? Roughly what share of requests fail?

Then bring it back:
```bash
lxc start web1
```

Stop the loop with `Ctrl+C`.

### Step 8: Write up

From the real results:
- What did two copies fix, compared with INFRA-018?
- What happened when a copy disappeared without warning, and why?
- What would be needed to get close to zero downtime? For example, health checks, which this LXD version may or may not support.

**Lab-vs-reality gap:** production load balancers (for example AWS's Application Load Balancer) health-check targets by default and stop routing to unhealthy ones. A load balancer without health checks only spreads traffic. It doesn't protect it.

---

## Definition of Done

- [ ] `web2` running on a different machine, at `10.10.10.51`
- [ ] Load balancer answering at `192.168.82.34` from both copies
- [ ] Health check support checked and recorded
- [ ] Planned-move result recorded
- [ ] Sudden-failure result recorded
- [ ] Write-up completed

---

## Retention Check

You have two copies behind a load balancer, and one copy crashes. What does the load balancer need in order to stop sending visitors to the dead copy, and what happens to visitors if it doesn't have it?