# INFRA-020: Install MicroK8s and Deploy Your First Pod

**Priority:** P2 (Week 5, Day 22: Kubernetes Foundations)
**Component:** Kubernetes (MicroK8s) running inside an LXD container on the private cloud
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), Ceph pool `remote`, OVN network `ovntest` (MTU 1500). Kubernetes **v1.35.6** (MicroK8s)
**Linked tickets:** INFRA-019 (control loops). Sets up INFRA-021 (Day 23: Services and networking)

> **Note:** this ticket was rewritten after the session to record what actually happened, including a real problem: **pods stuck for 15+ minutes**, fixed by a clean MicroK8s restart.

---

## Summary

Kubernetes is now running on the private cloud: MicroK8s, inside an LXD container (`k8s1`) on `mc3`. We ran a first pod, then deleted it to see whether Kubernetes brings it back.

**Results:**

| Test | Result |
|---|---|
| First pod (`hello`, nginx) | Running at `10.1.166.197`; welcome page served |
| Delete a **bare pod** | **Gone for good.** Nothing brought it back |
| Delete one pod of a **Deployment** (2 replicas) | **Replaced automatically**: new pod created within 7 s, Running by 12 s, new name and new address. The other copy kept running throughout |

**Why inside a container, not a VM?** VMs can't run in this lab (INFRA-017). In production, Kubernetes nodes are almost always VMs or bare metal; containers as nodes are a lab and testing pattern.

---

## Acceptance Criteria

- [x] Enough memory and storage confirmed before starting
- [x] The MicroK8s LXD profile downloaded, **read**, and its security trade-off explained
- [x] Container `k8s1` running on `mc3` with that profile
- [x] MicroK8s installed and the node `Ready`
- [x] First pod running and answering
- [x] A bare pod deleted: it did **not** come back
- [x] A Deployment's pod deleted: it **was replaced**
- [x] Plain-English write-up
- [x] **Added:** stuck pods diagnosed from logs and fixed

---

## Comments Thread

**Reviewer:** Where are you putting Kubernetes?

**NwaChi:** Inside an LXD container, since VMs can't run here. Does that mean in production it would run on a VM or bare metal instead?

**Reviewer:** Yes. On AWS, EKS worker nodes are EC2 instances, which are VMs. On premises it's often bare metal. Containers as nodes are for labs and testing, like `kind`. Production avoids them for isolation, and because Kubernetes wants to own its kernel, networking and storage.

**NwaChi:** And the profile?

**Reviewer:** What did Kubernetes need from LXD, and what did it cost?

**NwaChi:** It needed the microk8s profile, which required a lot of privileged access to the host.

**Reviewer:** Right. In a privileged container, root inside is root on the host. A compromise of `k8s1` is a compromise of `mc3`.

**NwaChi:** So the profile's privileges wouldn't matter so much on a VM or bare metal, because it's abstracted from the host?

**Reviewer:** On a VM or bare metal you don't need the profile at all, because Kubernetes has its own kernel. A VM's hypervisor is the wall; bare metal has no host above it. That limits how far a break-in spreads. It doesn't prevent the break-in.

**NwaChi:** My pods have been Pending for 15 minutes.

**Reviewer:** Don't guess. Ask Kubernetes, then read the logs.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Step 1: Check there's room

```bash
free -m                    # on mc3
sudo microceph.ceph df     # on mc1
```

**Real result:** `mc3` had **3,069 MB available** (we want at least ~1.5 GB). Ceph had **7.6 GiB MAX AVAIL** (we want at least ~3 GiB). That's up from 4.5 GiB before the INFRA-018 cleanup.

### Step 2: Get the MicroK8s profile, and read it (`mc1`)

```bash
wget https://raw.githubusercontent.com/canonical/microk8s/master/tests/lxc/microk8s.profile -O microk8s.profile
cat microk8s.profile
```

**What it contains, in plain words:**

| Part | What it does |
|---|---|
| `boot.autostart` | Starts the container when the host boots |
| `linux.kernel_modules` (`ip_tables`, `nf_nat`, `ip_vs*`, `br_netfilter`, `overlay`, `netlink_diag`, ...) | Loads kernel features **on the host** for Kubernetes' networking and container images. A container can't load them itself |
| `security.privileged: "true"` | Root inside the container is root on the host |
| `security.nesting: "true"` | Allows containers inside this container (pods) |
| `lxc.cap.drop=` (empty) | Keeps all of root's powers |
| `lxc.apparmor.profile=unconfined` | Turns off AppArmor, the security guard |
| `lxc.cgroup.devices.allow=a` | Allows access to all devices |
| `lxc.mount.auto=proc:rw sys:rw cgroup:rw` | Allows changing kernel and resource settings |
| `devices` (`/dev/kmsg`, `nf_conntrack` settings, `/sys/fs/bpf`) | Shares specific host files Kubernetes reads or adjusts |

