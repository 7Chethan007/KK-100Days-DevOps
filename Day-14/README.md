# Day 14 — Linux Process Troubleshooting

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the
diagnosis yourself*.

This lab's root cause is almost identical to Day 12's Apache
connectivity failure — same conflict (a system service squatting on
Apache's port), same culprit (`sendmail`), different port number. If
Day 12 is fresh, this should feel like pattern recognition, not a new
investigation from scratch.

---

## 1. Scenario

Monitoring reports Apache unavailable on one of three app servers.
Requirements:
- Apache running on all three servers (`stapp01`, `stapp02`, `stapp03`).
- Apache listening on port `6200`.
- Identify and fix the faulty server.

---

## 2. Reasoning model — how to *derive* the diagnosis, not memorize the fix

### 2.1 Check all three servers before touching any of them — comparative diagnosis, again

```bash
for server in stapp01 stapp02 stapp03; do
  case "$server" in
    stapp01) USER="tony"; PASSWORD="Ir0nM@n" ;;
    stapp02) USER="steve"; PASSWORD="Am3ric@" ;;
    stapp03) USER="banner"; PASSWORD="BigGr33n" ;;
  esac
  echo "===== $server ====="
  sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
    "$USER@$server" "echo '$PASSWORD' | sudo -S systemctl is-active httpd"
done
```
```text
stapp01 → failed
stapp02 → active
stapp03 → active
```

Same comparative-troubleshooting instinct as Day 12: checking all three
before investigating any single one immediately narrows the problem to
`stapp01` specifically, rather than assuming (or having to separately
discover) which server is actually affected.

### 2.2 Read the actual service error — don't assume Apache's own config is broken

```bash
sshpass -p 'Ir0nM@n' ssh ... "echo 'Ir0nM@n' | sudo -S systemctl status httpd --no-pager -l"
```
```text
AH00072: make_sock: could not bind to address [::]:6200
AH00072: make_sock: could not bind to address 0.0.0.0:6200
(98)Address already in use
```

"Apache failed to start" is the symptom; `Address already in use` is
the actual diagnostic fact — it tells you the problem is a **port
conflict**, not a misconfigured Apache. This is the identical error
shape as Day 12's `:5002` failure, just at port `6200` instead.

### 2.3 Why `Address already in use` means "ask who owns the port," not "restart Apache"

```text
Apache attempts bind() on :6200
        │
        ▼
Some OTHER process already owns :6200
        │
        ▼
bind() fails → Apache refuses to start
```

Only one process can hold a given IP:port combination at a time. The
actionable next question isn't "why won't Apache start" — it's "**who
already has port 6200**." Restarting Apache repeatedly without first
answering that question would just fail identically every time.

### 2.4 Finding the owning process — `ss` first, `lsof` as independent confirmation

```bash
ss -lntp | grep ':6200'
```
```text
LISTEN 0 10 127.0.0.1:6200 0.0.0.0:* users:(("sendmail",pid=25212,fd=4))
```
```bash
lsof -nP -iTCP:6200 -sTCP:LISTEN
```
```text
sendmail 25212 root ... TCP 127.0.0.1:6200 (LISTEN)
```

Using two independent tools to confirm the same fact (`sendmail`, PID
`25212`) is stronger evidence than trusting either alone — if they
disagreed, that mismatch itself would be worth investigating further.
Exactly the same diagnostic move, and the same culprit, as Day 12's
`:5002` conflict — `sendmail` apparently defaults to binding low,
memorable-looking ports on these lab images, which is worth knowing as
a pattern if you hit yet another "Address already in use" in a future
lab.

### 2.5 Confirm what you're about to stop is safe to stop — check the service, not just the PID

```bash
systemctl status sendmail --no-pager -l
```
```text
Active: active (running)
```
and confirmed `disabled` — not configured to start automatically at
boot.

Before stopping an unfamiliar process, confirming it's a known systemd
service (not some ad-hoc or security-relevant process) and that it's
`disabled` (so stopping it won't fight against something trying to
restart it automatically, and won't re-occur after a reboot) is a
reasonable due-diligence check — same "understand before you act"
discipline as every lab in this series, applied here to "is it safe to
stop this" rather than "is this the right fix."

### 2.6 Stop the conflict, verify the port is actually free, *then* start Apache

```bash
systemctl stop sendmail
ss -lntp | grep ':6200' || echo 'PORT 6200 IS FREE'
```
```text
PORT 6200 IS FREE
```

Verifying the port is free *before* attempting to start Apache again is
worth doing explicitly rather than assuming `systemctl stop` worked —
same "verify the fix landed before retrying the operation" discipline
as Day 12.

### 2.7 Start Apache and check both the status *and* its actual bound port

```bash
systemctl start httpd
systemctl status httpd --no-pager -l
```
```text
Active: active (running)
Status: "Started, listening on: port 6200"
```

The status line explicitly names the port Apache bound to — stronger
confirmation than `active (running)` alone, the same "logs prove the
*specific* thing you need, not just that the process is alive" lesson
from Day 11's Tomcat lab (`"Starting ProtocolHandler [...]"`) and
Day 12 (`"Starting ProtocolHandler [...:5002]"`).

### 2.8 Don't chase a cosmetic warning as if it were the actual failure

```text
Could not reliably determine the server's fully qualified domain name
```

This FQDN warning appears in Apache's startup output but is **not**
what caused the earlier failure — the real cause was the port conflict,
fully resolved already. Apache started successfully despite this
warning being present. Learning to recognize "this message looks
alarming but isn't the actual problem" (vs. `Address already in use`,
which genuinely was) is itself a skill — treating every line of output
as equally significant leads to chasing symptoms that were never the
real issue.

### 2.9 Final verification — all three servers, both service and port

```bash
for server in stapp01 stapp02 stapp03; do
  ...
  systemctl is-active httpd
  ss -lntp | grep ':6200'
done
```

Re-checking **all three** servers at the end (not just the one that was
fixed) confirms the fix didn't inadvertently affect the two that were
already working — the same "verify the whole picture, not just the one
thing you changed" discipline as Day 7's instance-type-change lab.

### 2.10 The compressed reasoning chain

```text
Requirement (Apache active + listening on :6200, on all 3 servers)
   → Comparative check across all 3 servers    → stapp01 is the only failure
   → systemctl status httpd on stapp01            → "Address already in use" :6200
   → Diagnose: this is a PORT CONFLICT, not an Apache config bug
   → ss -lntp | grep :6200                           → sendmail (PID 25212) owns it
   → lsof -iTCP:6200 -sTCP:LISTEN                      → independent confirmation
   → systemctl status sendmail                          → confirm it's a real, disabled service — safe to stop
   → systemctl stop sendmail
   → ss -lntp | grep :6200                                → confirm port now free
   → systemctl start httpd
   → systemctl status httpd                                 → "listening on: port 6200" (not just "active")
   → (ignore the unrelated FQDN warning — not the actual cause)
   → Re-verify ALL THREE servers                              → confirm nothing else regressed
```

---

## 3. Concepts (reference)

### 3.1 "Address already in use" as a specific, actionable diagnosis
This exact error means a `bind()` call failed because another process
already holds the requested IP:port — it is never a generic "something's
wrong with this service" message; it specifically redirects the next
step toward "find the other process," not toward the failing service's
own configuration.

### 3.2 `ss -lntp` vs. `lsof -iTCP:<port>`
Both answer "who's listening on this port," using different underlying
mechanisms — using both as independent cross-checks is stronger
evidence than either alone, the same principle as bidirectional
verification elsewhere in this series (just applied to a local process
fact instead of a remote resource relationship).

### 3.3 Checking a conflicting service's own health before stopping it
Confirming a process belongs to a legitimate, known systemd unit (not
stopping an unidentified PID blindly) and checking whether it's
`enabled`/`disabled` is reasonable due diligence before taking any
stop/kill action on a shared system.

### 3.4 Status line detail vs. "active (running)" alone
`systemctl status`'s free-text status line (when a service provides
one) can confirm a more specific fact than the bare `active` state —
here, confirming the *actual bound port*, not just that the process
exists and hasn't crashed.

