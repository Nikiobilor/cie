# INFRA-019: What Container Orchestration Solves, and Why Kubernetes

**Priority:** P2 (Week 5, Day 21: Kubernetes Foundations)
**Component:** Concepts: container orchestration, desired state, control loops, health checks
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), `web1` on `ovntest` (from INFRA-018), front door at `192.168.82.33`
**Linked tickets:** INFRA-018 (capstone). Sets up INFRA-020 (Day 22: install MicroK8s, deploy first pod)

> **Note:** this ticket was rewritten after the session to record the real results.

---

## Summary

In INFRA-018, **you** were the orchestrator. You chose where `web1` ran, pinned its address, built its front door, and decided when to move it.

Today we felt the problem Kubernetes solves, before installing it. We crashed the website and saw that nothing brought it back except a person. Then we wrote a ten-line loop that brought it back automatically. That loop is the core idea behind Kubernetes.

**Results:**

| Crash test | Down for | Who brought it back |
|---|---|---|
| No watcher | **~31 seconds** | A person (and only because one was watching) |
| With a 5-second watcher loop | **~14 seconds** | The loop |

**Also found:** the watcher checked whether the **container** was running, not whether the **website** answered. A web server can die inside a running container, and this watcher would never notice. Kubernetes solves that with health checks, which it calls **probes**.

---

## Acceptance Criteria

- [x] `web1` crashed on purpose, with what the visitor saw recorded
- [x] The manual chores from INFRA-018 listed and counted
- [x] A small "watcher" loop written that restarts `web1` automatically
- [x] The crash repeated with the watcher running, and downtime compared
- [x] A plain-English table mapping each manual chore to the Kubernetes feature that automates it
- [x] A short note on when Kubernetes is *not* worth it
- [x] **Added:** the gap between "container running" and "service working", and how probes close it

---

## Comments Thread

**Reviewer:** Your website survived maintenance in INFRA-018. What happens if it just crashes at 3 a.m.?

**NwaChi:** It stays down until someone notices and starts it again. And that someone is me.

**Reviewer:** Now imagine fifty services, three copies each, ten machines, and weekly updates. That's the job orchestration takes over. You stop giving **instructions** ("start this", "move that") and declare a **desired state** ("I want 3 copies of this running"). A program then loops forever: look at what's actually running, compare it with what you asked for, fix any difference. That's a **control loop**.

**NwaChi:** So a control loop is code that checks whether a service is on, starts it if it's off, and does nothing if it's on. Today the watcher checked whether the website was running, said when it was down, and started it back up.

**Reviewer:** That's the core of it. Two sharpenings. First, it's always *look, compare, fix*, and "on/off" is just the simplest case. In Kubernetes the desired state might be "3 copies"; if one dies, the loop sees 2 and starts another. Second, your watcher asked LXD "is the container running?", not "does the website answer?". Those aren't the same.

**NwaChi:** What would make the web server crash while the container keeps running?

**Reviewer:** Plenty: a bug, running out of memory (Linux kills the biggest process, the container survives), a bad config change, a full disk, someone stopping the service. Or it's frozen: still running, never answering. That's why Kubernetes checks the actual service, using probes.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Step 1: Check the starting point (`mc1`)

```bash
lxc list web1
curl -s --max-time 3 http://192.168.82.33
```

Plain English: `curl` fetches a web page from the command line. `-s` hides its progress meter, so only the page shows. `--max-time 3` gives up after 3 seconds instead of hanging. The address is the front door from INFRA-018. A reply of `Hello from web1 on web1` proves the whole path works.

### Step 2: Crash the website, and watch nothing happen

**Second terminal**, the visitor loop:
```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s --max-time 1 http://192.168.82.33 || echo "DOWN"
  sleep 1
done
```

**First terminal**, simulate a crash, then bring it back by hand after a while:
```bash
lxc stop web1 --force     # --force = stop immediately, like a crash
lxc start web1
```

**Real result:**
```
09:36:32 Hello from web1 on web1
09:36:33 DOWN
...
09:37:01 DOWN
09:37:03 Hello from web1 on web1
```
**Down for about 31 seconds.** That number was decided by how quickly a person reacted. Nothing would have brought it back on its own.

### Step 3: Count the chores

Getting **one** copy of the website running and reachable in INFRA-018 took roughly **10 commands**: launch, stop, pin the address, start, update packages, install the web server, write the page, test it locally, create the front door, add its port. That doesn't count the one-time uplink setting. Maintenance then added evacuate, check, and restore.

