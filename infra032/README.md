# INFRA-032: Incident Drill: A Tool Gives the Wrong Answer (pdb and git bisect)

**Priority:** P2 (Plan v3, Day 8: troubleshooting-first)
**Status:** Resolved (2026-10-17)
**Component:** Python debugging, git history. Tools: `python3 -m pdb`, `git log`, `git bisect`, `git revert`
**Environment:** `mc1` only, in a throwaway folder `~/drill-032`. **No cluster changes.**
**Linked tickets:** INFRA-031 (strace: asking the kernel). Sets up INFRA-033 (gdb and C)

> Rewritten after the session with the real outputs.

---

## Summary

`diskcheck` is a tiny Python tool: it reads a list of disks and their usage, and warns about any disk **over 80%**. It worked at version `v1.0`. After 5 more changes (**commits**), it **missed a disk that was 100% full**, without crashing.

- **`pdb`** (Python's debugger) paused the tool on the line that decides "over the limit?" and showed the cause: usage `'100'` was **text**, compared with the text `'80'`. Text is compared letter by letter, and `"1"` comes before `"8"`, so `"100"` counted as smaller than `"80"`.
- **`git bisect`** searched the history and named the commit that broke it: `parse: handle trailing spaces; fix type error`. The manual run went wrong (results weren't fed back to git); the automatic run (`git bisect run`) found it in 3 tests.
- **`git revert`** undid that commit. Both full disks are reported again.

## Why

A wrong answer is harder than a crash: there's no error to search for. `pdb` shows what the program really holds at the moment it decides. `git bisect` answers "what changed?" without reading every commit. Together they turn "it used to work" into "this line, in this commit".

---

## Runbook (with real results)

All commands ran on **`mc1`**.

### Step 1: Build the repo (one paste)

Created `~/drill-032` with `diskcheck.py`, `usage.txt` and 6 commits, the oldest tagged `v1.0`. (Full script in the plan version of this ticket.) Built successfully.

### Step 2: Symptom

```bash
cat usage.txt
python3 diskcheck.py usage.txt
```
```
sda 45%
sdb 100%
sdc 92%

WARNING: sdc is over 80%
1 disk(s) over 80%
```
`sdb` and `sdc` are both over 80%, but only `sdc` is reported. **`sdb`, 100% full, is missed.**

### Step 3: Diagnose with pdb

```bash
python3 -m pdb diskcheck.py usage.txt
```
```
> /home/nicky/drill-032/diskcheck.py(2)<module>()
(Pdb) b 20, name == "sdb"
Breakpoint 1 at /home/nicky/drill-032/diskcheck.py:20
(Pdb) c
> /home/nicky/drill-032/diskcheck.py(20)over_limit()
-> if pct > str(threshold):
(Pdb) p name, pct, threshold
('sdb', '100', 80)
(Pdb) p type(pct), type(threshold)
(<class 'str'>, <class 'int'>)
(Pdb) p pct > str(threshold)
False
(Pdb) q
```
**Reading it:**
- `b 20, name == "sdb"`: a **conditional breakpoint**, a pause at line 20 only when the disk is `sdb`. It skipped `sda` and stopped exactly where it mattered.
- `('sdb', '100', 80)`: `'100'` has quotes, so it's **text**; `80` is a **number**.
- `type(...)` confirms it: `str` (text) and `int` (number).
- Line 20 is `if pct > str(threshold):`, which turns the limit into text too, so it compares `"100"` with `"80"` as text, letter by letter. `"1"` comes before `"8"`, so the answer is **`False`**.

> `p int(pct) > threshold` (comparing as numbers) wasn't run in the session. It returns `True` (100 > 80), which is what the code should be doing.

### Step 4: Find the commit that broke it, with git bisect

**Manual run (went wrong, kept as a lesson):**
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0
```
```
Bisecting: 2 revisions left to test after this (roughly 1 step)
[490b9b9...] add .gitignore
```
The test was run 6 times (`GOOD` each time), but **`git bisect good` was never typed**, so git stayed on the same commit. `git bisect reset` then `git show 490b9b9` showed `add .gitignore`, which only adds a `.gitignore` file. **That commit was git's first midpoint, not the culprit.**

> **Lesson:** bisect is a conversation. After each test you must tell git `good` or `bad`, or it can't move on.

**Automatic run (correct):**
```bash
git bisect start HEAD v1.0
git bisect run sh -c 'python3 diskcheck.py usage.txt | grep -q sdb'
```
```
[490b9b9...] add .gitignore
running 'sh' '-c' 'python3 diskcheck.py usage.txt | grep -q sdb'
Bisecting: 0 revisions left to test after this (roughly 1 step)
[6340128...] add CHANGELOG
running 'sh' '-c' 'python3 diskcheck.py usage.txt | grep -q sdb'
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[50a96d9...] parse: handle trailing spaces; fix type error
running 'sh' '-c' 'python3 diskcheck.py usage.txt | grep -q sdb'
50a96d9568b4c7e13547ccb8addd0630941a7887 is the first bad commit
    parse: handle trailing spaces; fix type error
 diskcheck.py | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
bisect found first bad commit
```
| Round | Commit tested | Result | What it rules out |
|---|---|---|---|
| 1 | `add .gitignore` | good | everything before it |
| 2 | `add CHANGELOG` | bad | everything after it |
| 3 | `parse: handle trailing spaces; fix type error` | bad | it's the first bad one |

`git bisect run` uses the command's **exit code** as the answer: `0` (`grep` found `sdb`) means good, anything else means bad. It feeds each result back to git by itself, so the manual mistake can't happen.

```bash
git bisect reset
```
```
Switched to branch 'master'
```

### Step 5: Fix and prove it

```bash
git revert --no-edit 50a96d9568b4c7e13547ccb8addd0630941a7887
python3 diskcheck.py usage.txt
git log --oneline | head -3
```
```
Auto-merging diskcheck.py
[master 6d08394] Revert "parse: handle trailing spaces; fix type error"
 1 file changed, 2 insertions(+), 2 deletions(-)

WARNING: sdb is over 80%
WARNING: sdc is over 80%
2 disk(s) over 80%

6d08394 Revert "parse: handle trailing spaces; fix type error"
894515d print summary count
6340128 add CHANGELOG
```
- `git revert` adds a **new commit that reverses** the bad one. The bad commit stays in history, so everyone can see what happened and when.
- The later commit (`print summary count`) was kept: `2 disk(s) over 80%` still prints.

### Step 6: Clean up

```bash
cd ~ && rm -rf ~/drill-032
```
**Real result:** _not yet run_

---

## Incident write-up

| | |
|---|---|
| **Symptom** | `diskcheck` reported only `sdc` (92%); `sdb` (100%) was missed. No error |
| **What I checked** | `pdb` with a conditional breakpoint on line 20: `pct` was text `'100'`, compared with text `'80'` → `False`. `git bisect run`: 3 tests named commit `50a96d9` |
| **Root cause** | Commit `50a96d9` removed `int(...)` from `parse()`, so usage stayed as text, then "fixed" the resulting type error by turning the limit into text too. Text comparison is letter by letter, so `"100" < "80"` |
| **Fix** | `git revert 50a96d9`: both disks reported, later work kept |
| **Prevention** | A test with a 3-digit value (`100%`) would have caught it before merge. When a change "fixes a type error", check *why* the types differ instead of converting until the error goes away |

**Lessons:**
1. **Look at the real values.** The code read fine; only pdb showed `'100'` was text.
2. **A "type error fix" can hide a real bug.** Turning the limit into text silenced the crash but broke the logic.
3. **Bisect is a conversation:** test, then tell git `good` or `bad`. Or let `git bisect run` do both.
4. **Revert, don't rewrite.** `git revert` fixes the code and keeps an honest history.

**AWS comparison:** `git bisect` is the same idea as finding which **CloudFormation stack update** or **Lambda version** introduced a regression: start from a known-good version and a known-bad one, and narrow down by halves.

---

## Definition of Done

- [x] Repo built; wrong answer seen (`sdb` at 100% not reported)
- [x] pdb showed `pct` was text (`'100'`) compared with text (`'80'`)
- [x] git bisect named the first bad commit (automatic run; manual run mistake recorded)
- [x] Fixed with `git revert`; both disks reported
- [ ] Folder cleaned up

## Retention Check

`git bisect` found the bad commit in 3 tests out of 5 suspects. How does it do that, and why does it need one "good" and one "bad" starting point?

**Answer:** each test checks the commit in the middle of the suspects. If it's good, the break must be after it; if it's bad, at or before it. Each answer throws away half the list. It needs a known-good and a known-bad point because the search only works between two ends: "somewhere between this working version and this broken one".