# INFRA-025: Follow One Web Request Through the Lab with tcpdump

**Priority:** P2 (Plan v3, Day 1: troubleshooting-first)
**Component:** Networking: OVN network forward, Kubernetes NodePort, pod network. Tool: `tcpdump`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1` 192.168.20.140, `mc2` .141, `mc3` .142). `k8s1` (MicroK8s, Kubernetes v1.35.6) on `mc3` at `10.10.10.2` on OVN network `ovntest`. Deployment `hello` (2 nginx pods) behind NodePort Service `hello-np` on port `31986`. Front door: OVN forward `192.168.82.35:80 → 10.10.10.2:31986`.
**Linked tickets:** INFRA-021 (built this request path). Sets up INFRA-026 (Day 2: break the OVN forward)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

INFRA-021 built a path from the Windows browser to a Kubernetes pod. We know it works because the page loads. Today we **watch** it work: capture the same request at three points with `tcpdump` and note what changes at each hop (addresses and ports) and what stays the same.

**The path:**

| Hop | Where we capture | What we expect to see |
|---|---|---|
| 1 | `mc1`, uplink interface `ens37` | `192.168.82.139` → `192.168.82.35` port `80` |
| 2 | `k8s1`, interface `eth0` | same request, now going to `10.10.10.2` port `31986` |
| 3 | `k8s1`, all interfaces | same request, now going to a pod (`10.1.x.x`) port `80` |

## Why

When a site is down, "the page doesn't load" says nothing about **where** it stops. `tcpdump` shows whether a packet reached a point, and what it looked like when it got there. Capturing hop by hop is how you find the exact layer that is broken. Today, with nothing broken, we learn what "healthy" looks like, so Day 2 (a broken forward) has something to compare against.

---

## Runbook

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check (about 5 min)

Power on `mc1`, `mc2`, `mc3` in VMware. On **each node**:
```bash
sudo chronyc makestep
```
On `mc1`:
```bash
sudo microceph.ceph -s | head -5
```
**Expected:** `health: HEALTH_OK`. If a node is missing and its console shows `Detected Tx Unit Hang`, reset that VM.

**Real result:** _pending_

### Step 2: Start the containers and Kubernetes

```bash
lxc start --all
lxc exec k8s1 -- microk8s status --wait-ready | head -3
lxc exec k8s1 -- bash -c 'time kubectl get pods -A > /dev/null'
```
**Expected:** `microk8s is running`, and `real` under 1 second.

**Real result:** _pending_

### Step 3: Baseline: does the page load?

```bash
curl -s --max-time 3 http://192.168.82.35
lxc exec k8s1 -- kubectl get pods -o wide
```
**Expected:** `Hello from pod hello-...`, and two `hello` pods `Running`, each with a `10.1.x.x` address. Note those addresses; they show up in Step 6.

**Real result:** _pending_

### Step 4: Make sure tcpdump is in `k8s1`

```bash
lxc exec k8s1 -- which tcpdump || lxc exec k8s1 -- apt-get install -y tcpdump
```
**Expected:** `/usr/bin/tcpdump` (or a short install).

**Real result:** _pending_

### Step 5: Hop 1: the request leaving `mc1`

Starts a capture in the background, sends one request, then waits for the capture to finish.
```bash
sudo timeout 10 tcpdump -ni ens37 -c 6 host 192.168.82.35 and tcp port 80 &
sleep 1; curl -s --max-time 3 http://192.168.82.35; wait
```
**What the flags mean:** `-n` show numbers, not names. `-i ens37` listen on the uplink. `-c 6` stop after 6 packets. `timeout 10` stop after 10 seconds anyway.

**Expected:** lines like `192.168.82.139.xxxxx > 192.168.82.35.80: Flags [S]` (`[S]` is the first packet of a connection, a SYN) and replies from `192.168.82.35.80`.

**Real result:** _pending_

### Step 6: Hop 2: the request arriving at `k8s1`

```bash
lxc exec k8s1 -T -- timeout 10 tcpdump -ni eth0 -c 6 tcp port 31986 < /dev/null &
sleep 2; curl -s --max-time 3 http://192.168.82.35; wait
```
**Expected:** the destination is now `10.10.10.2.31986`. **Check the source address:** is it still `192.168.82.139` (`mc1`), or something else? Write down which.

**Real result:** _pending_

### Step 7: Hop 3: the request reaching a pod

```bash
lxc exec k8s1 -T -- timeout 10 tcpdump -ni any -c 6 tcp port 80 and net 10.1.0.0/16 < /dev/null &
sleep 2; curl -s --max-time 3 http://192.168.82.35; wait
```
**Expected:** the destination is one of the pod addresses from Step 3, port `80`. **Check the source again:** did it change this time?

**Real result:** _pending_

### Step 8: Shut down (about 2 min)

On `mc1`:
```bash
lxc stop --all
```
Then `sudo shutdown -h now` on `mc3`, then `mc2`, then `mc1`.

---

## Findings (fill in after the session)

| Hop | Source seen | Destination seen | What changed it |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |

---

## Definition of Done

- [ ] Start-up check passed (`HEALTH_OK`, Kubernetes ready)
- [ ] Page loads through `192.168.82.35`
- [ ] Request captured at hop 1 (`mc1`, `ens37`)
- [ ] Request captured at hop 2 (`k8s1`, port `31986`)
- [ ] Request captured at hop 3 (a pod, port `80`)
- [ ] Findings table filled in from real output
- [ ] Lab shut down cleanly

## Retention Check

A visitor says the site is down. You capture on `mc1` and see the request leave, but nothing arrives on `k8s1` port `31986`. Which layer do you look at next, and why?