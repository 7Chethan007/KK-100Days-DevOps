# Day 3 — Disable Direct Root SSH Login

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning Model** and **§3 Concepts**
first — the runbook is just mechanical execution once you understand *why*
each command exists and *how you'd have arrived at it yourself*.

---

## 1. Scenario

xFusionCorp's security team, following an audit, requires that root can no
longer log in to any app server directly over SSH. Task: disable direct
root SSH login on all three app servers (`stapp01`, `stapp02`, `stapp03`)
in the Stratos Datacenter. Access to those servers is only via `jump-host`.

| Server | Hostname | User   | Role                 |
| --------| ----------| --------| ----------------------|
| App 1  | stapp01  | tony   | Hosts Nautilus App 1 |
| App 2  | stapp02  | steve  | Hosts Nautilus App 2 |
| App 3  | stapp03  | banner | Hosts Nautilus App 3 |

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

The real skill in a lab like this isn't remembering a command list — it's
translating an English requirement into "which subsystem controls this?"
step by step. Here's the chain of questions that gets you from the task
description to every command in this runbook, in order.

### 2.1 "Connect to stapp01" → how does the name resolve?

Before you can SSH anywhere, ask: **how does this machine know what IP
`stapp01` means?** Linux has to translate a hostname to an IP somehow.
One source of that mapping is a static file:

```bash
cat /etc/hosts
```

We didn't know in advance it would contain the answer — we were
*investigating name resolution*, not confirming something we already knew.
In this lab, `stapp01` wasn't in `/etc/hosts`. So the next question is:

> If not `/etc/hosts`, where else does Linux look?

```bash
getent hosts stapp01
```

`getent` asks the system's configured name service (whatever `/etc/nsswitch.conf`
points at — files, DNS, etc.) to resolve the name, instead of you having to
know which source it comes from.

**The transferable intuition:** don't memorize "use `getent` in Kubernetes."
Memorize: *if I have a hostname and don't know how it resolves, investigate
name resolution* — `/etc/hosts` first (static, local), then `getent hosts`
(whatever the system is actually configured to use).

### 2.2 The result told us *where* we were

```text
10.244.240.170  stapp01.y5nbca4tl25kr73z.svc.cluster.local
```

Two details are diagnostic:
- `.svc.cluster.local` — this is Kubernetes' internal DNS domain suffix.
- `10.244.x.x` — a private range commonly used for pod networking.

So without being told, we could infer: `stapp01` isn't a bare VM, it's
resolving through Kubernetes cluster DNS. That doesn't change the SSH
commands, but it explains *why* plain `/etc/hosts` had nothing and *why*
`getent` was the tool that actually answered.

### 2.3 "Disable root SSH login" → which subsystem owns that?

Now translate the actual requirement. Walk down from the general to the
specific:

```text
"Disable root SSH login"
        ↓
   SSH (the protocol/technology)
        ↓
   Client or server? → the SERVER decides who may log in
        ↓
   The SSH server process is called sshd
        ↓
   sshd reads its behavior from a config file
        ↓
   /etc/ssh/sshd_config
```

Client vs. server matters: `ssh` (lowercase, no d) is the program *you* run
to connect out; `sshd` is the daemon listening on the target machine that
decides whether to accept you. A login policy is enforced by whoever is
being connected *to* — so it's always a server-side setting.

### 2.4 "Which setting, specifically?"

`sshd_config` has many directives (`Port`, `PasswordAuthentication`,
`PubkeyAuthentication`, `AllowUsers`, ...). You don't need to have them
memorized — you need to know how to *find* the one that matches the English
requirement:

```bash
grep -i '^PermitRootLogin' /etc/ssh/sshd_config
```

The name `PermitRootLogin` is close enough to the plain-English requirement
("permit root login") that grepping for the concept in the file gets you
there even the first time you see this config.

### 2.5 "I edited the file — how do I know it's safe to apply?"

Config files are just text; a typo can break the whole service. Before
telling the running daemon to pick up a change, ask the daemon to check its
own homework:

