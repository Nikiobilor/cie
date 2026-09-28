# INFRA-017: How Many Workloads Fit on One Node? Containers vs. Virtual Machines

**Priority:** P2 (Week 4, Day 19)
**Component:** Compute density (LXD system containers vs. LXD virtual machines)
**Environment:** VMware Workstation Pro on Windows, 3-node MicroCloud cluster (`mc1`, `mc2`, `mc3`), Ceph pool `remote`, OVN network `ovntest`. Test node: `mc3` (2 vCPU, 5.3 GB RAM)
**Linked tickets:** INFRA-016 (network identity on move). Sets up INFRA-018 (Day 20: Project 1 capstone, "Build Your Own Private Cloud")

> **Note:** this ticket was rewritten after the session to record what actually happened. The VM half was **blocked by the Windows host**, and that is documented as a finding. The container half was fully measured.

---

## Summary

LXD runs two kinds of workload: **system containers**, which share the host's Linux kernel, and **virtual machines (VMs)**, which boot their own kernel on emulated hardware. The goal was to turn "containers are lighter" into measured numbers.

**Results:**

| Measurement | Result |
|---|---|
| Can the nodes run LXD VMs? | **No.** The Windows host can't provide nested virtualization |
| Host memory cost per idle container | **~150 MB** (5 containers: −735 MB `available`) |
| Memory each container *reports* | 165–325 MiB (includes reclaimable cache, so it overstates the real cost) |
| Storage per idle container | **~140 MiB** (copy-on-write clone of the image) |
| One-time image cost on Ceph | **1.2 GiB** per image version |
| Container start to ready | **2.8 s** |
| Container stop | **4.0 s** |
| Platform's own memory footprint on `mc3` | **~1.65 GB**, about a third of the node's RAM |
| Rough idle capacity left on `mc3` | ~14 more containers, limited by memory |

---

## Acceptance Criteria

- [x] Confirmed whether the nodes can run LXD VMs (nested virtualization check), with the real result recorded
- [x] Baseline memory, CPU and storage recorded for the test node
- [x] Memory cost of 5 idle containers measured on one node
- [ ] ~~Memory cost of 1 idle VM measured~~ **Blocked**: the host doesn't support nested virtualization (see Step 1)
- [x] Container start time measured with a readiness check (VM comparison blocked)
- [x] Ceph storage growth recorded and broken down per disk
- [x] Write-up: measured density, platform overhead, and when you'd still choose a VM
- [x] Test instances cleaned up

---

## Comments Thread

**Reviewer:** A client asks: "how many workloads can this cluster hold?" What do you tell them?

**NwaChi:** Containers are lighter than VMs, so a lot more if we use containers.

**Reviewer:** "A lot more" isn't an answer you can put in a proposal. Measure it. And there's a catch: your nodes are themselves VMs inside VMware. Running an LXD VM inside them needs nested virtualization, which isn't always available. Check before you assume.

**NwaChi:** The check failed. No CPU virtualization flags, no `/dev/kvm`. And when I enabled VT-x in VMware, it said it isn't supported on this platform.

**Reviewer:** Then that's your first finding: the host OS blocks it, not LXD. Record it and measure the container side properly.

**NwaChi:** The containers report 200-ish MiB each, but the host only lost about 150 MB per container.

**Reviewer:** Good catch. Which number would you put in a capacity plan?

**NwaChi:** The host's. The per-container number includes cache the host can take back.

**Reviewer:** Right. And what else eats the node's memory?

**NwaChi:** The platform itself. Ceph, LXD, MicroOVN and MicroCloud hold about 1.65 GB on a 5.3 GB node before any workload runs.

**Reviewer:** That's the part people forget. "Workloads per node" always starts with "after the platform takes its share."

---

## Implementation Runbook

> **Standing rule:** record the real output at every step. **Stop launching if the node's `available` memory drops below about 500 MB**, because Ceph and the cluster services need headroom.

### Step 1: Can this cluster run VMs? (`mc3`)

Plain English: an LXD VM needs KVM (Kernel-based Virtual Machine), the Linux feature that uses the CPU's hardware virtualization. Inside a VMware VM, that only works if VMware passes it through. This is called nested virtualization.

