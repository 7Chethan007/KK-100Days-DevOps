# Day 25 — Git Branching, Merging, and Pushing to a Remote Repository

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 24 created `devops` as a bare reference pointing at the exact same
commit as `master` — the branch existed, but nothing had changed yet.
This lab is that branch's actual purpose fulfilled: commit real work on
it, merge it back, and push both branches to the remote — while also
hitting a genuinely new category of obstacle for this series,
Linux filesystem ownership blocking Git itself.

---

## 1. Scenario

On the Storage Server (`ststor01`), as `natasha`, in
`/usr/src/kodekloudrepos/games`: create `devops` from `master` (if not
already done), add `/tmp/index.html`, commit it, merge `devops` into
`master`, and push both branches to the remote `/opt/games.git`.

| Property | Value |
|---|---|
| Repository | `/usr/src/kodekloudrepos/games` |
| Remote | `/opt/games.git` |
| Feature branch | `devops` |
| Commit message | `Add index.html` |

```text
master (5dc37bc)
   │
   ├──── devops (same commit, from Day 24)
   │          │  cp index.html + add + commit → 6266cb3
   │          ▼
   │      devops @ 6266cb3
   │          │  merge devops → master (fast-forward)
   ▼          ▼
master @ 6266cb3  ══  devops @ 6266cb3
   │
   └── push BOTH → /opt/games.git
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 `dubious ownership` and `Permission denied` are two different failures with two different fixes

```text
fatal: detected dubious ownership in repository   → Git's OWN trust check (software-level)
fatal: cannot lock ref '...': Permission denied     → Linux FILESYSTEM permissions (OS-level)
```

Fixing `safe.directory` only resolves Git's own ownership-mismatch
safety check — it tells Git "trust this repository despite the
ownership mismatch." It does nothing to the actual Linux file
permissions underneath. The second error appeared *after* the first
was fixed precisely because they're unrelated: Git was willing to
operate, but the Linux filesystem itself still refused the write.
Treating these as the same problem (or assuming fixing one fixes the
other) is the specific mistake this lab is built to correct.

### 2.2 `safe.directory` changes trust; it does not change permissions

```text
git config --global --add safe.directory <path>
```

This setting only affects whether Git is *willing* to read a
repository's config/hooks despite an ownership mismatch — a
protection against a malicious repository owned by someone else
tricking you into running their hooks. It has zero effect on whether
the Linux filesystem actually grants `natasha` write access to
`root`-owned files. Expecting this single command to also fix the
subsequent `Permission denied` error is exactly the conflation §2.1
warns against.

### 2.3 Root's Git configuration is separate from `natasha`'s — `git -c` sidesteps needing to persist it anywhere

```text
natasha's ~/.gitconfig:  safe.directory entries added here
root's   ~/.gitconfig:   a COMPLETELY SEPARATE file — doesn't automatically inherit natasha's entries
```

Running Git via `sudo` executes as `root`, which has its own global
Git configuration, independent of whatever `natasha` configured.
`git -c safe.directory=<path> <command>` supplies the setting for
*that one invocation only*, avoiding the need to edit root's
persistent config just to get one command to run — a scoped,
minimal-footprint fix rather than a standing configuration change.

### 2.4 `cp /tmp/index.html .` failing is a *destination permissions* problem, not a *source* problem

```text
cp /tmp/index.html .                                          → Permission denied
sudo cp /tmp/index.html /usr/src/kodekloudrepos/games/index.html → succeeds
```

The source file (`/tmp/index.html`) was readable the whole time — the
failure was always about writing into the repository directory, which
`natasha` doesn't own. The explicit destination path in the working
command isn't what fixed it (the `.` and the explicit path are
equivalent as *paths*) — `sudo` is what fixed it, by running the copy
as the user that actually has write permission.

### 2.5 A branch isn't merged just because it exists or was pushed — merging is its own explicit action, in a specific direction

```text
git switch master      ← checkout the TARGET first
git merge devops         ← merges devops INTO whatever is currently checked out
```

`git merge devops` always merges *into* the currently checked-out
branch — running it while still on `devops` would attempt to merge
`devops` into itself, which is meaningless. Switching to `master`
*first* is what makes "merge `devops` into `master`" actually happen
in the direction the task requires, rather than relying on which
branch happens to be checked out by habit.

### 2.6 A fast-forward merge is possible specifically because `master` never diverged from `devops`

```text
master:  A ────────────▶ (never advanced past A)
devops:  A ─── B          (advanced by exactly one commit)
```

Because `A` (where `master` still sits) is a direct ancestor of `B`
(where `devops` now sits), Git doesn't need to reconcile two divergent
histories — it can just move `master`'s pointer forward to `B`. This
is only possible because nothing else was committed on `master` in the
meantime; if it had been, Git would need an actual three-way merge
(and possibly conflict resolution) instead of this simple pointer
move.

### 2.7 A local branch and its remote counterpart are separate references — neither updates the other automatically

```text
Local devops  @ 6266cb3   (after merge, locally)
Local master  @ 6266cb3   (after merge, locally)
origin/devops  ?           ← still whatever it was BEFORE any push
origin/master  ?           ← still whatever it was BEFORE any push
```

Creating, committing, and merging a branch all happen entirely in the
local repository — none of it touches the remote `/opt/games.git`
until an explicit `git push` names that branch. This is why the task
requires *two* separate pushes (`devops`, then `master`), and why
checking only local state would never catch a missed push.

### 2.8 `ls-remote` queries the remote directly — it doesn't trust potentially-stale local remote-tracking refs

```text
git branch -r                       → reads LOCAL cached remote-tracking refs (could be stale)
git ls-remote --heads origin ...     → asks the REMOTE REPOSITORY ITSELF, right now
```

After pushing, the decisive verification is asking `/opt/games.git`
directly what its `devops` and `master` references currently point
to — not trusting a local cache that could be out of date if, say, a
push silently failed or was run against the wrong remote. Both
returning the identical hash (`6266cb37...`) is the actual proof the
push succeeded on both branches.

### 2.9 Both branches pointing at the same final commit is the *expected*, correct outcome here — not a sign something is missing

```text
devops @ 6266cb3  ==  master @ 6266cb3
```

Because the merge was a fast-forward (§2.6) and both branches were
pushed immediately afterward, there was never a point where they
should have diverged — two branches sharing a commit is not evidence
one of them "didn't get pushed" or "is missing something," it's
exactly what a clean fast-forward-then-push sequence produces.

### 2.10 The compressed reasoning chain

```text
Requirement: devops branch gets index.html, merged + pushed to master
   → connect, cd into the repo, CHECK starting state (status/branch -a) before touching anything
   → fatal: dubious ownership → git config --global --add safe.directory <path> (TRUST, §2.2)
   → fatal: Permission denied writing refs → sudo (ELEVATED PRIVILEGE, different problem, §2.1)
   → root's config ≠ natasha's → use git -c safe.directory=... per-invocation instead of persisting (§2.3)
   → sudo git switch -c devops (if not already done in a prior lab)
   → sudo cp /tmp/index.html <explicit full path>  — sudo fixes WRITE permission, not the path syntax (§2.4)
   → sudo git add + commit -m "Add index.html"  — ALL on devops
   → sudo git switch master FIRST, then sudo git merge devops  — direction matters (§2.5)
   → fast-forward succeeds BECAUSE master never diverged (§2.6)
   → push devops, THEN push master — two SEPARATE pushes, neither implies the other (§2.7)
   → verify via ls-remote --heads origin devops master — queries the REMOTE directly (§2.8)
   → both branches sharing the same final hash is EXPECTED, not a red flag (§2.9)