### 3.5 Distinguishing a cosmetic warning from the actual root cause
Not every line in a service's startup output is equally significant —
learning to recognize which warnings are pre-existing, harmless noise
(the FQDN warning here) versus the actual failure (`Address already in
use`) prevents wasted effort chasing the wrong thing.

### 3.6 Comparative troubleshooting across a fleet (recap from Day 12)
Checking every instance of a service across all hosts it should be
running on — both before and after a fix — narrows the problem faster
than investigating one host in isolation, and confirms a fix didn't
introduce a regression elsewhere.

---

## 4. Runbook

### 4.1 Check Apache's state across all three servers
```bash
for server in stapp01 stapp02 stapp03; do
  case "$server" in
    stapp01) USER="tony"; PASSWORD="Ir0nM@n" ;;
    stapp02) USER="steve"; PASSWORD="Am3ric@" ;;
    stapp03) USER="banner"; PASSWORD="BigGr33n" ;;
  esac
  echo "===== $server ====="
  sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
    "$USER@$server" "echo '$PASSWORD' | sudo -S systemctl is-active httpd"
done
```
```text
stapp01 → failed
stapp02 → active
stapp03 → active
```

### 4.2 Inspect the failure on the affected server
```bash
sshpass -p 'Ir0nM@n' ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  tony@stapp01 "echo 'Ir0nM@n' | sudo -S systemctl status httpd --no-pager -l"
```
```text
AH00072: make_sock: could not bind to address [::]:6200
AH00072: make_sock: could not bind to address 0.0.0.0:6200
(98)Address already in use
```

