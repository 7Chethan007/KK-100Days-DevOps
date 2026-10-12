# Day 21 — Set Up a Bare Git Repository

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This lab is shorter than most in the series, but it's a direct
illustration of the "exact path matters, a succeeding command doesn't
prove you hit the right target" lesson that's run through every prior
lab since Day 19's `appsv` typo — here it happens to Git itself,
creating a perfectly valid, perfectly wrong repository one keystroke
away from the required one.

---

## 1. Scenario

Install Git on the Storage Server (`ststor01`) and create a bare
repository at the exact path `/opt/demo.git`.

```text
ststor01
 └── /opt/demo.git  (bare — no working tree, just Git's internal objects/refs)
        │
        ▼
 git --git-dir=/opt/demo.git rev-parse --is-bare-repository
        │
        ▼
      "true"
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 What "bare" actually removes, not just what it adds

```text
Normal repo:  working tree (editable files)  +  .git/ (internal data)
Bare repo:    .git's internal data ONLY — no working tree at all
```

A bare repository isn't a normal repository with extra restrictions —
it's a normal repository's `.git` *contents* (`HEAD`, `config`,
`objects/`, `refs/`) placed directly at the top level, with no
checked-out files sitting alongside them. That absence is exactly
what makes it suitable as a shared central remote: there's no single
developer's working copy pretending to be the canonical source, which
is the problem a bare repo is specifically designed to avoid.

### 2.2 The `.git` suffix is a naming convention; `--bare` is the actual mechanism

```text
/opt/demo.git      ← just a directory name ending in ".git" — convention, not magic
--bare             ← the flag that actually omits the working tree
```

Naming a directory `something.git` doesn't make Git treat it as bare
— plenty of ordinary (non-bare) repositories are also conventionally
named with a `.git` suffix in some workflows. The flag is what
determines the repository's actual structure; the name is purely a
human/tooling convention for recognizing bare repos at a glance.

### 2.3 A single mistyped character creates a different, equally valid target

```text
Required:   sudo git init --bare /opt/demo.git
Typed:      sudo git init --bare /opt/demo.gitv
```

`git init` doesn't know or care that `/opt/demo.gitv` wasn't the
intended path — Linux treats it as a completely distinct, valid
directory, and `git init --bare` succeeds there exactly as it would
anywhere else. The command returning success is therefore *zero*
evidence that the required resource now exists — this is the Git
analogue of Day 19's `appsv`/`apps` mismatch: a successful operation
at the wrong exact path is indistinguishable, from the command's own
output, from a successful operation at the right one.

### 2.4 Why `rev-parse --is-bare-repository` is the decisive check, not `ls`

```text
ls -ld /opt/demo.git           → proves the directory EXISTS (and its permissions)
rev-parse --is-bare-repository → proves Git itself recognizes it as a BARE repository
```

A directory existing at the right path is necessary but not
sufficient — you'd also want confirmation that Git considers it a
valid, bare repository rather than, say, an empty directory someone
`mkdir`'d by hand. `--git-dir=<path>` explicitly tells Git which
repository's metadata to inspect (rather than searching upward from
the current directory, which is Git's normal behavior), and
`rev-parse --is-bare-repository` asks the specific yes/no question
the lab requirement is actually about.

### 2.5 `sudo` here is about the destination's permissions, not Git's behavior

```text
/opt  → typically writable only by root or privileged users
```

`sudo git init --bare /opt/demo.git` needs elevated privileges purely
because `/opt` isn't writable by an unprivileged user by default —
`git init --bare` itself behaves identically with or without `sudo`;
the privilege requirement comes entirely from the filesystem location
chosen, not from anything about bare-repository initialization.

### 2.6 The compressed reasoning chain

```text
Requirement: bare Git repo at the EXACT path /opt/demo.git
   → sudo yum install -y git (sudo needed for package install, same as every prior dnf/yum lab)
   → git --version to confirm installation succeeded
   → sudo git init --bare /opt/demo.git  (sudo needed because /opt requires privilege, §2.5)
   → double-check the typed path character-by-character before running — a typo creates
     a DIFFERENT, equally "successful" repository elsewhere (§2.3)
   → verify with git --git-dir=/opt/demo.git rev-parse --is-bare-repository → "true" (§2.4)
   → ls -ld /opt/demo.git as a secondary existence/permissions check, not the primary proof