```bash
sshd -t     # "-t" = test: is this config syntactically valid?
```

No output = valid. This step exists specifically to prevent locking
yourself out over a typo.

### 2.6 "What will actually be enforced?"

A subtlety: what's *written* in the file and what the daemon will
*actually use* can differ, because `/etc/ssh/sshd_config` can `Include`
other files (`sshd_config.d/*.conf`) that override earlier values. So:

```bash
sshd -T | grep permitrootlogin
```

`-T` (capital) dumps the fully-resolved, effective configuration — the
ground truth of what sshd would enforce right now. `-t` answers "is this
valid?"; `-T` answers "what does this actually mean once resolved?".

### 2.7 "The file changed — does the running process know yet?"

Editing a file on disk does **not** change what an already-running process
has loaded into memory:

```text
   DISK                      MEMORY
/etc/ssh/sshd_config   →   sshd (already running,
   (edited)                 still has old config)
```

A running daemon needs to be told to re-read. That's a **service
management** question now, not a config question — which is why the tool
changes from `sshd` to `systemctl` (systemd, the process supervisor):

```bash
systemctl reload sshd
```

`reload` sends `SIGHUP` — "re-read your config" — without killing the
process or dropping existing connections. `restart` would stop and start
the process instead, which can drop your *own* SSH session if you're
connected through anything tied to that daemon. Since you're managing this
remotely over SSH, `reload` is the non-suicidal choice.

### 2.8 "Is the service still healthy after that?"

```bash
systemctl is-active sshd      # "is it running?" → active
systemctl status sshd         # "give me everything" → PID, recent logs, ExecReload event
```

### 2.9 "Does the *requirement* actually hold now?"

Every step above only proves the **configuration** is correct. It doesn't
prove the **behavior** changed. The only real proof is reproducing the
scenario the requirement was written to prevent:

```bash
ssh root@stapp01
# expect: Permission denied
```

Config-correct and behavior-correct are two different claims — verify both.

### 2.10 The chain, compressed

```text
Requirement  →  Which technology?  →  Client or server?  →  Daemon name
    →  Its config file  →  Which directive?  →  Edit
    →  Is the edit valid? (sshd -t)  →  What's actually effective? (sshd -T)
    →  Tell the running process to reload (systemctl reload)
    →  Is the service still up? (systemctl is-active/status)
    →  Does the real-world behavior match the requirement? (ssh root@host)
```

Every lab in this series follows some version of this shape. When you hit a
new one, re-run this chain with the new requirement instead of searching
for "the right command."

---

## 3. Concepts (reference)

### 3.1 Why root SSH login is a security risk
Root is a fixed, well-known username. If root login is allowed, an attacker
only needs to brute-force a password — they already know the account name.
Disabling it forces attackers (and admins) through a named, audited user
account plus `sudo`, which gives you accountability (who ran what) and one
fewer thing to guess.

### 3.2 `sshd_config` — the daemon's config file
`/etc/ssh/sshd_config` controls the SSH **server** (`sshd`), not the client.
Each directive is `Directive value`. Lines starting with `#` are comments —
often showing the *compiled-in default*, so a commented `#PermitRootLogin
prohibit-password` means "this is the default even though it's not
explicitly set here."

The directive that matters here is `PermitRootLogin`, with values:
- `yes` — root can log in with password or key (unsafe)
- `prohibit-password` (aka `without-password`) — root can log in only with
  an SSH key, never a password
- `no` — root cannot log in over SSH at all, period
- `forced-commands-only` — root can log in via key only to run one fixed command

### 3.3 `sshd -t` vs `sshd -T`
- `-t` (test) — is the config **syntactically valid**? No output = OK.
- `-T` (dump effective config) — what will sshd **actually enforce**,
  after resolving all `Include`d files? Always prefer this over grepping
  the raw file when you need ground truth.

### 3.4 `reload` vs `restart`
- `systemctl reload sshd` sends `SIGHUP` to the running daemon: it re-reads
  config in place, existing connections are untouched.
