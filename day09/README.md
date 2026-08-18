# INFRA-009: MicroCloud Overview — What It Bundles and Why

| Field | Value |
|---|---|
| **Ticket ID** | INFRA-009 |
| **Priority** | Medium |
| **Component** | Clustering Foundations / MicroCloud |
| **Environment** | VMware Workstation Pro (Windows host) — 3-node LXD cluster (`node1`, `node2`, `node3`) |
| **Reporter / Assignee** | NwaChi (Infrastructure Engineer track) |
| **Sprint** | Week 2 — Clustering Foundations (Day 9 of 50) |
| **Linked tickets** | INFRA-008 (failure detection + recovery) — direct motivation for this ticket. Week 3 (MicroCloud compute/storage) — this ticket's groundwork feeds it directly. |

---

## Summary

Days 6 through 8 built an LXD cluster by hand: joined three nodes one at a time, reasoned through dqlite quorum math, wrote a custom polling script to detect a failed node, and manually verified recovery. That was compute clustering only.

MicroCloud is Canonical's answer to "what if you had to do that same amount of work for storage and networking too" — it bundles LXD (compute), MicroCeph (distributed storage), and MicroOVN (managed networking) behind one guided setup process, so all three get clustered, made highly available, and kept in sync together instead of three separate manual builds.

This ticket is deliberately an **overview**, not a rebuild. The goal is to understand what MicroCloud is and install its tooling to explore it — not to bootstrap a new cluster on top of the one Week 2 already built. Reasoning for that boundary is in the Comments thread below.

---

## Acceptance Criteria

- [ ] Can explain what MicroCloud is and how it relates to LXD (wrapper vs. replacement vs. distribution)
- [ ] Can name each bundled component and which layer it owns: LXD (compute), MicroCeph (storage), MicroOVN (networking)
- [ ] Can explain, using Day 8 as the reference point, why bundling the three matters operationally
- [ ] Required snaps installed and explored on `node1` without disrupting the existing manual cluster
- [ ] Can describe, from documentation, what actually happens during `microcloud init` — without having run it against the live Week 2 cluster
- [ ] Comparison table completed: manual Week 2 build vs. MicroCloud's bundled equivalent
- [ ] Retention check answered

---

## Comments (conceptual explanation thread)

**NwaChi:**
Opening this to get ready for Week 3, which is going to lean on MicroCloud directly. Before touching it hands-on, I want to actually understand what it's doing under the hood rather than just following an init wizard and hoping it works.

**Senior Infra Engineer (review):**
Good sequencing. Here's the "why" worth internalizing before you install anything: think back to INFRA-008. You built a small monitoring script to detect when one LXD node dropped, and a manual runbook to recover it. That was real work, for one layer.

MicroCloud's pitch is that LXD, Ceph, and OVN each have their own version of that same problem — their own cluster membership, their own quorum/consensus mechanics, their own failure modes. Built manually, that's not one INFRA-008, that's three, each with different tooling and different failure semantics to learn. MicroCloud's `init` process orchestrates all three together: joins the nodes, sets up Ceph storage across them, sets up OVN networking across them, in one guided flow, using consistent, version-synced tooling across every node.

One caution before you touch a single command: **do not run `microcloud init` against `node1`/`node2`/`node3`.** They're already a manually-built, working LXD cluster from Days 6 through 8. MicroCloud's init process expects to bootstrap fresh — running it against nodes that are already clustered manually risks conflicting with or damaging the work you've already verified. A real hands-on bootstrap deserves clean nodes, which is exactly what Week 3 is for. Today, install the tooling and read what it would do. Don't execute the thing that changes cluster state.

**NwaChi:**
Makes sense — install and inspect, not initialize. That also means I get to actually see whether the snaps interfere with anything just by being installed, before Week 3 needs to trust that.

---

## Implementation Runbook

### Phase 0 — Why we're not running `microcloud init` today

Before typing anything: the reasoning above is the whole point of this ticket being scoped as an overview. If you're tempted to run `microcloud init` because it looks quick, stop — re-read the Comments thread first. Everything below is safe to run without changing your existing cluster's state.

---

### Phase 1 — Understand what's already installed

MicroCloud needs four snaps present on every intended member: `lxd`, `microceph`, `microovn`, and `microcloud` itself. You almost certainly already have `lxd` — it's what built the Week 2 cluster. Check before installing anything, rather than assuming:

```bash
snap list | grep -E 'lxd|microceph|microovn|microcloud'
```

You should see `lxd` listed. `microceph`, `microovn`, and `microcloud` are very likely absent — that's expected, that's what this phase installs next.

---

