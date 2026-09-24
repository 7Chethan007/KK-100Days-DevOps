# Day 5 — SELinux Installation and Configuration

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Following a security audit, the xFusionCorp security team wants SELinux
prepared on **App Server 3** (`stapp03`) in the Stratos Datacenter.

Requirements:
1. Install the required SELinux packages.
2. Permanently disable SELinux — it will be re-enabled later once
   configuration work is done.
3. **Do not reboot** the server; a maintenance reboot is already
   scheduled for tonight.
4. Ignore SELinux's *current* runtime status entirely — the only thing
   that matters is that the status is **disabled after that reboot**.

Requirement 4 is the crux of the whole lab: this is a task about
configuring what happens at the *next* boot, not about anything you can
observe right now.

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 What SELinux actually is

Regular Linux permissions (`rwx`, owner/group/others — see the Day 4
permissions guide in this repo) answer *"is this user/process allowed to
touch this file at all?"* SELinux is a second, independent layer sitting
on top of that:

```text
Application
     │
     ▼
Linux DAC permissions (rwx, owner/group)   ← "traditional" permissions
     │
     ▼
SELinux policy                              ← a SECOND, separate check
     │
     ▼
Allow / Deny
     │
     ▼
Kernel
```

The critical implication: even if normal Unix permissions say `rwx` and
would allow an action, SELinux can still say **DENIED**, because its
*policy* — a separate rule set about which process types may touch which
resource types — doesn't permit it. This is why "permissions look fine
but I still get denied" is a classic SELinux symptom, not a `chmod` bug.

### 2.2 The three SELinux modes

```text
Enforcing   → policy violations are actively blocked
Permissive  → policy violations are logged, but NOT blocked (useful for
              testing a policy before turning enforcement on)
Disabled    → SELinux isn't active at all
```

The task requires the third: `disabled`.

### 2.3 The distinction the entire lab hinges on: runtime vs. boot-time state

This is the one idea worth over-learning from this lab. SELinux has
**two independent questions**, each with its own separate answer:

```text
Question A: "What is SELinux doing RIGHT NOW, in the running kernel?"
        →  getenforce

Question B: "What will SELinux be set to on the NEXT boot?"
        →  cat /etc/selinux/config   (the SELINUX= line)
```

```text
              Runtime                        Boot configuration
                 │                                   │
                 ▼                                   ▼
             getenforce                    /etc/selinux/config
                 │                                   │
                 ▼                                   ▼
           CURRENT STATE                     NEXT-BOOT STATE
     (can only be Enforcing/            (Enforcing/Permissive/Disabled —
      Permissive at runtime;             read fresh by the kernel at
      "Disabled" here means the          every boot)
      running kernel loaded with
      SELinux off, set at ITS boot)
```

These two can legitimately **disagree** at any given moment — that
mismatch is exactly what this lab's transcript exposes (§2.5), and it's
why requirement 4 explicitly tells you to ignore the runtime value: it's
answering the wrong question for this task.

### 2.4 Inspect before touching anything

```bash
hostname
whoami
```

Confirms you're on the right box (`stapp03`) as the right user (`root`)
before changing any security configuration — a one-letter hostname typo
on a fleet of similarly-named app servers is a classic way to misconfigure
the wrong machine.

```bash
getenforce
```

reported `Disabled` — but per §2.3, that's the *runtime* answer, and this
task cares about the *boot-time* answer instead.

```bash
cat /etc/selinux/config
```

returned `No such file or directory`. That's a real, informative
finding, not an error to work around: the SELinux *configuration file*
didn't exist yet, meaning the SELinux *packages* that ship it probably
weren't installed either — which is exactly what the task's first bullet
point asks you to fix.

### 2.5 Confirm the OS and package manager before installing anything

```bash
cat /etc/os-release
```

```text
NAME="CentOS Stream"
VERSION_ID="9"
```

Different Linux families use different package managers and package
names — installing SELinux on Debian/Ubuntu looks nothing like installing
it on RHEL/CentOS. Confirming the distro *before* reaching for a package
manager avoids running a command that's simply wrong for this OS.

