# Day 7 — Linux SSH Authentication

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

The xFusionCorp sysadmin team runs scheduled scripts on the **jump host**
that operate against every app server in the Stratos Datacenter. For
these scripts to work unattended, `thor` (on the jump host) needs
**password-less** SSH access to each app server, through that server's
own sudo user:

| Server | Sudo user | Purpose |
|---|---|---|
| `jump-host` | `thor` | SSH gateway, runs the scripts |
| `stapp01` | `tony` | Application Server 1 |
| `stapp02` | `steve` | Application Server 2 |
| `stapp03` | `banner` | Application Server 3 |

Required end state:

```text
thor@jump-host
      ├──────► tony@stapp01     (no password prompt)
      ├──────► steve@stapp02    (no password prompt)
      └──────► banner@stapp03   (no password prompt)
```

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 Why unattended scripts specifically need this, not just convenience

A script running on a schedule (cron, systemd timer, or similar) has no
human present to type a password when SSH prompts for one:

```text
script → ssh user@host → password prompt → nobody there to type it → script hangs/fails
```

Public-key authentication removes the interactive prompt entirely,
replacing "type a password" with "cryptographically prove you hold a
specific private key" — a proof the SSH client can perform on its own,
with no human involved. This is why key-based auth, not "just remember
the password," underlies essentially all real automation (Ansible,
CI/CD, deployment scripts) — the same theme called out in §5 of this
runbook.

### 2.2 Public/private key pair — what each half is for

```text
~/.ssh/id_ed25519          → PRIVATE key — stays on thor's machine, never leaves it
~/.ssh/id_ed25519.pub      → PUBLIC key — copied to every server thor wants to reach
```

```text
Jump host (thor)                        App server (tony/steve/banner)
   id_ed25519  🔒                             ~/.ssh/authorized_keys
       │                                              │
       └──────────── authentication exchange ─────────┘
```

The private key never travels over the network, even during
authentication — the server issues a cryptographic challenge that only
the holder of the matching private key can answer correctly, so
possession is proven without transmission. This is the same public-key
principle as Day 6's EC2 key pair (Cloud-AWS), just applied to Linux user
accounts instead of instance login.

### 2.3 Inspect before generating anything

```bash
ls -la ~/.ssh
```

Before running `ssh-keygen`, check whether `thor` already has a key pair
— generating a new one unnecessarily could orphan an existing setup
elsewhere, or overwrite a key other automation already depends on. In
this lab, `thor` had no `~/.ssh` directory or key material yet, which is
what justified generating fresh keys rather than reusing something
already there. Same "inspect current state before changing it" habit as
every prior lab in this series.

### 2.4 Why `~/.ssh` needs `700` before anything goes in it

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

`700` = owner: `rwx`, group/others: nothing. SSH is deliberately strict
about the permissions on its config directory and key files — an
`~/.ssh` directory that's group- or world-writable/readable is a
plausible tampering vector (someone else could plant or read
authentication material), and OpenSSH will often refuse to use keys
sitting in an improperly-permissioned directory at all. Same "credential
material needs the tightest permissions that still let the owner use it"
principle as `.pem` files (Cloud-AWS Day 6) and `.env` files (MLOps
Day 4).

### 2.5 Choosing Ed25519 over older key types

```bash
ssh-keygen -t ed25519
```

Ed25519 is a modern elliptic-curve signature algorithm, supported by all
current OpenSSH versions, generally preferred over older RSA keys for
new setups: shorter keys, faster verification, and no known practical
weaknesses at commonly-used key sizes. For this lab specifically, an
empty passphrase was chosen deliberately — a passphrase would reintroduce
exactly the "someone must type something interactively" problem this
whole task exists to eliminate (§2.1).

### 2.6 A shell gotcha: a bare path is "run this," not "show me this"

```bash
~/.ssh/id_ed25519.pub
```

typed directly at the prompt fails with `Permission denied` — the shell
interprets any bare path as *a program to execute*, and a `.pub` file
isn't executable. To read a file's contents, you always need an explicit
command that does the reading:

```bash
cat ~/.ssh/id_ed25519.pub
```

```text
/path/to/file        →  "run this as a program"
cat /path/to/file     →  "read this file's contents"
```

Small, but a genuinely common early mistake — worth internalizing once so
it stops costing debugging time later.

### 2.7 `ssh-copy-id` — what it actually does, and doesn't do

```bash
ssh-copy-id tony@stapp01
```

```text
thor's PUBLIC key
       │
       ▼
   tony@stapp01
       │
       ▼
~/.ssh/authorized_keys   (appended, not overwritten)
```

`ssh-copy-id` logs in **interactively** (using Tony's password, one time)
purely to append thor's public key to `tony`'s `authorized_keys` file —
after that one password-authenticated session, the public key is in
place and all *future* logins as `tony` from `thor`'s key can skip the
password entirely. It never touches or transmits the private key — only
ever the public half.

### 2.8 `authorized_keys` — the server-side allow-list

```text
/home/tony/.ssh/authorized_keys
```

Each line in this file is one public key permitted to authenticate as
that specific Linux user. When `thor` connects as `tony`, SSH checks this
file, issues a challenge, and verifies `thor` holds the matching private
key — if the public key isn't listed here, no amount of correct private
key material lets the connection through. This file is *per destination
user*, not per machine — which is exactly the trap in §2.10.

### 2.9 The abstract passwordless-auth exchange

```text
1. thor → stapp01: "authenticate me as tony"
2. stapp01: "I have a public key on file for tony — here's a challenge"
3. thor: signs the challenge using id_ed25519 (the private key)
4. stapp01: verifies the signature against tony's authorized_keys entry
5. Authentication succeeds — no password was ever exchanged
```

"Passwordless" doesn't mean "unauthenticated" — it means authentication
happened via cryptographic proof instead of a shared secret typed over
the wire.

### 2.10 The mistake worth learning from: identity is per-(user, host), not per-host

```bash
ssh steve@stapp01
```

```text
steve@stapp01's password:
Permission denied
```

The configured trust relationship was `thor → tony@stapp01`, not
`thor → steve@stapp01` — `stapp01` and `stapp02` are different machines,
and `tony` and `steve` are different accounts, so `steve@stapp01` is a
*fourth*, never-configured identity distinct from all three intended
ones:

```text
Configured:              tony@stapp01, steve@stapp02, banner@stapp03
Accidentally attempted:  steve@stapp01     ← never set up, correctly rejected
```

The general lesson: an SSH trust relationship is scoped to **exactly one
(destination user, destination host) pair** — the same public key can be
authorized for many such pairs, but each pair is independently configured
and independently verified. Never assume "I set up passwordless SSH to
this machine" without specifying which account.

### 2.11 Verifying success is behavioral, not just configurational

Copying a key (§2.7) is a *configuration* action — it doesn't, by itself,
prove passwordless login actually works. The real verification is
attempting the connection and observing **no password prompt appears**:

```bash
ssh tony@stapp01
whoami; hostname
```

```text
tony
stapp01
```

— reached with zero password prompts. This is the same "don't stop at
'the command succeeded', verify the actual resulting behavior" discipline
as every `describe-*`/`get-*` verification step elsewhere in this series,
just applied to an interactive login instead of a cloud API response.

### 2.12 `ssh user@host 'command'` — the shape automation actually uses

```bash
ssh tony@stapp01 'whoami && hostname'
```

Runs a single remote command non-interactively and returns, without
opening a persistent interactive shell — exactly the invocation shape a
cron script or automation tool uses in a loop over multiple hosts. This
is worth testing explicitly, separately from an interactive `ssh
user@host` session, since it's the actual usage pattern the whole task
exists to enable (§1's "scripts... perform operations on all app
servers").

### 2.13 The compressed reasoning chain

```text
Requirement (thor passwordless to tony@stapp01, steve@stapp02, banner@stapp03)
   → Inspect: ls -la ~/.ssh                       → no key pair exists yet for thor
   → mkdir -p ~/.ssh && chmod 700 ~/.ssh
   → ssh-keygen -t ed25519 (empty passphrase — needed for unattended use)
   → For each (user, host) pair:
        → ssh-copy-id user@host                    (one-time, password-authenticated)
        → ssh user@host  (interactive)              → confirm NO password prompt
        → whoami && hostname                         → confirm correct identity
   → ssh user@host 'command' (non-interactive)       → confirm the automation-shaped invocation also works
   → Repeat verification independently for all THREE (user, host) pairs
```

---

## 3. Concepts (reference)

### 3.1 Private key vs. public key vs. `authorized_keys`
```text
PRIVATE KEY      → stays on the client; proves identity; never transmitted
PUBLIC KEY       → distributed to servers; used to verify a claimed identity
AUTHORIZED_KEYS  → server-side, per-user allow-list of public keys trusted
                    to authenticate as that account
```

### 3.2 Why `~/.ssh` and its contents need strict permissions
OpenSSH actively checks and can refuse to use SSH directories/files that
are writable by anyone other than the owner — this isn't paranoia, it's a
concrete defense against another local user tampering with your
authentication material. `700` on `~/.ssh`, `600` on private keys, and
`644` on public keys/`authorized_keys` are the conventional safe values.

### 3.3 Ed25519 vs. RSA (context, not required for this lab)
Both are valid modern choices for SSH key generation; Ed25519 is
generally preferred for new keys due to shorter key size, faster
operations, and no configurable (and therefore no misconfigurable) key
length — RSA remains common mainly for compatibility with older systems
that don't support Ed25519.

### 3.4 `ssh-copy-id` vs. plain `ssh`
- `ssh-copy-id user@host` — a one-time, password-authenticated action
  that installs a public key into the target's `authorized_keys`.
- `ssh user@host` — the actual connection attempt; used both to test
  interactively and, in scripts, to run remote commands directly.

### 3.5 SSH identity is scoped per (user, host) pair
There is no such thing as "passwordless SSH to `stapp01`" in the
abstract — trust is always established for one specific account on one
specific host. The same client key can be authorized for many such pairs
independently, but each must be set up and verified on its own (§2.10).

### 3.6 `ssh -v` for authentication debugging
```bash
ssh -v user@host
```
Prints the SSH client's authentication negotiation step by step
(which keys it offered, whether the server accepted or rejected each,
which method ultimately succeeded) — the first tool to reach for when
passwordless auth "just doesn't work" and the cause isn't obvious from
the plain error message.

---

## 4. Runbook

### 4.1 Inspect existing SSH state on the jump host
```bash
ssh thor@jump-host
ls -la ~/.ssh
```
```text
ls: cannot access '/home/thor/.ssh': No such file or directory
```
Confirms no key pair exists yet — justifies generating one fresh.

### 4.2 Create the SSH directory with correct permissions
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

### 4.3 Generate an Ed25519 key pair (no passphrase, for unattended use)
```bash
ssh-keygen -t ed25519
```
Accept the default path (`/home/thor/.ssh/id_ed25519`), leave the
passphrase empty.
```bash
ls -la ~/.ssh
```
```text
-rw-------  id_ed25519
-rw-r--r--  id_ed25519.pub
```

### 4.4 Read the public key (not execute the path directly)
```bash
cat ~/.ssh/id_ed25519.pub
```

### 4.5 Install the public key on App Server 1
```bash
ssh-copy-id tony@stapp01
```
```text
Number of key(s) added: 1
```

### 4.6 Verify passwordless login to App Server 1
```bash
ssh tony@stapp01
whoami
hostname
```
```text
tony
stapp01
```
No password prompt appeared. ✅

### 4.7 Repeat for App Server 2
```bash
ssh-copy-id steve@stapp02
```
(entered Steve's password once, interactively)
```bash
ssh steve@stapp02
whoami
hostname
```
```text
steve
stapp02
```
No password prompt on this login. ✅

### 4.8 Repeat for App Server 3
```bash
ssh-copy-id banner@stapp03
```
(entered Banner's password once, interactively)
```bash
ssh banner@stapp03
whoami
hostname
```
```text
banner
stapp03
```
No password prompt on this login. ✅

### 4.9 A wrong-identity mistake surfaced during testing (worth reproducing once)
```bash
ssh steve@stapp01
```
```text
steve@stapp01's password:
Permission denied
```
Expected — `thor → steve@stapp01` was never configured; only
`thor → tony@stapp01` was (§2.10). Confirms the trust relationship is
scoped per (user, host), not per host alone.

### 4.10 Final verification — all three, non-interactively
```bash
ssh tony@stapp01 'whoami && hostname'
ssh steve@stapp02 'whoami && hostname'
ssh banner@stapp03 'whoami && hostname'
```
```text
tony
stapp01

steve
stapp02

banner
stapp03
```
All three ran with zero password prompts — this is the exact invocation
shape the sysadmin team's scheduled scripts will actually use.

### 4.11 Click "Check" in the lab UI to validate.

---

## 5. Final state

```text
                         jump-host
                     ┌───────────────┐
                     │     thor      │
                     │  id_ed25519 🔒│
                     └───────┬───────┘
                             │
               ┌─────────────┼─────────────┐
               ▼             ▼             ▼
           stapp01        stapp02       stapp03
             tony          steve         banner
               │             │             │
        authorized_keys authorized_keys authorized_keys
        (thor's pubkey) (thor's pubkey) (thor's pubkey)
```

| Requirement | Result |
|---|---|
| `thor` → `tony@stapp01` passwordless | ✅ |
| `thor` → `steve@stapp02` passwordless | ✅ |
| `thor` → `banner@stapp03` passwordless | ✅ |
| Non-interactive `ssh user@host 'cmd'` works for all three | ✅ |

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` typing a `.pub` file path directly | Shell tries to execute the path as a program, not read it | Use `cat ~/.ssh/id_ed25519.pub` |
| `ssh-keygen` overwrites or is refused on an existing key | A key pair already existed and wasn't inspected first | `ls -la ~/.ssh` before generating anything new |
| SSH still prompts for a password after `ssh-copy-id` | Wrong destination user or host targeted, or `authorized_keys` permissions too permissive | Re-check the exact `(user, host)` pair (§2.10); confirm `~/.ssh` is `700` and `authorized_keys` is `600`/`644` on the target |
| `steve@stapp01` (or similar cross-mapped combo) asks for a password | That specific (user, host) pair was never configured — trust doesn't transfer across hosts or users | Confirm which exact pairs were set up; configure the missing one explicitly if it's actually required |
| Automation script hangs waiting for input | A password prompt is still occurring somewhere in the chain | Test the exact non-interactive form (`ssh user@host 'command'`) manually first, before wrapping it in a script |
| Unsure why authentication fails at all | Error message alone doesn't show which step failed | `ssh -v user@host` to see the full negotiation and pinpoint the failing step |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "thor needs passwordless SSH to three app servers" to
      "verified via non-interactive `ssh user@host 'command'` on all
      three."
- [ ] Explain, in one sentence, why the private key never needs to be
      copied anywhere, only the public key.
- [ ] Explain why SSH trust is scoped to a specific `(user, host)` pair,
      using the `steve@stapp01` mistake as the concrete example.
- [ ] Explain the difference between `ssh-copy-id` succeeding and
      passwordless login actually working — why are they two separate
      things to verify?
- [ ] Explain why an empty passphrase was the correct choice here, and
      under what circumstances it would NOT be the correct choice.
- [ ] Set up a fourth (user, host) pair from memory (pick any two
      accounts), then verify with both an interactive `ssh` and a
      non-interactive `ssh user@host 'command'`.

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