### Phase 2 — Install the MicroCloud tooling (on `node1` only, for now)

Run these on `node1`. The official install guide has you install these on *every* intended cluster member with a shared `--cohort` flag so all machines stay on matching versions — but since we're only exploring (not clustering anything new today), one node is enough to inspect the tooling itself:

```bash
sudo snap install microceph --channel=squid/stable --cohort="+"
sudo snap install microovn --channel=24.03/stable --cohort="+"
sudo snap install microcloud --channel=2/stable --cohort="+"
```

The `--cohort="+"` flag matters for later: it locks a machine to a specific release track so that when you eventually add more nodes in Week 3, every node installs the exact same version rather than whatever happens to be "latest" on install day. Skipping it now is fine since you're not clustering yet, but it's worth typing it anyway so the habit is already there when it does matter.

If `lxd` reports as already installed at a different channel than `5.21/stable` (the version MicroCloud's docs are written against), that's fine to note but not fix today — just be aware version mismatches are something MicroCloud cares about, which Phase 3 will show you directly.

---

### Phase 3 — Explore each tool's CLI without changing state

This is the "read the actual commands" habit from Day 8 applied here — don't assume what each tool does, look:

```bash
microcloud --help
microceph --help
microovn --help
```

Look specifically for the subcommands each one exposes for cluster membership (`init`, `join`, `status`, `cluster list` or similar) — you'll recognize the shape immediately, since it mirrors `lxc cluster list` from Week 2. That similarity isn't an accident: MicroCeph and MicroOVN are built on the same underlying clustering approach LXD uses.

Also check what MicroCloud reports about your current environment without initializing anything:

```bash
microcloud --version
snap list microcloud microceph microovn lxd
```

Compare the four version/channel numbers. This is the real-world version of the `--cohort` concern from Phase 2 — in a genuine MicroCloud deployment, mismatched versions across these four snaps between machines is a documented source of failed or inconsistent initialization.

---

### Phase 4 — Read (don't run) the initialization process

Based on MicroCloud's own documentation, here's what `microcloud init` actually does when you eventually run it for real in Week 3, so you're reading the prompts with understanding instead of guessing:

1. Run from one member (the "initiator") — it scrapes the local subnet for other machines running the same snaps.
2. Prompts you to select which discovered machines join the cluster.
3. If MicroCeph is installed on the selected machines, prompts you to choose local or distributed storage and select which disks to hand to Ceph.
4. If MicroOVN is installed, prompts you to configure the underlay networking.
5. Bootstraps all three — LXD cluster, Ceph cluster, OVN cluster — together, using the answers you gave.

Notice what's conditional in steps 3 and 4: if a machine doesn't have MicroCeph or MicroOVN installed, MicroCloud just skips those prompts and configures only what's present. That's worth remembering going into Week 3 — the "overview" story is real, but MicroCloud's install is also flexible, not all-or-nothing.

---

### Phase 5 — Comparison: manual build vs. bundled equivalent

Fill in from what you now know, using Week 2 as the left column:

| Layer | What you built manually (Week 2) | MicroCloud's bundled equivalent |
|---|---|---|
| Compute clustering | Joined `node1`–`node3` into LXD one at a time; reasoned through dqlite quorum by hand (INFRA-006/007) | Same LXD clustering, joined automatically as part of one `microcloud init` run |
| Failure detection | Hand-written `cluster-watch.sh` polling script, built from scratch (INFRA-008) | Would need an equivalent for MicroCeph and MicroOVN too, if built manually — this is the actual cost MicroCloud removes |
| Storage | Not yet built — scoped for Week 3 | MicroCeph — distributed, HA-aware storage, orchestrated by the same `init` |
| Networking | Host-only adapter + `lxdbr0` bridge, configured node by node (INFRA-006/007) | MicroOVN — managed virtual networking layer, orchestrated by the same `init` |

---

## Definition of Done

- [ ] `microceph`, `microovn`, `microcloud` snaps installed on `node1`, `lxd` confirmed already present
- [ ] `--help` output reviewed for all three new tools, subcommands mapped conceptually back to Week 2 equivalents
- [ ] Version/channel check run across all four snaps
- [ ] Comparison table completed
- [ ] Existing Week 2 cluster confirmed untouched — `lxc cluster list` still shows all three nodes as it did at the end of INFRA-008
- [ ] Retention check answered

---

## Retention check

If you had to build INFRA-008's failure-detection script separately for MicroCeph and MicroOVN too — not reuse it, actually build the equivalent from scratch for each — what specifically would you need to learn and build twice more? And what part of that repeated effort does MicroCloud's single `init` process take off your plate?