```

---

## 3. Concepts (reference)

### 3.1 Git's `dubious ownership` safety check vs. Linux filesystem permissions
Git's own ownership check (addressed by `safe.directory`) protects
against trusting config/hooks from a repository owned by someone
else. It is entirely independent of the OS-level filesystem
permissions that actually gate whether a write succeeds — fixing one
has no effect on the other.

### 3.2 `git -c <key>=<value>` for a single invocation
Supplies a Git configuration value scoped to just that one command
run, rather than persisting it to a config file — useful when running
as a different user (e.g. via `sudo`) whose own config shouldn't need
permanent edits for a one-off task.

### 3.3 Merge direction
`git merge <branch>` always merges the named branch *into* the
currently checked-out branch. The checked-out branch at merge time is
what determines the direction of integration.

### 3.4 Fast-forward merge
Possible only when the target branch's tip is a direct ancestor of
the source branch's tip — Git simply advances the target's pointer,
with no new merge commit and no possibility of conflicts.

### 3.5 Local branch vs. remote-tracking branch vs. the actual remote
A local branch, its local remote-tracking reference, and the actual
reference on the remote repository are three distinct things. Local
operations (commit, merge) don't touch the remote at all; only an
explicit `push` does, and only `ls-remote` queries the remote's
current state directly rather than a potentially stale local cache.

---

## 4. Runbook

### 4.1 Connect and enter the repository
```bash
ssh natasha@ststor01
cd /usr/src/kodekloudrepos/games
pwd
```

### 4.2 Inspect the starting state
```bash
git status
git branch -a
git branch --show-current
```
```text
On branch master
Your branch is up to date with 'origin/master'.
nothing to commit, working tree clean
```

### 4.3 Resolve Git's ownership trust check
```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/games
```
(For root-run commands, instead supply it per-invocation: `git -c
safe.directory=/usr/src/kodekloudrepos/games ...`, since root's
global config is separate from `natasha`'s — §2.3.)

### 4.4 Create (or confirm) the `devops` branch
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games switch -c devops
sudo git -c safe.directory=/usr/src/kodekloudrepos/games branch --show-current
```
```text
devops
```