### 4.3 Find who owns port 6200
```bash
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S ss -lntp | grep ':6200'"
```
```text
LISTEN 0 10 127.0.0.1:6200 0.0.0.0:* users:(("sendmail",pid=25212,fd=4))
```
```bash
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S lsof -nP -iTCP:6200 -sTCP:LISTEN"
```
```text
sendmail 25212 root ... TCP 127.0.0.1:6200 (LISTEN)
```

### 4.4 Confirm the conflicting service is safe to stop
```bash
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S systemctl status sendmail --no-pager -l"
```
```text
Active: active (running)
Main PID: 25212
```
Confirmed `disabled` (won't auto-restart at boot either).

### 4.5 Stop sendmail and verify the port is free
```bash
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S systemctl stop sendmail"
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S ss -lntp | grep ':6200' || echo 'PORT 6200 IS FREE'"
```
```text
PORT 6200 IS FREE
```

### 4.6 Start Apache and confirm the specific bound port
```bash
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S systemctl start httpd"
sshpass -p 'Ir0nM@n' ssh ... tony@stapp01 \
  "echo 'Ir0nM@n' | sudo -S systemctl status httpd --no-pager -l"
```
```text
Active: active (running)
Status: "Started, listening on: port 6200"
```
(The FQDN warning also present is unrelated noise — §2.8.)

### 4.7 Final verification across all three servers
```bash
for server in stapp01 stapp02 stapp03; do
  case "$server" in
    stapp01) USER="tony"; PASSWORD="Ir0nM@n" ;;
    stapp02) USER="steve"; PASSWORD="Am3ric@" ;;
    stapp03) USER="banner"; PASSWORD="BigGr33n" ;;
  esac
  echo "===== $server ====="
  sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
    "$USER@$server" "echo '$PASSWORD' | sudo -S bash -c '
      systemctl is-active httpd
      ss -lntp | grep \":6200\"
    '"
done
```
```text
stapp01: active, *:6200 httpd LISTEN
stapp02: active, *:6200 httpd LISTEN
stapp03: active, *:6200 httpd LISTEN
```

### 4.8 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `stapp01` identified as the faulty server | ✅ |
| Root cause identified (`sendmail` owning `:6200`) | ✅ |
| `sendmail` stopped, port confirmed free | ✅ |
| `httpd` started, confirmed listening on `:6200` specifically | ✅ |
| All three servers re-verified (no regression) | ✅ |
| Cosmetic FQDN warning correctly identified as not the cause | ✅ |

```text
stapp01                             stapp02 / stapp03
   │                                      │
sendmail :6200 (conflict)            httpd :6200 (already fine)
   │ stop
   ▼
:6200 free
   │ start
   ▼
httpd :6200 (active, listening)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AH00072: ... Address already in use` | Another process already bound to the required port | `ss -lntp \| grep :<port>` to identify the owner before touching Apache's config |
| Restarted Apache repeatedly with no success | Didn't identify the actual port owner first; retried the symptom instead of the cause | Diagnose with `ss`/`lsof` before any further `systemctl start/restart` attempts |
| Stopped a process without checking what it was | Risk of disrupting an unrelated, legitimate service | `systemctl status <service>` first to confirm identity and `enabled`/`disabled` state |
| Apache `active (running)` but still not reachable on the expected port | Didn't confirm the status line's specific "listening on: port N" detail | Check the full status output, not just the `active`/`inactive` summary |
| Chased the FQDN warning as if it were the failure | Treated every warning line as equally significant | Distinguish cosmetic/pre-existing warnings from the actual blocking error (`Address already in use`) |
| Fixed `stapp01` but didn't re-check `stapp02`/`stapp03` | Assumed the fix was isolated and couldn't affect the others | Always re-verify the full fleet after a fix, not just the host that was broken |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.10) out loud
      from "Apache unavailable on one server" to "all three verified,
      listening on :6200."
- [ ] Explain, in one sentence, why "Address already in use" points you
      toward a different process, not toward Apache's own configuration.
- [ ] Explain why using both `ss` and `lsof` to identify the port owner
      is better practice than trusting either alone.
- [ ] Explain why checking `systemctl status sendmail` before stopping
      it mattered, even though the fix turned out to be straightforward.
- [ ] Explain why the FQDN warning in Apache's output wasn't the actual
      problem, and how you'd distinguish a real blocking error from
      cosmetic noise in general.
- [ ] Compare this lab's root cause to Day 12's — what's identical,
      and what's different (the port number, the application)?
- [ ] Reproduce a similar port conflict on a test system (bind a dummy
      process to a port a real service needs) and diagnose it from
      memory using only `ss`/`lsof`.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: comparative check across the fleet → isolate the failing
host → read the actual error precisely → identify the specific
resource/conflict → confirm safety before acting → fix → verify the
specific detail, not just a summary state → re-verify the whole fleet.
End with a compressed arrow-chain version.>

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