```bash
command -v dnf
```

`dnf` is the package manager on RHEL 8+/CentOS Stream 9 (the successor to
`yum`). `command -v` here answers "is this tool available and where," the
same habit as checking `which python`/`which jupyter` in the JupyterLab
lab — know your tooling before invoking it blind.

### 2.6 Check what's already installed before installing anything

```bash
rpm -qa | grep -E '^selinux|^policycoreutils'
```

```text
policycoreutils-3.6-2.1.el9.x86_64
```

Only `policycoreutils` (SELinux's general management utilities) was
present — the actual **policy** packages (`selinux-policy`,
`selinux-policy-targeted`) were missing. This confirms the diagnosis from
§2.4: no policy packages installed → no `/etc/selinux/config` file → no
boot-time SELinux setting to even read yet. `rpm -qa | grep` is the
general RHEL-family pattern for "is X already installed" — always check
before installing, since installing something already present is at best
a no-op and at worst masks a different real problem.

### 2.7 What `selinux-policy` and `selinux-policy-targeted` actually are

```text
selinux-policy            → the SELinux policy framework itself
                             (shared files, base policy structure)
selinux-policy-targeted   → the "targeted" policy — the specific,
                             practical policy model RHEL/CentOS ships
                             by default (confines specific services
                             rather than locking down the entire system)
```

Installing both together is the standard, expected pairing on this OS
family — `selinux-policy` alone provides the framework but not a usable
default policy to load.

### 2.8 After installing, the boot-config file appears — and disagrees with runtime

```bash
ls -l /etc/selinux/config
```

Now it exists — installing the policy packages is what created it (they
ship the default config as part of the package).

```bash
grep -E '^[[:space:]]*SELINUX=' /etc/selinux/config
```

```text
SELINUX=enforcing
```

Here's the mismatch predicted in §2.3, made concrete:

```text
Runtime (getenforce)             →  Disabled
Boot config (/etc/selinux/config) →  SELINUX=enforcing
```

If nothing further were done, tonight's scheduled reboot would load
whatever `/etc/selinux/config` says — `enforcing` — directly contradicting
the task's requirement that the *post-reboot* state be `disabled`. This is
the single fact that makes the rest of the task non-optional.

### 2.9 Why `setenforce` is the wrong tool here — a deliberate trap

A natural but incorrect instinct is:

```bash
setenforce 0
```

`setenforce` only ever changes the **runtime** mode (§2.3, Question A),
and only ever toggles between `Enforcing` (`1`) and `Permissive` (`0`) —
it **cannot** set `Disabled` at all, and any runtime change it makes is
silently undone at the next boot anyway, since the kernel re-reads
`/etc/selinux/config` from scratch at boot time. Given the task explicitly
says "no reboot, but the state after reboot must be disabled," `setenforce`
cannot satisfy the requirement even in principle — the fix has to target
the boot-time config file directly.

### 2.10 The actual fix: edit the boot-time config, not the runtime state

```bash
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

```text
sed -i          → edit the file in place
's/PATTERN/REPLACEMENT/'  → substitute
^SELINUX=.*      → match a line starting with "SELINUX=", capturing
                    (and discarding) whatever follows
SELINUX=disabled → the replacement line
```

This changes only the **boot-time** answer — deliberately leaving the
current runtime `Disabled` state completely alone (which is fine, since
the task says to disregard runtime status entirely, §2.3).

### 2.11 The compressed reasoning chain

```text
Requirement (packages installed; SELinux disabled AFTER tonight's reboot,
             current runtime status irrelevant, no reboot performed now)
   → Inspect: hostname/whoami                     → confirmed stapp03, root
   → Inspect: getenforce                          → Disabled (runtime — not the target metric)
   → Inspect: /etc/selinux/config                 → doesn't exist yet
   → Diagnose: OS + package manager                → CentOS Stream 9, dnf
   → Inspect: rpm -qa for selinux/policycoreutils  → only policycoreutils present
   → Install: selinux-policy, selinux-policy-targeted
   → Re-inspect: /etc/selinux/config now exists    → SELINUX=enforcing (mismatch!)
   → Diagnose: setenforce can't help (runtime-only, can't set "disabled")
   → Fix: sed the SELINUX= line in /etc/selinux/config → disabled
   → Verify: grep the config line                 → SELINUX=disabled
   → Verify: packages present via rpm -qa
   → Do NOT reboot — the scheduled maintenance reboot will apply it
