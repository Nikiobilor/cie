# INFRA-021: Services and Networking in Kubernetes: Getting a Pod to the Browser

**Priority:** P2 (Week 5, Day 23: Kubernetes Foundations)
**Component:** Kubernetes Services (ClusterIP, NodePort), cluster DNS, plus an OVN front door
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster, `k8s1` (MicroK8s, Kubernetes v1.35.6) on `mc3` at `10.10.10.2` on OVN network `ovntest`, Deployment `hello` (2 nginx pods)
**Linked tickets:** INFRA-020 (first pod; why its address changes). Sets up INFRA-022 (Day 24: persistent storage)

> **Note:** this ticket was rewritten after the session to record what actually happened. It includes a real problem: **MicroK8s didn't start after a reboot**, a side effect of running Kubernetes in a privileged container. It also corrects one step in the original plan.

---

## Summary

INFRA-020 left two problems: pods get a new address whenever they're replaced, and the browser couldn't reach them because they sit three networks deep. Today a Kubernetes **Service** gave the pods one stable address and name, a **NodePort** opened a door on `k8s1`, and an **OVN forward** on the uplink opened a front door to the outside.

**Result: the Windows browser opened a page served by a Kubernetes pod**, at `http://192.168.82.35`.

**The request path, from the browser to the pod:**

| Hop | What moves it to the next layer |
|---|---|
| Windows browser → `192.168.82.35:80` | VMware's NAT network |
| `192.168.82.35:80` → `10.10.10.2:31986` | OVN network forward (the front door on the uplink) |
| `k8s1:31986` → Service `hello` | NodePort |
| Service `hello` → one of the pods | The Service's live list of pods |

**Unplanned finding:** after the machines restarted, MicroK8s was completely down. Its AppArmor security rules weren't reloaded at boot inside the privileged container. A small start-up service now loads them, and that was proven with a real reboot.

---

## Acceptance Criteria

- [x] Each `hello` pod given its own page, so we can see which one answers
- [x] A **ClusterIP** Service created, with visitors shared between both pods
- [x] A pod deleted, with the Service's address shown to stay the same
- [x] The Service reached **by name** from another pod (cluster DNS)
- [x] A **NodePort** Service created (`hello-np`, port **31986**) and reached from inside `k8s1`
- [x] An OVN front door created, and the page opened in a **Windows browser**
- [x] Plain-English write-up of the path a browser request takes
- [x] **Added:** MicroK8s boot failure diagnosed and fixed permanently

> Steps 3–5 were completed, but their exact output wasn't captured in the session. On a repeat run, record the ClusterIP, the curl replies showing both pod names, the endpoint slices before and after the deletion, and the DNS reply.

---

## Comments Thread

**Reviewer:** Yesterday's replacement pod had a new address. If a website pointed at the old one, what happens?

**NwaChi:** It breaks. The address points at nothing.

**Reviewer:** So Kubernetes adds a layer that doesn't change: a Service. One fixed address and a name, a live list of the pods with the right label, and traffic spread across them.

**NwaChi:** `kubectl` says it can't connect to the server at all.

**Reviewer:** Then Kubernetes isn't running. Find out why before restarting anything.

**NwaChi:** containerd says it's missing its AppArmor profile, and `snapd.apparmor` says it's *inside container environment without internal policy*.

**Reviewer:** There it is. Our profile made this container's AppArmor `unconfined`, so snapd decided loading the rules wasn't its job. The first install worked because installing a snap loads its rules directly. A reboot is the first time snapd had to do it. That's a direct cost of the privileged-container workaround.

**NwaChi:** A pod behind a Service is deleted and replaced. The pod's address and name change, but the Service's address stays the same. Visitors don't notice because they reach the pods through the Service address.

**Reviewer:** Right. Add two things: the Service keeps a live list of its pods and updates it automatically, and with two replicas the other copy keeps answering while the replacement starts.

**NwaChi:** If `k8s1` is recreated with the same name, does it get the same address?

**Reviewer:** No. A pinned address is saved in that container's own configuration. Delete the container and the setting goes with it. A new container with the same name is brand new to OVN.

---

## Implementation Runbook

> **Standing rule:** check the real output at every step before moving on.
> **(k8s1)** = inside `lxc shell k8s1`. **(mc1)** = on the host `mc1`.

### Step 1: Check the starting point, and the problem that appeared

**(k8s1)**
```bash
microk8s kubectl get deployment hello
```

**Real result:**
```
The connection to the server 127.0.0.1:16443 was refused - did you specify the right host or port?
```
Port 16443 is Kubernetes' API server, which every `kubectl` command talks to. Nobody was answering.

#### 1a. Are the services running?

```bash
snap services microk8s
microk8s status --wait-ready --timeout 60
```

**Real result:** every MicroK8s service was `enabled` (set to start at boot) but `inactive`. `microk8s status` said `microk8s is not running`. `microk8s inspect` listed `FAIL` for containerd, kubelite, k8s-dqlite, cluster-agent and apiserver-kicker.

