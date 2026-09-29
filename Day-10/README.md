# Day 10 — Website Archiving, Secure File Transfer & Automation

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the script
yourself*.

---

## 1. Scenario

xFusionCorp needs the ecommerce website on **App Server 3** (`stapp03`,
user `banner`) archived and shipped to the **Nautilus Storage Server**
(`ststor01`, user `natasha`), via a script at `/scripts/ecommerce_archive.sh`.

Requirements:
1. Archive `/var/www/html/ecommerce` into `xfusioncorp_ecommerce.zip`.
2. Store the archive locally under `/archives/`.
3. Copy the archive to `ststor01:/archives/`.
4. The copy must happen **without a password prompt**.
5. The script must be executable by the appropriate app-server user.
6. The script must **not use `sudo`**.
7. `zip` must be installed manually beforehand — **not** by the script
   itself.

```text
                 Stratos Datacenter

        ┌───────────────────────────────┐
        │  App Server 3 (stapp03/banner)│
        │                                │
        │ /var/www/html/ecommerce        │
        │              │ zip -r          │
        │              ▼                 │
        │ /archives/xfusioncorp_ecommerce.zip │
        └───────────────┬───────────────┘
                        │ scp (SSH, passwordless)
                        ▼
        ┌───────────────────────────────┐
        │ Storage Server (ststor01/natasha) │
        │ /archives/xfusioncorp_ecommerce.zip │
        └───────────────────────────────┘
```

---

## 2. Reasoning model — how to *derive* the script, not memorize it

### 2.1 The general shape: manual procedure → verified steps → automation

```text
Manual operational procedure
        ↓
Understand + verify EACH operation independently
        ↓
Encode the procedure in Bash
        ↓
Remove every interactive dependency
        ↓
Make repeated execution safe (idempotent)
        ↓
Verify the final artifact, both ends
```

This shape — backup, deployment, log rotation, artifact publishing,
CI/CD — recurs across almost all DevOps automation. The script at the
end of this lab is small; the *process* of arriving at it correctly is
the actual content being taught.

### 2.2 Confirm environment before touching anything

```bash
hostname
whoami
pwd
```

