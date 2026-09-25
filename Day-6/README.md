# Day 6 — Create a Cron Job

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the
automation yourself*.

---

## 1. Scenario

The Nautilus admins want a scheduled task deployed on **all application
servers** in the Stratos Datacenter:

1. Install `cronie` on `stapp01`, `stapp02`, `stapp03`.
2. Start `crond` (and have it start on boot).
3. Add this cron job for **root** on each server:
   ```text
   */5 * * * * echo hello > /tmp/cron_text
   ```

The infrastructure list also included `stlb01`, `stdb01`, `ststor01`,
`stbkp01`, `stmail01`, and others — but the task says "app servers," so
scoping correctly to just the three `stapp0N` hosts is itself part of the
task, not a given.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 What a cron job actually is

A cron job is a command Linux runs automatically on a schedule, instead
of a human typing it every time:

```text
                  Linux
                    │
                 crond
              (daemon/service)
                    │
             reads crontab
                    │
                    ▼
          scheduled cron entry
                    │
                    ▼
             shell command
                    │
                    ▼
        echo hello > /tmp/cron_text
```

Two terms worth keeping distinct from the start:

```text
crontab = the schedule/configuration (what to run, and when)
crond   = the daemon that watches the clock and actually executes it
```

This is the same "config file vs. the process that reads it" split as
`/etc/selinux/config` vs. the running kernel (Day 5) — the configuration
and the thing that acts on it are two separate concerns.

### 2.2 Cron is a systemd-managed daemon, same as any other service

```bash
systemctl status crond
systemctl enable crond    # start automatically on every future boot
systemctl start crond     # start it right now, this session
systemctl enable --now crond   # both, in one command
```

Nothing cron-specific here — it's the identical `enable`/`start` pattern
you'd use for any systemd service. Worth internalizing once, generically,
rather than treating cron's service management as a special case.

### 2.3 Reading a cron expression — five fields, left to right

```text
┌──────── minute        (0–59)
│ ┌────── hour           (0–23)
│ │ ┌──── day of month   (1–31)
│ │ │ ┌── month          (1–12)
│ │ │ │ ┌ day of week    (0–6, Sunday=0)
│ │ │ │ │
* * * * * command
```

`*/5` in the minute field means "every value divisible by 5" — so the
job fires at `:00, :05, :10, :15, ...` past every hour, not "5 minutes
after whenever this line was saved." A few reference points worth having
memorized rather than re-deriving each time:

```text
* * * * *        every minute
*/5 * * * *      every 5 minutes
0 * * * *        top of every hour
0 2 * * *        02:00 every day
0 3 * * 0        03:00 every Sunday
0 9 * * 1-5      09:00, Monday through Friday
```

### 2.4 A cron expression is configuration, not a shell command

```bash
*/5 * * * * echo hello > /tmp/cron_text
```

typed directly at a Bash prompt fails:

```text
bash: */5: No such file or directory
```

Bash tries to glob-expand `*/5` and run it as a command — because a cron
line only has meaning *inside a crontab file*, parsed by `crond`'s own
five-field syntax, not by the shell. This is the same category of mistake
as pasting a `.gitignore` pattern into a shell and expecting it to do
something — configuration syntax for one program is not automatically
valid input to another (Day 4 MLOps, §2.4).

### 2.5 Cron jobs belong to a specific user

```bash
crontab -e     # edit the CURRENT user's crontab
crontab -l     # list the CURRENT user's crontab
```

Whichever user runs `crontab -e` owns the resulting job — if `tony` runs
it, the job is `tony`'s; if `root` runs it, the job is `root`'s. The task
explicitly requires the **root** crontab, so the edit has to happen as
root (`sudo crontab -e` / `sudo crontab -l`), not as whatever user you
initially SSH in as.

### 2.6 Adding a cron line without an interactive editor

```bash
(crontab -l 2>/dev/null; echo '*/5 * * * * echo hello > /tmp/cron_text') | crontab -
```

```text
crontab -l 2>/dev/null   → print the existing crontab (silently produce
                             nothing, not an error, if none exists yet)
echo '...'                → the new line to add
( ... ; ... )              → run both, concatenating their output
| crontab -                → feed that combined text back in as the
                             NEW complete crontab; "-" means "read from
                             stdin" instead of opening an editor
```

