# INFRA-030: Incident Drill: Pods That Keep Crashing, and a Pod That Never Gets Traffic

**Priority:** P2 (Plan v3, Day 6: troubleshooting-first)
**Status:** Resolved (2026-10-16)
**Component:** Kubernetes Deployments, ConfigMaps, readiness probes, Services. Tools: `kubectl get`, `kubectl logs --previous`, `kubectl describe`
**Environment:** `k8s1` (MicroK8s, Kubernetes v1.35.6), LXD container on `mc3`. Everything ran in a throwaway namespace `drill`. **The `hello` app and the front door were not touched.**
**Linked tickets:** INFRA-020 (first pod), INFRA-021 (Services), INFRA-029 (LXD won't start). Sets up INFRA-031 (strace and systemd)

> Rewritten after the session with the real outputs.

---

## Summary

Two drills, both very common Kubernetes tickets:

**Part A: CrashLoopBackOff.** An app started, couldn't find its config file, and exited with an error. Kubernetes kept restarting it, waiting longer before each retry (**CrashLoopBackOff**). The logs showed `ERROR: /etc/app/config.yaml not found`, and `describe` showed `Exit Code: 1`. Fixed by giving the app its file through a **ConfigMap** (Kubernetes' way of storing config and mounting it into pods as files): `1/1 Running`, `0` restarts.

**Part B: a failing readiness probe.** A **readiness probe** is a regular check: "is this pod ready for traffic?" It was pointed at `/healthz`, a page nginx doesn't have. The pod ran but stayed `0/1` ready, its events said `Readiness probe failed: ... statuscode: 404`, and the Service had **no endpoints**, so no traffic could reach it. Fixed by pointing the probe at `/`: `1/1` ready, and the Service listed the pod.

## Why

"The pod keeps restarting" and "the pod is running but the site doesn't work" are two of the most common Kubernetes tickets. The skills: reading the crashed run's logs (`--previous`), and reading **events** (`describe`), which explain what Kubernetes itself did and why.

---

## Runbook (with real results)

All commands ran **inside `k8s1`** (`lxc shell k8s1` from `mc1`).

### Step 1: Start-up check

```bash
microk8s status --wait-ready | head -3
kubectl get nodes
kubectl get pods
```
```
microk8s is running
NAME   STATUS   ROLES    AGE    VERSION
k8s1   Ready    <none>   7d1h   v1.35.6
NAME                        READY   STATUS    RESTARTS       AGE
hello-9ff96cc5-hfswg        1/1     Running   3 (150m ago)   4d3h
hello-9ff96cc5-jrd5d        1/1     Running   3 (150m ago)   4d3h
notes-pv-569c96c9ff-9m784   1/1     Running   9 (150m ago)   6d1h
```
> **Side note:** `RESTARTS` isn't always a crash count. These pods restarted "150m ago", about when the lab was powered on. Every lab shutdown and start-up restarts them. Read the "ago" before worrying about the number.

## Part A: CrashLoopBackOff

### Step 2: Deploy the broken app

Namespace `drill` created, and `crashy.yaml` applied (busybox: prints `starting`, exits with an error if `/etc/app/config.yaml` is missing).
```
namespace/drill created
deployment.apps/crashy created
```

### Step 3: Symptom

```bash
kubectl -n drill get pods
```
```
NAME                      READY   STATUS   RESTARTS      AGE
crashy-7956c99f99-92x5m   0/1     Error    2 (17s ago)   30s
crashy-7956c99f99-92x5m   0/1     Error    3 (41s ago)   68s
```
`RESTARTS` kept climbing (2, then 3, then 5 by the next step). The status flips between `Error` (it just crashed) and `CrashLoopBackOff` (waiting before the next retry). Both checks happened to catch `Error`.

### Step 4: Diagnose