#### 1b. Did they fail, or never start?

```bash
systemctl status snap.microk8s.daemon-kubelite --no-pager | head -8
journalctl -u snap.microk8s.daemon-containerd -b --no-pager | tail -15
```

`-b` limits the log to the current boot.

**Real result:**
- kubelite: `failed (Result: exit-code)`, after 8 restart attempts.
- containerd, repeatedly:
  ```
  missing profile snap.microk8s.daemon-containerd.
  Please make sure that the snapd.apparmor service is enabled and started
  ...
  Start request repeated too quickly.
  ```

Kubernetes needs containerd, so kubelite failed too.

#### 1c. Why weren't the rules loaded?

Plain English: snap packages run under **AppArmor** rules, which list what each program may touch. They must be loaded into the kernel at every boot, by a service called `snapd.apparmor`.

```bash
systemctl status snapd.apparmor --no-pager | head -8
```

**Real result:** `active (exited)`, status `0/SUCCESS`, with the message:
```
Inside container environment without internal policy
```

It "succeeded" by deciding to do nothing. Our `microk8s` profile set this container's AppArmor to `unconfined`, so snapd saw no AppArmor setup of its own here to load rules into.

> The `Warning: The unit file ... changed on disk` line in these outputs is unrelated and harmless. The snap had updated itself, and `systemctl daemon-reload` clears it.

#### 1d. Load the rules by hand

```bash
apparmor_parser -r /var/lib/snapd/apparmor/profiles/snap.microk8s.*
systemctl daemon-reload
microk8s start
microk8s status --wait-ready --timeout 120
```

**Real result:** `microk8s is running`. Enabled add-ons: **dns** (CoreDNS), **ha-cluster**, **helm**, **helm3**. That closes an INFRA-020 follow-up.

#### 1e. Make it permanent

A small start-up service that loads MicroK8s' rules before containerd and kubelite start:

```bash
cat > /etc/systemd/system/microk8s-apparmor.service << 'UNIT'
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
UNIT

systemctl daemon-reload
systemctl enable microk8s-apparmor.service
```

**Prove it with a real reboot (mc1):**
```bash
lxc restart k8s1
sleep 60
lxc exec k8s1 -- microk8s status --wait-ready --timeout 60 | head -3
lxc exec k8s1 -- microk8s kubectl get pods -o wide -l app=hello
lxc list
```

**Real result:**
- `microk8s is running`, with no manual steps.
- `hello` was at 2/2. Both pods kept their names, but showed `RESTARTS 2` and **new addresses** (`10.1.166.207`, `.208`). Even the same pod can come back with a different address.
- `k8s1` is at **`10.10.10.2`** on `ovntest`. `lxc list` also shows `vxlan.calico` (`10.1.166.192`): Calico's tunnel interface, the doorway between `k8s1` and its pod network.

### Step 2: Give each pod its own page

**(k8s1)**
```bash
for p in $(microk8s kubectl get pods -l app=hello -o name); do
  microk8s kubectl exec $p -- sh -c 'echo "Hello from pod $(hostname)" > /usr/share/nginx/html/index.html'
done
```

### Step 3: Create a ClusterIP Service

**(k8s1)**
```bash
microk8s kubectl expose deployment hello --port=80
microk8s kubectl get service hello
for i in 1 2 3 4 5 6; do curl -s <CLUSTER-IP>; done
microk8s kubectl get endpointslices -l kubernetes.io/service-name=hello
```

`expose` creates a Service that finds the Deployment's pods by their label (`app=hello`). The `CLUSTER-IP` is the stable address. The endpoint slices are the Service's live list of pod addresses.

### Step 4: Delete a pod: does the Service care?

**(k8s1)**
```bash
microk8s kubectl delete pod <one hello pod name>
microk8s kubectl get endpointslices -l kubernetes.io/service-name=hello
microk8s kubectl get service hello
for i in 1 2 3 4 5 6; do curl -s <CLUSTER-IP>; done
```

The Service's address stays the same, and its list of pods updates. The replacement pod shows nginx's default page, because only the original pods were relabelled. That's one more sign that a replacement is a brand-new pod.

### Step 5: Reach it by name

**(k8s1)**
```bash
microk8s kubectl run tmp --rm -it --image=busybox --restart=Never -- wget -qO- http://hello
```

Kubernetes' own DNS (CoreDNS) turns the name `hello` into the Service's address.

### Step 6: Open a door on the node (NodePort)

**(k8s1)**
```bash
microk8s kubectl expose deployment hello --type=NodePort --name=hello-np --port=80
microk8s kubectl get service hello-np
curl -s http://10.10.10.2:31986
```

**Real result:**
```
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-np   NodePort   10.152.183.142  <none>        80:31986/TCP   93m
```
and `Hello from pod hello-9ff96cc5-6z4nq`.

