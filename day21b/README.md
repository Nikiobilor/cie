# INFRA-019: What Container Orchestration Solves, and Why Kubernetes

**Priority:** P2 (Week 5, Day 21: Kubernetes Foundations)
**Component:** Concepts: container orchestration, desired state, control loops
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), `web1` on `ovntest` (from INFRA-018), front door at `192.168.82.33`
**Linked tickets:** INFRA-018 (capstone). Sets up INFRA-020 (Day 22: install MicroK8s, deploy first pod)

---

## Summary

In INFRA-018, **you** were the orchestrator. You chose which machine `web1` ran on, pinned its address, set up its front door, and decided when to move it. LXD did the moving, but only because you told it to.

Today, before installing Kubernetes, we feel the problem Kubernetes exists to solve. We'll crash the website and see that nobody brings it back. Then we'll write a tiny loop that does bring it back automatically. That loop is the core idea behind Kubernetes, in about ten lines.

No Kubernetes is installed today. That's Day 22.

---

## Acceptance Criteria

- [ ] `web1` crashed on purpose, with what the visitor saw recorded (does anything bring it back?)
- [ ] The manual chores from INFRA-018 listed, with a count of how many steps one more copy would take
- [ ] A small "watcher" loop written that restarts `web1` automatically when it stops
- [ ] The crash repeated with the watcher running, and downtime compared
- [ ] A plain-English table mapping each manual chore to the Kubernetes feature that automates it
- [ ] A short note on when Kubernetes is *not* worth it

---

## Comments Thread

**Reviewer:** Your website survived maintenance in INFRA-018. What happens if it just crashes at 3 a.m.?

**NwaChi:** It stays down until someone notices and starts it again.

**Reviewer:** And who is "someone"?

**NwaChi:** Me.

**Reviewer:** Right. And that's one website. Imagine fifty services, each with three copies, spread across ten machines, with updates going out every week. What are you doing all day?

**NwaChi:** Placing containers, restarting them, updating addresses, moving things around.

**Reviewer:** That's the job orchestration takes over. The key idea is simple: you stop giving **instructions** ("start this", "move that") and start declaring a **desired state** ("I want 3 copies of this website running, reachable at this address"). Then a program runs a loop forever: look at what's actually running, compare it with what you asked for, and fix any difference. That's called a **control loop**.

**NwaChi:** So Kubernetes is basically a very sophisticated version of that loop?

**Reviewer:** Many of them, cooperating. Build a tiny one by hand today and you'll understand Kubernetes far better than by reading about it.

**NwaChi:** Is Kubernetes always the answer, then?

**Reviewer:** No. It solves real problems at scale, and it adds real complexity. Part of today's write-up is when it's worth it.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Step 1: Check the starting point (`mc1`)

```bash
lxc list web1
curl -s --max-time 3 http://192.168.82.33
```

Expected: `web1` RUNNING, and the page answering.

### Step 2: Crash the website, and watch nothing happen

Start the visitor loop in a **second terminal** on `mc1`:
```bash
while true; do
  printf "%s " "$(date +%T)"
  curl -s --max-time 1 http://192.168.82.33 || echo "DOWN"
  sleep 1
done
```

In the **first terminal**, simulate a crash:
```bash
lxc stop web1 --force
```

`--force` stops it immediately, like a crash, rather than a clean shutdown. Wait 60 seconds and watch the loop.

**Record:** does the website ever come back on its own?

Then bring it back by hand:
```bash
lxc start web1
```

**Record:** how long was it down in total? Notice what decided that number: how quickly *you* reacted.

### Step 3: Count the chores

Plain English: list every manual step from INFRA-018 that it took to get **one** copy of the website running and reachable. Count them. Then imagine doing it for a third copy, a tenth, a hundredth.

From INFRA-018, one copy needed: launch the container, choose its machine, pin its address, install the web server, write the page, create the front door, and check it works. And during maintenance: evacuate, check where it went, restore.

**Record:** how many commands per copy? Who would remember to do all of them, every time, at 3 a.m.?

### Step 4: Build a tiny control loop

Plain English: this loop does what you did by hand in Step 2. Every 5 seconds it checks whether `web1` is running, and starts it if it isn't. The desired state is "`web1` is RUNNING".

In a **third terminal** on `mc1` (keep the visitor loop running):
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

Watch all three terminals.

**Record:**
- In the watcher: when did it notice, and what did it do?
- In the visitor loop: how long was the website down this time?
- Compare with Step 2.

Then stop the watcher and the visitor loop with `Ctrl+C`.

### Step 6: Write up

**Map the chores to Kubernetes.** Fill this in, in your own words:

| Manual chore (INFRA-018 / today) | What Kubernetes calls it |
|---|---|
| Choosing which machine runs it | Scheduling |
| Noticing a crash and restarting it | Self-healing (a controller keeping the desired number of copies) |
| Running several copies | Replicas |
| One fixed address in front of the copies | A Service |
| Spreading visitors across copies | Service load balancing |
| Updating copies one at a time, without downtime | A rolling update (a Deployment) |
| Moving work off a machine before maintenance | Draining a node |

**Why Kubernetes specifically:** it has become the standard way to run containers across many machines. The same concepts work on AWS (EKS), Google Cloud (GKE), Azure (AKS), and on your own hardware (MicroK8s, which you'll install on Day 22). Learn it once, and it applies everywhere.

**When it's not worth it:** Kubernetes adds its own machines, components, configuration and learning curve. For a handful of simple services, a single machine or a simpler tool may be faster and cheaper to run. It earns its cost when there are many services, many copies, frequent changes, and a need to survive failures without a human in the loop.

**Lab-vs-reality gap:** today's watcher is a toy. It handles one container, one kind of failure, and it would break if the machine it runs on died. Kubernetes runs many such loops, keeps its desired state in a replicated database, and runs the loops themselves in a highly available way. That's Week 6.

---

## Definition of Done

- [ ] Crash with no watcher: result recorded
- [ ] Chores counted
- [ ] Watcher loop built and working
- [ ] Crash with the watcher: result recorded and compared
- [ ] Chore-to-Kubernetes table filled in
- [ ] "When it's not worth it" note written

---

## Retention Check

In your own words: what is a control loop, and which of today's manual chores did your ten-line watcher take over?