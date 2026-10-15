# INFRA-026: Incident Drill: Broken Front Door (OVN Network Forward)

**Priority:** P2 (Plan v3, Day 2: troubleshooting-first)
**Status:** Resolved (2026-10-15)
**Component:** Networking: OVN network forward (LXD), OVN load balancer, Kubernetes NodePort. Tools: `curl`, `tcpdump`, `ovn-nbctl`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). `k8s1` (MicroK8s) on `mc3` at `10.10.10.2`. NodePort Service `hello-np` on `31986`. Front door: OVN forward `192.168.82.35:80 → 10.10.10.2:31986`.
**Linked tickets:** INFRA-021 (built the front door), INFRA-025 (what a healthy request looks like). Sets up INFRA-027 (Ceph)

> Rewritten after the session with the real outputs.

---

## Summary

The front door was broken on purpose: its rule was changed to send traffic to port `31987` instead of `31986`, a one-digit typo. The site went down. It was then diagnosed one layer at a time, as if the cause were unknown.

**Result:** the cause was proven from three angles (LXD's config, OVN's own database, and packets on the wire), fixed, and the page loaded again.

## Why

"The site is down" is the most common ticket there is. The fastest path to the cause is to test one layer at a time: is the app up, does the traffic arrive, and where is it being sent? This drill shows how to tell "broken in front of the app" apart from "the app is broken".

---

## Runbook (with real results)

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check

```bash
sudo chronyc makestep                       # on each node
sudo microceph.ceph -s | head -5            # on mc1
lxc start --all
lxc exec k8s1 -- microk8s status --wait-ready | head -3
```
**Real result:** completed; output not captured in the session.

### Step 2: Record the healthy state

```bash
curl -s --max-time 3 http://192.168.82.35
lxc network forward show ovntest 192.168.82.35
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**What `ovn-nbctl lb-list` does:** LXD builds each network forward as a small **load balancer inside OVN**. This command lists those load balancers straight from OVN's own database (the "northbound" database, which holds the desired network setup).

**Real result:** completed; output not captured. The healthy rule is the same as the Step 6 result below.

### Step 3: Break it (the incident)

```bash
lxc network forward port remove ovntest 192.168.82.35 tcp 80
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31987
```

### Step 4: Symptom

```bash
curl -sS --max-time 3 http://192.168.82.35; echo "exit code: $?"
```
**Real result:**
```
curl: (7) Failed to connect to 192.168.82.35 port 80 after 5 ms: Could not connect to server
exit code: 7
```
**Reading it:** failing in **5 ms** means something actively refused the connection. A blocked or dead path would hang until the 3-second timeout instead.

### Step 5: Diagnose, one layer at a time

**5a. Where is the front door sending traffic?**
```bash
lxc network forward show ovntest 192.168.82.35
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**Real result:**
```
listen_address: 192.168.82.35
ports:
- protocol: tcp
  listen_port: "80"
  target_port: "31987"
  target_address: 10.10.10.2

65ec42d6-6dfa-4d47-b2d0-deffab09242c    lxd-net8-lb-192.    tcp        192.168.82.35:80    10.10.10.2:31987
```
LXD and OVN agree: traffic goes to port `31987`.

**5b. Does the request reach `k8s1`, and who refuses it?**

First attempt (2-second wait before curl):
```
0 packets captured
```
> **Gotcha:** the whole failed connection takes 5 ms. `tcpdump` starting inside the container through `lxc exec` wasn't ready yet, so it missed it. **0 packets doesn't always mean "nothing arrived".** Check the capture was really running first.

Second attempt (5-second wait):
```bash
lxc exec k8s1 -T -- timeout 15 tcpdump -ni eth0 -c 2 'tcp port 31987' < /dev/null &
sleep 5; curl -sS --max-time 3 http://192.168.82.35; wait
```
**Real result:**
```
09:43:28.965180 IP 192.168.82.139.34326 > 10.10.10.2.31987: Flags [S], ...
09:43:28.965296 IP 10.10.10.2.31987 > 192.168.82.139.34326: Flags [R.], ...
```
- `[S]`: `mc1` asks to connect; the front door has changed the destination to `10.10.10.2:31987`.
- `[R.]`: `k8s1` replies with a **reset** in 0.1 ms: "nothing is listening on this port".
- The source is still `192.168.82.139` (`mc1`): the forward changes only the **destination**, not the sender.

**5c. Is the app itself healthy?** Go around the front door:
```bash
lxc exec k8s1 -- curl -s --max-time 3 http://10.10.10.2:31986 | head -4
```
**Real result:** `<title>Welcome to nginx!</title>`. Kubernetes and the app are fine.

### Step 6: Fix and prove it

```bash
lxc network forward port remove ovntest 192.168.82.35 tcp 80
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31986
curl -s --max-time 3 http://192.168.82.35
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**Real result:** the nginx welcome page, and:
```
c99954c8-65e4-42b6-b108-4285446d76b4    lxd-net8-lb-192.    tcp        192.168.82.35:80    10.10.10.2:31986
```
The OVN load balancer has a **new ID** (`c99954c8...`, was `65ec42d6...`): LXD didn't edit the old rule, it deleted it and built a new one.

### Step 7: Shut down

On `mc1`: `lxc stop --all`. Then `sudo shutdown -h now` on `mc3`, `mc2`, then `mc1`.

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `http://192.168.82.35` refused in 5 ms (`curl: (7) Could not connect to server`) |
| **What I checked** | LXD forward config and OVN's load balancer (`ovn-nbctl lb-list`): both sent traffic to `10.10.10.2:31987`. `tcpdump` on `k8s1`: SYN arrived on 31987, reset sent back. Direct test of NodePort 31986: app healthy |
| **Root cause** | The front door's target port was `31987`; the Service's NodePort is `31986`. Nothing listens on 31987, so `k8s1` reset every connection |
| **Fix** | Re-created the forward's port rule with target port `31986` |
| **Prevention** | Keep the NodePort pinned (INFRA-024) and record it next to the forward in the docs. After any front-door change, run one `curl` through it, and compare `ovn-nbctl lb-list` with the Service's port (`kubectl get svc hello-np`) |

**Lessons:**
1. **A fast failure is a clue.** Refused in milliseconds means "reached a machine, nothing listening". A timeout means "blocked or lost".
2. **Go around the broken layer.** One `curl` straight to the NodePort proved Kubernetes was fine, so there was no need to debug it.
3. **Check the tool before trusting it.** The first `0 packets` was a capture that started too late, not missing traffic.

**AWS comparison:** this is a load balancer **target group pointing at the wrong port**. On AWS the health checks would mark the target unhealthy. LXD's forward has no health checks, so nothing warned us; only the users would notice.

---

## Definition of Done

- [x] Incident created and the exact symptom recorded
- [x] Wrong port proven in LXD, in OVN and on the wire (tcpdump)
- [x] App proven healthy by going around the front door
- [x] Fix applied and proven (curl, OVN)
- [ ] Page confirmed in the Windows browser
- [ ] Lab shut down cleanly
- [ ] Steps 1–2 output captured (on a repeat run)

## Retention Check

The app answered on `10.10.10.2:31986` but not through `192.168.82.35`. Why did that one test save you from debugging Kubernetes?