**In one line:** this container can change settings on its host's own kernel.

Create it:
```bash
lxc profile create microk8s
cat microk8s.profile | lxc profile edit microk8s
lxc profile show microk8s
```

(Use `microk8s-zfs.profile` instead on ZFS storage. Ceph uses the plain one.)

### Step 3: Launch `k8s1` on `mc3`

```bash
lxc launch ubuntu:24.04 k8s1 -p default -p microk8s --network ovntest --storage remote --target mc3
lxc list k8s1
```

### Step 4: Install MicroK8s

```bash
lxc exec k8s1 -- snap install microk8s --classic
lxc exec k8s1 -- microk8s status --wait-ready
```

**Real result:** Kubernetes **v1.35.6**. (Enabled add-ons weren't captured; record `microk8s status` on a repeat run.)

### Step 5: Look around (inside `k8s1`)

```bash
lxc shell k8s1
microk8s kubectl get nodes
microk8s kubectl get pods -A
```

> **Gotcha:** plain `kubectl` gives `Command 'kubectl' not found`, and `apt install kubectl` gives `Unable to locate package`. MicroK8s ships its own `kubectl`, matched to the cluster's version, so use `microk8s kubectl`. For a lasting shortcut: `snap alias microk8s.kubectl kubectl`.

### Step 6: Run your first pod, and the problem that appeared

```bash
microk8s kubectl run hello --image=nginx
microk8s kubectl get pods -o wide
```

**Real result:** `hello` was **Pending for 15+ minutes**, with `NODE <none>`.

#### 6a. Is the node the problem?

```bash
microk8s kubectl get nodes
microk8s kubectl describe pod hello | tail -10
```

**Real result:** the node was `Ready`, and `Events: <none>`. If Kubernetes had *tried* to place the pod and failed, it would have written a reason. Silence suggested the part that places pods (the scheduler) wasn't acting.

#### 6b. What about Kubernetes' own pods?

```bash
microk8s kubectl get pods -A
microk8s kubectl describe pod -n kube-system <coredns pod> | tail -15
microk8s kubectl describe node k8s1 | grep -i -A3 taints
```

**Real result:**
- `calico-node` was Running.
- `coredns` and `calico-kube-controllers` were stuck at `ContainerCreating` for 15 minutes.
- Every `Events` section was empty.
- The node had no taints (no "keep away" signs).

Total silence, even for clearly stuck pods, pointed below individual pods.

#### 6c. Read the service logs

Plain English: in MicroK8s, Kubernetes' brain (API server, scheduler, controllers, node agent) runs as one service called **kubelite**. **containerd** is what actually starts containers.

```bash
journalctl -u snap.microk8s.daemon-kubelite --no-pager -n 500 | grep -iE "error|fail" | tail -15
journalctl -u snap.microk8s.daemon-containerd --no-pager -n 500 | grep -iE "error|fail" | tail -15
```

**Real result, in time order:**
- **13:16–13:17 (containerd):** `plugin type="calico" failed (add): stat .../calico/nodename: no such file or directory: check that the calico/node container is running`. Pods asked for network addresses before Calico's agent had finished starting. That's normal start-up noise.
- **13:18:54–13:19:05 (containerd):** `context canceled` on running commands and on image downloads (CoreDNS, Calico controllers). Something stopped mid-way. **After this, containerd logged nothing more**: nothing was even trying to create containers.
- **13:31–13:32 (kubelite), every ~10 s:** `Error updating node status, will retry ... node "k8s1" not found`. The node agent was being told its own node didn't exist, while `kubectl get nodes` showed `k8s1` Ready.

**Interpretation:** around 13:19, MicroK8s' services ended up in an inconsistent state. The node agent couldn't see its node, the scheduler wasn't placing pods, and no events were recorded. **Root cause not determined.**

#### 6d. The fix: a clean restart

```bash
microk8s stop
microk8s start
microk8s status --wait-ready
microk8s kubectl get pods -A
```

**Real result:** all four pods moved to `Running` (`hello`, `calico-kube-controllers`, `calico-node` with 1 restart, and `coredns`). Kubernetes also finished the image downloads it had abandoned.

> If it happens again, read the kubelite log around the moment things stopped, before restarting.
> `journalctl -u snap.microk8s.daemon-kubelite --since "HH:MM" --until "HH:MM" --no-pager`

#### 6e. Visit the pod

```bash
microk8s kubectl get pods -o wide
curl -s 10.1.166.197 | head -5
```

**Real result:** `<title>Welcome to nginx!</title>`.

Note the address: **10.1.166.197** is not on `ovntest` (10.10.10.x). It's on Kubernetes' **own** pod network, run by Calico, inside `k8s1`. That gives three networks stacked: VMware's network, then OVN's `ovntest`, then Kubernetes' pod network.

> **Why a browser can't open it:** pod addresses only exist inside `k8s1`. Windows can reach uplink front doors, the cluster machines can reach `ovntest`, and only `k8s1` can reach its pods. Reaching a pod from outside needs a Kubernetes **Service**, plus a front door. That's INFRA-021.

### Step 7: Delete the bare pod

```bash
microk8s kubectl delete pod hello
microk8s kubectl get pods
```

**Real result:** `No resources found in default namespace`, and it stayed that way. A bare pod has no desired state, so no control loop is watching it.

### Step 8: Do it again with a Deployment

Plain English: a **Deployment** declares a desired state, "keep N copies running", and Kubernetes runs a control loop for it.

```bash
microk8s kubectl create deployment hello --image=nginx --replicas=2
microk8s kubectl get pods -o wide
microk8s kubectl delete pod hello-9ff96cc5-47r7s
microk8s kubectl get pods -o wide
```

**Real result:**
```
hello-9ff96cc5-6z4nq   0/1   ContainerCreating   0   7s    <none>         k8s1
hello-9ff96cc5-tj4z6   1/1   Running             0   74s   10.1.166.198   k8s1
...
hello-9ff96cc5-6z4nq   1/1   Running             0   12s   10.1.166.200   k8s1
hello-9ff96cc5-tj4z6   1/1   Running             0   79s   10.1.166.198   k8s1
```

- A replacement was created within **7 seconds** and was **Running by 12 seconds**.
- It's a **new pod**, with a new name and a new address (`.200`), not the old one restarted.
- The other copy (`tj4z6`) **kept running the whole time**.
- `9ff96cc5` in the names identifies the **ReplicaSet**. The Deployment creates a ReplicaSet, and the ReplicaSet is the control loop that keeps the count at 2.

Leave the shell with `exit`.

---

## Write-up

**What Kubernetes needed from LXD, and the cost:** the `microk8s` profile. It loads kernel features on the host, allows nesting, and switches off key security layers: privileged mode, all root powers, AppArmor off, all devices, and writable kernel settings. **Cost:** root inside `k8s1` is root on `mc3`, so a compromise of `k8s1` is a compromise of its host and everything on it. On a VM or bare metal, the profile isn't needed at all, because Kubernetes has its own kernel. A VM's hypervisor limits how far a break-in spreads; it doesn't prevent one.

**Bare pod vs. Deployment:** the bare pod had no desired state, so when it was deleted, nothing noticed. The Deployment declared `replicas: 2`, and its control loop (via the ReplicaSet) saw 1, compared it with 2, and created a new pod.

**Compared with the INFRA-019 watcher:**

| | INFRA-019 watcher | Kubernetes Deployment |
|---|---|---|
| Who wrote the loop | Me, 10 lines | Built in |
| On a loss | Restarts the *same* container | Creates a *new* pod |
| Service during recovery | Down (~14 s) | The other copy kept serving |

**Why this leads to Services:** a replaced pod gets a new address, so anything pointing at the old address breaks. Kubernetes' answer is a **Service**: one stable address in front of pods that come and go (INFRA-021).

**Lab-vs-reality gaps:**
- One Kubernetes node, inside a privileged container. Production runs several nodes, on VMs or bare metal.
- The 13:19 break was fixed by a restart, but not explained. In production, you'd want the root cause before trusting the system again.

---

## Definition of Done

- [x] Memory and storage checked (3,069 MB; 7.6 GiB)
- [x] Profile read; privileged trade-off explained in plain words
- [x] `k8s1` running with MicroK8s, Kubernetes v1.35.6, node Ready
- [x] Stuck pods diagnosed from logs; fixed with a clean restart; root cause left open
- [x] First pod answering at `10.1.166.197`
- [x] Bare pod deleted: not replaced
- [x] Deployment pod deleted: replaced in ~12 s, with a new name and address

**Open follow-ups (optional):**
- Read the kubelite log for 13:18–13:20 to find what stopped.
- Record the enabled add-ons with `microk8s status`.

---

## Retention Check

You deleted a bare pod and a pod belonging to a Deployment. Only one came back. Which one, and what was running in the background that made the difference?

> **Answer given:** the Deployment's pod came back, because the Deployment says `replicas: 2`, and Kubernetes watches that and reinstates it.
> **Refinement:** the "watcher" is a control loop run by the Deployment's ReplicaSet: it looks (1 pod), compares (want 2), and fixes (creates one). It creates a *new* pod, with a new name and address, rather than reviving the old one.