```

---

## 3. Concepts (reference)

### 3.1 DAC vs. MAC — why SELinux exists at all
Standard Unix permissions are **Discretionary Access Control (DAC)** — the
resource owner decides who can access it (`chmod`, `chown`). SELinux is
**Mandatory Access Control (MAC)** — access is governed by a system-wide
policy that even the resource's owner (or root) can't unilaterally
override by just changing permissions. This is why a `777` file can still
be inaccessible under SELinux — DAC and MAC are independent, additive
gates, both of which must allow an action.

### 3.2 `getenforce` / `setenforce` vs. `/etc/selinux/config`
- `getenforce` — reads the **current, in-memory** SELinux mode.
- `setenforce {0|1}` — changes the **current, in-memory** mode only
  (Permissive/Enforcing); cannot set `Disabled`; not persisted across
  reboot.
- `/etc/selinux/config` — the file the kernel reads **once, at boot**, to
  decide the mode for that entire session (`enforcing`/`permissive`/
  `disabled`). This is the only place a `disabled` state can be
  configured, and the only place a boot-persistent change can be made.

### 3.3 `selinux-policy` vs. `selinux-policy-targeted`
`selinux-policy` provides the policy framework and shared resources;
`selinux-policy-targeted` provides the actual default policy RHEL/CentOS
ships and expects — "targeted" means it specifically confines a defined
set of services/processes rather than locking down the whole system
indiscriminately. On this OS family, install both together.

### 3.4 Why a missing `/etc/selinux/config` was itself a useful signal
A missing config file wasn't a bug to route around — it was direct
evidence that the SELinux policy packages weren't installed yet, which
is precisely what led straight to the correct next step (install them)
rather than guessing at unrelated fixes.

### 3.5 `rpm -qa | grep <pattern>`
The general RHEL/CentOS pattern for "is a package (or package family)
already installed." Checking before installing avoids redundant work and,
more importantly, avoids misdiagnosing an issue as "package missing" when
it's actually already present but misconfigured.

---

## 4. Runbook

### 4.1 Confirm target host and identity
```bash
hostname
whoami
```
```text
stapp03
root
```

### 4.2 Inspect current runtime SELinux state (informational only — not the target)
```bash
getenforce
```
```text
Disabled
```

### 4.3 Check for the boot-time config file
```bash
cat /etc/selinux/config
```
```text
cat: /etc/selinux/config: No such file or directory
```

### 4.4 Identify the OS and package manager
```bash
cat /etc/os-release
```
```text
NAME="CentOS Stream"
VERSION_ID="9"
```

```bash
command -v dnf
```
```text
/bin/dnf
```

### 4.5 Check currently installed SELinux-related packages
```bash
rpm -qa | grep -E '^selinux|^policycoreutils'
```
```text
policycoreutils-3.6-2.1.el9.x86_64
```
Confirms the SELinux policy packages themselves are missing.

### 4.6 Install the required SELinux packages
```bash
dnf install -y selinux-policy selinux-policy-targeted
```
Resolves and installs `selinux-policy`, `selinux-policy-targeted`, and
their dependencies (`policycoreutils` gets upgraded as part of this).

Verify:
```bash
rpm -qa | grep -E '^selinux-policy'
```
```text
selinux-policy-38.1.86-1.el9.noarch
selinux-policy-targeted-38.1.86-1.el9.noarch
```

### 4.7 Re-check the boot-time config — now it exists
```bash
ls -l /etc/selinux/config
```
```text
-rw-r--r-- 1 root root 1175 Sep 24 14:58 /etc/selinux/config
```

```bash
grep -E '^[[:space:]]*SELINUX=' /etc/selinux/config
```
```text
SELINUX=enforcing
```
This is the mismatch (§2.8) that must be corrected before tonight's
reboot.

### 4.8 Set the persistent (boot-time) SELinux mode to disabled
```bash
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