```bash
kubectl -n drill logs deploy/crashy
kubectl -n drill logs deploy/crashy --previous
kubectl -n drill describe pod -l app=crashy | grep -A6 -E "Last State|^Events"
```
```
starting
ERROR: /etc/app/config.yaml not found

starting
ERROR: /etc/app/config.yaml not found

    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Fri, 16 Oct 2026 12:34:39 +0000
      Finished:     Fri, 16 Oct 2026 12:34:39 +0000
    Restart Count:  5
Events:
  Normal   Scheduled  3m23s                default-scheduler  Successfully assigned drill/crashy-... to k8s1
  Normal   Pulled     3m10s                kubelet            Successfully pulled image "busybox:1.36" in 11.635s ...
  Normal   Created    11s (x6 over 3m10s)  kubelet            Container created
```
**Reading it:**
- Both `logs` and `logs --previous` show the reason. Here the current run had already crashed too. When a pod is waiting to retry, plain `logs` can be empty, and `--previous` is where the answer is.
- `Exit Code: 1`: the app itself exited with an error. Kubernetes didn't kill it.
- `Started` and `Finished` in the **same second**: it dies immediately on start, a typical sign of a missing file or bad setting.
- `Created (x6 over 3m10s)`: Kubernetes has created the container 6 times. (The `Back-off restarting failed container` warning was below the 6 lines `grep` showed.)

### Step 5: Fix: give the app its config file

```bash
kubectl -n drill create configmap app-config --from-literal=config.yaml='mode: lab'
# crashy.yaml re-applied with a volume that mounts app-config at /etc/app
kubectl apply -f crashy.yaml
sleep 15
kubectl -n drill get pods
kubectl -n drill logs deploy/crashy
```
```
configmap/app-config created
deployment.apps/crashy configured
NAME                      READY   STATUS    RESTARTS   AGE
crashy-6b86574d9b-72lfb   1/1     Running   0          31s
starting
config loaded:
```
- **A new pod** (`...-72lfb`): changing a Deployment replaces its pods.
- `1/1 Running`, `0` restarts, and the log reached `config loaded:`, so the file was found.