In a multi-server environment, executing a perfectly correct command
against the *wrong machine* is a real, common failure mode — the same
"confirm identity first" discipline as every prior lab (AWS's `sts
get-caller-identity`, the MariaDB lab's `hostname` check).

### 2.3 Inspect source and destinations before writing any automation

```bash
ls -ld /var/www/html/ecommerce
find /var/www/html/ecommerce -maxdepth 2 -type f -print
ls -ld /scripts /archives
```

Never automate an assumption you haven't verified — confirming the
source directory actually exists, contains what you expect, and that
`/scripts/` and `/archives/` are real, writable locations, all *before*
a single line of the script is written, is what prevents debugging three
compounded problems simultaneously later.

### 2.4 Archive vs. compression — and why `zip` specifically

```text
Archive      → combines multiple files/directories into ONE container
Compression  → reduces the STORAGE SIZE of that container
```

`tar` is primarily archiving; `gzip` is primarily compression; `tar.gz`
combines both as two separate tools; `.zip` is a single format that does
both at once. The practical reason to bundle into one artifact at all:
transferring and verifying **one file** is operationally simpler than
transferring and verifying a thousand.

### 2.5 `zip -r` — why recursion is not optional

```bash
zip -r "$ARCHIVE" "$SOURCE"
```

`-r` (recursive) is required because the source directory can contain
nested subdirectories (`images/`, `css/`, etc.) — without it, `zip`
would only capture files at the top level, silently producing an
incomplete archive that *looks* successful (exits 0, produces a file)
while missing most of the actual website.

### 2.6 Idempotency — why the script must delete before it creates

Running `zip -r existing.zip source/` against an *already-existing*
archive doesn't overwrite it cleanly — it can **append**, producing
duplicate or path-inconsistent entries across repeated runs (this lab's
draft transcript hit exactly this: manual testing from `/var/www/html`
vs. the script's absolute-path invocation left two different path
prefixes for the same files inside one archive). The fix:

```bash
rm -f "$ARCHIVE"
zip -r "$ARCHIVE" "$SOURCE"
```

```text
run 1  →  fresh archive
run 2  →  fresh archive   (not "old archive + more stuff")
run 3  →  fresh archive
```

`rm -f` (not plain `rm`) matters specifically because `-f` suppresses
the error if the archive doesn't exist yet — without it, the *first*
ever run of the script would fail on `rm` before it even got to
creating anything. This is the concrete, minimal version of
**idempotent automation**: rerunning the script produces the same
result every time, rather than compounding state.

### 2.7 Why the script cannot ever prompt for a password

```text
Cron job / scheduled automation
   → script starts
   → scp asks "password:"
   → nobody is there to type it
   → job hangs or fails
```

This is the exact same reasoning as Day 7's SSH-authentication lab,
applied here to `scp` instead of interactive `ssh` — any automation
that can run unattended must authenticate without a human present,
which means SSH public-key authentication, not a stored or typed
password.

### 2.8 Public-key authentication, recapped for this lab's specific tools

```text
~/.ssh/id_ed25519       → PRIVATE key, stays on banner's machine
~/.ssh/id_ed25519.pub   → PUBLIC key, installed on ststor01 for natasha
```

```bash
ssh-keygen -t ed25519
ssh-copy-id natasha@ststor01
```

Same rule as Day 7: **private key never leaves the client.**
`ssh-copy-id` appends the public key to `~natasha/.ssh/authorized_keys`
on the remote host — a one-time, password-authenticated action that
enables every future connection to skip the password. If
`ssh-copy-id` reports "All keys were skipped because they already
exist on the remote system," that's *useful information* (the key is
already installed), not an error to work around.

### 2.9 Proving passwordless auth actually works, not just hoping

```bash
ssh -o BatchMode=yes natasha@ststor01 'hostname && whoami'
```

`BatchMode=yes` forces SSH to **fail immediately** rather than fall back
to an interactive password prompt — this makes the test unambiguous: if
key auth doesn't work, you get a clean failure instead of a hung
terminal waiting for input that will never come. This exact invocation
shape (`ssh ... 'command'`, non-interactive) is also precisely what the
production script itself will rely on via `scp`.

### 2.10 `scp` is a different tool from `ssh`, sharing the same transport

```text
ssh user@host              → remote LOGIN / remote COMMAND execution
ssh user@host 'command'    → run one command remotely, non-interactively
scp file user@host:/path/  → COPY a file, over the same SSH auth/transport
```

Both authenticate identically (same keys, same `authorized_keys`) —
proving passwordless `ssh` works is a reliable predictor that
passwordless `scp` will also work, since they share the underlying
authentication mechanism.

### 2.11 `set -e` — stopping the script at the first real failure

```bash
#!/bin/bash
set -e
```

Without it: if `zip` fails partway through, the script would still
proceed to `scp` — transferring either a stale archive from a previous
run or a corrupt/partial one. With `set -e`, any command returning a
non-zero exit status stops the script immediately:

```text
zip fails   →  script stops   →   scp never runs
```

This is **failure propagation** — the general principle that a
multi-step automation shouldn't silently continue past a failed step
and produce a misleadingly "complete" but wrong result.

### 2.12 Why `sudo` is explicitly disallowed in the script

`sudo` inside a script reintroduces the exact problem `set -e` and
key-based auth are solving: potential interactive prompting, plus
broader-than-necessary privilege. The correct approach is ensuring the
executing user (`banner`) already has whatever filesystem permissions
the script needs — least privilege applied to automation, not just to
individual commands (the same principle behind services running as
unprivileged users, covered in Days 5 and 9).

### 2.13 Why `zip` is installed *before*, not *inside*, the script

This separates two different concerns:

```text
Environment preparation   → installing zip (provisioning/config-mgmt concern)
Application automation    → what the script itself does (archive + transfer)
```

A script that silently installs its own dependencies hides
infrastructure state inside application logic — in a larger system,
dependency provisioning belongs to configuration management, image
building, or CI infrastructure, not buried inside a task script that
someone might run without realizing it's also modifying installed
packages.

### 2.14 Verify local *and* remote — and verify content, not just existence

```bash
ls -lh /archives/xfusioncorp_ecommerce.zip           # exists?
unzip -l /archives/xfusioncorp_ecommerce.zip          # contains the right files?
ssh natasha@ststor01 'ls -lh /archives/xfusioncorp_ecommerce.zip'   # arrived, remotely?
```

`scp` reporting `100%` only proves a transfer occurred — not that the
source was correct, the contents are right, or the remote copy is
identical. A file can exist and still be empty, corrupt, or built from
the wrong source. This is the same "response accepted ≠ verified
state" discipline running through every lab in this series, here
checked at **two separate locations** rather than one.

### 2.15 The compressed reasoning chain

```text
Requirement (archive ecommerce/, ship to ststor01, no password, no sudo, executable)
   → Confirm environment: hostname/whoami/pwd
   → Inspect source (exists? contents?) and destinations (/scripts, /archives exist?)
   → Confirm zip is already installed (manual prerequisite, not scripted)
   → Manually test EACH step before writing the script:
        zip -r manually → unzip -l to inspect → ssh BatchMode=yes → scp a test file
   → Write the script: shebang, set -e, variables, rm -f, zip -r, scp
   → chmod +x the script
   → Run it once; verify LOCAL artifact (ls + unzip -l)
   → Verify REMOTE artifact (ssh ... ls -lh)
   → Re-run the script; confirm the archive is FRESH each time, not accumulating (idempotency)
```

---

## 3. Concepts (reference)

### 3.1 Shebang (`#!/bin/bash`)
Tells the OS which interpreter should execute the script's contents —
without it (or with the execute bit unset, see §3.2), `./script.sh`
either fails outright or gets interpreted by whatever shell happens to
be running it, which may not be what the script assumes.

### 3.2 `chmod +x`
A script is an ordinary text file until the execute bit is set. Before:
`-rw-r--r--` (not runnable as `./script.sh`); after `chmod +x`:
`-rwxr-xr-x` (runnable). See the Day 4 file-permissions guide for the
full `rwx`/octal breakdown.

### 3.3 Variables and quoting
```bash
SOURCE="/var/www/html/ecommerce"
zip -r "$ARCHIVE" "$SOURCE"
```
Naming values instead of repeating literal strings improves
readability and makes the script's configuration editable in one place.
**Always quote variable expansions** (`"$VAR"`, not `$VAR`) when used as
command arguments — an unquoted variable containing a space or
shell-special character gets word-split by Bash into multiple
arguments, silently breaking the command.

### 3.4 Exit codes and `$?`
```bash
zip -r ...
echo $?
```
`0` conventionally means success; any non-zero value means failure.
`set -e` relies entirely on this convention — every command's exit
status is what it checks to decide whether to keep going.

### 3.5 `rm -f`
`-f` (force) suppresses the error that would otherwise occur if the
target doesn't exist — critical for a "delete before recreate" step
that must succeed identically whether or not a previous artifact is
present.

### 3.6 `command -v <name>`
Answers "does this executable exist, and where does the shell's `PATH`
resolve it?" — the fast way to confirm a dependency (here, `zip`) is
actually installed and reachable before a script tries to use it. Same
tool used in Day 8's Ansible-install lab.

### 3.7 `bash -x` for debugging a misbehaving script
```bash
bash -x /scripts/ecommerce_archive.sh
```
Prints every command Bash actually executes, with variable expansions
resolved — the fastest way to see exactly what a script is doing versus
what you assumed it was doing, especially useful for catching an
unexpectedly empty variable or a wrong resolved path.

### 3.8 File ownership differs per filesystem
The local archive is owned by `banner` (who created it); the copy on
`ststor01` is owned by `natasha` (the account `scp` authenticated as) —
ownership metadata belongs to the filesystem the file currently sits
on, not some property that travels with the file's "identity" across
machines.

---

## 4. Runbook

### 4.1 Confirm environment
```bash
hostname; whoami; pwd
```
```text
stapp03
banner
/home/banner
```

### 4.2 Inspect source and destinations
```bash
ls -ld /var/www/html/ecommerce
find /var/www/html/ecommerce -maxdepth 2 -type f -print
```
```text
ecommerce/
├── .gitkeep
└── index.html
```
```bash
ls -ld /scripts /archives
```

### 4.3 Confirm `zip` is already installed (prerequisite, not scripted)
```bash
command -v zip
zip -v
```
```text
/bin/zip
```

### 4.4 Manually test the archive step before scripting it
```bash
cd /var/www/html
zip -r /archives/xfusioncorp_ecommerce.zip ecommerce
unzip -l /archives/xfusioncorp_ecommerce.zip
```

### 4.5 Generate a passwordless SSH key pair (if not already present)
```bash
ssh-keygen -t ed25519
ssh-copy-id natasha@ststor01
```
```text
All keys were skipped because they already exist on the remote system.
```
(informative, not an error — see §2.8)

### 4.6 Prove passwordless authentication works, non-interactively
```bash
ssh -o BatchMode=yes natasha@ststor01 'hostname && whoami'
```
```text
ststor01
natasha
```

### 4.7 Manually test the transfer step
```bash
scp /archives/xfusioncorp_ecommerce.zip natasha@ststor01:/archives/
ssh natasha@ststor01 'ls -lh /archives/xfusioncorp_ecommerce.zip'
```

### 4.8 Write the script
```bash
cat > /scripts/ecommerce_archive.sh <<'EOF'
#!/bin/bash
set -e

SOURCE="/var/www/html/ecommerce"
ARCHIVE="/archives/xfusioncorp_ecommerce.zip"
REMOTE="natasha@ststor01:/archives/"

rm -f "$ARCHIVE"

zip -r "$ARCHIVE" "$SOURCE"
scp "$ARCHIVE" "$REMOTE"
EOF
```

### 4.9 Make it executable
```bash
chmod +x /scripts/ecommerce_archive.sh
ls -l /scripts/ecommerce_archive.sh
```
```text
-rwxr-xr-x 1 banner banner ... /scripts/ecommerce_archive.sh
```

### 4.10 Run it
```bash
/scripts/ecommerce_archive.sh
```

### 4.11 Verify locally — existence AND contents
```bash
ls -lh /archives/xfusioncorp_ecommerce.zip
unzip -l /archives/xfusioncorp_ecommerce.zip
```
```text
var/www/html/ecommerce/
var/www/html/ecommerce/.gitkeep
var/www/html/ecommerce/index.html
```

### 4.12 Verify remotely
```bash
ssh natasha@ststor01 'ls -lh /archives/xfusioncorp_ecommerce.zip'
```

### 4.13 Confirm idempotency — run it again
```bash
/scripts/ecommerce_archive.sh
unzip -l /archives/xfusioncorp_ecommerce.zip
```
Same clean contents each time — no duplicate or stale entries (§2.6).

### 4.14 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `/scripts/ecommerce_archive.sh` exists, executable | ✅ `-rwxr-xr-x` |
| `zip` installed manually (not by the script) | ✅ |
| Local archive at `/archives/xfusioncorp_ecommerce.zip` | ✅ |
| Remote copy at `ststor01:/archives/xfusioncorp_ecommerce.zip` | ✅ |
| No password prompt during transfer | ✅ (SSH key auth) |
| No `sudo` inside the script | ✅ |
| Script safely rerunnable | ✅ (`rm -f` before `zip -r`) |

```text
SOURCE                          LOCAL ARTIFACT                    REMOTE
/var/www/html/ecommerce  --zip-->  /archives/xfusioncorp_ecommerce.zip  --scp-->  ststor01:/archives/
     (banner, stapp03)              (banner-owned)                        (natasha-owned)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `zip warning: name not matched` / `zip error: Nothing to do` | `$SOURCE` (or another variable) was empty — ran in a shell where it was never set | `echo "SOURCE=$SOURCE"` before the `zip` call to confirm variables actually hold values |
| Archive contains duplicate or inconsistently-pathed entries | Re-ran `zip -r` against an already-existing archive without deleting it first | `rm -f "$ARCHIVE"` before every `zip -r` (§2.6) |
| `scp` prompts for a password inside the script | Public key not installed on the remote account, or testing skipped `BatchMode=yes` verification | `ssh-copy-id`, then re-verify with `ssh -o BatchMode=yes ... 'hostname'` before trusting the script |
| `No such file or directory` on a `chmod`/script invocation | Two commands accidentally typed/pasted together with no separator, or a typo'd filename | `ls -l /scripts/` to confirm the actual filename before re-running; never guess |
| Script "succeeds" (`scp` shows `100%`) but the task still fails verification | Only checked that the transfer completed, not that the content was correct on both ends | `unzip -l` locally AND `ssh ... ls -lh` remotely — verify content, not just transfer completion |
| Script fails partway and a stale/partial archive gets transferred anyway | Missing `set -e`, so a failed `zip` didn't stop the script before `scp` ran | Add `set -e` at the top of the script |
| Unsure exactly what the script is doing at runtime | Reading the script source isn't the same as seeing what it actually executes with real variable values | `bash -x /scripts/ecommerce_archive.sh` to trace every resolved command |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.15) out loud
      from "archive this website and ship it, unattended" to "verified
      on both ends, safely rerunnable."
- [ ] Explain, in one sentence, why `rm -f "$ARCHIVE"` has to run before
      `zip -r`, not after.
- [ ] Explain why `ssh -o BatchMode=yes` is a better passwordless-auth
      test than a plain `ssh` command that might silently fall back to
      a password prompt.
- [ ] Explain why `sudo` and installing `zip` were both explicitly kept
      out of the script — what principle do both violations share?
- [ ] Explain why `scp` reporting `100%` doesn't prove the task
      succeeded, and what two separate checks would.
- [ ] Delete the script and rebuild it from memory, testing each
      manual step (archive, SSH auth, transfer) independently before
      writing a single line of the script itself.

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