This pattern — read existing config, append, pipe the whole thing back
in — is worth recognizing generally: it's how you script an edit to
something an interactive tool would otherwise require a human to type
into an editor for. Skipping `crontab -l` and only `echo`-ing the new
line would silently **overwrite** any existing cron jobs for that user —
worth understanding *why* the existing-list step is there, not just
copying the one-liner.

### 2.7 Configuration vs. execution — two different questions, two different checks

```text
crontab -l           → IS the job configured?     (a static fact)
cat /tmp/cron_text   → HAS the job actually run?   (a dynamic fact)
```

Confirming the crontab line exists proves nothing about whether it has
executed yet — if you configure it at `17:12`, the first `*/5`-aligned
run happens around `17:15`, not immediately. Treating "the config is
right" as equivalent to "the behavior is verified" is a common shortcut
worth resisting — the same "response received ≠ resource ready" caution
from the EBS volume lab (Day 5, Cloud-AWS), applied to time-based
automation instead of a create-API response.

### 2.8 Scoping the requirement before automating anything

The visible infrastructure list included load balancers, databases,
storage, backup, and mail servers alongside the three app servers — but
the task says "all application servers," not "every server you can see."
Automating against the full list would apply the change to hosts that
were never in scope. This is a general lesson, not cron-specific:
**determine exactly which hosts a requirement applies to before writing
any loop that touches more than one of them.**

### 2.9 Repetition across hosts is the actual signal to automate

Doing `stapp01` manually first (§4.2–§4.4) before writing a loop for
`stapp02`/`stapp03` is deliberate, not wasted effort: it proves the
*procedure itself* is correct on one host, in an environment where
mistakes are easy to see and fix interactively, before wrapping it in a
loop where mistakes get repeated silently across every remaining host.

### 2.10 Diagnosing the first automation failure — wrong identity, not wrong logic

```bash
ssh "$host"
```

run as `thor@jump-host` implicitly tries `thor@stapp02` — but `stapp02`'s
actual login user was `steve`, and `stapp03`'s was `banner`. The loop
logic itself was fine; the *identity* assumption baked into a bare
`ssh "$host"` was wrong. General lesson: when automation fails, check
**which user, on which host, running as what** before assuming the
command or its logic is broken — same "confirm identity before assuming
the resource is misconfigured" instinct as `aws sts get-caller-identity`
in the AWS labs.

### 2.11 Diagnosing the second automation failure — two separate authentication boundaries

```bash
sshpass -p "$password" ssh ...
sudo ...          # fails: "a terminal is required to read the password"
```

There are **two independent** authentication steps here, easy to
conflate into one:

```text
Jump host
   │
   │ SSH password           ← sshpass supplies THIS one
   ▼
Remote user (steve/banner)
   │
   │ sudo password           ← sshpass does NOT supply this one
   ▼
root
```

`sshpass` only ever answers the SSH login prompt; `sudo` prompts
separately, and without an interactive terminal (`ssh` running a remote
command non-interactively), there's nothing to type the password into —
hence the "a terminal is required" error. The fix, `sudo -S`, makes
`sudo` read its password from **stdin** instead of prompting a terminal,
which is exactly the kind of input a non-interactive SSH command *can*
supply:

```bash
echo "$password" | sudo -S sh -c '...'
```

### 2.12 The compressed reasoning chain

```text
Requirement (cronie + crond + root cron job, on app servers ONLY)
   → Scope: identify stapp01/02/03 among the full host list, exclude the rest
   → Manually configure stapp01 first, to validate the procedure itself
       → dnf install -y cronie
       → systemctl enable --now crond
       → (crontab -l 2>/dev/null; echo '...') | crontab -   (as root)
       → verify: crontab -l, then later cat /tmp/cron_text
   → Recognize the remaining 2 hosts as repetition → write a loop
   → Attempt 1: ssh "$host"                  → fails, wrong assumed user (thor)
   → Fix: map host → correct login user (steve/banner)
   → Attempt 2: sshpass + plain sudo          → fails, no terminal for sudo's password
   → Fix: sudo -S, feeding the password via stdin instead
   → Re-run the loop for stapp02, stapp03
   → Verify EVERY host independently: package, service active, service
     enabled, root crontab present, /tmp/cron_text eventually populated
```

