# Day 26 — Manage Git Remotes

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 25 pushed to the *single* remote (`origin`) a repository already
had configured. This lab's new concept is that a repository can have
*multiple independent remotes* — adding a second one (`dev_blog`)
doesn't touch, replace, or affect `origin` in any way. The ownership
obstacles (`dubious ownership`, root-owned `.git`, `sudo` vs.
`safe.directory`) are the exact same two-layered problem from Day 25,
now additionally blocking `git remote add` itself, not just commits.

---

## 1. Scenario

In `/usr/src/kodekloudrepos/blog` on `ststor01`, register a second
remote `dev_blog` pointing at `/opt/xfusioncorp_blog.git`, commit
`/tmp/index.html` on `master`, push `master` to `dev_blog`
specifically, and verify the push against the remote directly.

| Property | Value |
|---|---|
| Repository | `/usr/src/kodekloudrepos/blog` |
| Existing remote | `origin` → `/opt/blog.git` (untouched) |
| New remote | `dev_blog` → `/opt/xfusioncorp_blog.git` |
| Commit message | `Add index.html` |

```text
/usr/src/kodekloudrepos/blog  (local)
      │
      ├── origin    → /opt/blog.git            (unchanged, untouched)
      └── dev_blog  → /opt/xfusioncorp_blog.git (new — receives THIS push)

git push dev_blog master
      │
      ▼
/opt/xfusioncorp_blog.git  refs/heads/master → 5ad6e2f...
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 `git push <remote> <branch>` — the remote name and the branch name are never the same kind of thing

```text
git push dev_blog master
         ^^^^^^^^  ^^^^^^
         WHERE      WHAT (local branch to send)
```

`dev_blog` is an *alias* for a destination repository, configured
locally; `master` is the *local branch* whose commits get sent. The
full unambiguous form `git push dev_blog master:master` makes explicit
that the first `master` is the *source* (local) and the second is the
*destination* (remote) — they happen to share a name here, but
nothing requires that. Misreading `dev_blog` as a branch name (or
`master` as a remote name) is the most common way to misunderstand
this command's syntax.

### 2.2 Multiple remotes are completely independent — adding one touches nothing about any other

```text
Before: origin → /opt/blog.git          (the only remote)
After:  origin → /opt/blog.git           (STILL exactly as before)
        dev_blog → /opt/xfusioncorp_blog.git  (newly added)
