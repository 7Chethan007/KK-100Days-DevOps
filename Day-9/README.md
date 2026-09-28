# Day 9 — MariaDB Troubleshooting

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the diagnosis
yourself*.

---

## 1. Scenario

The Nautilus application couldn't connect to its database. Production
support identified that **MariaDB was down** on the database server.

```text
Database Server : stdb01
Database Service: mariadb
Login user       : peter
```

Task: troubleshoot why MariaDB is down, fix the underlying issue, and
verify MariaDB is genuinely operational afterward — not just "the start
command didn't error."

---

## 2. Reasoning model — how to *derive* the diagnosis, not memorize the fix

### 2.1 Trace the dependency chain, not just the symptom

```text
Nautilus Application
       │
       │ SQL connection
       ▼
    MariaDB
       │
       ▼
/var/lib/mysql
```

The application reporting "can't connect to database" doesn't mean the
*application* is broken — it means something *along this chain* is. The
general troubleshooting principle worth internalizing: **the component
reporting the error is often several layers downstream of the actual
cause.** This lab's real failure ends up being a filesystem permission
problem two layers below where the symptom was first observed.

### 2.2 Confirm you're on the right machine before doing anything else

```bash
ssh peter@stdb01
sudo su
hostname
```
```text
stdb01
```

A one-letter hostname mixup on a fleet of similarly-named servers (the
same risk flagged in the AWS EC2/SELinux labs) would make every
subsequent diagnostic step meaningless — confirming identity first is
cheap insurance, not a formality.

### 2.3 SSH host-key verification on first connection

```text
The authenticity of host 'stdb01 ...' can't be established.
```

SSH is asking you to confirm the server's cryptographic identity before
trusting it — a defense against a man-in-the-middle presenting itself as
`stdb01`. Accepting stores the key fingerprint in `~/.ssh/known_hosts`;
subsequent connections are then checked silently against that stored
fingerprint, and a *changed* fingerprint later would be a real red flag,
not just an annoyance to click past.

### 2.4 Three distinct service facts that are easy to conflate

```text
installed  →  does the package/systemd unit exist at all?
enabled    →  will it start automatically on the NEXT boot?
active     →  is it running RIGHT NOW?
```

