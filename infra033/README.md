# INFRA-033: Incident Drill: A C Program Crashes (gdb and core dumps), plus How Ubuntu Packages Carry Fixes

**Priority:** P2 (Plan v3, Day 9: troubleshooting-first)
**Component:** C debugging, core dumps, Debian/Ubuntu packaging. Tools: `gcc`, `gdb`, `ulimit`, `apt source`, `debian/patches`
**Environment:** `mc1` only, in a throwaway folder `~/drill-033`. One system setting (where core dumps are saved) is changed temporarily and restored. **No cluster changes.**
**Linked tickets:** INFRA-028 (chrony), INFRA-031 (strace), INFRA-032 (pdb and git). Sets up INFRA-034 (capstone)
**Timebox:** 30 minutes. Troubleshooting cap: 10 minutes.

> **Status:** plan. This file gets rewritten after the session with the real outputs.

---

## Summary

**Part A: a crash.** `diskreport` is a tiny C program that prints a line for each disk. It crashes with `Segmentation fault`, which means it tried to read memory it isn't allowed to. We let Linux save a **core dump** (a snapshot of the program's memory at the moment it died), then open it with **`gdb`** (the standard debugger for C programs). The **backtrace** (the chain of functions that were running at the crash) leads straight to the bad data.

**Part B: packaging, 10 minutes.** Ubuntu ships programs as **packages**. When Ubuntu fixes a bug, it often doesn't change the original code directly. It adds a small **patch** file on top, listed in `debian/patches`. We download the source of `chrony` (the time-sync service from INFRA-028) and look at how its fixes are recorded.

## Why

Many low-level tools are written in C: the Linux kernel, `systemd`, `chrony`, much of Ceph and Open vSwitch. When one crashes in production, you usually don't get to watch it happen. You get a **core file** afterwards. Reading it with `gdb` is a core skill for sustaining and escalation roles. And a fix for a customer often ships as a patch in an Ubuntu package, so knowing where patches live is part of the job.

---

## Runbook

All commands run on **`mc1`**.

## Part A: The crash

### Step 1: Install the tools, and build the program

```bash
sudo apt-get install -y gcc gdb
mkdir -p ~/drill-033 && cd ~/drill-033
cat > diskreport.c <<'EOF'
#include <stdio.h>
#include <string.h>

struct disk {
    const char *name;
    int used;
};

static struct disk disks[] = {
    {"sda", 45},
    {"sdb", 100},
    {NULL, 92},
};

static int name_length(const char *name)
{
    return strlen(name);
}

static void report(const struct disk *d)
{
    printf("%-*s %3d%%\n", name_length(d->name) + 2, d->name, d->used);
}

int main(void)
{
    for (int i = 0; i < 3; i++)
        report(&disks[i]);
    return 0;
}
EOF
gcc -g -O0 -Wall -o diskreport diskreport.c
```
- The program has a list of 3 disks. The third one has **no name** (`NULL`, meaning "nothing").
- `gcc` compiles the C code into a program you can run.
- `-g` adds **debug information** (line numbers, variable names) so `gdb` can show them.
- `-O0` turns off optimisation, so the program runs line by line, exactly as written.
- `-Wall` shows all common warnings.

**Expected:** the install finishes; `gcc` prints nothing (no warnings).

From here on, pretend you don't know what's wrong with the data.

**Real result:** _pending_

### Step 2: Turn on core dumps (temporarily)

