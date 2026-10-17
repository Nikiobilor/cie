# INFRA-034: Capstone: Incident Portfolio and OpenStack Map

**Priority:** P2 (Plan v3, Day 10: capstone)
**Status:** Done (2026-10-17)
**Component:** Communication, OpenStack concepts
**Environment:** The whole lab: 3-node Canonical MicroCloud (LXD, MicroCeph, MicroOVN) on VMware Workstation Pro, Ubuntu Server, MicroK8s in an LXD container
**Linked tickets:** INFRA-015 to INFRA-033

> This ticket is the front page of the lab. It can be used as the repository's `README.md`.

---

## Summary

Over the challenge, a 3-node private cloud was built, broken (by accident and on purpose), diagnosed and repaired, and every session was written up as a ticket with real outputs. This page collects:
1. **The incident portfolio:** the best incidents, one card each.
2. **The skills map:** which tools and skills each ticket proves.
3. **The OpenStack map:** what each OpenStack service does, and the closest thing built in this lab.
4. **Honest gaps:** what wasn't done, and why.

## Why

A hiring manager won't read 20 tickets. One page should show, in two minutes, that this person can find a root cause with evidence, fix it safely, and explain it clearly.

---

## 1. The lab

| Layer | What runs it | Detail |
|---|---|---|
| Hosts | 3 VMware VMs: `mc1`, `mc2`, `mc3` | Ubuntu Server, chrony time sync, one laptop with 32 GB RAM |
| Compute | **LXD** (clustered) | Containers placed across nodes; profiles; `local` and `remote` storage pools |
| Storage | **MicroCeph** | 3 OSDs × 10 GB, 3 monitors, every pool keeps 3 copies (`size 3`, `min_size 2`) |
| Networking | **MicroOVN** | Private network `ovntest` behind an OVN router, `UPLINK` to the outside, a network forward as the public "front door" |
| Kubernetes | **MicroK8s** v1.35.6 in LXD container `k8s1` | `hello` (nginx, 2 replicas) behind a NodePort, reachable from a browser through the front door |

**The request path, browser to pod:** browser → front door `192.168.82.35:80` (OVN network forward) → `k8s1:31986` (NodePort) → Service `hello` → one of the nginx pods.

---

## 2. Incident portfolio

### INFRA-023 (P1): Power loss corrupted the whole cluster's state
- **Symptom:** after the laptop slept and lost power, all 3 Ceph monitors crashed in a loop; no storage, no container could start. Later: "Storage pool is unavailable", then OVN "connection refused".
- **Evidence:** monitor crash reasons on each node (`ceph_assert(is_consistent())`, `std::invalid_argument`); LXD log `Placeholder volume does not exist`; zero-byte and corrupt MicroOVN state files on `mc2`, last written at the moment of the power loss.
- **Decisions:** backed up monitor databases first. Rejected OSD-based recovery because the only repair tools available were for a newer Ceph major version, which risked making the damage unrecoverable. Rebuilt Ceph, since everything on it could be rebuilt from the tickets.
- **Fixes:** Ceph rebuilt (`HEALTH_OK`); LXD's missing `lxd_remote` marker recreated; `mc2`'s OVN files restored from a healthy peer (`mc3`), copying only the shared parts.
- **Lesson:** **quorum isn't a backup.** 2-of-3 survives one failure, not all three dying mid-write. Then **INFRA-024** rebuilt Kubernetes and the app from the tickets alone, and the front door worked again unchanged.

### INFRA-022: "The cluster wasn't broken, it was starved"
- **Symptom:** a Kubernetes add-on stuck in `ContainerCreating` with **no events**; new Deployments created nothing; `kubectl get pods` took 8.2 s.
- **Evidence:** kubelite log `Failed to renew lease ... context deadline exceeded` → `leaderelection lost` → restart; `top` showed CPU idle at 2% with no disk wait; `nproc` = **2**.
- **Root cause:** one node had 2 vCPUs for Ceph, OVN, LXD **and** Kubernetes.
- **Fix:** 4 vCPUs. `kubectl get pods` went from 8.2 s to **0.34 s**.
- **Also found:** Kubernetes' database was too slow on network storage (moved to local disk), and Ceph block volumes can't be mounted inside an LXD container, since containers share the host's kernel. Recorded as a known limit.

### INFRA-026: The front door pointed one digit off
- **Symptom:** site refused in **5 ms**.
- **Evidence:** LXD forward config and OVN's own load balancer (`ovn-nbctl lb-list`) both said port `31987`; `tcpdump` on `k8s1` showed the SYN arrive and a reset (`[R.]`) sent back; the app answered fine on `31986` directly.
- **Fix:** forward re-pointed to `31986`.
- **Lessons:** a refusal in milliseconds means "reached a machine, nothing listening"; a timeout means "blocked or lost". And `0 packets` once meant the capture started too late, not that nothing arrived.

