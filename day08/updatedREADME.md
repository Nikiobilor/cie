# INFRA-008: Node Failure Detection and Recovery — LXD Cluster

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-008 |
| **Priority** | High |
| **Component** | Clustering / High Availability |
| **Environment** | VMware Workstation Pro (Windows host) — 3-node LXD cluster (`node1`, `node2`, `node3`) |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 2 — Clustering Foundations (Day 8 of 50) |
| **Linked tickets** | INFRA-007 (cluster join + quorum test) — prerequisite. Day 9 (MicroCloud overview) — this ticket's findings feed it directly. |
| **Status** | Detection + recovery complete. Final write-based quorum re-verification (Phase 6) not yet confirmed run — see Definition of Done. |

---

## Summary

INFRA-007 proved that the 3-node cluster tolerates exactly 1 node failure — quorum survives with 2 of 3 up, and is lost with only 1 up. That test was done manually: nodes were powered off by hand and `lxc cluster list` checked by hand.

This ticket set out to build two things instead:
1. A small monitoring script that polls cluster member status and flags anything that isn't healthy.
2. A recovery procedure for bringing a downed node back into the cluster and confirming the cluster is whole again.

What actually happened deviated from the plan almost immediately — in a useful way. Before the detection script was even written, running the Phase 0 pre-flight check on a cluster that had genuinely lost quorum (all three nodes had been powered off for several days) produced a real incident: the command hung instead of erroring. Diagnosing that hang, live, turned into the most valuable part of the day.

---

## What Actually Happened (incident log)

**Starting state:** all three VMs had been powered off for roughly 3 days (away from the machine). Only `node1` was powered back on to begin the day's work.

**1. Pre-flight check hung.**
Ran `lxc cluster list` from `node1` with only 1 of 3 members up. The command did not return an error — it just sat there with no output.

**2. Diagnosis.**
With only 1 of 3 nodes up, there is no majority to elect a dqlite (Raft-based) leader. A *write* fails fast, because there's clearly no leader to accept it — that's what INFRA-007 already demonstrated. A *read* like `lxc cluster list` behaves differently: it still has to route through the cluster leader for a consistent answer, so with no leader available it doesn't fail immediately — it retries the connection attempt repeatedly. Left running long enough, LXD does eventually surface an explicit error (`failed to create dqlite connection: no available dqlite leader server found`), confirmed against Canonical's own cluster-recovery documentation and LXD's GitHub issue tracker — but on a short timeframe it looks and behaves like an indefinite hang.
**Correction to the original mental model:** the read isn't waiting on something that can genuinely never arrive — it's retrying against a real (if generous) retry budget before failing loud. Worth remembering for any Raft/dqlite-backed system, not just LXD.

**3. Powered `node2` back on** to restore majority (2 of 3). `lxc cluster list` returned immediately once quorum was available again — no more hanging.

**4. Unexpected findings on `node2`'s reconnect:**
- **Leadership had moved.** `node2` came back holding the `database-leader` role; `node1` held only `database` (voter). Raft/dqlite elects leadership dynamically — nothing in this cluster assumes `node1` stays leader permanently, and this was a live example of that.
- **`node2`'s host-only IP had drifted.** `ip a` on `node2` showed `192.168.20.129` (marked `dynamic`) on interface `ens33` — not the documented static `192.168.20.130` on `ens37` from the INFRA-006/007 netplan fix. Root cause: that fix bound the static config to an *interface name*, and VMware's NIC enumeration order isn't guaranteed stable across a full power-off/power-on cycle (as opposed to a simple reboot). This time the host-only adapter enumerated as `ens33` instead of `ens37`, the static config never matched, and DHCP filled the gap. **This is flagged as a follow-up, not fixed in this ticket** — see Follow-ups below.

**5. Phase 2 — inspected real JSON before writing any parsing logic**, per the reviewer note in the Comments thread. Actual captured output (fields matched what the script was written against, with one extra field worth using):

```json
{
  "server_name": "node3",
  "url": "https://192.168.20.131:8443",
  "status": "Offline",
  "message": "No heartbeat for 86h54m17.632414265s (2026-08-10 21:18:34.642141311 +0000 UTC)",
  "roles": ["database"]
}
```

The `message` field's heartbeat duration was real, not staged — `node3` had genuinely been offline for the full 3 days away, which is why it made a better test case than a contrived 15-second outage.

**6. Detection script written and run** against that real state — see Implementation below for the final version and captured output.