```bash
grep -Ec '(vmx|svm)' /proc/cpuinfo
ls -l /dev/kvm
lxc info | grep -A3 "driver: "
```

**Real result:**
```
0
ls: cannot access '/dev/kvm': No such file or directory
  driver: lxc
  driver_version: 6.0.6
  instance_types:
  - container
```

No virtualization CPU flags, no `/dev/kvm`, and LXD lists only the `lxc` driver and the `container` instance type. **VMs can't run.**

> Tip: `lxc info | grep -A3 driver` also matches every API extension with "driver" in its name. Grep for `"driver: "` (with the colon and space) to get only the relevant block.

#### Attempted fix: enable nested virtualization in VMware

Protect Ceph before taking a node down (`mc1`):
```bash
sudo microceph.ceph osd set noout
```
Then shut down `mc3` (`sudo shutdown -h now`), and in VMware open **VM → Settings → Processors** and tick **"Virtualize Intel VT-x/EPT or AMD-V/RVI"**.

**Real result:** VMware warned:
> Virtualized Intel VT-x/EPT is not supported on this platform. Continue without virtualized Intel VT-x/EPT?

After clicking **Yes**, the VM **refused to start**:
> Feature 'hv.capable' was 0, but must be at least 0x1. Module 'FeatureCompatLate' power on failed.

**Why:** this most commonly happens when Windows is running Microsoft's own hypervisor underneath, which features such as WSL2, Hyper-V, or Memory Integrity use. VMware then runs on top of that hypervisor and can't pass hardware virtualization down to its guests. Enabling it would mean turning Microsoft's hypervisor off on the host, which would also disable WSL2. That trade-off was declined for this lab.

**Recovery:** with `mc3` powered off, **untick** the VT-x option and power on. If VMware still refuses, edit `mc3`'s `.vmx` file and set `vhv.enable = "FALSE"`. Then, on `mc1`:
```bash
sudo microceph.ceph osd unset noout
sudo microceph.ceph -s
```

#### Side finding: `BLUESTORE_SLOW_OP_ALERT`

After `mc3` returned, Ceph showed `HEALTH_WARN: 1 OSD(s) experiencing slow operations in BlueStore`. BlueStore is the engine each OSD uses to write data to its disk. All PGs stayed `active+clean`, so no data was at risk.

Investigation, step by step:
```bash
sudo microceph.ceph health detail     # -> osd.3
sudo microceph.ceph osd tree          # -> osd.3 is on host mc2
sudo zgrep -h "slow operation observed" /var/snap/microceph/common/logs/ceph-osd.3.log* | tail -10
uptime -s                             # on each node
```

Notes for replicators:
- MicroCeph writes daemon logs to `/var/snap/microceph/common/logs/`, not the system journal, so `journalctl` came back empty.
- `grep -i slow` on the log only matched harmless RocksDB statistics lines ("slowdown" with every counter at `0`). The specific phrase is `slow operation observed`.
- Use `zgrep` to include rotated `.gz` logs.

**Real result:** `osd.3` recorded write commits (`_txc_committed_kv`) taking 5 to 6 seconds on **08-19**, **09-24** (the Phase 2 reboot day in INFRA-015) and **today at 09:58 UTC**. Normally these take milliseconds.

Two hypotheses were tested and **ruled out** using boot times:
- `mc3` rejoining: it booted at 11:48, about two hours *after* the stall.
- `mc1`/`mc2` booting: they booted at 03:40 and 03:43, about six hours *before* it.

**Cause: not determined.** The leading suspect is the Windows host briefly starving the virtual disk, but that's unproven. By default the alert fires on a single slow operation and stays up for 24 hours, so it clears on its own if the stall doesn't recur. Don't raise the thresholds just to silence it.

### Step 2: Baseline the test node (`mc3`)

```bash
nproc
free -m
lxc list --format csv -c nL | grep mc3
sudo microceph.ceph df
```

Plain English: in `free -m`, track the **`available`** column, not `free`. Linux lends spare memory to disk cache and gives it back on demand, so `available` is the honest "what's left" number.

