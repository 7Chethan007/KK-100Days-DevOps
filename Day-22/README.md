# Day 22 — Clone a Git Repository on the Storage Server

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 21 created the bare central repository at `/opt/demo.git`. This
lab is its direct sequel: cloning that same repository into a working
copy — the other half of the "Developer A/B push/pull against the
central bare repo" diagram from Day 21 §6. It also repeats, in a new
form, Day 21's own lesson about trusting a stated path over verifying
the actual server state.

---

## 1. Scenario

As the `natasha` user on the Storage Server (`ststor01`), clone the
existing repository at `/opt/demo.git` into
`/usr/src/kodekloudrepos`, without modifying the source repository or
any existing permissions.

| Parameter | Value |
|---|---|
| Storage server user | `natasha` |
| Source repository | `/opt/demo.git` |
| Destination parent | `/usr/src/kodekloudrepos` |
| Expected clone directory | `/usr/src/kodekloudrepos/demo` |

The task text named `/opt/games.git`, but the actual server had
`/opt/demo.git` — the repository this runbook documents is the one
that was actually present and successfully cloned.

```text
/opt/demo.git  (bare, from Day 21)
      │  git clone /opt/demo.git   (run FROM /usr/src/kodekloudrepos)
      ▼
/usr/src/kodekloudrepos/demo/
      ├── .git/   (origin → /opt/demo.git)
      └── (working tree — empty, since the source has no commits)
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 A stated path in a task description is a claim, not a fact — check the server before acting on it

```text
Task text said:     /opt/games.git
Server actually had: /opt/demo.git
```

The correct response to a missing path was to inspect the server
(`ls -ld`), not to invent the missing repository or silently
substitute a different one without confirming it's the one the task
actually means. This is the exact same discipline Day 21 built around
its own typo (`demo.gitv` vs `demo.git`) — a path in a requirement is
only ever as good as what's actually checked against, never assumed.

### 2.2 Cloning a local filesystem path is still "cloning," just without a network hop

```text
git clone /opt/demo.git     ← source is a LOCAL path, same machine
git clone user@host:/path   ← source is a REMOTE path, over SSH
git clone https://...        ← source is a REMOTE path, over HTTPS
```

Git's clone mechanics (copying objects/refs, configuring a remote
named `origin`, checking out a default branch if one exists) are
identical regardless of whether the source is local or remote — only
the transport differs. This matters here because `/opt/demo.git` and
`/usr/src/kodekloudrepos` are both ordinary filesystem paths on the
*same* server; nothing about SSH or a URL scheme is involved in the
clone operation itself, only in *reaching* `ststor01` in the first
place.

### 2.3 Omitting the destination argument derives the directory name from the source's last path component

```text
git clone /opt/demo.git        (run from /usr/src/kodekloudrepos)
   → creates /usr/src/kodekloudrepos/demo/   (strips the trailing .git)
```

versus:

```text
git clone /opt/demo.git /usr/src/kodekloudrepos
   → clones DIRECTLY INTO /usr/src/kodekloudrepos itself — no "demo" subdirectory
```

These produce genuinely different directory structures from the same
source — the lab's required structure
(`/usr/src/kodekloudrepos/demo/`) depends specifically on *omitting*
the destination argument while already being inside the parent
directory, not on supplying the parent as an explicit destination
argument. Getting this backwards would still produce "a successful
clone," just at the wrong resulting path — another instance of §2.1's
general pattern.

### 2.4 "Empty repository" is a property of the source having no commits, not a cloning failure

```text
warning: You appear to have cloned an empty repository.
```

This warning reflects that `/opt/demo.git` (as created in Day 21, via
plain `git init --bare`, with nothing committed to it) has no commit
history to check out — Git still successfully creates the destination
repository and configures `origin` correctly. Adding a commit to the
source purely to silence this warning would violate the lab's
explicit constraint not to modify the source repository; the warning
is informational, not something to engineer around.

### 2.5 `rev-parse --is-inside-work-tree` and `remote -v` verify two different claims

```text
rev-parse --is-inside-work-tree  → IS this directory a Git working tree at all?
remote -v                         → does origin point at the CORRECT source?
```

A directory could be a perfectly valid Git working tree while its
`origin` points at the wrong repository (if, say, the clone command
had been run against a different source by mistake) — these are
independently-checkable facts, the same "existence vs. correctness"
split as Day 21's `ls -ld` (directory exists) vs.
`rev-parse --is-bare-repository` (Git recognizes it as bare) check on
the source side.

### 2.6 `-C <path>` makes a Git command location-independent

```bash
git -C /usr/src/kodekloudrepos/demo rev-parse --is-inside-work-tree
```

`-C` tells Git to behave as though the current directory were
`<path>`, without an actual `cd`. This matters for running
verification commands reliably from a script or a different shell
location than the repository itself, rather than depending on the
caller already being inside the right directory.

### 2.7 Running the clone as `natasha`, not `sudo`, preserves the intended ownership model

```text
sudo git clone ...   → files could end up owned by root, depending on how sudo is invoked
natasha git clone ... → files owned by natasha, matching the task's explicit account requirement
```

The lab specifically requires cloning *as* `natasha` — using `sudo`
risks silently changing file ownership away from what the task
requires, for no benefit the task actually asked for. This is the
same "don't reach for elevated privilege beyond what's explicitly
needed" discipline as every prior lab's careful, scoped use of `sudo`
only where a specific destination (like `/opt` in Day 21) genuinely
required it.

### 2.8 The compressed reasoning chain

```text
Requirement: clone /opt/demo.git into /usr/src/kodekloudrepos AS natasha
   → connect as natasha, confirm whoami/hostname (right account, right machine)
   → ls -ld the SOURCE path before trusting the task description's stated path (§2.1)
   → ls -ld the destination PARENT, confirm it exists
   → cd into the parent, THEN git clone <source> with NO destination argument (§2.3)
   → an "empty repository" warning is expected here — do NOT modify the source to fix it (§2.4)
   → verify: rev-parse --is-inside-work-tree (IS a working tree) (§2.5)
   → verify: remote -v (origin points at the CORRECT source) (§2.5)
   → run the clone as natasha, never sudo, to preserve ownership (§2.7)