- `systemctl restart sshd` stops and starts the process: any in-flight
  session tied to the old process can be dropped.

Reload is strictly safer for remote config changes — if you get locked out
mid-restart on a box you only reach via SSH, you're in trouble. Never
restart sshd remotely if reload will do.

### 3.5 Config change ≠ verified fix
Editing a file only proves intent. The task isn't "done" until you've
observed the *behavior* change: an actual `ssh root@host` attempt from
outside must be rejected. Config verification (`sshd -T`) and behavioral
verification (attempt real login, expect `Permission denied`) are two
different checks — do both.

### 3.6 Jump host pattern
`jump-host` (a bastion) is the only machine with direct network access to
the app servers; you SSH into it first, then SSH onward to each app server.
This is why `getent hosts` / `/etc/hosts` matter — DNS/hostnames for
`stapp01`–`stapp03` are only resolvable from inside that network.

---

## 4. Runbook

Repeat **section 4.2** for all three servers.

### 4.1 (Optional) Recon from jump host
```bash
cat /etc/hosts                              # locally configured host mappings
getent hosts stapp01 stapp02 stapp03        # resolve app server IPs
```

### 4.2 Per-server: disable root login
```bash
ssh <user>@stappXX                                    # e.g. ssh tony@stapp01

sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config  # 1. check current value

sudo vi /etc/ssh/sshd_config                          # 2. edit:
#   change/add:  PermitRootLogin no

sudo grep -i '^PermitRootLogin' /etc/ssh/sshd_config  # 3. confirm file says "no"

sudo sshd -t                                          # 4. syntax check
#   no output = OK; any output = fix before continuing

sudo sshd -T | grep permitrootlogin                   # 5. confirm EFFECTIVE config
#   expect: permitrootlogin no

sudo systemctl reload sshd                            # 6. apply live, no dropped sessions

systemctl is-active sshd                              # 7. confirm daemon still healthy
#   expect: active

exit                                                   # back to jump-host
```

### 4.3 Behavioral verification (from jump host)
```bash
ssh root@stappXX
# expect: Permission denied (publickey,password,...)
```

### 4.4 Repeat
Do 4.2–4.3 for `stapp01` (tony), `stapp02` (steve), `stapp03` (banner).

---

## 5. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `sshd -t` prints an error | Typo/duplicate directive in config | Re-open file, fix the line, re-run `sshd -t` before reloading |
| `sshd -T` still shows old value after edit | Reload not run yet, or a file in `sshd_config.d/*.conf` overrides your edit | `systemctl reload sshd`; grep `sshd_config.d/` for a conflicting `PermitRootLogin` line |
| Locked out of root entirely and need root for something else | Working as intended — use `sudo` from the named user instead | Don't restart/reload again to "fix" this; it's the goal |
| `ssh root@host` still succeeds after reload | Change applied to wrong file/host, or reload didn't fire | Re-check `sshd -T`, re-check `systemctl status sshd` for reload timestamp |

---

## 6. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud from
      "disable root SSH login" to the final `ssh root@host` check.
- [ ] Explain what `PermitRootLogin no` allows vs `prohibit-password`.
- [ ] Explain why `sshd -T` is more trustworthy than grepping the file.
- [ ] Explain why `reload` was chosen over `restart`.
- [ ] Repeat the runbook end-to-end on a fresh VM/container with `sshd`
      installed, without copy-pasting — type each command from memory,
      deriving it from the reasoning chain if you forget.
- [ ] Verify with a real rejected `ssh root@...` attempt, not just a config grep.

---

## 7. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which servers/users, constraints>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem, step by step, in the order you'd
actually discover it: "what does this English requirement translate to?"
"client or server?" "what's the process/daemon called?" "where's its config?"
"which setting?" "how do I validate before applying?" "how do I apply
without breaking my own access?" "how do I prove the behavior, not just the
config, changed?" End with the compressed arrow-chain version.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands, grouped by step, comments explaining intent inline>

## 5. Troubleshooting
<symptom / cause / fix table>

## 6. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting" prompt>
```