### INFRA-027: One of three storage disks died, and nothing went down
- **Drill:** `noout` set, the OSD on `mc2` stopped.
- **Evidence:** `324/972 objects degraded (33.333%)`, exactly one copy of everything; placement groups `active+undersized+degraded`; app and containers kept working.
- **Recovery:** OSD restarted; Ceph caught up by itself in under a minute; `HEALTH_OK`.
- **Lesson:** **`active` is the word that matters.** It means reads and writes still work.

### INFRA-028: The clock was wrong for minutes before anything noticed
- **Drill:** chrony stopped on `mc2`, clock pushed 1 second ahead.
- **Evidence:** no warning at first (monitors only compare clocks every 5 minutes); after a forced check, `MON_CLOCK_SKEW` 0.98 s > 0.05 s. On `mc2`, `chronyc tracking` → `506 Cannot talk to daemon`.
- **Catch:** after restarting chrony, `makestep` said `200 OK` but nothing was fixed yet (`synchronized: no`, Ceph still measured 0.98 s).
- **Lesson:** **`200 OK` means "command accepted", not "problem solved".** Check the result.

### INFRA-029: A container wouldn't start, and its own config looked clean
- **Evidence:** start error `Missing source path "/srv/drill-data"`; the container's log was empty (it never ran); its own config had no such device; **`lxc config show --expanded`** revealed it, inherited from a profile.
- **Fix:** removed the device from the profile.
- **Also found:** the `default` profile has no root disk or network card, and LXD refuses a bad path at the moment a profile is attached.

### INFRA-030: Pods crashing, and a pod that ran but got no traffic
- **CrashLoopBackOff:** `logs --previous` showed `config.yaml not found`, `Exit Code: 1`; fixed by mounting a ConfigMap.
- **Readiness probe:** pod `Running` but `0/1` ready, events said `Readiness probe failed ... 404`, and the Service had **no endpoints**; fixed the probe path.
- **Lesson:** **`Running` isn't the same as working.** Check `READY` and the endpoints.

### INFRA-031: A service that failed silently
- **Symptom:** `status=1/FAILURE` after 29 ms; `journalctl` had only systemd's own lines.
- **Evidence:** `strace -u nobody` showed `openat(".../notes.conf") = -1 EACCES (Permission denied)`; `namei -l` showed the file was root-only. The script discarded its own errors.
- **Fix:** least privilege (group read only).
- **Lesson:** **when the logs are silent, ask the kernel.**