**7. Recovery:** `node3` powered on, `sudo lxd waitready` confirmed the daemon before assuming boot meant ready, `lxc cluster list` confirmed all three `ONLINE`, and the running detection script itself flipped `node3` from `ALERT` to `OK` without manual intervention — the actual verification moment, not just something described on paper.

**8. Monitoring script stopped** with `Ctrl+C` once recovery was confirmed (foreground process, no backgrounding used this run).

---

## Acceptance Criteria

- [x] Can explain, in plain English, what "cluster member state" means in LXD and how a node goes from Online → Offline
- [x] Detection script correctly identifies an offline node using the cluster's real state (used the genuine 3-day `node3` outage, not a staged one)
- [x] Detection script logs alerts with a timestamp
- [x] Recovery executed: `node2` and `node3` powered back on and confirmed rejoined
- [x] `lxc cluster list` shows all three nodes Online after recovery
- [ ] Quorum re-verified with a real write after recovery (Phase 6 — not yet confirmed run)
- [x] Lab-vs-reality gap documented — expanded beyond the planned scope to also cover the dqlite hang-vs-error correction found mid-lab
- [ ] Leftover INFRA-007 housekeeping (test cluster groups, old VirtualBox VMs) — still open, tracked as follow-up
- [x] New finding from this ticket (netplan static IP not surviving a full power cycle) — logged as follow-up, not silently dropped

---

## Comments (conceptual explanation thread)

**NwaChi:**
Opening this because INFRA-007 showed *that* the cluster can lose a node, but everything after "the write failed" was me watching it happen. I want to actually catch a failure automatically and have a real procedure for fixing it.

**Senior Infra Engineer (review):**
Good instinct — and worth being precise about what "automated recovery" can honestly mean here before you build anything. In a cloud environment, "recovery" can mean the platform notices a node is gone and replaces it without a human touching anything. That's not available on a laptop lab — nothing except you can power `node2` back on. So split it honestly: **detection** is genuinely automatable today; **recovery** is semi-automated — a human performs the physical action, a script verifies the outcome instead of you eyeballing it.

One more thing: don't hardcode assumptions about LXD's JSON output. Pull the raw JSON first, look at it, write parsing logic against what you actually see.

**NwaChi (mid-lab update):**
Turned out to matter faster than expected — the very first command I ran today hung instead of returning anything, before I'd even gotten to the detection script.

**Senior Infra Engineer:**
That's not a detour from the ticket, that's the ticket happening a step early. A hang with no error is arguably a more realistic incident than a clean failure — production doesn't always tell you what's wrong. Good that you diagnosed it instead of just killing it and moving on. That instinct (confirm the mechanism before treating something as "just broken") is the same instinct that separates debugging from guessing, and it's exactly the kind of judgment that doesn't show up in a resume bullet point but does show up in an interview conversation.

---

## Implementation

### Pre-flight incident (unplanned, see incident log above)
`lxc cluster list` hung with 1 of 3 nodes up. Resolved by restoring majority (powered `node2` back on). No code changed for this — it was a diagnosis, not a fix.

### Detection script (final version, as actually run)

```bash
#!/bin/bash
# cluster-watch.sh
# Polls LXD cluster member status on a fixed interval and logs an
# alert line the moment a member isn't Online. Detects only — does
# not attempt to fix anything.

INTERVAL=15
LOGFILE="$HOME/lxd-cluster-watch.log"

echo "Starting LXD cluster watch (every ${INTERVAL}s). Logging to $LOGFILE"
echo "Press Ctrl+C to stop."

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    STATUS_JSON=$(lxc cluster list --format json)

    echo "$STATUS_JSON" | jq -r '.[] | "\(.server_name)|\(.status)|\(.message)"' | while IFS='|' read -r NAME STATE MSG; do
        if [ "$STATE" != "Online" ]; then
            echo "[$TIMESTAMP] ALERT: $NAME is $STATE — $MSG" | tee -a "$LOGFILE"
        else
            echo "[$TIMESTAMP] OK: $NAME is Online" >> "$LOGFILE"
        fi
    done

    sleep "$INTERVAL"
done
```

**Actual captured output** (against the real `node3` outage, before recovery):

```
[2026-08-14 14:13:19] ALERT: node3 is Offline — No heartbeat for 88h54m44.89066341s (2026-08-10 21:18:34.642141311 +0000 UTC)
[2026-08-14 14:13:34] ALERT: node3 is Offline — No heartbeat for 88h55m0.136386308s (2026-08-10 21:18:34.642141311 +0000 UTC)
[2026-08-14 14:13:49] ALERT: node3 is Offline — No heartbeat for 88h55m15.353146929s (2026-08-10 21:18:34.642141311 +0000 UTC)
```