```

---

## 3. Concepts (reference)

### 3.1 Bare vs. non-bare repository
A non-bare repository pairs a working tree with a `.git` metadata
directory. A bare repository contains only that metadata — no working
tree — making it suitable as a shared central remote that multiple
developers push to and fetch from, rather than one developer's own
working copy doubling as the shared source.

### 3.2 Git's internal structures
`HEAD` identifies the default branch reference; `config` stores
repository-specific settings; `objects/` stores Git's content-
addressed objects (blobs, trees, commits); `refs/` stores named
pointers to commits (branches, tags). A bare repository has all of
these at its top level, with nothing else alongside them.

### 3.3 `--git-dir` as an explicit override
By default, Git commands search upward from the current directory to
find a `.git` directory. `--git-dir=<path>` bypasses that search,
telling Git exactly which repository's metadata to operate on —
useful for inspecting a repository you're not currently inside of.

### 3.4 Exact-path discipline
A command succeeding at *some* path is not evidence it succeeded at
the *required* path. Verification needs to target the exact path the
requirement names, not just confirm "a thing was created somewhere
plausible."

---

## 4. Runbook

### 4.1 Install Git
```bash
sudo yum install -y git
git --version
```
```text
git version 2.52.0
```

### 4.2 Create the bare repository at the exact required path
```bash
sudo git init --bare /opt/demo.git
```

### 4.3 Verify Git recognizes it as bare
```bash
git --git-dir=/opt/demo.git rev-parse --is-bare-repository
```
```text
true
```

### 4.4 Verify the exact directory exists
```bash
ls -ld /opt/demo.git
```
```text
drwxr-xr-x 6 root root ... /opt/demo.git
```
`root root` ownership is expected here because the repository was
created with `sudo`; a real shared repository's ownership/permissions
should be set according to the team's actual access model.

---

## 5. A mistake encountered, and how it was caught

```bash
sudo git init --bare /opt/demo.gitv
```
Created `/opt/demo.gitv` — a distinct, valid, successfully-initialized
bare repository at the *wrong* path. The required repository was then
created separately at the correct path (§4.2) and independently
verified (§4.3-4.4). Linux treats `demo.git` and `demo.gitv` as
completely unrelated directory names; nothing about the typo produced
an error, which is exactly why exact-path verification (§2.4), not
just "the command succeeded," is the thing that catches this.

---

## 6. Final state

| Requirement | Result |
|---|---|
| Git installed via `yum` | ✅ |
| Bare repository created at the exact path `/opt/demo.git` | ✅ |
| `rev-parse --is-bare-repository` → `true` | ✅ |
| No stray `/opt/demo.gitv` left mistaken for the required repo | ✅ (created the correct one separately) |

```text
ststor01
 └── /opt/demo.git  (bare, root:root)
        ├── HEAD
        ├── config
        ├── objects/
        └── refs/
```

```text
Developer A                    Central bare repo                 Developer B
edit → commit → push  ───────▶   /opt/demo.git   ◀───────  clone → fetch → pull
```

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `rev-parse --is-bare-repository` fails or returns `false` | Wrong `--git-dir` path, or the directory isn't actually a Git repository | Re-check the exact path; re-run `git init --bare` at the correct location if needed |
| A directory exists at a similar but wrong path (e.g. `demo.gitv`) | Typo during `git init --bare` | Create the correctly-named repository separately; leave or clean up the stray one deliberately |
| `Permission denied` creating under `/opt` | Missing `sudo` — `/opt` typically requires elevated privileges | Re-run with `sudo` |
| `git --version` not found after install | Install step didn't complete, or `yum` targeted the wrong package | Re-run `sudo yum install -y git`, check for errors in the output |
| Developers can't push to the bare repo | Remote/SSH access or permissions not yet configured — a separate step from creating the repo | Configure server-side access (SSH keys, repository permissions) per the team's access model |

---

## 8. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.6) out loud
      from "install Git" to "verified via `rev-parse
      --is-bare-repository`."
- [ ] Explain, in one sentence, what a bare repository is missing
      compared to a normal one, and why that absence is the point.
- [ ] Explain why the `.git` suffix in `/opt/demo.git` is not what
      makes the repository bare.
- [ ] Explain why `sudo git init --bare /opt/demo.gitv` succeeding
      doesn't prove the lab requirement was satisfied.
- [ ] Explain the difference between `ls -ld /opt/demo.git` and
      `git --git-dir=/opt/demo.git rev-parse --is-bare-repository` —
      what does each actually prove?
- [ ] From memory, create a second bare repository at a different
      path, clone it, make a commit in the clone, push it back, and
      confirm the push landed — without copying any command from this
      file.

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