### INFRA-032: A tool gave the wrong answer, with no error
- **Evidence:** `pdb` with a conditional breakpoint showed usage `'100'` was **text**, compared letter by letter with `'80'`. `git bisect run` named the bad commit in 3 tests.
- **Fix:** `git revert`.
- **Honest note:** the manual bisect went wrong (results weren't fed back to git); recorded as a lesson.

### INFRA-033: A C program crashed
- **Evidence:** core file read with `gdb`. Frame `#0` was in `strlen` (C library), but frame `#1` showed our code passing `name=0x0`.
- **Lesson:** **frame `#0` is where it crashed, not where the bug is.**
- **Also:** read chrony's Ubuntu packaging (changelog, `debian/patches`, DEP-3 headers). Ubuntu's own change keeps chrony serving time inside containers.

---

## 3. Skills map

| Area | Tools used for real | Tickets |
|---|---|---|
| Linux systems | systemd, journalctl, netplan, chrony, AppArmor, file permissions, `namei` | 018, 021, 028, 031 |
| Networking | OVN networks, router, network forward, MTU, NAT, `tcpdump`, `ovn-nbctl` | 015, 018, 021, 025, 026 |
| Storage | Ceph monitors, OSDs, PGs, pools, `noout`, RBD, LXD storage pools | 017, 022, 023, 027 |
| Containers | LXD clustering, profiles, devices, `--expanded`, privileged containers | 016, 019, 024, 029 |
| Kubernetes | Deployments, Services, NodePort, ConfigMaps, PVCs, probes, `logs --previous`, events | 020, 021, 022, 024, 030 |
| Low-level debugging | `strace`, `gdb`, core dumps, `pdb` | 031, 032, 033 |
| Change tracking | git history, `git bisect`, `git revert`, Debian patches | 032, 033 |
| Incident handling | Evidence first, backups before changes, safe-recovery decisions, rebuild from runbooks | 022, 023, 024 |

---

## 4. OpenStack map

OpenStack is a set of services that together run a private cloud. Canonical's OpenStack (deployed with **Sunbeam**) uses **Ceph** for storage and **OVN** for networking, the same building blocks as this lab. Here is each main service next to its closest equivalent in this lab.

| OpenStack service | What it does | Closest thing in this lab | How close |
|---|---|---|---|
| **Nova** | Runs virtual machines | LXD instances (containers here; LXD can also run VMs) | Close in role; this lab ran containers, not VMs |
| **Neutron** (with OVN) | Virtual networks, routers, IP addresses | MicroOVN: `ovntest`, its router, `UPLINK` | Very close: same OVN underneath |
| Neutron **floating IP** | A public address mapped to a private instance | LXD **network forward** `192.168.82.35` → `10.10.10.2` | Close; the forward maps chosen ports, a floating IP maps the whole address |
| Neutron **security groups** | Firewall rules per instance | LXD network ACLs (OVN ACLs) | Not built |
| **Cinder** | Block storage volumes | Ceph RBD pool `remote` | Very close: Cinder commonly uses Ceph RBD |
| **Glance** | Stores machine images | LXD image store (`ubuntu:24.04` images) | Close in role |
| **Swift** / Ceph RGW | Object storage (like S3) | Nothing | Not built |
| **Keystone** | Users, projects, logins, permissions | LXD's trust certificates | Far: no multi-user identity here |
| **Placement / scheduler** | Picks which server runs an instance | LXD cluster placement (`--target`, or automatic) | Close in role |
| **Horizon** | Web dashboard | LXD has a web UI (not used) | Not used |
| **Octavia** | Load balancers | OVN load balancer (built automatically under the network forward) | Partly: seen in `ovn-nbctl lb-list`, not managed directly |
| **Heat** | Builds whole environments from templates | Terraform (known from AWS work, not used here) | Not built |
| **Magnum** | Runs Kubernetes clusters | MicroK8s in an LXD container | Partly: one manual cluster |
| **Telemetry** | Usage and health metrics | Ceph's Prometheus module | Partly |

**AWS view, for readers who know AWS:** Nova ≈ EC2, Neutron ≈ VPC, floating IP ≈ Elastic IP, Cinder ≈ EBS, Glance ≈ AMIs, Swift ≈ S3, Keystone ≈ IAM, Octavia ≈ ELB, Heat ≈ CloudFormation.

**Why OpenStack wasn't installed:** Sunbeam needs at least 4 cores, 16 GiB RAM and 100 GiB of SSD for a single node, a long install, and nested virtualisation, which this laptop lab couldn't provide alongside the existing cluster. Source: https://canonical.com/openstack/docs/latest/reference/enterprise-requirements/

---

## 5. Honest gaps and next steps

- **No OpenStack hands-on.** Concepts mapped above; a Sunbeam install needs a bigger machine.
- **No virtual machines.** LXD VMs were blocked in this nested setup, so KVM was never exercised.
- **Single-node Kubernetes.** No high availability yet, and no load balancer in front of multiple nodes.
- **Ceph volumes inside Kubernetes** couldn't be mounted (container kernel limit). The test needs Kubernetes on VMs.
- **Open lab follow-ups:** switch the VMs' network cards from `e1000` to `vmxnet3` (two hangs so far); give `default` a root disk and network card, or make `--storage`/`--network` the house rule; finish INFRA-025's real outputs.
- **Deeper code debugging** (Go, C in real projects) and **building** a Debian package are still ahead.

---

## Ticket index

| Ticket | Topic |
|---|---|
| INFRA-015 to INFRA-019 | MicroCloud networking, containers, density, MTU, front doors |
| INFRA-020, INFRA-021 | MicroK8s, Services, NodePort, front door, AppArmor boot fix |
| INFRA-022 | Persistent storage, CPU starvation, Ceph limits |
| INFRA-023, INFRA-024 | Power-loss incident and rebuild from runbooks |
| INFRA-025 | Following one request with tcpdump (real outputs pending) |
| INFRA-026 to INFRA-033 | The troubleshooting drills (cards above) |
| INFRA-034 | This page |

---

## Definition of Done

- [x] Incident portfolio written from real ticket outputs
- [x] Skills map linked to tickets
- [x] OpenStack map, with an AWS view and honest "how close" ratings
- [x] Gaps and next steps listed
- [ ] Published as the GitHub repository's `README.md`

## Retention Check

An interviewer asks: "Your lab has no OpenStack. What do you actually know about running it?" Give a two-sentence answer using this page.

**Suggested answer:** "I've run the same building blocks Canonical OpenStack is built on: OVN for networking, which is what Neutron uses, and Ceph for block storage, which is what Cinder commonly uses. I've broken and repaired both, including a full Ceph rebuild after a power-loss incident. I haven't installed the OpenStack services on top yet; the hardware requirements were beyond my laptop lab, and that's my next step."