**Real result:**
- 2 CPUs, 5360 MB total, **3324 MB available**
- Already running: `ovn-c`
- Ceph `remote` pool: **2.7 GiB STORED**, 8.0 GiB USED. USED is about 3× STORED because of 3 replicas. The pool list also confirms the PG count: `remote` has 32 and `.mgr` has 1.

### Step 3: Launch 5 containers on `mc3`

```bash
for i in 1 2 3 4 5; do
  lxc launch ubuntu:24.04 dens-c$i --storage remote --network ovntest --target mc3
done
```

> **Don't press `Ctrl+C` during a launch.** In the original run it was pressed during the slow first launch. The client printed `Remote operation canceled by user` and then `This operation can't be cancelled (interrupt two more times to force)`. Forcing it can leave half-created storage behind. `lxc list dens-` showed that all five were created anyway: the cancel only stopped the client from *waiting*.

Measure after about 60 seconds:
```bash
lxc list dens- -c nsmL
free -m
sudo microceph.ceph df
```

**Real result:**

| | Baseline | After 5 containers | Change |
|---|---|---|---|
| `available` memory | 3324 MB | 2589 MB | **−735 MB (~147 MB each)** |
| Memory reported per container | – | 165–325 MiB (≈1,144 MiB total) | – |
| Ceph `remote` STORED | 2.7 GiB | 4.4 GiB | +1.7 GiB |

**Two numbers that disagree, and why:** the per-container memory figure comes from the kernel's cgroup accounting and includes disk cache the container has touched. The host counts that cache as reclaimable. **The drop in the host's `available` memory is the honest cost.**

### Step 3b: Break down the storage growth

The +1.7 GiB looked too large for copy-on-write clones, so check it rather than assume:
```bash
lxc storage volume list remote type=image
lxc image list
sudo microceph.rbd du -p remote
```

**Real result:**
- `lxc image list` showed a new image, **`ubuntu 24.04 LTS (20260926)`, uploaded today at 12:24 UTC**. Canonical had published a newer build, and LXD fetched it at launch instead of reusing the Sept 11 image. That also explains why the first launch was slow.
- `rbd du` (disk usage per RBD image; RBD stands for RADOS Block Device, a Ceph disk) showed:

| Disk | USED |
|---|---|
| New image snapshot (`image_ee3016…@readonly`) | **1.2 GiB** |
| Each `container_dens-cN` | **140–144 MiB** |

So today's growth was about 1.2 GiB for the image, plus about 0.7 GiB for five containers. **An idle container costs about 140 MiB of storage**, because it's a copy-on-write clone of the image snapshot and only stores the blocks it changes.

Also visible in `rbd du`:
- **Thin provisioning:** every disk is `PROVISIONED 10 GiB`, which adds up to **130 GiB promised on 30 GiB of raw storage**. This works only while disks stay mostly empty. In production, it has to be monitored.
- **Old images stay:** the Sept 11 image (1.2 GiB) is still on the pool, because older containers are clones of it.
- **`container_flat-a` uses 932 MiB**, about four times its siblings. Something inside it wrote a lot. It's an open item, not investigated.
- `fast-diff map is not enabled` warnings are harmless. `rbd du` just has to scan each disk instead of using a shortcut index.

### Step 4: VM measurement

**Blocked** (see Step 1). No VM memory, storage or start-time figures were taken.

### Step 5: Start and stop times

Plain English: `lxc start` returns when the workload is *launched*, not when it's *ready*. The loop below retries a command inside the container until it succeeds.

```bash
lxc stop dens-c1
time (lxc start dens-c1 && until lxc exec dens-c1 -- true 2>/dev/null; do sleep 1; done)
time lxc stop dens-c2
```

**Real result:** start to ready **2.8 s**, stop **4.0 s**.

**Correction to an INFRA-016 hypothesis:** stop plus start takes only about 7 s, so it doesn't explain INFRA-016's ~43-second outage. In INFRA-016, `stop`, `move` and `start` were typed as three separate commands, so human typing gaps were counted in the outage. The move and network recovery time were also never measured on their own. To measure the machine-only cost, run the move as one command while pinging:
```bash
time (lxc stop ovn-a && lxc move ovn-a --target mc1 && lxc start ovn-a)
```
(Not run in this session.)

### Step 6: Clean up

```bash
lxc delete dens-c1 dens-c2 dens-c3 dens-c4 dens-c5 --force
free -m
sudo microceph.ceph df
```

**Real result:**
- Ceph `remote` STORED went back down to **3.8 GiB**. The clones are gone, and the new image (1.2 GiB) stays.
- `available` memory recovered to **2883 MB**, not the 3324 MB baseline. About **440 MB is still held.**

### Step 6b: What is the platform using?

```bash
ps -eo rss,comm --sort=-rss | head -8
```

**Real result:** RSS (Resident Set Size, the memory a process actually holds):

| Process | Memory |
|---|---|
| `ceph-osd` | ~555 MB |
| `ceph-mgr` (standby on this node) | ~456 MB |
| `microcloudd` | ~139 MB |
| `microcephd` | ~135 MB |
| `lxd` | ~132 MB |
| `microovnd` | ~127 MB |
| `ceph-mon` | ~106 MB |
| **Total** | **~1.65 GB** |

The leftover ~440 MB most plausibly sits in `ceph-osd`, which caches recently used data up to a memory target. This is consistent with it being the largest process, but **unproven**, since there was no per-process baseline. To prove it, capture this same `ps` output *before* the test on a repeat run.

---

## Write-up

**Density, measured:** on `mc3`, an idle Ubuntu container costs the host about **150 MB of memory** and **140 MiB of storage**, and is ready **2.8 seconds** after start. It's that light because it shares the host's running kernel: there's no firmware, no kernel boot, and only the changed blocks are stored on disk.

**Two traps for capacity planning:**
1. **Per-container memory overstates the real cost.** Use the host's `available` drop, not the per-container figure.
2. **The platform takes its share first.** Ceph, LXD, MicroOVN and MicroCloud hold about 1.65 GB, roughly a third of this node, before any workload runs. On `mc3`, memory runs out first: about **14 more idle containers** fit with the 500 MB reserve, while storage alone would allow about 30.

**VMs:** not measured, because the Windows host blocks nested virtualization. By design, a VM boots its own kernel and reserves its configured memory, so you'd expect a higher per-workload cost and a slower start. Measuring it requires a host without Microsoft's hypervisor, or bare metal.

**When you'd still choose a VM:** you need a different kernel or OS, stronger isolation between tenants (a separate kernel rather than a shared one), or kernel-level features a container can't have. Density isn't the only criterion.

**Lab-vs-reality gap:** these nodes are small VMs inside VMware on Windows, so the numbers are lower and noisier than on real servers. The slow-operation alert and the blocked VMs both come from that nesting. The platform's share of memory would be much smaller on a server with 256 GB of RAM. That's why density claims from small labs overstate platform overhead, and understate how many workloads a real node holds.

---

## Definition of Done

- [x] Nested virtualization check recorded; VMs blocked by the host OS, with the exact VMware errors
- [x] Recovery from the failed VT-x attempt documented
- [x] `BLUESTORE_SLOW_OP_ALERT` investigated; two causes ruled out, root cause recorded as undetermined
- [x] Baseline recorded for `mc3`
- [x] Container memory recorded, both reported and host cost
- [x] Storage broken down per disk (image vs. containers)
- [x] Start and stop times recorded
- [x] Test instances deleted; storage returned, memory partly returned and explained as a hypothesis
- [x] Write-up completed

**Open follow-ups (optional):**
- Check `osd.1`/`osd.2` logs for similar stalls, to see whether `mc2`'s disk is specifically slow.
- Find out what wrote ~930 MiB inside `flat-a`.
- Time the INFRA-016 move as a single command, with a ping running.
- Repeat the density test with a per-process memory baseline.

---

## Retention Check

An idle container on `mc3` reported using over 200 MiB, yet the host only lost about 150 MB per container, and a third of the node's memory was gone before any container started. What two corrections do you need to make before telling a client how many workloads a node can hold?