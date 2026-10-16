# INFRA-030: Incident Drill: Pods That Keep Crashing, and a Pod That Never Gets Traffic

**Priority:** P2 (Plan v3, Day 6: troubleshooting-first)
**Component:** Kubernetes Deployments, ConfigMaps, readiness probes, Services. Tools: `kubectl get`, `kubectl logs --previous`, `kubectl describe`
**Environment:** `k8s1` (MicroK8s, Kubernetes v1.35.6), LXD container on `mc3`. Everything runs in a throwaway namespace `drill`. **The `hello` app and the front door are not touched.**
**Linked tickets:** INFRA-020 (first pod), INFRA-021 (Services), INFRA-029 (LXD won't start). Sets up INFRA-031 (strace and systemd)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

Two drills, both very common Kubernetes tickets:

**Part A: CrashLoopBackOff.** An app starts, can't find its config file, and exits. Kubernetes restarts it, it crashes again, and Kubernetes waits longer before each retry. That waiting state is called **CrashLoopBackOff**. We find the reason in the logs of the *previous* (crashed) run, then fix it by giving the app its config file through a **ConfigMap** (Kubernetes' way of storing config and mounting it into pods as files).

**Part B: a failing readiness probe.** A **readiness probe** is a regular check Kubernetes runs: "is this pod ready for traffic?" If the check fails, the pod keeps running but gets **no traffic** from its Service. We point the probe at a page that doesn't exist, see the pod stuck at `0/1` ready, find the reason in its events, and fix the probe.

## Why

"The pod keeps restarting" and "the pod is running but the site doesn't work" are two of the most common Kubernetes tickets. The skills: reading the crashed run's logs (`--previous`), and reading **events** (`describe`), which explain what Kubernetes itself did and why.

---

## Runbook

All commands run **inside `k8s1`**. From `mc1`:
```bash
lxc shell k8s1
```

### Step 1: Start-up check

```bash
microk8s status --wait-ready | head -3
kubectl get nodes
kubectl get pods
```
**Expected:** `microk8s is running`; node `Ready`; `hello` pods `Running`.

**Real result:** _pending_

## Part A: CrashLoopBackOff

### Step 2: Deploy the broken app

```bash
kubectl create namespace drill
cat > crashy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashy
  namespace: drill
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashy
  template:
    metadata:
      labels:
        app: crashy
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo starting; if [ ! -f /etc/app/config.yaml ]; then echo 'ERROR: /etc/app/config.yaml not found'; exit 1; fi; echo 'config loaded:'; cat /etc/app/config.yaml; sleep 3600"]
EOF
kubectl apply -f crashy.yaml
```
**What the app does:** prints `starting`, checks for `/etc/app/config.yaml`, and **exits with an error** if the file is missing. If the file is there, it prints it and keeps running.

**Expected:** `namespace/drill created`, `deployment.apps/crashy created`.

From here on, pretend you don't know what the app needs.

**Real result:** _pending_

### Step 3: Symptom

Run this 3 or 4 times, about 20 seconds apart:
```bash
kubectl -n drill get pods
```
**Expected:** `STATUS` moves between `Error` and `CrashLoopBackOff`; `RESTARTS` keeps climbing.

**Real result:** _pending_

### Step 4: Diagnose

```bash
kubectl -n drill logs deploy/crashy
kubectl -n drill logs deploy/crashy --previous
kubectl -n drill describe pod -l app=crashy | grep -A6 -E "Last State|^Events"
```
**What each shows:**
- `logs`: the **current** run. During the wait between retries it can be empty, or show the same few lines.
- `logs --previous`: the run that **crashed**. This is usually where the answer is.
- `describe`: `Last State: Terminated`, `Reason: Error`, `Exit Code: 1`, and events such as `Back-off restarting failed container`.

**Expected:** `ERROR: /etc/app/config.yaml not found`, and `Exit Code: 1`.

**Real result:** _pending_

### Step 5: Fix: give the app its config file

```bash
kubectl -n drill create configmap app-config --from-literal=config.yaml='mode: lab'
cat > crashy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashy
  namespace: drill
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashy
  template:
    metadata:
      labels:
        app: crashy
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["sh", "-c", "echo starting; if [ ! -f /etc/app/config.yaml ]; then echo 'ERROR: /etc/app/config.yaml not found'; exit 1; fi; echo 'config loaded:'; cat /etc/app/config.yaml; sleep 3600"]
        volumeMounts:
        - name: config
          mountPath: /etc/app
      volumes:
      - name: config
        configMap:
          name: app-config
EOF
kubectl apply -f crashy.yaml
sleep 15
kubectl -n drill get pods
kubectl -n drill logs deploy/crashy
```
- The ConfigMap `app-config` holds one file, `config.yaml`, containing `mode: lab`.
- The new `volumeMounts` and `volumes` sections mount it into the pod at `/etc/app`.

**Expected:** a new pod `Running` with `RESTARTS 0`; the log shows `config loaded:` and `mode: lab`.

**Real result:** _pending_

## Part B: A failing readiness probe

### Step 6: Deploy a web app with a wrong health check

```bash
cat > web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: drill
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: drill
spec:
  selector:
    app: web
  ports:
  - port: 80
EOF
kubectl apply -f web.yaml
sleep 20
kubectl -n drill get pods -l app=web
```
The probe asks nginx for `/healthz` every 5 seconds. nginx has no such page, so it answers **404 (not found)**.

**Expected:** `READY 0/1`, `STATUS Running`, `RESTARTS 0`. Running, but not ready.

**Real result:** _pending_

### Step 7: Diagnose

```bash
kubectl -n drill describe pod -l app=web | grep -A8 "^Events"
kubectl -n drill describe svc web | grep -i endpoints
```
**Expected:**
- Events: `Readiness probe failed: HTTP probe failed with statuscode: 404`.
- The Service shows **no endpoints** (empty): it has no ready pod to send traffic to.

**Real result:** _pending_

### Step 8: Fix the probe

```bash
sed -i 's#path: /healthz#path: /#' web.yaml
kubectl apply -f web.yaml
sleep 20
kubectl -n drill get pods -l app=web
kubectl -n drill describe svc web | grep -i endpoints
```
`/` is nginx's welcome page, which returns **200 (OK)**.

**Expected:** `READY 1/1`; the Service now lists the pod's IP as an endpoint.

**Real result:** _pending_

### Step 9: Clean up

```bash
kubectl delete namespace drill
kubectl get pods
exit
```
**Expected:** `namespace "drill" deleted`; the `hello` pods still `Running`. `exit` returns you to `mc1`.

Then shut down: `lxc stop --all`, then `sudo shutdown -h now` on `mc3`, `mc2`, `mc1`.

**Real result:** _pending_

---

## Incident write-up (fill in after the session)

| | Part A: CrashLoopBackOff | Part B: readiness probe |
|---|---|---|
| **Symptom** | | |
| **What I checked** | | |
| **Root cause** | | |
| **Fix** | | |
| **Prevention** | | |

---

## Definition of Done

- [ ] Part A: CrashLoopBackOff seen; cause found with `logs --previous` and `describe`; fixed with a ConfigMap
- [ ] Part B: pod `Running` but `0/1` ready; cause found in events; Service had no endpoints; fixed the probe
- [ ] Namespace `drill` deleted; `hello` untouched
- [ ] Lab shut down cleanly

## Retention Check

A pod shows `Running` but `0/1` ready, and `RESTARTS 0`. Is the app crashing? Where do you look first, and what does the Service do with this pod?