```

`git remote add` only appends a new name→URL entry to `.git/config` —
it has no mechanism to modify, replace, or interact with any
previously-configured remote. This is why pushing to `dev_blog` has
zero effect on `origin`'s state: they are two entirely separate
named connections, and an operation naming one never implicitly
touches the other.

### 2.3 Registering a remote records a *destination*; it transfers nothing by itself

```text
git remote add dev_blog /opt/xfusioncorp_blog.git
```

This command writes configuration only — no commits move, no
network/filesystem transfer happens, and `/opt/xfusioncorp_blog.git`
is not read or written to at all by this specific command. It's the
exact same "identity vs. action" split as Day 18's `git tag` (creates
a reference, doesn't push, commit, or rerun anything) and Day 20
AWS's `create-role` (establishes identity, not authorization) — here
applied to remote configuration instead of a resource or a release.

### 2.4 `git remote add` failing with `Permission denied` is the *same two-layer problem* as Day 25, now hitting config instead of commits

```text
error: could not lock config file .git/config: Permission denied
```

`git remote add` writes to `.git/config` — if that file (or its
directory) is root-owned and `natasha` lacks write permission, the
command fails for the exact same Linux-filesystem-permission reason
Day 25's `cp`/`commit` failed, just on a different file within the
same repository. `safe.directory` (Git's ownership *trust* check)
still does nothing here either — it's still a completely separate
concern from the actual write permission, exactly as established in
Day 25 §2.1-§2.2.

### 2.5 Root's Git config is still separate from `natasha`'s — `git -c` still the scoped fix

```text
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog remote add dev_blog ...
```

Same reasoning as Day 25 §2.3: running via `sudo` executes as root,
which has its own independent global Git config. Supplying
`safe.directory` per-invocation via `-c` avoids needing to persist
anything into root's config just to register one remote.

### 2.6 `git remote add` vs. `set-url` vs. `remove` — three distinct operations on the same configuration entry

```text
remote add <name> <url>     → CREATES a new entry — fails if <name> already exists
remote set-url <name> <url>  → UPDATES an existing entry's URL
remote remove <name>         → DELETES the local config entry ONLY
```

Re-running `add` on an already-configured name is the wrong tool —
Git will reject it precisely because `add` is specifically for
*new* entries. `remove` only deletes the local alias/config; it has
no effect whatsoever on the actual remote repository
(`/opt/xfusioncorp_blog.git`) or any commits already pushed there —
removing the local pointer doesn't un-push anything, the same
"configuration metadata vs. actual remote content are separate"
principle the runbook draws out explicitly.

### 2.7 A push's report (`[new branch] master -> master`) tells you what changed on the *remote side specifically*

```text
[new branch] master -> master   → the REMOTE had no master branch before this push
```

This message is Git reporting on the destination's state transition,
not just confirming "your local commits exist somewhere now" — it
specifically means `dev_blog`'s `master` reference didn't exist prior
to this push and was just created. A different report (e.g. a commit
range like `abc123..def456 master -> master`) would instead mean the
remote branch already existed and simply advanced.

### 2.8 `ls-remote` queries the destination directly — the decisive check, same as Day 25

```text
git ls-remote --heads dev_blog master
```

Exactly Day 25's §2.8 lesson, reapplied to a second remote: this
command asks `/opt/xfusioncorp_blog.git` itself what its `master`
reference currently points to, rather than trusting a local
remote-tracking cache (`dev_blog/master`) that could be stale if a
push partially failed or was run against the wrong remote.

### 2.9 Local `HEAD -> master, dev_blog/master` both pointing at one commit is simply the *local record* of a successful push — not independent proof

```text
5ad6e2f (HEAD -> master, dev_blog/master) Add index.html
```

`dev_blog/master` is a *local* remote-tracking reference, updated by
Git itself immediately after a successful push — it reflects what
Git *believes* the remote now has, based on the push's own result. It
is useful context, but `ls-remote` (§2.8) remains the only check that
independently confirms the actual remote state, rather than trusting
Git's own bookkeeping about that push.

### 2.10 The compressed reasoning chain

```text
Requirement: push master to a SECOND, newly-registered remote (dev_blog)
   → connect, cd into repo, check STARTING state: status, branch --show-current, remote -v
   → git remote add dev_blog <url>  — may hit the SAME ownership/permission problem as Day 25 (§2.4)
   → safe.directory fixes TRUST, sudo fixes PERMISSION — still two separate fixes (§2.4, §2.5)
   → verify registration: git remote -v  — confirm origin is UNCHANGED, dev_blog is new (§2.2)
   → cp the file (sudo if needed, same as Day 25), add, commit -m "Add index.html"
   → git push dev_blog master  — explicit remote name, explicit branch (§2.1)
   → read the push report: [new branch] vs. a commit-range update (§2.7)
   → verify via ls-remote --heads dev_blog master — the DECISIVE, remote-side check (§2.8)
   → local log --decorate showing dev_blog/master is CONFIRMATORY, not independently decisive (§2.9)
```

---

## 3. Concepts (reference)

### 3.1 Remote as a named alias
A Git remote is a local name bound to a URL/path in `.git/config` —
purely configuration, carrying no commits or content of its own until
a `push`/`fetch`/`pull` actually moves data to or from it.

### 3.2 Multiple independent remotes
A repository can have any number of remotes, each entirely
independent — adding, removing, or pushing to one has no effect on
any other.

### 3.3 `remote add` vs. `set-url` vs. `remove`
Three distinct operations: creating a new entry, updating an
existing entry's URL, and deleting a local entry — none of which
touches the actual remote repository's content, only the local
alias/configuration pointing at it.

### 3.4 Remote-tracking branch vs. the actual remote
A remote-tracking reference (e.g. `dev_blog/master`) is a local
record of what Git last observed about a remote branch. The actual,
current state of the remote is only authoritatively known by querying
the remote directly (`ls-remote`).

### 3.5 Push report semantics
Git's push output communicates the specific state transition that
occurred on the remote — `[new branch]` for a previously
nonexistent branch, or a commit-range for an existing branch that
advanced.

---

## 4. Runbook

### 4.1 Connect and inspect the starting state
```bash
ssh natasha@ststor01
cd /usr/src/kodekloudrepos/blog
git status
git branch --show-current
git remote -v
```
```text
origin  /opt/blog.git (fetch)
origin  /opt/blog.git (push)
```

### 4.2 Register the new remote
```bash
git remote add dev_blog /opt/xfusioncorp_blog.git
```
If this fails with `Permission denied` on `.git/config`:
```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/blog
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog \
  remote add dev_blog /opt/xfusioncorp_blog.git