> **Correction to the original plan:** the original step said to test the NodePort from `mc1`. That **doesn't work, and isn't meant to**. `mc1` isn't on `ovntest`; that network lives behind OVN's router. On `mc1`, `curl -s http://10.10.10.2:31986` returned nothing, and:
> ```
> $ ip route get 10.10.10.2
> 10.10.10.2 via 192.168.82.2 dev ens37 src 192.168.82.139
> ```
> `mc1` sends the request out to VMware's gateway, toward the internet, where it dies. It's the same lesson as the INFRA-015 false ping. From outside `ovntest`, the only way in is a front door on the uplink. (`-s` hides curl's error, which is why the reply was empty.)

### Step 7: Add a front door on the uplink, and open it in a browser

**(mc1)**
```bash
lxc network forward create ovntest 192.168.82.35
lxc network forward port add ovntest 192.168.82.35 tcp 80 10.10.10.2 31986
curl -s http://192.168.82.35
```

> **Gotcha:** the first attempt used `10.10.10.2:31986` and failed with `Error: Failed updating forward: Invalid target address in port specification 0`. The target address and port are **two separate arguments**: `10.10.10.2 31986`.

**Real result:** `Hello from pod hello-9ff96cc5-6z4nq` from `mc1`, and the page **opened in the Windows browser** at `http://192.168.82.35`.

#### Pin `k8s1`'s address (recommended)

The forward is hard-wired to `10.10.10.2`. OVN usually gives a container the same address again (INFRA-016), but nobody has *promised* it. To make it a guarantee:
```bash
lxc stop k8s1
lxc config device set k8s1 eth0 ipv4.address=10.10.10.2
lxc start k8s1
```

The pin is saved in **that container's own configuration**. If `k8s1` is deleted and recreated, the pin is gone, so set it at creation time:
```bash
lxc init ubuntu:24.04 k8s1 -p default -p microk8s --network ovntest --storage remote --target mc3
lxc config device set k8s1 eth0 ipv4.address=10.10.10.2
lxc start k8s1
```
(`lxc init` creates the container without starting it.)

---

## Write-up

**What a Service gives you:** one stable address and a name in front of pods that come and go, plus traffic spread across them. Pods change address when replaced, and even after a restart (Step 1e). The Service's address doesn't.

**When a pod is replaced:** its name and address change. The Service's address stays the same, and its live list of pods updates automatically. With two replicas, the other copy keeps answering. Visitors only ever use the Service (by address, by name, or through the NodePort and front door), so they don't notice.

**Three ways in:**
- **Inside the cluster:** the ClusterIP address, or the name `hello` via DNS.
- **From outside:** OVN forward → NodePort → Service → pod.

**Mapped to AWS:**

| In this lab | Closest AWS equivalent |
|---|---|
| `ovntest` | A private subnet in a VPC |
| `UPLINK` | Internet Gateway plus a public subnet |
| The OVN router's uplink address, for outbound access | NAT Gateway |
| The reserved uplink block (`192.168.82.32/28`) | Elastic IPs |
| Forward `192.168.82.35:80` → `k8s1:31986` | A Network Load Balancer: listener on port 80, target is the node's port 31986 |

On EKS, a `LoadBalancer`-type Service builds exactly this path automatically: an AWS load balancer whose targets are the NodePort on each worker node. Today it was built by hand.

**The cost of the privileged-container workaround, again:** with AppArmor unconfined, snapd skips loading MicroK8s' rules at boot, so Kubernetes fails to start after every reboot unless something else loads them. On a VM or bare metal, this problem doesn't exist.

**Lab-vs-reality gaps:**
- The front door points at **one** node's NodePort. If that node failed, the site would go dark even though Kubernetes moved the pods. **Planned for Week 6 (Day 27/28):** an OVN load balancer on the uplink, in front of every node's NodePort. It's the closest match to an AWS NLB in front of EKS nodes. Note that LXD 6.0's load balancer doesn't health-check its targets, which Day 28 will test.
- `k8s1`'s address is not yet pinned.

---

## Definition of Done

- [x] MicroK8s boot failure diagnosed (AppArmor rules not loaded in an unconfined container) and fixed with a start-up service, proven by a real reboot
- [x] Pods labelled; ClusterIP Service shared traffic across both
- [x] Service address stayed fixed through a pod replacement
- [x] Service reached by name from a pod
- [x] NodePort `31986` reached from inside `k8s1`; the plan's `mc1` test corrected, with routing evidence
- [x] Page opened in a Windows browser through `192.168.82.35`
- [x] Write-up completed, including the request path and the AWS mapping

**Open follow-ups:**
- Pin `k8s1`'s address.
- Capture Steps 3–5 output on a repeat run.

---

## Retention Check

A pod behind a Service is deleted and replaced. What changes, what stays the same, and why don't the visitors notice?