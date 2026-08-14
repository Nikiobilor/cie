# INFRA-008: Automated Node Failure Detection and Recovery Runbook — LXD Cluster

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-008 |
| **Priority** | High |
| **Component** | Clustering / High Availability |
| **Environment** | VMware Workstation Pro (Windows host) — 3-node LXD cluster (`node1`, `node2`, `node3`) |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 2 — Clustering Foundations (Day 8 of 50) |
| **Linked tickets** | INFRA-007 (cluster join + quorum test) — prerequisite. Day 9 (MicroCloud overview) — this ticket's findings feed it directly. |

---

## Summary

INFRA-007 proved that the 3-node cluster tolerates exactly 1 node failure — quorum survives with 2 of 3 up, and is lost with only 1 up. That test was done manually: nodes were powered off by hand and `lxc cluster list` checked by hand.

That doesn't scale. In a real environment, nobody is sitting at a terminal running `lxc cluster list` in a loop, waiting to notice a node dropped. You need something watching the cluster for you, and a documented, repeatable way to bring a failed node back once you're alerted.

This ticket builds both:
1. A small monitoring script that polls cluster member status and flags anything that isn't healthy.
2. A recovery runbook for bringing a downed node back into the cluster and confirming the cluster is whole again.

Convenient timing: `node2` and `node3` are already powered off (left that way at the close of INFRA-007). Instead of artificially simulating a failure, this ticket starts by pointing the detection script at a cluster that's *actually* in a degraded state right now.

---

## Acceptance Criteria

- [ ] Can explain, in plain English, what "cluster member state" means in LXD and how a node goes from Online → Offline
- [ ] Detection script correctly identifies `node2` and `node3` as not-Online using the cluster's real current state
- [ ] Detection script logs alerts with a timestamp, not just prints to screen
- [ ] Recovery runbook executed: `node2` and `node3` powered back on and confirmed rejoined
- [ ] `lxc cluster list` shows all three nodes Online after recovery
- [ ] Quorum re-verified with a real write after recovery (not just assumed)
- [ ] Lab-vs-reality gap for "recovery automation" explicitly documented (see Comments)
- [ ] Leftover INFRA-007 housekeeping (test cluster groups, old VirtualBox VMs) logged as a separate follow-up, not silently forgotten

---

## Comments (conceptual explanation thread)

**NwaChi:**
Opening this because INFRA-007 showed *that* the cluster can lose a node, but everything after "the write failed" was me watching it happen. I want to actually catch a failure automatically and have a real procedure for fixing it, not just knowledge that it's fixable.

**Senior Infra Engineer (review):**
Good instinct, and worth being precise about what "automated recovery" can honestly mean here before you build anything.

In a cloud environment — AWS Auto Scaling, a Kubernetes node pool, MicroCloud itself, "recovery" can mean the platform notices a node is gone and provisions a replacement without a human touching anything. That's not available to you on a laptop lab. `node2` is a VM file sitting on your Windows host, nothing except you (or a script calling into VMware's own CLI, which is a heavier lift than this ticket needs) can power it back on. So be honest in the README about the split:

- **Detection** — genuinely automatable today. A script can poll and alert without you.
- **Recovery** — semi-automated. A human (you) performs the physical action (power on), and a script verifies the outcome (confirms the node rejoined, confirms quorum is back) instead of you eyeballing it.

That distinction is exactly the kind of thing that separates someone who copies a "how to build a monitoring script" tutorial from someone who understands what they built and its limits. If you're ever asked in an interview "does this scale to production," you want to be the one who volunteers the gap, not the one who gets caught not knowing it.

**NwaChi:**
Makes sense. So detection is the "real" automation win here, recovery is a tightened-up manual process with automated verification at the end.

**Senior Infra Engineer:**
Exactly. One more thing — don't hardcode assumptions about what LXD's JSON output looks like. Field names and exact status strings can shift slightly between LXD versions. Pull the raw JSON first, actually look at it, then write your parsing logic against what you see, not against what a tutorial says you'll see. That habit will save you far more time than it costs.

---

## Implementation Runbook

### Phase 0 — Pre-flight check

Confirm where things stand before touching anything.

```bash
# Run this from node1
lxc cluster list
```

You should see `node1` as `ONLINE`, and `node2`/`node3` showing something other than `ONLINE` (likely `OFFLINE`), because they're still powered off from INFRA-007. That's expected. That's your live test case.

---

### Phase 1 — Understand cluster member state (concept, no commands yet)

Plain English first, before any more typing:

Every LXD cluster member sends a periodic "heartbeat" to the cluster database, roughly every few seconds, to say "I'm still here." If the cluster hasn't heard from a member within a configurable window (`cluster.offline_threshold`, default around 20 seconds), that member's state flips from `ONLINE` to `OFFLINE`. The cluster doesn't guess or ping the node directly — it's purely "have I heard from you recently or not."

This matters because it tells you *what* your detection script actually needs to watch: not "is the VM powered on" (LXD has no way to know that), but "has this member's state stopped being ONLINE."

---

### Phase 2 — Inspect the real JSON output before writing anything

Don't assume the field names. Look.

```bash
lxc cluster list --format json | jq .
```

If `jq` isn't installed:

```bash
sudo apt install jq -y
```

Look specifically for two things in the output: the field holding each node's name, and the field holding its status value (and what the actual string looks like — `"Online"`, `"ONLINE"`, `"Offline"`, etc. — casing varies by version). Write those two field names down. You'll use your *actual* observed field names in Phase 3, not the ones below verbatim if they differ.

---

### Phase 3 — Build the detection script

Create the script on `node1` (the node most likely to stay up, since it's your database-leader):

```bash
nano cluster-watch.sh
```

```bash
#!/bin/bash
# cluster-watch.sh
# Polls LXD cluster member status on a fixed interval and logs an
# alert line the moment a member isn't Online. This script only
# detects and logs — it does not attempt to fix anything.

INTERVAL=15
LOGFILE="$HOME/lxd-cluster-watch.log"

echo "Starting LXD cluster watch (every ${INTERVAL}s). Logging to $LOGFILE"
echo "Press Ctrl+C to stop."

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    STATUS_JSON=$(lxc cluster list --format json)

    # Adjust the jq field names below to match what you found in Phase 2
    # if your LXD version reports them differently.
    echo "$STATUS_JSON" | jq -r '.[] | "\(.server_name)|\(.status)"' | while IFS='|' read -r NAME STATE; do
        if [ "$STATE" != "Online" ]; then
            echo "[$TIMESTAMP] ALERT: $NAME is $STATE" | tee -a "$LOGFILE"
        else
            echo "[$TIMESTAMP] OK: $NAME is Online" >> "$LOGFILE"
        fi
    done

    sleep "$INTERVAL"
done
```

Save, then make it executable:

```bash
chmod +x cluster-watch.sh
```

---

### Phase 4 — Run it against the real degraded state

```bash
./cluster-watch.sh
```

You should immediately see `node1` reported OK and `node2`/`node3` reported as ALERT lines, refreshing every 15 seconds. Let it run for a minute or two so you can see it's genuinely polling, not just printing once. Leave it running in this terminal, or background it with `&` and open a second terminal for Phase 5.

Check the log file directly too, so you're not just trusting the live terminal output:

```bash
tail -n 20 ~/lxd-cluster-watch.log
```

---

### Phase 5 — Recovery runbook (the human + verification part)

**Step 1 — Power the nodes back on**
In VMware Workstation, power on the `node2` and `node3` VMs. Give them a minute or two to fully boot.

**Step 2 — Confirm LXD is actually ready on each node**
SSH into `node2` (192.168.20.130) and `node3` (192.168.20.131) individually:

```bash
sudo lxd waitready
```

This command blocks until the LXD daemon on that node is fully up — don't skip it and assume boot = ready.

**Step 3 — Verify from node1 that they rejoined**

```bash
lxc cluster list
```

Because these nodes were never formally removed from the cluster (just powered off), they should reconnect automatically once the daemon is back up and reachable on the host-only network — no manual `lxd cluster join` needed. If either one is stuck in `OFFLINE` after a few minutes, check basic reachability first before assuming a cluster-level problem:

```bash
ping 192.168.20.130
ping 192.168.20.131
```

**Step 4 — Watch your own script confirm it**
Go back to the terminal running `cluster-watch.sh` and watch the ALERT lines for `node2`/`node3` turn into OK lines on their own, without you touching the script. That's your automated verification working — you didn't have to manually re-run `lxc cluster list` and eyeball it.

---

### Phase 6 — Re-confirm quorum with a real write

State showing `ONLINE` isn't proof the cluster can actually take writes — confirm it the same way INFRA-007 did:

```bash
lxc cluster group create day8-recovery-check
lxc cluster group delete day8-recovery-check
```

If both commands succeed, quorum and write capability are genuinely restored, not just assumed from status output.

---

## Lab-vs-reality gap

Detection here is real, general-purpose automation — the polling/alerting pattern is the same shape you'd use against a production cluster, just pointed at Slack/PagerDuty instead of a log file. Recovery is not fully automated and, on a single laptop hypervisor, honestly can't be without significantly more infrastructure (a hypervisor API integration to auto-power-on VMs, which is out of scope for a fundamentals week). In a real cloud environment, an orchestrator (Auto Scaling, Kubernetes, MicroCloud) closes that last gap for you — which is exactly the "why" Day 9's MicroCloud overview picks up from here.

---

## Definition of Done

- [ ] `cluster-watch.sh` written, executable, and committed to the repo
- [ ] Script demonstrated catching the real node2/node3 offline state (log excerpt saved)
- [ ] node2 and node3 powered back on, `lxd waitready` confirmed on each
- [ ] `lxc cluster list` shows all three nodes ONLINE
- [ ] Quorum re-verified via a real cluster group create/delete
- [ ] Script's own log shows the automatic OK transition after recovery
- [ ] Lab-vs-reality gap section written
- [ ] INFRA-007 housekeeping (leftover test cluster groups, retiring old VirtualBox VMs) opened as a separate tracked follow-up, not folded silently into this ticket

---

## Retention check

If you woke up tomorrow and `node3` had silently dropped out of the cluster three hours ago, how would you know — and what's the actual first command you'd run to confirm it's a heartbeat issue and not a network issue?