```

---

## 3. Concepts (reference)

### 3.1 Cloning a local filesystem path
`git clone` treats a plain filesystem path as a valid source exactly
like a remote URL — Git's internal clone mechanics don't change based
on transport; only how the source is *reached* differs.

### 3.2 Destination argument vs. derived directory name
Supplying an explicit destination path to `git clone` clones directly
into that path. Omitting it derives a new subdirectory name from the
source's final path component (with a trailing `.git` stripped),
created inside the current working directory.

### 3.3 `origin` as a conventional, not special, remote name
`git clone` names the source remote `origin` by convention — there's
nothing privileged about that name beyond being the default a plain
clone assigns.

### 3.4 Empty repository clones
A repository with no commits has nothing to check out into a working
tree. Cloning it still succeeds and still configures the remote
correctly; the resulting working tree is simply empty until something
is committed and pushed.

### 3.5 `-C <path>` for location-independent Git commands
Running `git -C <path> <subcommand>` executes as though the shell had
already changed into `<path>`, without needing an actual `cd` or
depending on the caller's current directory.

---

## 4. Runbook

### 4.1 Connect to the Storage Server as `natasha`
```bash
sshpass -p 'Bl@kW' ssh \
  -o StrictHostKeyChecking=accept-new \
  natasha@ststor01
```
```bash
whoami
hostname
```
```text
natasha
ststor01
```

### 4.2 Inspect the source and destination before cloning
```bash
ls -ld /opt/demo.git
ls -ld /usr/src/kodekloudrepos
```
Confirmed `/opt/demo.git` exists (owned by `natasha`), and the
destination parent `/usr/src/kodekloudrepos` exists — the task text's
stated `/opt/games.git` did not exist on this server.

### 4.3 Clone the repository
```bash
cd /usr/src/kodekloudrepos
git clone /opt/demo.git
```
Or in one command from the jump host:
```bash
sshpass -p 'Bl@kW' ssh natasha@ststor01 \
  'cd /usr/src/kodekloudrepos && git clone /opt/demo.git'
```
```text
warning: You appear to have cloned an empty repository.
```
Expected and not a failure (§2.4) — the source has no commits.

### 4.4 Verify the clone is a valid working tree
```bash
git -C /usr/src/kodekloudrepos/demo rev-parse --is-inside-work-tree
```
```text
true
```

### 4.5 Verify `origin` points at the correct source
```bash
git -C /usr/src/kodekloudrepos/demo remote -v
```
```text
origin  /opt/demo.git (fetch)
origin  /opt/demo.git (push)
```

### 4.6 Confirm the directory and repository status
```bash
ls -ld /usr/src/kodekloudrepos/demo
git -C /usr/src/kodekloudrepos/demo status
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Connected to `ststor01` as `natasha` | ✅ |
| Verified actual source path (`/opt/demo.git`, not the task text's `/opt/games.git`) | ✅ |
| Cloned into `/usr/src/kodekloudrepos` with no explicit destination argument | ✅ |
| `/usr/src/kodekloudrepos/demo` created | ✅ |
| `rev-parse --is-inside-work-tree` → `true` | ✅ |
| `origin` → `/opt/demo.git` (fetch + push) | ✅ |
| Source repository and existing permissions left unmodified | ✅ |

```text
ststor01
 ├── /opt/demo.git                        (bare, from Day 21 — unchanged)
 └── /usr/src/kodekloudrepos/
       └── demo/                           (cloned working copy)
              └── .git/  origin → /opt/demo.git
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `repository ... does not exist` | Source path missing, or stated path in the task doesn't match the server | `ls -ld` the actual server state before assuming the path is wrong |
| `destination path ... already exists` | A non-empty directory already occupies the target clone path | Inspect the existing directory; don't delete it blindly |
| `You appear to have cloned an empty repository` | Source has no commits | Expected for a freshly `git init --bare`'d source; not an error to fix |
| SSH host-authenticity prompt | Host key not yet in `known_hosts` | Verify the fingerprint, accept only the intended host |
| `Permission denied` | Wrong account, or authentication failed | Confirm the login account; don't change permissions without authorization |
| `origin` points at an unexpected path | Clone was run against the wrong source | `remote -v`, compare against the task's required source path |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.8) out loud
      from "connect as `natasha`" to "verified via `remote -v`."
- [ ] Explain, in one sentence, why `git clone /opt/demo.git` run from
      inside `/usr/src/kodekloudrepos` produces a different result
      than `git clone /opt/demo.git /usr/src/kodekloudrepos`.
- [ ] Explain why "cloned an empty repository" is expected here and
      not something to fix by modifying the source.
- [ ] Explain why `rev-parse --is-inside-work-tree` and `remote -v`
      check two different things, and why passing one doesn't imply
      the other passed too.
- [ ] Explain why running the clone as `natasha` instead of with
      `sudo` matters for this specific lab's requirement.
- [ ] From memory, clone `/opt/demo.git` a second time into a
      *different* destination name (e.g. `demo-copy`), confirm both
      clones' `origin` point at the same source, then make a commit
      in one, push it, and fetch it into the other — without copying
      any command from this file.

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