```

### 4.3 Verify both remotes
```bash
git remote -v
git remote get-url dev_blog
```
```text
dev_blog  /opt/xfusioncorp_blog.git (fetch)
dev_blog  /opt/xfusioncorp_blog.git (push)
origin    /opt/blog.git (fetch)
origin    /opt/blog.git (push)
```
`origin` unchanged (§2.2).

### 4.4 Copy, stage, and commit the file
```bash
ls -l /tmp/index.html
cat /tmp/index.html
sudo cp /tmp/index.html /usr/src/kodekloudrepos/blog/index.html
cat index.html
```
```bash
git branch --show-current
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog add index.html
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog commit -m "Add index.html"
```
```text
5ad6e2f Add index.html
```

### 4.5 Push to the new remote specifically
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog push dev_blog master
```
```text
[new branch] master -> master
```
`dev_blog`'s `master` didn't exist before this push (§2.7).

### 4.6 Verify against the remote directly
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog \
  ls-remote --heads dev_blog master
```
```text
5ad6e2f950f483902cf1d85431e6ec2e35f45064 refs/heads/master
```

### 4.7 Confirm the local view
```bash
sudo git -c safe.directory=/usr/src/kodekloudrepos/blog log -1 --oneline --decorate
```
```text
5ad6e2f (HEAD -> master, dev_blog/master) Add index.html
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| `dev_blog` remote registered → `/opt/xfusioncorp_blog.git` | ✅ |
| `origin` remote unchanged | ✅ |
| `index.html` committed on `master` (`Add index.html`) | ✅ |
| `master` pushed to `dev_blog` specifically | ✅ |
| Remote `dev_blog`'s `master` confirmed via `ls-remote` → `5ad6e2f...` | ✅ |
| Local `dev_blog/master` tracking ref matches | ✅ |

```text
/usr/src/kodekloudrepos/blog  (local)
 ├── origin    → /opt/blog.git            (untouched throughout this lab)
 └── dev_blog  → /opt/xfusioncorp_blog.git
                     │ git push dev_blog master
                     ▼
            refs/heads/master @ 5ad6e2f950f483902cf1d85431e6ec2e35f45064
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `remote dev_blog already exists` | Remote name already configured | `git remote get-url dev_blog` to confirm its current destination before re-adding |
| `Permission denied` on `.git/config` | Linux filesystem permissions, same root cause as Day 25 | `sudo git -c safe.directory=<path> remote add ...` |
| `dubious ownership` | Git's own trust check | `safe.directory`, separate fix from the permission issue above |
| `src refspec master does not match any` | No local branch/commit named `master` to push | `git branch --show-current`, `git log -1` |
| `Everything up-to-date` | Destination already has the commits being pushed | Confirm with `git ls-remote --heads dev_blog master` |
| Push rejected, non-fast-forward | Remote branch has history the local branch lacks | `git fetch dev_blog`, inspect before integrating |
| Commit exists locally but not on `dev_blog` | Pushed to the wrong remote, or not pushed at all | `git push dev_blog master`, then `ls-remote` to confirm |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out
      loud from "`git remote add dev_blog ...`" to "verified via
      `ls-remote`."
- [ ] Explain, in one sentence, why adding `dev_blog` has zero effect
      on `origin`'s configuration or state.
- [ ] Explain why `git remote add` failing with `Permission denied`
      is the same underlying problem as Day 25's `cp`/`commit`
      failures, just hitting a different file.
- [ ] Explain the difference between `git remote remove dev_blog` and
      actually deleting commits from `/opt/xfusioncorp_blog.git`.
- [ ] Explain what `[new branch] master -> master` in a push's output
      specifically tells you, versus what a commit-range report would
      tell you instead.
- [ ] Explain why `dev_blog/master` appearing in `git log --decorate`
      is not, by itself, independent proof the push succeeded.
- [ ] From memory, register a third remote, push a different local
      branch to it under a *different* remote branch name (using the
      `local:remote` syntax), and verify it via `ls-remote` — without
      copying any command from this file.

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