`node1`/`node2` OK lines were written to the log file only (script only prints ALERT lines to the terminal, to avoid spamming healthy-status noise every 15 seconds) — confirmed present via `tail -n 20 ~/lxd-cluster-watch.log`.

### Recovery (as actually executed)

1. Powered on `node3` in VMware Workstation.
2. `sudo lxd waitready` on `node3` — confirmed daemon ready before assuming boot meant ready.
3. `lxc cluster list` from `node1` — all three members `ONLINE`.
4. Watched `cluster-watch.sh`'s own output flip `node3` from `ALERT` to `OK` on its own — the actual automated-verification moment.
5. Stopped the script with `Ctrl+C` once confirmed.

### Phase 6 — quorum re-verification (documented, not yet confirmed executed)

```bash
lxc cluster group create day8-recovery-check
lxc cluster group delete day8-recovery-check
```

`ONLINE` status alone confirms heartbeats are being received again — it doesn't by itself prove the cluster can take a write. This step closes that gap and should be run before marking the ticket fully done.

---

## Lab-vs-reality gap

Detection built here is real, general-purpose automation — the polling/alerting pattern is the same shape used against a production cluster, just pointed at a log file instead of Slack/PagerDuty. Recovery is not fully automated and, on a single laptop hypervisor, honestly can't be without a hypervisor-API integration to auto-power-on VMs (out of scope for a fundamentals week) — that gap is exactly what Day 9's MicroCloud overview picks up.

A second, unplanned lab-vs-reality note surfaced mid-ticket: the assumption that "the read will hang forever with no leader" was slightly wrong. dqlite retries on a real (generous) budget before surfacing an explicit error. The practical lesson carries either way — a monitoring script calling this command with no timeout will stall for a long stretch right alongside the cluster it's supposed to be watching, which is why the detection script here should eventually be wrapped in `timeout` with the timeout itself treated as a signal, not just a parse failure.

---

## Follow-ups (opened during this ticket, not yet actioned)

1. **Netplan static IP isn't durable across a full power cycle.** Bound to interface name (`ens37`), which isn't guaranteed stable across VMware NIC re-enumeration. Needs a MAC-address-based netplan match instead. Should be fixed before Day 9 relies on stable node IPs again.
2. **INFRA-007 housekeeping still open:** delete leftover test cluster groups, formally retire the old VirtualBox `node1`/`node2` VMs.

---

## Definition of Done

- [x] `cluster-watch.sh` written, executable, run against a genuine (not staged) node outage
- [x] node2 and node3 powered back on, `lxd waitready` confirmed on each
- [x] `lxc cluster list` shows all three nodes ONLINE
- [ ] Quorum re-verified via a real cluster group create/delete (Phase 6 — pending confirmation)
- [x] Script's own log shows the automatic OK transition after recovery
- [x] Lab-vs-reality gap documented, including the dqlite hang-vs-error correction
- [ ] Follow-up tickets opened for netplan MAC fix and INFRA-007 housekeeping (logged here, not yet filed as separate tickets)

---

## Retention check

**Question:** If you woke up tomorrow and `node3` had silently dropped out of the cluster three hours ago, how would you know — and what's the actual first command you'd run to confirm it's a heartbeat issue and not a network issue?

**First answer given:** check the log the script writes to; run `bash lxd-cluster-watch.log`.
**Gap:** correctly identified the log as the source of truth, but the command itself was invalid (a log file isn't executable), and the network-vs-heartbeat distinction the question was actually asking for wasn't addressed.

**Second answer given:** check the log; run `ping node3` for network, `lxc cluster` to read `node3`'s state for heartbeat.
**Corrected to:** `ping 192.168.20.131` (no DNS/hostname resolution configured for these nodes — `ping node3` fails on name resolution, not connectivity) and `lxc cluster list` (`lxc cluster` alone only lists subcommands, doesn't show member state).
**Final understanding confirmed correct:** ping failing points to a network-layer problem (adapter, routing, powered-off VM); ping succeeding while `lxc cluster list` still shows the node offline isolates it to the LXD/heartbeat layer specifically — which tells you whether to go check VMware networking or `systemctl status snap.lxd.daemon` on that node, instead of guessing.