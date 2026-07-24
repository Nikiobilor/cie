# 50-Day Cloud Infrastructure Lab Challenge

A weekday-only, fundamentals-first lab series building the open-source virtualization, distributed storage, and networking skills behind the **Infrastructure Engineer → Canonical Junior CFE → Cloud Field Engineer → Cloud Solutions Architect** path.

MicroCloud, Ceph, and OpenStack aren't the goal here — they're how the underlying concepts (virtualization, clustering, distributed storage, SDN, high availability, migration) get taught hands-on. Those concepts transfer to Proxmox, VMware, bare OpenStack, or any public cloud's own internal design.

This is the fast, fundamentals-level companion track to the deeper [cloud-field-engineer-labs](https://github.com/Nikiobilor/cloud-field-engineer-labs) series — short daily reps here, longer client-simulation sessions there.

## Environment

- **VirtualBox** (Windows, already installed) — local VMs, cloned from a one-time base image
- **AWS Free Tier** — strictly capped to free-tier-eligible resources (t2/t3.micro), used for networking/IAM-flavored days
- No resources outside free tier, no new paid tooling

## Daily Format

Every day is one sitting, ~45–60 min:
1. **Why this matters** — plain-English, real-world tie-in
2. **The concept** — analogy first, technical term second
3. **Hands-on** — one task, one artifact
4. **Recap** — what you did and why
5. **Retention check** — one question, answered in your own words

## Progress Tracker

<details>
<summary><strong>Week 1 — Virtualization Foundations (Days 1–5)</strong></summary>

- [x] Day 1: VM vs. container — why LXD is denser than VMware
- [x] Day 2: Install LXD, launch first container, compare footprint to Day 1
- [x] Day 3: LXD networking basics (bridges, profiles)
- [ ] Day 4: LXD storage pools — how a container's disk actually works
- [ ] Day 5: Recap + build: multiple containers on one LXD host

</details>

<details>
<summary><strong>Week 2 — Clustering Foundations (Days 6–10)</strong></summary>

- [ ] Day 6: What clustering and quorum actually mean, plain English
- [ ] Day 7: LXD clustering — join two nodes
- [ ] Day 8: Networking VirtualBox nodes together (Host-only adapter)
- [ ] Day 9: MicroCloud overview — what it bundles (LXD + MicroCeph + MicroOVN) and why
- [ ] Day 10: Recap + build: 2-node LXD cluster

</details>

<details>
<summary><strong>Week 3 — MicroCloud Compute & Storage (Days 11–15)</strong></summary>

- [ ] Day 11: `microcloud init` walkthrough
- [ ] Day 12: MicroCeph fundamentals — what distributed storage means
- [ ] Day 13: CRUSH maps — where your data actually lives
- [ ] Day 14: Simulate node failure and recovery
- [ ] Day 15: Recap + build: resilient storage demo on the cluster

</details>

<details>
<summary><strong>Week 4 — MicroCloud Networking & Density (Days 16–20)</strong></summary>

- [ ] Day 16: MicroOVN / software-defined networking fundamentals
- [ ] Day 17: Network topology choices — why not just flat networking
- [ ] Day 18: Live migration of an instance between nodes
- [ ] Day 19: Density comparison — Day 1's VM footprint vs. LXD at scale
- [ ] Day 20: **Portfolio artifact — Project 1: Build Your Own Private Cloud**

</details>

<details>
<summary><strong>Week 5 — Kubernetes Foundations (Days 21–25)</strong></summary>

- [ ] Day 21: What container orchestration solves, and why Kubernetes
- [ ] Day 22: Install MicroK8s, deploy first pod
- [ ] Day 23: Services and networking in Kubernetes
- [ ] Day 24: Persistent storage in Kubernetes (ties back to Ceph)
- [ ] Day 25: Recap + build: simple app on MicroK8s

</details>

<details>
<summary><strong>Week 6 — Kubernetes High Availability (Days 26–30)</strong></summary>

- [ ] Day 26: What high availability means for a control plane
- [ ] Day 27: Multi-node MicroK8s cluster, running on your Week 3–4 private cloud
- [ ] Day 28: Simulate a node failure in the Kubernetes cluster
- [ ] Day 29: Monitoring fundamentals — Prometheus/Grafana concepts
- [ ] Day 30: **Portfolio artifact — Project 3: HA Kubernetes on Your Private Cloud**

</details>

<details>
<summary><strong>Week 7 — OpenStack/Sunbeam Foundations (Days 31–35)</strong></summary>

*Note: nested virtualization isn't available on free-tier resources — expect QEMU-emulated (slower) performance this week. Designed around correctness, not speed.*

- [ ] Day 31: What OpenStack solves and why enterprises still run it
- [ ] Day 32: Sunbeam install walkthrough
- [ ] Day 33: Compute (Nova) fundamentals
- [ ] Day 34: Networking (Neutron) fundamentals
- [ ] Day 35: Recap + build: first OpenStack instance

</details>

<details>
<summary><strong>Week 8 — OpenStack Storage & Migration (Days 36–40)</strong></summary>

- [ ] Day 36: Storage (Cinder) fundamentals, Ceph backend
- [ ] Day 37: Image management (Glance) — exporting/importing VM images
- [ ] Day 38: VMware-to-OpenStack migration concepts
- [ ] Day 39: Networking considerations during migration
- [ ] Day 40: **Portfolio artifact — Project 2: VMware to OpenStack Migration**

</details>

<details>
<summary><strong>Week 9 — Automation & Observability (Days 41–45)</strong></summary>

- [ ] Day 41: Terraform fundamentals, applied to this stack
- [ ] Day 42: Ansible fundamentals, applied to this stack
- [ ] Day 43: **Portfolio artifact — Project 4: one-command environment build**
- [ ] Day 44: Monitoring/alerting build (health checks + Slack)
- [ ] Day 45: Backup and disaster recovery fundamentals (Ceph snapshots)

</details>

<details>
<summary><strong>Week 10 — Capstone (Days 46–50)</strong></summary>

- [ ] Day 46: Design day — define requirements for a simulated 500-employee org
- [ ] Day 47: Build — compute, storage, networking foundation
- [ ] Day 48: Build — Kubernetes and application layer
- [ ] Day 49: Validate — failover testing, backup/restore test
- [ ] Day 50: **Portfolio artifact — Project 5: consultant-style case study writeup**

</details>

## Portfolio Projects

| Project | Days | What it demonstrates |
|---|---|---|
| 1. Build Your Own Private Cloud | 1–20 | Virtualization, clustering, distributed storage, SDN |
| 3. HA Kubernetes on Private Cloud | 21–30 | Orchestration, high availability |
| 2. VMware → OpenStack Migration | 31–40 | Migration planning, storage/network cutover |
| 4. One-Command Automated Build | 41–43 | Terraform + Ansible, infrastructure as code |
| 5. Simulated Enterprise | 46–50 | End-to-end design, architectural thinking, case-study writeup |