Multiply that by every copy and every service, and add "remember to do it at 3 a.m.", and you have the job orchestration exists to take over.

### Step 4: Build a tiny control loop

Plain English: every 5 seconds, **look** at `web1`'s state, **compare** it with the desired state (RUNNING), and **fix** any difference.

**Third terminal:**
```bash
while true; do
  state=$(lxc list web1 -c s --format csv)
  if [ "$state" != "RUNNING" ]; then
    echo "$(date +%T) web1 is $state, wanted RUNNING: starting it"
    lxc start web1
  fi
  sleep 5
done
```

### Step 5: Crash it again, with the watcher running

```bash
lxc stop web1 --force
```

**Real result:**
```
09:39:02 Hello from web1 on web1
09:39:03 DOWN
...
09:39:15 DOWN
09:39:16 Hello from web1 on web1
```
**Down for about 14 seconds, with no human involved.** That's roughly half the Step 2 time.

Where those 14 seconds went:
- **Up to 5 s** until the watcher's next check noticed.
- **About 3 s** for the container to start (measured in INFRA-017).
- **The rest** for nginx to start and the first visitor request to land.

Stop all loops with `Ctrl+C`.

### Step 5b (optional): The gap the watcher can't see

Stop only the web server, and leave the container running:
```bash
lxc exec web1 -- systemctl stop nginx
lxc list web1                                                  # still RUNNING
curl -s --max-time 3 http://192.168.82.33 || echo "DOWN"       # DOWN
lxc exec web1 -- systemctl start nginx
```

The container says RUNNING, but visitors get DOWN, and the Step 4 watcher would report nothing wrong. (Not run in the original session; included to show the gap.)

---

## Write-up

**What orchestration solves:** keeping services running and reachable without a person doing the placing, restarting, scaling and moving. The core mechanism is a **control loop**: declare the desired state, and let software repeatedly look, compare and fix.

**Manual chores mapped to Kubernetes:**

| Manual chore (INFRA-018 / today) | What Kubernetes calls it |
|---|---|
| Choosing which machine runs it | Scheduling |
| Noticing a crash and restarting it | Self-healing (a controller keeping the desired number of copies) |
| Running several copies | Replicas |
| One fixed address in front of the copies | A Service |
| Spreading visitors across copies | Service load balancing |
| Updating copies one at a time, without downtime | A rolling update (a Deployment) |
| Moving work off a machine before maintenance | Draining a node |
| Checking the service itself, not just the container | Probes (health checks) |

**Probes: Kubernetes' health checks.**

| Probe | The question it asks | What happens if the answer is "no" |
|---|---|---|
| Liveness | Is it still alive, or stuck? | Kubernetes restarts it |
| Readiness | Is it ready to serve visitors right now? | It stops getting visitors until it's ready, without a restart |
| Startup | Has it finished starting? | The other probes wait until it has |

A probe can request a web page (like `curl`), open a connection to a port, or run a command inside the container. On AWS, an Application Load Balancer's target group health checks play the role of a readiness probe.

**Why Kubernetes specifically:** it's the standard way to run containers across many machines. The same concepts work on AWS (EKS), Google Cloud (GKE), Azure (AKS), and on your own hardware (MicroK8s, from Day 22).

**When it's not worth it:** Kubernetes brings its own components, configuration and learning curve. For a handful of simple services, a simpler setup is often cheaper and faster to run. It earns its cost when there are many services, many copies, frequent changes, and a need to survive failures without a human.

**Lab-vs-reality gap:** today's watcher is a toy. It handles one container, checks the wrong thing (the container, not the service), restarts instead of failing over, and would stop working if its own machine died. Kubernetes runs many loops, keeps desired state in a replicated database, checks services with probes, keeps spare copies already running, and runs its own control loops in a highly available way (Week 6).

---

## Definition of Done

- [x] Crash with no watcher: ~31 s, ended by a person
- [x] Chores counted: ~10 commands per copy, plus maintenance steps
- [x] Watcher loop built and working
- [x] Crash with the watcher: ~14 s, no person involved
- [x] Chore-to-Kubernetes table filled in
- [x] Probes explained, including the container-vs-service gap
- [x] "When it's not worth it" note written

---

## Retention Check

In your own words: what is a control loop, and which of today's manual chores did your ten-line watcher take over?