# Day 8 — Install Ansible

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The Nautilus DevOps team is starting to test Ansible, using the **jump
host** as the Ansible controller against the app servers. Task:

1. Install **Ansible `4.7.0`** on the jump host using **`pip3` only**.
2. Ensure the `ansible` binary is available **globally** — every user on
   the system must be able to run Ansible commands.

```text
                 Jump Host
              Ansible Controller
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       stapp01   stapp02   stapp03
```

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Inspect the environment before installing anything

```bash
python3 --version
pip3 --version
ansible --version 2>/dev/null || echo "Ansible is not installed"
```

Before installing, confirm which Python and `pip3` you're actually
targeting, and whether Ansible is already present under a different
version — installing blind risks either duplicating an existing install
or installing against the wrong Python interpreter entirely (the same
"confirm which environment you're actually in" habit as `which
python`/`which jupyter` in the JupyterLab lab, and `which pip` reasoning
in the `uv`/venv guide).

### 2.2 "Install with pip3 only" is itself a constraint worth taking literally

The task specifies `pip3`, not "however you'd normally install Ansible"
— on many systems Ansible is also installable via the OS package manager
(`dnf`/`apt`), which would land at a different version, in a different
location, managed by a different tool entirely. Taking the stated
constraint literally (`sudo pip3 install ansible==4.7.0`, not `dnf
install ansible`) matters here because the *verification* later depends
on where `pip3` actually put things (§2.5).

### 2.3 Why `ansible --version` shows a different number than what you installed

```bash
sudo pip3 install ansible==4.7.0
ansible --version
```
```text
ansible [core 2.11.12]
```

This isn't a contradiction — the `ansible` PyPI package (`4.7.0`) is a
distribution that bundles/depends on a specific `ansible-core` version
(`2.11.12`) as its actual engine:

```text
ansible (the package you asked pip for)   4.7.0
        │
        └── depends on
                │
                ▼
        ansible-core (the actual CLI/engine)   2.11.12
```

`ansible --version` reports the **core** engine version, because that's
literally what the CLI binary belongs to — checking that you got the
requested `4.7.0` requires asking `pip` directly, not the CLI:

```bash
pip3 show ansible | grep '^Version:'
```
```text
Version: 4.7.0
```

This is the same "the tool's own version output isn't always the thing
the requirement asked about" trap as checking a wheel's `.dist-info`
metadata directly rather than trusting a build tool's summary line
(MLOps Day 7).

### 2.4 What `pip3 install` actually does — two separate outputs, two separate locations

```text
pip3 install ansible==4.7.0
       │
       ├── Python library files  →  /usr/local/lib/python3.9/site-packages/ansible
       └── CLI executable         →  /usr/local/bin/ansible
```

The library is what Python imports internally; the executable is a
separate, generated entry-point script that the shell actually runs when
you type `ansible`. "Global availability" (the task's second
requirement) is a question about the *second* thing — where that
executable landed, and who can reach it.

### 2.5 "Available system-wide" decomposes into two independent questions

```text
1. Can every user EXECUTE the binary file itself?    → filesystem permissions
2. Can every user's shell FIND the binary at all?    → PATH
```

These are genuinely separate — a binary can be perfectly executable and
still be invisible to a user whose shell never looks in the directory it
lives in, and vice versa (though the second case is rarer). Both must be
true for "all users can run Ansible commands" to actually hold.

### 2.6 Checking executability

```bash
ls -l /usr/local/bin/ansible
```
```text
-rwxr-xr-x 1 root root ... /usr/local/bin/ansible
```

```text
-rwxr-xr-x
 │   │   │
 │   │   └── others: r-x  (read + execute)
 │   └────── group: r-x
 └────────── owner: rwx
```

Numerically `755` — owner can read/write/execute, group and others can
read/execute. This is exactly the conventional "public executable" mode
from the file-permissions guide (Day 4) — any user, not just root, can
execute this binary once their shell can locate it.

### 2.7 Checking `PATH` — and discovering it isn't the same everywhere

```bash
echo "$PATH"       # as thor, normal shell
```
```text
.../usr/local/bin...
```
`/usr/local/bin` is present — so `thor`'s ordinary shell finds `ansible`
fine. But:
```bash
sudo env | grep '^PATH='
```
```text
PATH=/sbin:/bin:/usr/sbin:/usr/bin
```
`sudo`'s own environment uses a **different, more restrictive `PATH`**
that doesn't include `/usr/local/bin` at all — this is a deliberate
`sudo` security default (`secure_path` in `/etc/sudoers`), not a bug or
an installation failure. The consequence:
```bash
sudo ansible --version
```
```text
sudo: ansible: command not found
```
while the binary itself is completely fine:
```bash
sudo -u root /usr/local/bin/ansible --version
```
works, because giving the **full path** bypasses `PATH` lookup entirely.

