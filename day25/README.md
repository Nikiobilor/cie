# INFRA-024: Rebuild Kubernetes and Restore the Service Path After the Incident

**Priority:** P2 (Week 5; follow-up to the INFRA-023 incident)
**Status:** Done (2026-10-10)
**Component:** LXD container `k8s1`, MicroK8s, Kubernetes Service (NodePort), OVN network forward
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), rebuilt MicroCeph pool `remote`, OVN network `ovntest`, front door `192.168.82.35`
**Linked tickets:** INFRA-020 (install MicroK8s), INFRA-021 (Services, boot fix, front door), INFRA-023 (incident that destroyed `k8s1`). Unblocks INFRA-022 (persistent storage)

---

## Summary

The INFRA-023 power-loss incident ended with Ceph rebuilt from scratch, so `k8s1` and its Kubernetes app were gone. This ticket rebuilds them **only from the earlier tickets**, as a test of whether those tickets really work as rebuild instructions.

**Results:**

| Check | Result |
|---|---|
| `microk8s` LXD profile survived the incident | Yes, including `rbd` (it lives in LXD's database, not on Ceph) |
| `k8s1` back at its old address | Yes, pinned at `10.10.10.2` before first start |
| Kubernetes version | v1.35.6, same as before |
| MicroK8s starts by itself after a restart | Yes (INFRA-021 boot fix) |
| `hello` app reachable through the **pre-incident** front door | Yes, **without touching the front door** |

The last row is the point: the front door at `192.168.82.35` was created days before the incident and worked again unchanged, because both values it depends on were rebuilt **pinned**: `k8s1`'s address and the NodePort.

---

## Acceptance Criteria

- [x] Leftover test container from the incident removed
- [x] `microk8s` profile checked
- [x] `k8s1` recreated on `mc3` with `10.10.10.2` pinned before first start
- [x] MicroK8s installed, node `Ready`
- [x] AppArmor boot fix installed and proven with a real restart
- [x] `hello` Deployment recreated, behind a NodePort Service pinned to `31986`
- [x] Page served through `192.168.82.35` with no change to the forward

---

## Comments Thread

**Reviewer:** Ceph is back. How do you get Kubernetes back?

**NwaChi:** Follow INFRA-020 and INFRA-021 again.

**Reviewer:** Good, and treat it as a test of those tickets. If a step is missing or wrong, the ticket is wrong. What's different this time?

**NwaChi:** Last time I pinned `k8s1`'s address after it already had one. This time I can pin it before it ever starts.

**Reviewer:** Right, `lxc init`, then set the address, then start. What about the front door?

**NwaChi:** It points at `10.10.10.2`, port `31986`. The address is pinned. But the NodePort was random last time.

**Reviewer:** So pin that too. Then nothing outside Kubernetes has to change. That's the real lesson: anything other systems point at should be fixed, not left to chance.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step.
> **(mc1)** = on host `mc1`. **(k8s1)** = inside `lxc shell k8s1`.

### Step 1: Clear the way and check what survived (mc1)

```bash
lxc list t1 -c n4L
lxc delete t1 --force
lxc profile get microk8s linux.kernel_modules
```

`t1` was the test container from INFRA-023. Deleting it also frees `10.10.10.2` if it took that address.

**Real result:** `ip_vs,ip_vs_rr,ip_vs_wrr,ip_vs_sh,ip_tables,ip6_tables,netlink_diag,nf_nat,overlay,br_netfilter,rbd`. The profile survived, with the `rbd` module added in INFRA-022.

### Step 2: Recreate `k8s1` with the address pinned first (mc1)

```bash
lxc init ubuntu:24.04 k8s1 -p default -p microk8s --network ovntest --storage remote --target mc3
lxc config device set k8s1 eth0 ipv4.address=10.10.10.2
lxc start k8s1
lxc list k8s1 -c n4L
```

Plain English: `lxc init` creates the container **without starting it**, so the pinned address is in place before the container ever asks OVN for one.

**Real result:** `k8s1 | 10.10.10.2 (eth0) | mc3`.

### Step 3: Install MicroK8s and the boot fix (k8s1)

```bash
snap install microk8s --classic
microk8s status --wait-ready
```

Then the start-up service from INFRA-021, which loads MicroK8s' AppArmor rules before Kubernetes starts (snapd skips them inside this unconfined container):
```bash
cat > /etc/systemd/system/microk8s-apparmor.service << 'EOF'
[Unit]
Description=Load MicroK8s AppArmor profiles (LXD container workaround)
After=snapd.apparmor.service
Before=snap.microk8s.daemon-containerd.service snap.microk8s.daemon-kubelite.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'apparmor_parser -r /var/lib/snapd/apparmor/profiles/snap.microk8s.*'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable microk8s-apparmor.service
microk8s kubectl get nodes
```

**Real result:** `k8s1   Ready   <none>   58m   v1.35.6`.

### Step 4: Prove the boot fix (mc1)

```bash
lxc restart k8s1
sleep 60
lxc exec k8s1 -- microk8s status --wait-ready --timeout 120 | head -1
```

**Real result:** `microk8s is running`, with no manual steps.

If it says `not running`, check `lxc exec k8s1 -- systemctl is-enabled microk8s-apparmor.service`.

### Step 5: Recreate the app, with the NodePort pinned (k8s1)

Plain English: Kubernetes normally picks a random NodePort between 30000 and 32767. The front door points at **31986**, so the Service asks for exactly that.

```bash
microk8s kubectl create deployment hello --image=nginx --replicas=2
cat > hello-np.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: hello-np
spec:
  type: NodePort
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31986
EOF
microk8s kubectl apply -f hello-np.yaml
microk8s kubectl get pods -l app=hello
microk8s kubectl get service hello-np
```

- `selector: app: hello` sends traffic to the Deployment's pods.
- `nodePort: 31986` requests that exact port.

**Real result:** both pods sat at `ContainerCreating` for about 3.5 minutes. That was only the nginx image downloading after the incident wiped every cached image; they became `Running` on their own.

> If pods stay at `ContainerCreating` much longer, check `microk8s kubectl describe pod <name> | tail -8`. `Pulling image` with no error means it's still downloading.

### Step 6: The end-to-end test (mc1, then Windows)

```bash
curl -s http://192.168.82.35 | head -4
```

**Real result:** `<title>Welcome to nginx!</title>`.

The request path, unchanged from INFRA-021:

| Hop | What moves it on |
|---|---|
| Browser / `mc1` → `192.168.82.35:80` | VMware's network |
| `192.168.82.35:80` → `10.10.10.2:31986` | OVN network forward (created before the incident, untouched) |
| `k8s1:31986` → Service `hello-np` | NodePort, pinned |
| Service → one of the `hello` pods | The Service's live list of pods |

---

## Write-up

**Did the tickets work as rebuild instructions?** Yes. INFRA-020 and INFRA-021 brought `k8s1` and its app back step by step, including the non-obvious AppArmor boot fix, which a restart then proved.

**What made the rebuild invisible to everything outside it:** pinning. `k8s1`'s address and the NodePort were both set explicitly, so the front door, built days before, didn't need to know anything had happened. In production this is the same reason load balancers, DNS records and firewall rules point at **stable** names and addresses, never at whatever a machine happened to get last time.

**What wasn't recovered:** the pods' content. INFRA-021 had labelled each pod's page with its name; the rebuilt pods serve nginx's default page. Nothing was persisted, because nothing in Kubernetes used persistent storage yet. That's exactly what INFRA-022 covers.

**AWS comparison:** on EKS, the stable pieces are usually a `LoadBalancer` Service and a DNS name in Route 53. Nodes and pods can be replaced freely behind them, and clients never notice.

---

## Definition of Done

- [x] `k8s1` rebuilt at `10.10.10.2`, Kubernetes v1.35.6, `Ready`
- [x] Boot fix proven by restart
- [x] `hello` behind NodePort `31986`
- [x] Front door `192.168.82.35` working unchanged
- [ ] `web1` (INFRA-018, `10.10.10.50`) not rebuilt; only needed if its front door `192.168.82.33` is used again

---

## Retention Check

The front door was created before the incident, and everything behind it was destroyed and rebuilt. Why did it work again without any change, and what would have broken it?