> **Small open point:** the line `mode: lab` (the file's content) didn't show in the log. Most likely the value has no line break at the end, so it was lost in the terminal output. To check on a repeat run: `kubectl -n drill exec deploy/crashy -- cat /etc/app/config.yaml`.

> **Paste note:** the terminal showed the pasted YAML scrambled (merged lines). That was only the screen display; `apply` succeeded and the pod ran, so the file was correct.

## Part B: A failing readiness probe

### Step 6: Deploy a web app with a wrong health check

`web.yaml` applied: nginx with a readiness probe on `/healthz` every 5 seconds, plus a Service `web`.
```
deployment.apps/web created
service/web created
NAME                   READY   STATUS              RESTARTS   AGE
web-57787b98b4-p4vdk   0/1     ContainerCreating   0          47s
```
> **Finding: `ContainerCreating` was not the fault.** The events (Step 7) show the nginx image (72 MB) took **48.7 seconds** to download. The 20-second wait wasn't long enough. When a pod sits in `ContainerCreating`, check the events for `Pulling` before assuming something is broken.

A few seconds later:
```
NAME                   READY   STATUS    RESTARTS   AGE
web-57787b98b4-p4vdk   0/1     Running   0          84s
```
**Running, but `0/1` ready, with `0` restarts.** The app isn't crashing; it just isn't trusted with traffic.

### Step 7: Diagnose

```bash
kubectl -n drill describe pod -l app=web | grep -A8 "^Events"
kubectl -n drill describe svc web | grep -i endpoints
```
```
  Normal   Pulling    73s               kubelet  Pulling image "nginx:1.27"
  Normal   Pulled     25s               kubelet  Successfully pulled image "nginx:1.27" in 48.708s ... Image size: 72406859 bytes.
  Normal   Started    25s               kubelet  Container started
  Warning  Unhealthy  3s (x6 over 24s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404

Endpoints:
```
- `Readiness probe failed ... 404`: the check asked for a page that doesn't exist. It failed 6 times in 24 seconds (every 5 seconds, as set).
- `Endpoints:` **empty**: the Service has no ready pod, so traffic sent to it goes nowhere.

### Step 8: Fix the probe

```bash
sed -i 's#path: /healthz#path: /#' web.yaml
kubectl apply -f web.yaml
kubectl -n drill get pods -l app=web
kubectl -n drill describe svc web | grep -i endpoints
```
```
deployment.apps/web configured
service/web unchanged
NAME                  READY   STATUS    RESTARTS   AGE
web-c74b5d988-p7ktb   1/1     Running   0          9s
Endpoints:                10.1.166.247:80
```
`1/1` ready within 9 seconds (the image was already downloaded), and the Service now sends traffic to the pod at `10.1.166.247:80`.

### Step 9: Clean up

```bash
kubectl delete namespace drill
kubectl get pods
exit
```
```
namespace "drill" deleted
hello-9ff96cc5-hfswg        1/1     Running   3 (161m ago)   4d3h
hello-9ff96cc5-jrd5d        1/1     Running   3 (161m ago)   4d3h
notes-pv-569c96c9ff-9m784   1/1     Running   9 (161m ago)   6d1h
logout
```
`drill` gone; `hello` and `notes-pv` untouched.

---

## Incident write-up

| | Part A: CrashLoopBackOff | Part B: readiness probe |
|---|---|---|
| **Symptom** | `0/1`, `Error` / `CrashLoopBackOff`, `RESTARTS` climbing | `Running` but `0/1` ready, `RESTARTS 0`; Service with no endpoints |
| **What I checked** | `logs` and `logs --previous`: `ERROR: /etc/app/config.yaml not found`. `describe`: `Exit Code: 1`, started and finished in the same second | `describe pod` events: `Readiness probe failed: HTTP probe failed with statuscode: 404`. `describe svc`: `Endpoints:` empty |
| **Root cause** | The app needs `/etc/app/config.yaml`, and nothing provided it | The probe checked `/healthz`, a page nginx doesn't have |
| **Fix** | ConfigMap `app-config` mounted at `/etc/app` | Probe path changed to `/` |
| **Prevention** | Ship config with the app (ConfigMap in the same manifest); test a new Deployment in a separate namespace first | Point probes at a page the app really serves; check `READY` and endpoints after every rollout, not just `STATUS` |

**Lessons:**
1. **`Running` isn't the same as working.** Check the `READY` column and the Service's endpoints too.
2. **`logs --previous` shows the crashed run**, which is where crash reasons usually are.
3. **Events explain Kubernetes' own actions:** restarts, failed checks, slow downloads.
4. **Not every odd state is a fault.** `ContainerCreating` was a slow image download; `RESTARTS 3` was the lab being switched on.

**AWS comparison:**
- A readiness probe works like an **ALB target group health check**: a failing target stays registered but gets no traffic.
- CrashLoopBackOff is like an **ECS task** that keeps stopping with `Essential container exited`: the service keeps starting new tasks, and the reason is in the stopped task's logs.

---

## Definition of Done

- [x] Part A: crash loop seen; cause found with `logs --previous` and `describe`; fixed with a ConfigMap
- [x] Part B: pod `Running` but `0/1` ready; cause found in events; Service had no endpoints; fixed the probe
- [x] Namespace `drill` deleted; `hello` untouched
- [ ] `mode: lab` file content confirmed with `kubectl exec` (on a repeat run)
- [ ] Lab shut down cleanly

## Retention Check

A pod shows `Running` but `0/1` ready, and `RESTARTS 0`. Is the app crashing? Where do you look first, and what does the Service do with this pod?

**Answer:** No, it isn't crashing (0 restarts); its readiness check is failing. Look first at the pod's events (`kubectl describe pod`), which say why the probe fails. The Service leaves the pod out of its endpoints, so it gets no traffic until the probe passes.