### 2.8 Why you should *not* "fix" this by editing `/etc/sudoers`

The task's requirement is "all users on this system are able to run
Ansible commands" — it does not say "running `ansible` through `sudo`
must also work identically to a plain user shell." `sudo`'s restricted
`PATH` is an intentional hardening default (limiting which binaries a
privilege-escalated command can silently pick up); loosening it globally
just to make `sudo ansible` type-comfortable is a disproportionate,
unrequested change with real security implications. Recognizing "this
observed behavior doesn't actually violate the stated requirement" is
itself part of getting this lab right — not every surprising finding
needs remediation.

### 2.9 The `nobody` test — a different failure, at a different layer

```bash
sudo -u nobody /usr/local/bin/ansible --version
```
```text
PermissionError: [Errno 13] Permission denied
/.ansible/tmp
```

This is **not** "Ansible isn't installed" or "the binary isn't
executable" — the process started, found the binary, and began running;
it failed only when it tried to create a working-directory file under a
home directory `nobody` doesn't have a normal writable one for. This is
a useful distinction to practice:

```text
"command not found"        → PATH/lookup problem, or binary genuinely missing
"permission denied on X"   → the program STARTED, then failed doing something else
```

Different error shapes point at completely different layers of the
stack — worth reading the actual error rather than pattern-matching
"Ansible isn't working" onto every failure.

### 2.10 The compressed reasoning chain

```text
Requirement (Ansible 4.7.0 via pip3 only, globally runnable)
   → Inspect: python3/pip3/ansible versions before installing
   → sudo pip3 install ansible==4.7.0
   → ansible --version shows ansible-core 2.11.12 — NOT a mismatch,
     core is a dependency; verify the actual package version separately:
     pip3 show ansible | grep Version
   → Locate the executable: command -v ansible → /usr/local/bin/ansible
   → Check permissions: ls -l → -rwxr-xr-x (755) — executable by all
   → Check PATH as a normal user (thor) → /usr/local/bin present, works
   → Check PATH under sudo → restricted, /usr/local/bin absent
        → sudo ansible fails, but the BINARY is fine (full-path invocation works)
        → recognize this doesn't violate the stated requirement — leave sudoers alone
   → Test as `nobody` → fails on home-directory write, a DIFFERENT
     layer's problem, not an installation defect
   → Conclude: installation and global executability are both correct
```

---

## 3. Concepts (reference)

### 3.1 `ansible` (the PyPI package) vs. `ansible-core`
The `ansible` package is a curated bundle: `ansible-core` (the actual
engine/CLI) plus a large collection of additional modules/collections.
`ansible --version`'s `[core X.Y.Z]` line always reports the core engine
version — checking the *bundle* version you asked `pip` for requires
`pip3 show ansible`, not the CLI's own version output.

### 3.2 Where `pip3 install` places files
- Library code → the interpreter's `site-packages` directory.
- Any console-script entry points (like `ansible`) → a `bin/` directory
  on that Python installation's prefix (`/usr/local/bin` for a
  system-wide install here).

### 3.3 `PATH` is per-shell/per-process, not a single global fact
Different processes can have different `PATH` values even on the same
machine at the same time — a login shell, a cron job, and `sudo`'s
internal environment are three separate contexts that can each resolve
the same command name differently, or not at all.

### 3.4 `sudo`'s restricted `PATH` (`secure_path`)
Many `sudo` configurations deliberately ignore the invoking user's
`PATH` and substitute a fixed, minimal one — this is a security measure
to prevent a privileged command from being tricked into running an
attacker-controlled binary earlier in a manipulated `PATH`. It is not a
sign of a broken install, and loosening it is a security-relevant change
that shouldn't be made just to satisfy an unrelated task.

### 3.5 Reading error messages by *layer*, not by vibe
"Command not found" (shell/PATH layer) and "permission denied writing to
a path" (application runtime layer, after the binary already started)
are different failure categories requiring completely different fixes.
Pattern-matching either onto "the installation is broken" leads to
wasted reinstalls that never address the actual cause.

---

## 4. Runbook

### 4.1 Inspect the environment
```bash
python3 --version
pip3 --version
ansible --version 2>/dev/null || echo "Ansible is not installed"
```
```text
Python 3.9.19
pip 21.3.1
Ansible is not installed
```

### 4.2 Install Ansible 4.7.0 via pip3
```bash
sudo pip3 install ansible==4.7.0
```
```text
Successfully installed ansible-4.7.0 ansible-core-2.11.12 ...
```

