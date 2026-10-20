# Day 27 — Git Revert Some Changes

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Days 25-26 established `safe.directory` (Git's trust check) and
`sudo` (Linux permission) as two separate, recurring fixes for a
root-owned repository — this lab hits that exact pattern a third time,
now applied to `git revert`. The new concept is `revert` itself:
undoing a commit's *changes* while explicitly preserving the commit
that made them, as opposed to erasing history.

---

## 1. Scenario

In `/usr/src/kodekloudrepos/news` on `ststor01`, revert the latest
commit (`add data.txt file`) while keeping full history intact, with
the new commit's message exactly `revert news` (lowercase).

```text
Before:                           After:
48f947f add data.txt file         c42920e revert news
c86c537 initial commit            48f947f add data.txt file
                                   c86c537 initial commit
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 `revert` undoes *changes*; it does not undo *history*

```text
git reset --hard HEAD~1   → ERASES the commit, branch pointer moves BACKWARD
git revert HEAD            → ADDS A NEW commit that reverses the changes, pointer moves FORWARD
```

The task explicitly requires preserving history — the original
`add data.txt file` commit must still exist afterward, which rules
out `reset` entirely. `revert` is the correct tool precisely because
its whole design goal is "undo what this commit did, as a new,
visible, auditable commit" rather than "make it as if this commit
never happened." This is a fundamentally different operation from
anything touched in this series' prior Git labs (branching, merging,
remotes) — all of those only ever *added* history; this is the first
lab whose entire point is to *counteract* prior history without
erasing it.

### 2.2 `--no-edit` accepts Git's default message as a deliberate intermediate step, not the final answer

```text
git revert HEAD --no-edit
→ default message: Revert "add data.txt file"
```

`--no-edit` just skips opening an interactive editor to confirm/change
the auto-generated message — it doesn't lock that message in
permanently. The task's exact required message (`revert news`)
differs from Git's default revert phrasing, which is *why* an
`--amend` step is still needed afterward — `--no-edit` and getting the
exact final message are two separate concerns, handled by two separate
commands.

### 2.3 `commit --amend` replaces the commit's content/message, producing a *new* hash — this is expected, not a sign something broke

```text
git revert HEAD --no-edit          → creates commit X (default message)
git commit --amend -m "revert news" → REPLACES commit X with commit Y (new hash, new message)
```

Amending doesn't edit commit X in place — Git commits are immutable
once created, so "amending" is really "create a new commit with the
desired changes/message, and move the branch pointer to point at it
instead." The hash changing between the revert and the amend is the
direct, necessary consequence of that immutability, not evidence the
amend did something unintended.

### 2.4 The same `safe.directory` vs. `sudo` split from Days 25-26, a third time

```text
fatal: detected dubious ownership  → safe.directory (Git's TRUST check)
Permission denied                   → sudo (Linux FILESYSTEM permission)
```

Exactly the same two-layer problem as Day 25's commit/merge and Day
26's `remote add` — configuring trust once per user
(`git config --global --add safe.directory ...`) and then running the
actual write operations (`revert`, `commit --amend`, `push`) via
`sudo` because the repository is root-owned. Once `safe.directory` is
configured for `natasha`, it applies to every subsequent command run
*as* `natasha` — it's `sudo`'s root identity that still needs its own
handling if root-run Git commands independently hit the same trust
check (as addressed via `git -c safe.directory=...` in Days 25-26).

### 2.5 `git log` *before* reverting identifies the exact target — don't revert "the latest commit" blindly without confirming which one that is

```text
sudo git log -2 --oneline
→ 48f947f (HEAD -> master, origin/master) add data.txt file
→ c86c537 initial commit
```

The task names the commit to revert by its *message*
("add data.txt file"), and checking history first confirms that
`HEAD` really does point at that specific commit before running
`revert HEAD` — the same "inspect the exact target before acting on
it" discipline as every exact-resource check throughout this series
(Day 20's exact IAM policy, Day 22's exact SSH account path, Day 26's
exact remote name).

### 2.6 A branch being "ahead of origin" after a local revert+amend is expected — pushing is a separate, explicit step

```text
git status → "Your branch is ahead of 'origin/master' by 1 commit."
```

Both `revert` and `--amend` are purely local operations — exactly
like every commit in this series since Day 25, nothing reaches the
remote until an explicit `push`. Being "ahead" here isn't a problem to
fix by any means other than deliberately choosing to push (or not,
if the task's scope doesn't require it) — the same local-vs-remote
separation as Day 25 §2.7/Day 26 §2.2's "local operations never touch
the remote by themselves."

### 2.7 An untracked file appearing mid-task is informational, not something to act on uninvited

```text
Untracked file: news.txt
```

Git surfacing an untracked file in `status` output is just it
reporting what's on disk that it isn't tracking — it has no bearing on
the revert/amend/push sequence and shouldn't be added, deleted, or
otherwise touched unless the task explicitly asks for it. This is the
same "don't modify what wasn't asked for" discipline as Day 22 MLOps's
"delete only the extra resource, never touch the required one," here
applied to leaving an unrelated file alone entirely.

### 2.8 The compressed reasoning chain

```text
Requirement: revert the latest commit, preserve history, exact message "revert news"
   → connect, cd into repo
   → safe.directory configured ONCE for natasha (TRUST, §2.4)
   → git log -2 --oneline — CONFIRM the exact commit to revert before acting (§2.5)
   → sudo git revert HEAD --no-edit  — ADDS a new commit, does NOT erase history (§2.1, §2.2)
   → sudo git commit --amend -m "revert news"  — replaces default msg, NEW hash expected (§2.3)
   → sudo git log -3 --oneline  — verify the resulting 3-commit chain
   → sudo git status  — "ahead by 1" is expected; an untracked file is unrelated, leave it (§2.6, §2.7)
   → sudo git push origin master  — a SEPARATE, explicit step, only if remote update is required
   → sudo git ls-remote --heads origin master  — the decisive remote-side check (same as Days 25-26)
```

---

## 3. Concepts (reference)

### 3.1 `git revert` vs. `git reset`
`revert` adds a new commit that reverses an earlier commit's changes,
preserving full history. `reset` moves the branch pointer (and
optionally the working tree) to an earlier commit, effectively
discarding later history from that branch's perspective. They solve
"undo the changes, keep the record" vs. "pretend it never happened,"
respectively.

### 3.2 Commit immutability and `--amend`
Git commits, once created, are never edited in place — `--amend`
creates a *new* commit object with the desired changes and moves the
branch pointer to it, which is why the commit hash always changes
after an amend.

### 3.3 `--no-edit`
Accepts whatever commit message Git would otherwise open an editor to
let you confirm or change — a convenience for scripting, not a
guarantee that the resulting message is the one ultimately required.

### 3.4 Local vs. remote state after a revert
Reverting and amending are entirely local operations; the remote
branch remains unchanged until an explicit `push`, which is why
`status` correctly reports the local branch as "ahead" afterward.

### 3.5 Untracked files
Files present on disk but not under Git's tracking — reported by
`git status` purely as information, with no inherent relationship to
whatever commit/revert operation is currently in progress.

---

## 4. Runbook

### 4.1 Connect and navigate to the repository
```bash
ssh natasha@ststor01
cd /usr/src/kodekloudrepos/news
```

### 4.2 Configure Git trust once
```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/news
git status
```

### 4.3 Inspect the initial history
```bash
sudo git log -2 --oneline
```
```text
48f947f (HEAD -> master, origin/master) add data.txt file
c86c537 initial commit
```

### 4.4 Revert the latest commit
```bash
sudo git revert HEAD --no-edit
```

### 4.5 Set the exact required commit message
```bash
sudo git commit --amend -m "revert news"
```

### 4.6 Verify the new history
```bash
sudo git log -3 --oneline
```
```text
c42920e (HEAD -> master) revert news
48f947f (origin/master) add data.txt file
c86c537 initial commit
```

### 4.7 Check status
```bash
sudo git status
```
```text
Your branch is ahead of 'origin/master' by 1 commit.
```
An untracked `news.txt` may also appear — leave it untouched (§2.7).

### 4.8 Push the revert commit
```bash
sudo git push origin master
```
```text
To /opt/news.git
   48f947f..c42920e  master -> master
```

### 4.9 Verify the remote directly
```bash
sudo git ls-remote --heads origin master
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Latest commit reverted (not reset/erased) | ✅ |
| `c86c537` and `48f947f` still present in history | ✅ |
| New commit message exactly `revert news` (lowercase) | ✅ |
| Local `master` ahead of `origin/master` before pushing | ✅ |
| Pushed, remote `master` confirmed via `ls-remote` | ✅ |
| Untracked `news.txt` left untouched | ✅ |

```text
/usr/src/kodekloudrepos/news  (local)          /opt/news.git  (remote)
 master:                                         master:
   c42920e revert news       ───push──────────▶    c42920e revert news
   48f947f add data.txt file                        48f947f add data.txt file
   c86c537 initial commit                            c86c537 initial commit
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `fatal: detected dubious ownership` | Git's own trust check | `git config --global --add safe.directory <path>` as the user running Git |
| `Permission denied` during a write | Linux filesystem permissions, separate from the above | `sudo` for the repository command |
| Revert opens an editor unexpectedly | Ran without `--no-edit` | Save/exit, amend the message afterward if needed |
| Commit message is Git's default, not `revert news` | Only used `--no-edit`, skipped the amend step | `sudo git commit --amend -m "revert news"` |
| Branch shows "ahead of origin" | Revert/amend are local-only; not yet pushed | `sudo git push origin master` if remote update is required |
| Untracked `news.txt` appears | Unrelated file on disk | Leave it — don't add or delete unless explicitly required |
| `Everything up-to-date` on push | No new commits to send | Verify with `git ls-remote --heads origin master` |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out
      loud from "`git log -2`" to "verified via `ls-remote`."
- [ ] Explain, in one sentence, why `git revert` was the correct
      choice here instead of `git reset --hard HEAD~1`.
- [ ] Explain why the commit hash changes after
      `git commit --amend`, even though the task only asked to change
      the message.
- [ ] Explain why `git status` showing "ahead of origin by 1 commit"
      right after the amend is expected, not an error.
- [ ] Explain why the untracked `news.txt` file should be left alone.
- [ ] Explain the difference between `--no-edit` on `git revert` and
      the later `--amend` step — what does each actually control?
- [ ] From memory, make two more commits, revert the *older* of the
      two (not `HEAD`) using `git revert <commit-hash>`, and confirm
      the newer commit's changes remain intact — without copying any
      command from this file.

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