### 4.5 Copy, verify, stage, and commit `index.html`
```bash
ls -l /tmp/index.html
cat /tmp/index.html
```
```text
Welcome to Nautilus Group !
```
```bash
sudo cp /tmp/index.html /usr/src/kodekloudrepos/games/index.html
ls -l index.html
cat index.html
```
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games add index.html
sudo git -c safe.directory=/usr/src/kodekloudrepos/games diff --cached
sudo git -c safe.directory=/usr/src/kodekloudrepos/games commit -m "Add index.html"
sudo git -c safe.directory=/usr/src/kodekloudrepos/games log -1 --oneline
```

### 4.6 Merge `devops` into `master`
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games switch master
sudo git -c safe.directory=/usr/src/kodekloudrepos/games merge devops
```
```text
Updating 5dc37bc..6266cb3
Fast-forward
 index.html | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```

### 4.7 Push both branches
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games push origin devops
sudo git -c safe.directory=/usr/src/kodekloudrepos/games push origin master
```
```text
 * [new branch]      devops -> devops
   5dc37bc..6266cb3  master -> master
```

### 4.8 Verify the remote directly
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games \
  ls-remote --heads origin devops master
```
```text
6266cb37165955aede9ba944c1431de0bee2f59a refs/heads/devops
6266cb37165955aede9ba944c1431de0bee2f59a refs/heads/master
```

### 4.9 Verify the committed file on `master`
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/games show master:index.html
```
```text
Welcome to Nautilus Group !
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `devops` branch created from `master` | ✅ |
| `index.html` copied, staged, committed on `devops` (`Add index.html`) | ✅ |
| `devops` merged into `master` via fast-forward | ✅ |
| `devops` pushed to `origin` | ✅ |
| `master` pushed to `origin` | ✅ |
| Remote `ls-remote` confirms both branches → `6266cb37...` | ✅ |
| `master:index.html` contains the expected content | ✅ |

```text
ststor01:/usr/src/kodekloudrepos/games   (local)       /opt/games.git  (remote)
 ├── master  @ 6266cb3 ───────────push──────────────▶   master  @ 6266cb3
 └── devops  @ 6266cb3 ───────────push──────────────▶   devops  @ 6266cb3
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `fatal: detected dubious ownership` | Git's own ownership-mismatch safety check | `git config --global --add safe.directory <path>`, or `git -c safe.directory=<path>` per-command |
| `fatal: cannot lock ref '...': Permission denied` | Linux filesystem permissions, NOT a Git trust issue | `sudo` for the write operation — a separate fix from `safe.directory` (§2.1) |
| `cp: Permission denied` | Destination directory not writable by current user | `sudo cp ...` with the full explicit destination path |
| `a branch named 'devops' already exists` | Branch already created in a prior step/lab | `git branch --list devops`; switch to it instead of recreating |
| `nothing to commit, working tree clean` | File not actually copied, or already committed | Re-check `ls -l index.html`; re-verify with `git status` |
| `Already up to date` during merge | Target already contains source's commits | Confirm via `git log --oneline --graph --decorate --all` |
| Push rejected, non-fast-forward | Remote has commits the local branch doesn't | `git fetch origin`, inspect before integrating — never blindly force-push |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out
      loud from "`dubious ownership`" to "verified via `ls-remote`."
- [ ] Explain, in one sentence, why fixing `safe.directory` did not
      also fix the subsequent `Permission denied` error.
- [ ] Explain why `git merge devops` has to be run *while checked out
      on* `master`, not on `devops`.
- [ ] Explain why this particular merge was a fast-forward, and what
      would have made it NOT a fast-forward.
- [ ] Explain why pushing `devops` doesn't also push `master`, even
      though both were updated locally.
- [ ] Explain why `ls-remote` is more trustworthy than `git branch -r`
      for confirming a push actually succeeded.
- [ ] From memory, create a third branch, intentionally diverge it
      from `master` with a conflicting change, attempt the merge, and
      resolve the resulting conflict — without copying any command
      from this file.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
