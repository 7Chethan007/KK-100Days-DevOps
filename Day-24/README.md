# Day 24 — Create a Git Branch from Master

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each action exists and *how you'd derive it yourself*.

Day 23 forked a repository at the *platform* layer. This lab returns to
the bare local Git CLI and isolates the smallest Git operation in the
series: creating a branch. Its whole point is that a branch is a
**reference** — a movable pointer to a commit — not a copy of code. The
correct result is therefore a new *name* pointing at the *same commit*
as `master`, with **zero** file changes.

---

## 1. Scenario

Create a branch `xfusioncorp_demo` from `master` in
`/usr/src/kodekloudrepos/demo` on the Storage Server (`ststor01`),
without modifying any application code.

| Property | Required value |
|---|---|
| Server | Storage Server (`ststor01`) |
| Repository | `/usr/src/kodekloudrepos/demo` |
| Source branch | `master` |
| New branch | `xfusioncorp_demo` |
| Code changes | None |
| New commit | Not required |
| Push to remote | Not required |

```text
Before                          After
master ─▶ commit C              master ─────────────┐
                                                    ▼
                                                 commit C
                                                    ▲
                                                    │
                                xfusioncorp_demo ───┘
```
Both refs point at the same commit; no files change.

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 The exact machine and directory are part of the requirement

```text
Jump Host ──ssh──▶ ststor01 ──cd──▶ /usr/src/kodekloudrepos/demo ──git──▶ ...
```

Running Git on the wrong host or in the wrong directory touches the
wrong repository (or none). Before anything else, `pwd` answers *"am I
in the right directory?"* and `git status` answers *"does Git recognize
this as a repository?"* — two different questions. A directory merely
*named* `demo` is not proof it is a Git repo. `fatal: not a git
repository` means Git found no `.git` context from the current
directory; it says nothing about whether branch creation is valid.

### 2.2 A branch is a reference, not a copy

```text
git branch xfusioncorp_demo master
```

writes one new pointer (`refs/heads/xfusioncorp_demo`) at the commit
`master` currently points to. No objects are copied, no second working
directory appears — which is exactly why branch creation is
instantaneous. The two names share history now and can diverge later:

```text
master            → C → D
xfusioncorp_demo  → C → E
```

### 2.3 Name the source branch explicitly

```text
git branch xfusioncorp_demo          # created at whatever HEAD currently is
git branch xfusioncorp_demo master   # created at master, regardless of current checkout
```

The task says "from `master`". The explicit form states the source *in
the command itself* instead of relying on being checked out on the right
branch — so the requirement is expressed, not assumed.

### 2.4 Do only the requested operation