### 4.3 Verify the CLI runs, and understand its version output
```bash
ansible --version
```
```text
ansible [core 2.11.12]
  config file = ...
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 3.9.19 ...
```

### 4.4 Verify the actual requested package version
```bash
pip3 show ansible | grep '^Version:'
```
```text
Version: 4.7.0
```

### 4.5 Locate the executable and check its permissions
```bash
command -v ansible
```
```text
/usr/local/bin/ansible
```
```bash
ls -l /usr/local/bin/ansible
```
```text
-rwxr-xr-x 1 root root ... /usr/local/bin/ansible
```
`755` — readable and executable by owner, group, and others.

### 4.6 Confirm a normal user can run it
```bash
echo "$PATH"
ansible --version
```
Works — `/usr/local/bin` is on `thor`'s `PATH`.

### 4.7 Investigate `sudo`'s different behavior (informational, not a defect)
```bash
sudo env | grep '^PATH='
```
```text
PATH=/sbin:/bin:/usr/sbin:/usr/bin
```
```bash
sudo ansible --version
```
```text
sudo: ansible: command not found
```
```bash
sudo -u root /usr/local/bin/ansible --version
```
Works — confirms the binary itself is fine; only `sudo`'s restricted
`PATH` doesn't include `/usr/local/bin` (§2.7, §2.8). No sudoers change
is required by the stated task.

### 4.8 (Optional exploration) Test as the `nobody` user
```bash
sudo -u nobody /usr/local/bin/ansible --version
```
```text
PermissionError: [Errno 13] Permission denied: '/.ansible/tmp'
```
Confirms the binary starts and runs — the failure is `nobody` lacking a
writable home directory, a different (and out-of-scope) problem, not an
installation issue (§2.9).

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Installed via `pip3` | ✅ |
| `ansible` package version `4.7.0` | ✅ (`pip3 show ansible`) |
| `ansible-core` version | `2.11.12` (bundled dependency, not the requirement itself) |
| Executable at `/usr/local/bin/ansible` | ✅ |
| Executable permissions | ✅ `755` (`-rwxr-xr-x`) |
| Normal users (e.g. `thor`) can run `ansible` | ✅ |
| `sudo ansible` works identically | ⚠️ Not required by the task — `sudo`'s restricted `PATH` is expected behavior |

```text
pip3 install ansible==4.7.0
         │
         ├── library   → /usr/local/lib/python3.9/site-packages/ansible
         └── executable → /usr/local/bin/ansible   (755, world-executable)
                              │
                    every user's shell PATH
                    includes /usr/local/bin
                              │
                              ▼
                   `ansible` runs for all users
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ansible --version` shows a different number than what you `pip install`ed | `ansible --version` reports `ansible-core`'s version, not the `ansible` package's own version | `pip3 show ansible \| grep Version` for the actual requested package version |
| `command not found: ansible` for a normal user | The executable's directory isn't on that user's `PATH` | `command -v ansible`; if empty, confirm the install location and add it to `PATH` |
| `sudo ansible` fails with `command not found` while plain `ansible` works | `sudo` uses its own restricted `PATH` (`secure_path`), which often excludes `/usr/local/bin` | Expected behavior, not a bug; use the full path (`sudo /usr/local/bin/ansible ...`) if `sudo` invocation is genuinely required |
| A specific user (e.g. `nobody`) fails with a `PermissionError` on some path under their home directory | The binary ran fine; it failed on an unrelated runtime file-write issue for that user | Distinguish "not found" from "started, then failed" — this is not an installation defect |
| Tempted to edit `/etc/sudoers` to fix `sudo ansible` | Confusing "not required by the task" with "must be fixed" | Re-read the actual requirement; don't make unrequested, security-relevant changes |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "install Ansible 4.7.0 via pip3, globally available" to
      "verified executable, permissions, and PATH for a normal user."
- [ ] Explain, in one sentence, why `ansible --version` and
      `pip3 show ansible` can report two different-looking version
      numbers without contradicting each other.
- [ ] Explain the difference between "the binary isn't executable" and
      "the binary isn't on this user's PATH" — how would you test each
      independently?
- [ ] Explain why `sudo ansible: command not found` is not itself
      evidence of a broken installation.
- [ ] Explain why the `nobody` test's `PermissionError` is a different
      category of failure than "command not found," and how you can tell
      from the error text alone.
- [ ] Redo the verification from memory on a fresh session: confirm the
      package version, the executable's location and permissions, and
      that a second, different user account can also run `ansible`.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: inspect current state → identify the gap vs. required state →
diagnose each root cause individually → fix → re-run → verify the actual
success state. End with a compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, with real intermediate
output/results inline where they mattered to the diagnosis>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions + "break it
again and fix it from memory" prompt>
```