---

## 3. Concepts (reference)

### 3.1 `cronie` vs. `crond`
`cronie` is the package (on RHEL/CentOS) providing the cron
implementation; `crond` is the daemon binary/service it installs and
that systemd manages. Installing the package is a prerequisite for the
service to exist at all — the same package-then-service-then-config
layering as any systemd-managed daemon.

### 3.2 `crontab -l` vs `sudo crontab -l`
Cron jobs are stored per-user (`/var/spool/cron/<username>`). Running
`crontab -l` as a non-root user shows *that* user's jobs, never root's —
if a task requires the root crontab specifically, you must either be
root or explicitly prefix with `sudo`.

### 3.3 `(existing; new) | crontab -` as a general scripting pattern
Reading current state, appending to it, then feeding the combined result
back into a tool that would otherwise expect interactive editor input is
a broadly reusable technique — not unique to cron. Anywhere a CLI tool's
"edit" subcommand normally opens `$EDITOR`, check whether it also accepts
input on stdin for exactly this kind of scripted append.

### 3.4 `sudo -S`
Reads the sudo password from standard input rather than prompting an
interactive terminal — required whenever `sudo` needs to run inside a
context with no TTY attached, such as a command passed non-interactively
over `ssh user@host "command"`. Distinct from passwordless `sudo`
(`NOPASSWD` in `/etc/sudoers`), which is a different (and often
preferable) way to avoid the same problem.