Not needed here: `git add` (stages), `git commit` (records a commit),
`git push` (publishes to a remote). The task asks for a **local
reference** only. An empty `git status --short` is the evidence that
nothing else was touched (it can't prove a file was never edited and
restored, but it's the observable check).

### 2.5 Existence ≠ correctness — verify the *starting commit*

```text
git branch                                          → name exists        (weak)
git show-ref --verify refs/heads/xfusioncorp_demo   → exact local ref    (stronger)
git rev-parse master ; git rev-parse xfusioncorp_demo → SAME commit ID?  (decisive)
```

A branch appearing in `git branch` does **not** prove it was created
from `master` — it could point anywhere. Comparing the two resolved
commit IDs does, which is why the lab's acceptance property is
"`xfusioncorp_demo` points at the commit `master` points at", not merely
"a branch of that name exists".

### 2.6 Compressed reasoning chain

```text
Requirement: xfusioncorp_demo from master, no code changes
   → ssh to ststor01, cd /usr/src/kodekloudrepos/demo
   → pwd + git status — confirm directory AND repository context (§2.1)
   → git switch master — make the source explicit (§2.3)
   → if refs/heads/xfusioncorp_demo exists: VERIFY it, don't recreate (§2.5)
   → else: git branch xfusioncorp_demo master (§2.2, §2.3)
   → verify: show-ref --verify + equal rev-parse commit IDs (§2.5)
   → git status --short — confirm no working-tree changes (§2.4)
   → submit
```

---

## 3. Concepts (reference)

### 3.1 Branch as a reference
A branch is a movable pointer to a commit, stored as `refs/heads/<name>`.
Creating one writes a new ref and copies no objects.

### 3.2 `HEAD`
Identifies the current checkout context — normally `refs/heads/master`
after switching. Creating a branch *without* checking it out leaves HEAD
unchanged, which is why the `*` stays on `master`.

### 3.3 Local vs. remote-tracking refs
`refs/heads/xfusioncorp_demo` is a **local** branch. `origin/master` is a
local *record* of the remote's state. Creating a local branch does not
publish it — `git push` (not required here) is what would.

### 3.4 `git branch` vs. `git switch` / `git checkout -b`
`git branch NAME master` creates **without** switching. `git switch -c
NAME` / `git checkout -b NAME` create **and** switch. Only the first
matches "create from `master`, stay on `master`".

### 3.5 `git rev-parse` and `git show-ref --verify`
`rev-parse <ref>` resolves a ref to its underlying object ID.
`show-ref --verify <full-ref>` asserts an *exact* reference exists.
Together they prove both existence and the correct starting commit.

### 3.6 Idempotency ≠ correctness
Re-running a creation command fails if the branch exists, so a safe
script checks first. But "already exists" is not automatically "correct":
if it points at the wrong commit, report it rather than silently
overwrite — the lab forbids code changes and the branch may hold work.

---

## 4. Runbook — reusable instructions

Generic form; substitute `<repo>`, `<source>`, `<branch>` for other labs.
Nothing here is lab-specific output — see **§5** for that.

### 4.1 Connect and enter the repo
```bash
ssh natasha@ststor01
cd /usr/src/kodekloudrepos/demo
pwd
git status
```

### 4.2 Confirm the source branch
```bash
git branch          # lists local branches; * marks the current one
git switch master   # make the source explicit
```

### 4.3 Create the branch — only if absent (idempotent)
```bash
if git show-ref --verify --quiet refs/heads/xfusioncorp_demo; then
  echo "Already exists: xfusioncorp_demo"
else
  git branch xfusioncorp_demo master
fi
```

### 4.4 Verify
```bash
git show-ref --verify refs/heads/xfusioncorp_demo

test "$(git rev-parse master)" = "$(git rev-parse xfusioncorp_demo)" \
  && echo "PASS: branch points at the same commit as master"

git branch --list master xfusioncorp_demo
git status --short
```

### 4.5 Submit
Run the lab page's **Check / Submit** — the platform checker is the final
acceptance test; local Git checks are necessary but not the grader.

---

## 5. Observed results — this lab (actual)

Reference transcript, not instructions.

```text
# directory / repository context settled first; an earlier malformed
# script had produced, before entering the repo:
fatal: not a git repository (or any of the parent directories): .git

# source branch
$ git switch master
Already on 'master'
Your branch is up to date with 'origin/master'.

# creation (branch had already been made in an earlier attempt)
Branch already exists: xfusioncorp_demo

# verification
$ git show-ref --verify refs/heads/xfusioncorp_demo
9718de20453548b39987f16d8e0d1ef00a002ae5 refs/heads/xfusioncorp_demo

$ git rev-parse master
9718de20453548b39987f16d8e0d1ef00a002ae5
$ git rev-parse xfusioncorp_demo
9718de20453548b39987f16d8e0d1ef00a002ae5

PASS: branch points at the same commit as master
=== Final verification ===
SUCCESS: Branch creation and verification completed.
```

```text
$ git branch --list master xfusioncorp_demo
* master
  xfusioncorp_demo

$ git status --short
(no output)
```

A later script printed `bash: USER: unbound variable` *after* its
`SUCCESS` line — a shell/heredoc error unrelated to the Git checks
(see §7). The final interactive run ended with:
```text
SUCCESS: xfusioncorp_demo verified
Connection to ststor01 closed.
```

---

## 6. Final state

| Requirement | Verification | Result |
|---|---|---|
| Correct server | connected to `ststor01` | Pass |
| Correct repository | `pwd` | Pass |
| Source branch | `git switch master` | Pass |
| Required branch exists | `git show-ref --verify` | Pass |
| Starts from `master` | equal commit IDs | Pass |
| No working-tree changes | `git status --short` empty | Pass |
| Code modifications | none required | satisfied |
| Push to remote | not required | not performed |

```text
ststor01 : /usr/src/kodekloudrepos/demo
  refs/heads/master            ─┐
                                 ├─▶ 9718de20453548b39987f16d8e0d1ef00a002ae5
  refs/heads/xfusioncorp_demo  ─┘
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `fatal: not a git repository` | Ran outside the repo, or no `.git` context | `pwd`, `cd` to the exact path, retry `git status` |
| `pathspec 'master' did not match` | No local `master`; state differs from expectation | `git branch -a`, inspect available branches |
| `branch named 'xfusioncorp_demo' already exists` | Branch already present | Verify it; compare commit IDs — do not recreate |
| Branch exists but commit IDs differ | Created from another commit, or advanced since | Inspect history; do not overwrite blindly |
| `git show-ref --verify` fails | Exact local reference missing | Recheck repository path and branch list |
| `sudo` prompt breaks a heredoc script | `sudo` and the remote script compete for stdin | Use an interactive session; separate authentication |
| `bash: USER: unbound variable` after `SUCCESS` | Malformed/unterminated heredoc leaked commands into the wrong shell context | Check the heredoc delimiter and quoting; rerun verification separately |
| Lab fails though the branch exists | Wrong server/repo/branch, or another acceptance condition | Recheck server, path, exact branch name, commit IDs |

Actual lessons: the first script hit `fatal: not a git repository`
*before* entering the correct directory, then succeeded once in the right
repo; a later script emitted `USER: unbound variable` **after** a
`SUCCESS` print — a script bug, not a Git failure.

---

## 8. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.6) out loud
      from "identify the server" to "verified via equal commit IDs."
- [ ] In one sentence: why must the Git commands run on `ststor01`, not
      the Jump Host?
- [ ] What is the difference between a branch and a working directory?
- [ ] Why does `git branch xfusioncorp_demo master` express the
      requirement better than `git branch xfusioncorp_demo` alone?
- [ ] Why does the `*` stay next to `master` after creating the branch?
- [ ] Why are matching commit IDs stronger evidence than the branch
      merely appearing in `git branch`?
- [ ] What is the difference between `refs/heads/xfusioncorp_demo` and a
      remote-tracking reference?
- [ ] Why is `git push` unnecessary for this lab?
- [ ] If the branch already exists but points to another commit, what do
      you inspect before taking any action?
- [ ] What does an empty `git status --short` tell you — and what does
      it *not* prove?
- [ ] Why can a script print `SUCCESS` and still return an error after?
- [ ] Practice: create `feature_logging` from `develop` in a *different*
      repository. Derive each step; don't just swap branch names in a
      script without checking which repo and source commit apply.

---

## 9. Document template for future lab days

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