### 4.9 Verify the fix landed
```bash
grep -E '^SELINUX=' /etc/selinux/config
```
```text
SELINUX=disabled
```

```bash
rpm -qa | grep -E '^selinux-policy|^policycoreutils'
```
```text
policycoreutils-3.6-9.el9.x86_64
selinux-policy-38.1.86-1.el9.noarch
selinux-policy-targeted-38.1.86-1.el9.noarch
policycoreutils-python-utils-3.6-9.el9.noarch
```

```bash
getenforce
```
```text
Disabled
```
Runtime happens to still read `Disabled` here too — but per §2.3, that
was never the metric being graded; the config file is.

### 4.10 Do NOT reboot
The scheduled maintenance reboot tonight will apply `SELINUX=disabled`
from the config file. Rebooting now is explicitly out of scope for this
task.

### 4.11 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| App server | `stapp03` ✅ |
| SELinux packages (`selinux-policy`, `selinux-policy-targeted`, `policycoreutils`) | Installed ✅ |
| `/etc/selinux/config` → `SELINUX=` | `disabled` ✅ |
| Reboot performed | Not performed (correct — none was required) ✅ |
| Current `getenforce` | `Disabled` (informational only, not the graded metric) |

```text
                    FINAL STATE

                 ┌───────────────┐
                 │    stapp03    │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
        Packages installed    Boot configuration
              │                     │
              ▼                     ▼
       selinux-policy         SELINUX=disabled
       selinux-policy-targeted
              │                     │
              └──────────┬──────────┘
                         ▼
                 Scheduled reboot (tonight)
                         ▼
                SELinux boots disabled
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `getenforce` says `Disabled` but the task still fails | The task grades the **boot-time** config, not the runtime mode (§2.3) | Check and fix `/etc/selinux/config`'s `SELINUX=` line, not `setenforce` |
| `cat /etc/selinux/config`: No such file or directory | SELinux policy packages aren't installed yet | `dnf install -y selinux-policy selinux-policy-targeted` |
| Tried `setenforce 0` expecting it to "disable" SELinux | `setenforce` only toggles Enforcing/Permissive at runtime; it cannot set `Disabled`, and doesn't persist across reboot anyway | Edit `/etc/selinux/config`'s `SELINUX=` line directly instead |
| `/etc/selinux/config` shows `SELINUX=enforcing` right after installing packages | This is the package's shipped default — installing SELinux policy packages does not itself disable SELinux | `sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config` |
| Unsure if the fix actually "worked" without rebooting | Verifying the boot-time effect of a config file, before that boot happens, isn't something you can observe directly | Verify by re-reading the config file itself (`grep '^SELINUX=' /etc/selinux/config`) — that IS the thing the next boot will read |
| `dnf install` fails or hangs | Wrong package manager assumed for the OS, or no network to the repos | Confirm OS via `/etc/os-release` and package manager via `command -v dnf` first |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "SELinux needs disabling before tonight's reboot" to "verified
      via the config file, without ever rebooting."
- [ ] Explain, in one sentence, the difference between what `getenforce`
      reports and what `/etc/selinux/config` configures.
- [ ] Explain why `setenforce 0` cannot satisfy this task's requirement,
      even though it sounds like "turning SELinux off."
- [ ] Explain why a missing `/etc/selinux/config` file was itself useful
      diagnostic information, not just an error to skip past.
- [ ] Explain the difference between `selinux-policy` and
      `selinux-policy-targeted`, and why both get installed together on
      RHEL/CentOS.
- [ ] Revert `/etc/selinux/config` back to `enforcing` on a test box (or
      just mentally), then redo the fix from memory using only the
      reasoning chain, not this file.

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