These are independent facts about a systemd service — a service can be
installed but never enabled, enabled but currently inactive (crashed), or
active without being enabled (started manually, won't survive a reboot).
This lab's actual final state was `installed=yes, enabled=no,
active=yes` — proving directly that **`enabled ≠ running`** and
**`disabled ≠ broken`**. Conflating these three is a common source of
wasted troubleshooting time ("it's enabled, so why isn't it running?" —
wrong question; enabled only governs boot-time behavior).

### 2.5 `systemctl status` vs. `journalctl` — current state vs. history

```bash
systemctl status mariadb --no-pager   # WHAT is the state right now?
journalctl -u mariadb -n 30 --no-pager # WHAT happened before, in the logs?
```

Two different questions, and in this lab `journalctl` genuinely returned
nothing useful (`-- No entries --`) — the service had never logged a
failure because it had never really *attempted* to run in a way that
generated log entries yet. When history is empty, the next move is to
*generate* new evidence by attempting a controlled start, rather than
continuing to search for evidence that doesn't exist yet.

### 2.6 `ExecStartPre` — a service can fail before its "real" process even begins

```bash
systemctl cat mariadb
```
```text
ExecStartPre=/usr/libexec/mariadb-check-socket
ExecStartPre=/usr/libexec/mariadb-prepare-db-dir %n
ExecStart=/usr/libexec/mariadbd ...
```

```text
mariadb.service startup sequence:
      │
      ├── ExecStartPre: check socket        ← must succeed first
      ├── ExecStartPre: prepare db dir       ← must succeed next
      └── ExecStart:    mariadbd             ← the actual DB server
```

`ExecStartPre` entries are prerequisite commands that must each succeed
*before* systemd even attempts to launch the main process
(`ExecStart`). This means "the service failed to start" can mean several
completely different things depending on *which* stage failed — reading
`systemctl cat` to understand a unit's actual startup pipeline, rather
than assuming `ExecStart` itself is always where a failure originates, is
what let this lab correctly localize the problem to `prepare-db-dir`
specifically, not to `mariadbd` itself.

### 2.7 Attempt a controlled start to generate real evidence

```bash
systemctl start mariadb
```
```text
Job for mariadb.service failed because the control process exited with error code.
```

This single attempt converts "MariaDB is down" (a vague symptom) into
"MariaDB fails specifically during startup, with a specific exit code" (a
concrete, investigable fact) — deliberately reproducing a failure under
observation, rather than only reasoning about an already-dead service, is
the same "reproduce before diagnosing" discipline as the JupyterLab and
Makefile labs.

### 2.8 Don't trust truncated output — always ask for the full error

```bash
systemctl status mariadb -l --no-pager
```

`-l` (`--full`) prevents systemd from truncating long lines in its
status output — the default, shortened view can hide exactly the error
text you need. This is where the actual root-cause lines appear:

```text
Process: ... ExecStartPre=/usr/libexec/mariadb-check-socket (code=exited, status=0/SUCCESS)
Process: ... ExecStartPre=/usr/libexec/mariadb-prepare-db-dir mariadb.service (code=exited, status=1/FAILURE)
...
chown: changing ownership of '/var/lib/mysql': Operation not permitted
chmod: changing permissions of '/var/lib/mysql': Operation not permitted
Cannot change ownership of the database directories to the 'mysql' user.
```

`status=0/SUCCESS` on the socket check, `status=1/FAILURE` on the
directory-prep step — this precisely pinpoints *which* `ExecStartPre`
stage failed (§2.6), and the accompanying `chown`/`chmod`/"Operation not
permitted" lines name the exact resource and exact operation that's
failing.

### 2.9 Why `/var/lib/mysql`'s ownership specifically matters

```text
User=mysql
Group=mysql
```

(from `systemctl cat mariadb`) — MariaDB's actual server process runs as
the unprivileged `mysql` user, never as `root`, for the same
least-privilege reasoning behind any service not running as root by
default. Its startup script (`mariadb-prepare-db-dir`) needs to be able
to `chown`/`chmod` `/var/lib/mysql` so the `mysql` user can actually read
and write its own data directory — if that directory's ownership has
drifted away from `mysql:mysql` (to `root:root`, for instance), the prep
step's attempt to fix it fails with exactly the "Operation not permitted"
error observed here.

### 2.10 Verify the actual ownership before changing anything

```bash
ls -ld /var/lib/mysql
```
```text
drwxr-xr-x 1 mysql mysql ...
```

Confirm what the error already implied, directly against the filesystem,
before running a fix — the same "inspect before act" discipline as every
other lab in this series, here applied to file ownership instead of
cloud resource state.

### 2.11 The targeted fix — matching the exact error, nothing more

```bash
chown mysql:mysql /var/lib/mysql
```

The error specifically named a **chown** failure on **this specific
directory** — the fix is exactly that operation, on exactly that path.
This is worth contrasting explicitly with a much more common
(and dangerous) instinct:

```bash
chmod 777 /var/lib/mysql    # WRONG — treats a symptom, ignores the cause
```

`chmod 777` might superficially "work" by making the directory writable
by literally everyone, but it doesn't address *why* ownership was wrong,
leaves the database directory world-writable (a real security
regression for a data directory), and doesn't match what the error
message actually said was failing (an ownership operation, not a
permission bits problem). **Good troubleshooting applies the smallest
change that matches the evidence, not the broadest change that might
plausibly help.**

### 2.12 Retry, then verify at two independent layers

```bash
systemctl start mariadb
systemctl is-active mariadb    # LAYER 1: does systemd consider it running?
mysqladmin ping                # LAYER 2: does the DB server itself respond?
```

```text
systemctl is-active   → "active"           (systemd's opinion)
mysqladmin ping        → "mysqld is alive"  (the database's own opinion)
```

These are genuinely different checks — systemd's `active` state reflects
whether the *process* is running and hasn't exited; `mysqladmin ping`
actually round-trips a request to the database server and gets a real
response back. A process can be `active` in systemd's eyes while
deadlocked or unresponsive internally — checking both layers is stronger
evidence of real health than checking either alone, the same "don't stop
at the API accepting your request, verify the actual resulting behavior"
discipline as every cloud-resource verification in this series.

### 2.13 The compressed reasoning chain

```text
Symptom (application can't reach the database)
   → Trace the dependency: app → MariaDB → /var/lib/mysql
   → Confirm identity: hostname == stdb01
   → systemctl status mariadb           → inactive, disabled (but NOT necessarily broken — §2.4)
   → journalctl -u mariadb              → no history; generate new evidence instead
   → systemctl cat mariadb              → learn the ExecStartPre pipeline (§2.6)
   → systemctl start mariadb            → fails with a control-process error
   → systemctl status mariadb -l        → FULL output: prepare-db-dir failed, chown "Operation not permitted"
   → ls -ld /var/lib/mysql              → confirm actual ownership
   → chown mysql:mysql /var/lib/mysql   → targeted fix matching the EXACT error
   → systemctl start mariadb            → succeeds
   → systemctl is-active mariadb        → active
   → mysqladmin ping                    → mysqld is alive (independent, deeper verification)
```

---

## 3. Concepts (reference)

### 3.1 `installed` / `enabled` / `active` — three independent systemd facts
See §2.4. None of the three implies either of the others — always check
the specific one that answers the question you're actually asking
("will this survive a reboot" is `enabled`; "is this running right now"
is `active`).

### 3.2 `ExecStartPre` and multi-stage service startup
A systemd unit can require several prerequisite commands to each succeed
before its main process starts. `systemctl cat <unit>` is how you learn
a service's actual startup pipeline rather than assuming failures always
originate in `ExecStart` itself.

### 3.3 `systemctl status` vs. `systemctl status -l` vs. `journalctl`
- `systemctl status <unit>` — current state, possibly **truncated**.
- `systemctl status <unit> -l` — current state, **full/untruncated**
  output — use this whenever a status line looks cut off.
- `journalctl -u <unit>` — historical log entries for the unit, useful
  when you need more context than the current snapshot provides.

### 3.4 Why MariaDB (and most services) run as an unprivileged user
Running as `mysql` rather than `root` limits the blast radius of a
compromise or bug in the database server process — it can only do what
the `mysql` user is permitted to do on the filesystem, not anything root
could do. This is the same least-privilege reasoning behind Jupyter's
root-user warning (Day 2) and `sudo`'s restricted `PATH` (Day 8) —
running as less-privileged-by-default is a deliberate security posture,
not an oversight to "fix."

### 3.5 `chown` vs. `chmod`
- `chown user:group <path>` — changes **who owns** the file/directory.
- `chmod <mode> <path>` — changes **what actions are permitted** for
  owner/group/others (see the Day 4 file-permissions guide for the full
  `rwx`/octal breakdown).

This lab's error was specifically an ownership failure — `chmod`ing more
permissively would not have fixed the actual cause and would have
introduced an unrelated, unnecessary security regression (§2.11).

### 3.6 Two layers of "is this service healthy"
- systemd's view (`is-active`) — is the process running, from the init
  system's perspective?
- The application's own view (`mysqladmin ping`, or any
  service-specific health check) — does the actual service respond
  correctly to a real request?

Checking only the first layer can miss a process that's running but
non-functional; checking both is meaningfully stronger verification.

---

## 4. Runbook

### 4.1 Connect and confirm identity
```bash
ssh peter@stdb01
```
```text
The authenticity of host 'stdb01 ...' can't be established.
```
Accept (`yes`) to trust and store the host key.
```bash
sudo su
hostname
```
```text
stdb01
```

### 4.2 Check current service state
```bash
systemctl status mariadb --no-pager
```
```text
Loaded: loaded (...; disabled; preset: disabled)
Active: inactive (dead)
```
```bash
systemctl is-active mariadb
```
```text
inactive
```

### 4.3 Check for historical log entries
```bash
journalctl -u mariadb -n 30 --no-pager
```
```text
-- No entries --
```
No prior failure history — proceed to generate fresh evidence.

### 4.4 Inspect the unit's startup pipeline
```bash
systemctl cat mariadb
```
```text
User=mysql
Group=mysql
ExecStartPre=/usr/libexec/mariadb-check-socket
ExecStartPre=/usr/libexec/mariadb-prepare-db-dir %n
ExecStart=/usr/libexec/mariadbd ...
```

### 4.5 Confirm configuration files exist (rule out "not configured")
```bash
ls -l /etc/my.cnf /etc/my.cnf.d/
```
Files present — configuration itself is not the (or at least not the
only) problem.

### 4.6 Attempt a controlled start
```bash
systemctl start mariadb
```
```text
Job for mariadb.service failed because the control process exited with error code.
```

### 4.7 Read the full, untruncated error
```bash
systemctl status mariadb -l --no-pager
```
```text
Process: ... ExecStartPre=/usr/libexec/mariadb-check-socket (code=exited, status=0/SUCCESS)
Process: ... ExecStartPre=/usr/libexec/mariadb-prepare-db-dir mariadb.service (code=exited, status=1/FAILURE)

chown: changing ownership of '/var/lib/mysql': Operation not permitted
chmod: changing permissions of '/var/lib/mysql': Operation not permitted
Cannot change ownership of the database directories to the 'mysql' user.
Perhaps /etc/my.cnf is misconfigured or there is some problem
with permissions of /var/lib/mysql.
```
Root cause localized: `prepare-db-dir` fails while trying to fix
ownership of `/var/lib/mysql`.

### 4.8 Verify current ownership
```bash
ls -ld /var/lib/mysql
```
```text
drwxr-xr-x 1 mysql mysql ...
```

### 4.9 Apply the targeted fix
```bash
chown mysql:mysql /var/lib/mysql
```
```bash
ls -ld /var/lib/mysql
```
```text
drwxr-xr-x 1 mysql mysql ...
```

### 4.10 Retry starting the service
```bash
systemctl start mariadb
```
```bash
systemctl status mariadb --no-pager -l
```
```text
Active: active (running)
Status: "Taking your SQL requests now..."

mariadb-check-socket       → SUCCESS
mariadb-prepare-db-dir     → SUCCESS
mariadb-check-upgrade      → SUCCESS
```

### 4.11 Verify at both layers
```bash
systemctl is-active mariadb
```
```text
active
```
```bash
mysqladmin ping
```
```text
mysqld is alive
```

### 4.12 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Correct host confirmed (`stdb01`) | ✅ |
| Root cause identified (ownership, not the DB engine itself) | ✅ |
| `/var/lib/mysql` owned by `mysql:mysql` | ✅ |
| `mariadb` service `active` | ✅ |
| `mysqladmin ping` responds | ✅ `mysqld is alive` |
| `mariadb` service `enabled` | ❌ — not required; `enabled ≠ running` (§2.4) |

```text
Application ──X──► MariaDB ──X──► /var/lib/mysql
                                         │
                                  ownership drifted
                                  away from mysql:mysql
                                         │
                                  chown mysql:mysql /var/lib/mysql
                                         │
                                         ▼
Application ─────► MariaDB ─────► /var/lib/mysql
                       │
                  active (running)
                       │
                  mysqld is alive
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `systemctl status mariadb` shows `disabled` and you assume that's the bug | Conflating `enabled` (boot-time behavior) with `active` (currently running) | Check `is-active` separately; `disabled` alone doesn't mean broken (§2.4) |
| `journalctl -u mariadb` shows `-- No entries --` | The service hasn't logged a failure yet, often because it hasn't really been attempted | Attempt a controlled `systemctl start` to generate fresh, inspectable evidence |
| `systemctl status` output looks cut off mid-error | Default status view truncates long lines | Re-run with `-l`/`--full` for the complete error text |
| `Job for mariadb.service failed...` with no further detail | Only the summary line was read, not the full process breakdown | `systemctl status mariadb -l --no-pager` to see which `ExecStartPre` stage actually failed |
| `chown: Operation not permitted` inside the service's own startup log | `/var/lib/mysql`'s ownership has drifted away from `mysql:mysql` | `chown mysql:mysql /var/lib/mysql`, then retry the start |
| Tempted to `chmod 777 /var/lib/mysql` to "just make it work" | Treating a permission-bits problem when the actual error was an ownership problem | Match the fix to the exact error (`chown`, not `chmod`); `777` on a DB data directory is also a security regression (§2.11) |
| `systemctl is-active` shows `active` but the app still can't connect | Checked only the systemd layer, not the database's own responsiveness | `mysqladmin ping` (or an app-level query) for a second, independent health signal |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "application can't reach the database" to "verified via
      `mysqladmin ping`."
- [ ] Explain, in one sentence, the difference between a service being
      `enabled` and being `active`, with an example of when they'd
      disagree.
- [ ] Explain what `ExecStartPre` means, and why reading `systemctl cat`
      was necessary to correctly localize this failure.
- [ ] Explain why `chmod 777 /var/lib/mysql` would have been the wrong
      fix, even if it might have superficially "worked."
- [ ] Explain the difference between what `systemctl is-active` confirms
      and what `mysqladmin ping` confirms — why check both?
- [ ] Deliberately break `/var/lib/mysql`'s ownership again
      (`chown root:root /var/lib/mysql`) on a test system, then diagnose
      and fix it from memory using only `systemctl status -l` output.

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
