# INFRA-026: Incident Drill: Broken Front Door (OVN Network Forward)

**Priority:** P2 (Plan v3, Day 2: troubleshooting-first)
**Component:** Networking: OVN network forward (LXD), OVN load balancer, Kubernetes NodePort. Tools: `curl`, `tcpdump`, `ovn-nbctl`
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud (`mc1`, `mc2`, `mc3`). `k8s1` (MicroK8s) on `mc3` at `10.10.10.2`. NodePort Service `hello-np` on `31986`. Front door: OVN forward `192.168.82.35:80 → 10.10.10.2:31986`.
**Linked tickets:** INFRA-021 (built the front door), INFRA-025 (what a healthy request looks like). Sets up INFRA-027 (Ceph)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

We break the front door on purpose by pointing it at the wrong port (`31987` instead of `31986`), a one-digit typo. Then we act like the on-call engineer who doesn't know what changed: check each layer, find the exact cause, fix it, and prove the fix.

## Why

"The site is down" is the most common ticket there is. The fastest way to the cause is to test one layer at a time: is the app up, does the traffic arrive, and where is it being sent? This drill uses the healthy capture from INFRA-025 as the "before" picture.

---

## Runbook

All commands run on **`mc1`** unless the step says otherwise.

### Step 1: Start-up check (about 5 min)

Power on `mc1`, `mc2`, `mc3`. On **each node**:
```bash
sudo chronyc makestep
```
On `mc1`:
```bash
sudo microceph.ceph -s | head -5
lxc start --all
lxc exec k8s1 -- microk8s status --wait-ready | head -3
```
**Expected:** `HEALTH_OK`, then `microk8s is running`.

**Real result:** _pending_

### Step 2: Record the healthy state (the "before" picture)

```bash
curl -s --max-time 3 http://192.168.82.35
lxc network forward show ovntest 192.168.82.35
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**What the last command does:** LXD builds each network forward as a small **load balancer inside OVN**. `ovn-nbctl lb-list` lists those load balancers straight from OVN's own database (the "northbound" database, which holds the desired network setup).

**Expected:** a nginx page; the forward shows `target_port: "31986"`; the OVN line shows `192.168.82.35:80` → `10.10.10.2:31986`.

**Real result:** _pending_

### Step 3: Break it (the incident)

Replace the forward's rule with one that has a typo in the port:
```bash
lxc network forward port remove ovntest 192.168.82.35 tcp 80
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31987
```
**Expected:** no output.

From here on, pretend you don't know what changed.

**Real result:** _pending_

### Step 4: Symptom

```bash
curl -sS --max-time 3 http://192.168.82.35; echo "exit code: $?"
```
`-S` makes curl show its error message (INFRA-021 used `-s`, which hides it).

**Expected:** an error, not a page. Write down the exact message.

**Real result:** _pending_

### Step 5: Diagnose, one layer at a time

**5a. Is the app itself healthy?** Skip the front door and go straight to the NodePort:
```bash
lxc exec k8s1 -- curl -s --max-time 3 http://10.10.10.2:31986
```
**Expected:** the nginx page. If yes, the app and Kubernetes are fine; the problem is in front of them.

**5b. Does the request reach `k8s1`, and on which port?** Capture both ports while sending one request:
```bash
lxc exec k8s1 -T -- timeout 10 tcpdump -ni eth0 -c 4 'tcp port 31986 or tcp port 31987' < /dev/null &
sleep 2; curl -s --max-time 3 http://192.168.82.35; wait
```
**Expected:** packets arrive at `10.10.10.2.31987`, not `31986`. A reply with `Flags [R]` (a reset) would mean "nothing is listening on that port".

**5c. Confirm where OVN is sending it:**
```bash
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**Expected:** `10.10.10.2:31987`. Compare with Step 2.

**Real result:** _pending_

### Step 6: Fix and prove it

```bash
lxc network forward port remove ovntest 192.168.82.35 tcp 80
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31986
curl -s --max-time 3 http://192.168.82.35
sudo microovn.ovn-nbctl lb-list | grep 192.168.82.35
```
**Expected:** the nginx page again, and OVN back to `10.10.10.2:31986`. Also open `http://192.168.82.35` in the Windows browser.

**Real result:** _pending_

### Step 7: Shut down (about 2 min)

On `mc1`: `lxc stop --all`. Then `sudo shutdown -h now` on `mc3`, `mc2`, then `mc1`.

---

## Incident write-up (fill in after the session)

| | |
|---|---|
| **Symptom** | |
| **What I checked** | |
| **Root cause** | |
| **Fix** | |
| **Prevention** | |

---

## Definition of Done

- [ ] Healthy state recorded (page, LXD forward, OVN load balancer)
- [ ] Incident created and the exact symptom recorded
- [ ] App proven healthy by going around the front door
- [ ] Wrong port proven with tcpdump and `ovn-nbctl`
- [ ] Fix applied and proven (curl, OVN, browser)
- [ ] Incident write-up filled in
- [ ] Lab shut down cleanly

## Retention Check

The app answered on `10.10.10.2:31986` but not through `192.168.82.35`. Why did that one test save you from debugging Kubernetes?