### 3.5 Two independent authentication layers in "SSH then sudo" automation
SSH login and `sudo` elevation are checked by two different mechanisms
(SSH's own auth, then PAM/sudoers on the remote host) and can each fail
independently. A script supplying credentials for one is not
automatically supplying credentials for the other — treat them as two
separate steps to verify, not one combined "login" step.

### 3.6 Why hardcoded passwords in a script are a lab-only shortcut
Plaintext passwords in a script are exposed via shell history, process
listings (`ps aux` can reveal command-line arguments briefly), logs, and
version control if ever committed. Production automation replaces this
with SSH keys, a configuration-management tool (Ansible, etc.) using
vaulted secrets, or short-lived credentials from a secrets manager — the
pattern in §4.6 is acceptable *only* because this is a disposable lab
environment with rotating, single-session credentials.

---

## 4. Runbook

### 4.1 Confirm scope — which hosts does "app servers" actually mean?
```text
Full inventory: stapp01, stapp02, stapp03, stlb01, stdb01, ststor01, stbkp01, stmail01, ...
In scope for this task: stapp01, stapp02, stapp03   (the "app servers" only)
```

### 4.2 Manually configure the first server (validate the procedure)
```bash
ssh tony@stapp01
sudo su
```
```bash
hostname
whoami
```
```text
stapp01
root
```
```bash
dnf install -y cronie
```
```bash
systemctl enable --now crond
systemctl status crond --no-pager
```
```text
Active: active (running)
```

### 4.3 Add the root cron job on stapp01
```bash
(crontab -l 2>/dev/null; echo '*/5 * * * * echo hello > /tmp/cron_text') | crontab -
crontab -l
```
```text
*/5 * * * * echo hello > /tmp/cron_text
```

### 4.4 Verify configuration now, execution later
```bash
crontab -l               # configuration check — passes immediately
cat /tmp/cron_text       # execution check — only meaningful once a
                          # */5-aligned minute has actually passed
```
```text
hello
```

### 4.5 Automate the remaining two servers — first attempt (fails, wrong user)
```bash
for host in stapp02 stapp03; do
  ssh "$host" "..."
done
```
```text
Permission denied (publickey,password)
```
Diagnosis: this ran as `thor@stapp02`/`thor@stapp03` by default — but the
actual login users are `steve` (stapp02) and `banner` (stapp03).

### 4.6 Automate the remaining two servers — corrected version
```bash
for host in stapp02 stapp03; do
  if [ "$host" = "stapp02" ]; then
    user="steve"
    password='Am3ric@'
  else
    user="banner"
    password='BigGr33n'
  fi

  sshpass -p "$password" ssh -o StrictHostKeyChecking=no "$user@$host" \
    "echo '$password' | sudo -S sh -c '
      dnf install -y cronie &&
      systemctl enable --now crond &&
      (crontab -l 2>/dev/null; echo \"*/5 * * * * echo hello > /tmp/cron_text\") | crontab -
    '"
done
```
```text
Installing:
  cronie
Installed:
  cronie
  cronie-anacron
  crontabs
```
(on both `stapp02` and `stapp03` — confirms package install succeeded on
each host independently.)

**Lab-only note:** hardcoded plaintext passwords in a script are
acceptable here only because this is a disposable KodeKloud environment
with rotating, single-session credentials — see §3.6 before reusing this
pattern anywhere real.

### 4.7 Verify every host independently — don't stop at "no error"
Run on each of `stapp01`, `stapp02`, `stapp03`:
```bash
rpm -q cronie
```
```text
cronie-...
```
```bash
systemctl is-active crond
```
```text
active
```
```bash
systemctl is-enabled crond
```
```text
enabled
```
```bash
sudo crontab -l
```
```text
*/5 * * * * echo hello > /tmp/cron_text
```
```bash
cat /tmp/cron_text
```
```text
hello
```
(only meaningful once at least one `*/5` boundary has passed since the
crontab was written — see §2.7.)

### 4.8 Click "Check" in the lab UI to validate.

---

## 5. Final state

```text
             Nautilus App Servers
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
     stapp01     stapp02     stapp03
        │           │           │
    cronie       cronie       cronie
        │           │           │
      crond       crond       crond
    (active,     (active,     (active,
     enabled)     enabled)     enabled)
        │           │           │
        └───────────┼───────────┘
                     │
               root crontab
                     │
       */5 * * * * echo hello > /tmp/cron_text
                     │
                     ▼
             /tmp/cron_text → "hello"
        (populated after the next */5 boundary)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `bash: */5: No such file or directory` | A cron expression was typed directly at the shell prompt instead of into a crontab | Use `crontab -e` (interactive) or the `(crontab -l; echo ...) \| crontab -` pattern (§2.6) |
| `crontab -l` shows nothing after editing | Edited the wrong user's crontab (e.g. edited as a regular user when the task requires root) | Use `sudo crontab -l` / `sudo crontab -e` explicitly |
| `cat /tmp/cron_text`: No such file or directory, despite a correct crontab | Not enough time has passed for a `*/5`-aligned minute to occur yet | Wait until the next 5-minute boundary, then re-check — configuration and execution are different checks (§2.7) |
| Loop-based SSH fails with `Permission denied` for every host | Assumed the wrong remote login user (defaulted to the local jump-host user) | Explicitly map each hostname to its correct login user before connecting |
| `sudo: a terminal is required to read the password` | `sudo` prompting for a password with no interactive TTY attached (common inside a non-interactive `ssh host "command"`) | Use `sudo -S` and pipe the password via stdin: `echo "$password" \| sudo -S ...` |
| Automation applied the change to servers outside the stated scope | Looped over every host in the visible inventory instead of just the ones the task actually named | Explicitly enumerate the in-scope hostnames (§2.8) before writing any loop |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "deploy this cron job on all app servers" to "verified on
      every host, both configuration and execution."
- [ ] Explain, in one sentence, why `*/5 * * * * echo hello` fails when
      typed directly into Bash but works inside a crontab.
- [ ] Explain the difference between `crontab -l` confirming the job is
      configured and `cat /tmp/cron_text` confirming it has actually run.
- [ ] Explain why `sshpass` supplying the SSH password wasn't enough to
      make a subsequent `sudo` command succeed, and what `sudo -S` fixes.
- [ ] Explain why the task's host list needed to be filtered down to just
      the app servers before writing the automation loop.
- [ ] Re-derive the `(crontab -l 2>/dev/null; echo '...') | crontab -`
      one-liner from scratch, explaining what happens if you skip the
      `crontab -l` part.

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
