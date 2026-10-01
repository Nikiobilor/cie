# INFRA-020: Install MicroK8s and Deploy Your First Pod

**Priority:** P2 (Week 5, Day 22: Kubernetes Foundations)
**Component:** Kubernetes (MicroK8s) running inside an LXD container on the private cloud
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), Ceph pool `remote`, OVN network `ovntest` (MTU 1500)
**Linked tickets:** INFRA-019 (control loops). Sets up INFRA-021 (Day 23: Services and networking)

---

## Summary

Today Kubernetes goes onto the private cloud. We install **MicroK8s**, Canonical's compact Kubernetes, inside an LXD container on `mc3`. Then we run our first **pod**, Kubernetes' smallest unit: one or more containers that run together.

Then we repeat INFRA-019's crash test the Kubernetes way, and see the control loop doing the work.

**Why inside a container, and not a VM?** VMs can't run in this lab (INFRA-017: the Windows host blocks nested virtualization). Running Kubernetes inside LXD containers is also how Week 6 will build a multi-node cluster on this private cloud.

---

## Acceptance Criteria

- [ ] Enough memory and storage confirmed before starting
- [ ] The MicroK8s LXD profile downloaded, **read**, and its security trade-off understood
- [ ] Container `k8s1` running on `mc3` with that profile
- [ ] MicroK8s installed and reporting ready
- [ ] First pod running and answering
- [ ] A bare pod deleted: does it come back?
- [ ] The same done with a Deployment: does it come back?
- [ ] Plain-English write-up

---

## Comments Thread

**Reviewer:** Yesterday you wrote a ten-line control loop. Today you install the real thing. Where are you putting it?

**NwaChi:** Inside an LXD container, since VMs can't run here.

**Reviewer:** Good. But Kubernetes inside a container isn't a normal workload. Kubernetes itself starts containers, adjusts networking, and loads kernel features, so its container needs far more freedom than `web1` ever had. The MicroK8s docs give an LXD profile for this, and it makes the container **privileged**.

**NwaChi:** What does privileged mean?

**Reviewer:** Normally, "root" inside a container is mapped to a powerless user on the host, so a break-out does little damage. In a privileged container, root inside is root on the host. That's fine for a lab, and a serious decision in production. So read the profile before you apply it. Never apply security settings you haven't read.

**NwaChi:** And the first pod?

**Reviewer:** Run one, then delete it, and see whether it comes back. Then do the same with a Deployment. The difference is yesterday's lesson, built into Kubernetes.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.

### Step 1: Check there's room (`mc3`, then `mc1`)

Plain English: Kubernetes needs real memory, and its software and images take up disk space. Check both before you start.

On `mc3`:
```bash
free -m
```
On `mc1`:
```bash
sudo microceph.ceph df
```

**Record:** `available` memory on `mc3` (we want at least about 1.5 GB), and Ceph `MAX AVAIL` (we want at least about 3 GiB).

### Step 2: Get the MicroK8s profile, and read it (`mc1`)

```bash
wget https://raw.githubusercontent.com/canonical/microk8s/master/tests/lxc/microk8s.profile -O microk8s.profile
cat microk8s.profile
```

Plain English: this is the list of extra freedoms the container will get. Look for these lines, and understand them before continuing:
- `security.privileged: "true"`: root inside the container is root on the host.
- `security.nesting: "true"`: containers are allowed to run inside this container.
- `linux.kernel_modules`: kernel features loaded on the host for Kubernetes' networking.
- `lxc.apparmor.profile=unconfined`: turns off a security layer that would otherwise block Kubernetes.

There's a separate `microk8s-zfs.profile` for ZFS storage. Our storage is Ceph, so we use the plain one.

Create the profile:
```bash
lxc profile create microk8s
cat microk8s.profile | lxc profile edit microk8s
lxc profile show microk8s
```

### Step 3: Launch `k8s1` on `mc3`

```bash
lxc launch ubuntu:24.04 k8s1 -p default -p microk8s --network ovntest --storage remote --target mc3
lxc list k8s1
```

Expected: `k8s1` RUNNING, with an address in `10.10.10.x`.

### Step 4: Install MicroK8s

```bash
lxc exec k8s1 -- snap install microk8s --classic
lxc exec k8s1 -- microk8s status --wait-ready
```

`--wait-ready` waits until Kubernetes has fully started. It can take a few minutes.

**Record:** the MicroK8s version (`lxc exec k8s1 -- snap list microk8s`), and which add-ons show as enabled.

> If the install or startup fails, don't retry blindly. Paste the exact error. Kubernetes inside containers has known pitfalls (kernel modules, security layers), and each one has a specific fix.

### Step 5: Look around (inside `k8s1`)

Open a shell inside the container, so commands are shorter:
```bash
lxc shell k8s1
```

Then:
```bash
microk8s kubectl get nodes
microk8s kubectl get pods -A
```

Plain English: `kubectl` is the command for talking to Kubernetes. `get nodes` lists the machines in the cluster: just one here, `k8s1`. `get pods -A` lists every pod in every namespace (a namespace is a named group). Kubernetes runs its own helpers as pods too, such as networking (Calico) and name lookups (CoreDNS).

**Record:** the node's status, and which system pods are running.

### Step 6: Run your first pod

```bash
microk8s kubectl run hello --image=nginx
microk8s kubectl get pods -o wide
```

Wait until `STATUS` says `Running`. Kubernetes first has to download the nginx image. `-o wide` adds the pod's address.

Then visit it, using the pod's address from the output:
```bash
curl -s <pod IP> | head -5
```

Expected: the start of nginx's welcome page.

### Step 7: Delete the bare pod, and watch

```bash
microk8s kubectl delete pod hello
microk8s kubectl get pods
```

**Record:** does `hello` come back?

### Step 8: Do it again with a Deployment

Plain English: a **Deployment** is a desired state: "keep N copies of this pod running." Kubernetes runs a control loop for it, just like yesterday's watcher, but built in.

```bash
microk8s kubectl create deployment hello --image=nginx --replicas=2
microk8s kubectl get pods -o wide
```

Delete one of the two pods, by its full name from the output:
```bash
microk8s kubectl delete pod <one of the hello-xxxxx names>
microk8s kubectl get pods -o wide
```

**Record:**
- How many pods are there right after the delete?
- Is there a new name?
- How long until it's `Running`?

Leave the shell with `exit`.

### Step 9: Write up

From the real results:
- What did Kubernetes need from LXD to run, and what's the security cost?
- What happened to the bare pod versus the Deployment's pod, and why?
- How does the Deployment compare with your INFRA-019 watcher?

**Lab-vs-reality gap:** this is one Kubernetes node, inside a privileged container. Production runs several nodes, usually on VMs or bare metal, so the Kubernetes layer is isolated from the host. Week 6 adds more nodes.

---

## Definition of Done

- [ ] Memory and storage checked
- [ ] Profile read; privileged trade-off explained in plain words
- [ ] `k8s1` running with MicroK8s ready, version recorded
- [ ] First pod running and answering on its own address
- [ ] Bare pod deletion result recorded
- [ ] Deployment pod deletion result recorded
- [ ] Write-up completed

---

## Retention Check

You deleted a bare pod and a pod belonging to a Deployment. Only one came back. Which one, and what was running in the background that made the difference?