```bash
cat /proc/sys/kernel/core_pattern | tee ~/drill-033/core_pattern.orig
sudo sysctl -w kernel.core_pattern=/tmp/core.%e.%p
ulimit -c unlimited
```
- `core_pattern` tells Linux what to do with core dumps. On Ubuntu it usually hands them to **apport** (Ubuntu's crash reporter). We save the original setting first, to restore it later.
- The new pattern saves them as plain files: `/tmp/core.<program name>.<process ID>`.
- `ulimit -c unlimited` allows core files of any size **in this terminal only**. The default size limit is often 0, which means "no core dumps".

**Expected:** the original pattern printed (probably starting with `|/usr/share/apport/apport`), then `kernel.core_pattern = /tmp/core.%e.%p`.

**Real result:** _pending_

### Step 3: Symptom

```bash
./diskreport; echo "exit code: $?"
ls -lh /tmp/core.diskreport.*
```
**Expected:**
```
sda    45%
sdb   100%
Segmentation fault (core dumped)
exit code: 139
```
plus one core file.
- `139` = 128 + 11, where **11 is the signal number for a segmentation fault** (`SIGSEGV`). The kernel stopped the program.
- `(core dumped)` means the snapshot was saved.
- The first two disks printed fine; it died on the third.

**Real result:** _pending_

### Step 4: Diagnose with gdb (reading the core file)

```bash
gdb -q -batch \
  -ex "bt" \
  -ex "frame 1" -ex "p name" \
  -ex "frame 2" -ex "p *d" \
  -ex "frame 3" -ex "p i" \
  ./diskreport /tmp/core.diskreport.*
```
- `gdb ./diskreport <core file>` opens the snapshot. Nothing runs; we only read what was in memory at the crash.
- `-batch -ex "..."` runs the listed commands one after another, then quits.
- `bt`: **backtrace**. The list of functions that were running, from the crash point (`#0`) back to the start (`main`).
- `frame N`: step into one of those functions; `p ...`: **print** a value as it was at the crash.

**Expected, roughly:**
```
#0  __strlen_... () at ../sysdeps/x86_64/multiarch/strlen-....S:...
#1  name_length (name=0x0) at diskreport.c:17
#2  report (d=0x... <disks+32>) at diskreport.c:22
#3  main () at diskreport.c:28
...
$1 = 0x0
$2 = {name = 0x0, used = 92}
$3 = 2
```
**Reading it (bottom to top):**
- `#3 main`, with `i = 2`: the loop was on the **third** disk (counting starts at 0).
- `#2 report`: that disk is `{name = 0x0, used = 92}`. **It has no name.**
- `#1 name_length (name=0x0)`: our code passed that empty name on, at line 17.
- `#0 __strlen_...`: the crash itself happened inside **`strlen`**, the C library's "how long is this text?" function, when it tried to read text at address `0x0`. (The exact name varies with the CPU; a `No such file` warning about its source is normal.)

> **Key lesson:** frame `#0` is where it crashed, **not** where the bug is. `strlen` is fine. Our code handed it a `NULL`. Walk up the backtrace to the first frame that's **your** code.

**Real result:** _pending_

### Step 5: Fix and prove it

Check for a missing name before using it:
```bash
sed -i 's|    return strlen(name);|    return name ? strlen(name) : 0;|' diskreport.c
sed -i 's|    printf("%-\*s %3d%%\\n", name_length(d->name) + 2, d->name, d->used);|    printf("%-*s %3d%%\\n", name_length(d->name) + 2, d->name ? d->name : "(unnamed)", d->used);|' diskreport.c
gcc -g -O0 -Wall -o diskreport diskreport.c && ./diskreport; echo "exit code: $?"
```
- `name ? strlen(name) : 0` means: "if there's a name, measure it; otherwise use 0".
- The second change prints `(unnamed)` instead of trying to print nothing.

**Expected:** 3 lines, the last one `(unnamed)  92%`, and `exit code: 0`.

**Real result:** _pending_

### Step 6: Restore the core dump setting

```bash
sudo tee /proc/sys/kernel/core_pattern < ~/drill-033/core_pattern.orig
cat /proc/sys/kernel/core_pattern
sudo rm -f /tmp/core.diskreport.*
```
**Expected:** the original pattern again (the `apport` one).

**Real result:** _pending_

## Part B: How Ubuntu packages carry fixes (10 minutes)

### Step 7: Enable source packages and download chrony's source

```bash
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt-get update -q
sudo apt-get install -y dpkg-dev
cd ~/drill-033 && apt source chrony
ls
```
- Ubuntu normally downloads only ready-to-run packages (`deb`). Adding `deb-src` also allows downloading their **source code**.
- `dpkg-dev` holds the tools `apt source` uses to unpack it.
- `apt source chrony` downloads chrony's original code **plus Ubuntu's changes**, and unpacks them into a folder.

**Expected:** a folder like `chrony-4.x/` and a few `.tar.xz` / `.dsc` files.

**Real result:** _pending_

### Step 8: Read the packaging

```bash
cd ~/drill-033/chrony-*/
head -12 debian/changelog
ls debian/patches/
cat debian/patches/series
head -20 debian/patches/$(head -1 debian/patches/series)
```
**What each shows:**
- `debian/changelog`: the package's history. The top entry is the version installed on `mc1`, with who changed what and why, often including security fix IDs (CVE numbers) and Launchpad bug numbers (`LP: #...`).
- `debian/patches/`: one file per change made on top of the original code.
- `series`: the **order** the patches are applied in.
- A patch's top lines: a header that describes it (`Description`, `Author`, `Origin`, `Bug`, `Forwarded`), then the actual changed lines (`-` removed, `+` added).

**Expected:** a changelog starting with something like `chrony (4.x-...ubuntu...) noble; urgency=...`, and a short list of patches.

> If chrony has no `debian/patches` folder, swap `chrony` for `openssh` in Step 7. It has many patches.

**Real result:** _pending_

### Step 9: Clean up

```bash
cd ~ && rm -rf ~/drill-033
sudo sed -i 's/^Types: deb deb-src$/Types: deb/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt-get update -q
```
**Expected:** folder gone; source downloads switched off again.

**Real result:** _pending_

---

## Incident write-up (fill in after the session)

| | |
|---|---|
| **Symptom** | |
| **What I checked** | |
| **Root cause** | |
| **Fix** | |
| **Prevention** | |

---

## Definition of Done

- [ ] Program built; crash reproduced with a core file saved
- [ ] Backtrace read; bad data found (`name = 0x0` on the third disk)
- [ ] Fixed with a NULL check; program runs to the end
- [ ] Core dump setting restored to the original
- [ ] chrony source downloaded; changelog and patches read
- [ ] Cleaned up (folder removed, `deb-src` switched off)

## Retention Check

The backtrace's top frame (`#0`) was inside `strlen`, a C library function used by millions of programs. Why wasn't that the bug, and how did you find the real one?