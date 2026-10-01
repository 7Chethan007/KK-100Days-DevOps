# Day 12 — Diagnosing an Apache Port Connectivity Failure

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the
diagnosis yourself*.

---

## 1. Scenario

In the Stratos Datacenter, Apache must listen on **TCP port 5002** on
all three app servers (`stapp01`, `stapp02`, `stapp03`), reachable from
the jump host:

```bash
curl http://stapp02:5002
```

`stapp01` fails to respond. Task: diagnose and fix it, **without
compromising the existing firewall security posture**.

The core lesson this lab is built around:

> **"Apache is running" does not necessarily mean "Apache is reachable."**

---

## 2. Reasoning model — how to *derive* the diagnosis, not memorize the fix

### 2.1 Every `curl` hides a chain of independent layers

```text
Jump Host (curl)
      │
      │ DNS / /etc/hosts
      ▼
10.244.189.204
      │
      │ TCP SYN → destination port 5002
      ▼
stapp01
      │
   Firewall
      │
   TCP :5002
      │
   httpd
```

For the request to succeed, **every** layer must work: name resolution,
routing, firewall, TCP port, application listener, HTTP response. The
discipline this lab teaches is walking this chain from the bottom up
(or systematically, layer by layer) instead of guessing at the top
(the application) first.

### 2.2 Comparative troubleshooting — use the servers that work as a control group

```bash
for server in stapp01 stapp02 stapp03; do
  echo "===== $server ====="
  nc -zv -w 3 "$server" 5002 2>&1
done
```
```text
stapp01 → No route to host
stapp02 → Connected
stapp03 → Connected
```

Two servers working and one failing is itself diagnostic evidence: this
is very unlikely to be a jump-host-wide or network-wide problem (those
would affect all three identically) — it narrows the search to
something specific to `stapp01`, before you've even looked at any
config on that host. This "find a working control to compare against"
instinct generalizes far beyond networking.

### 2.3 `nc -z` tests TCP only — deliberately below the HTTP layer

```bash
nc -zv -w 5 stapp01 5002
```
```text
nc
 ├── -z   scan/check connectivity, send no application data
 ├── -v   verbose
 └── -w 5 timeout after 5 seconds
```

Getting `No route to host` here means the failure happened **before**
HTTP even entered the picture — there's no value in inspecting Apache's
response content or config syntax yet, because the request never got
far enough to reach Apache in the first place. This is the single most
important triage step: confirm *which layer* failed before investigating
anything above it.

### 2.4 `No route to host` can mean "the firewall rejected it," not "there's no route"

```text
Chain INPUT
5 REJECT all ... reject-with icmp-host-prohibited
```

This is a genuinely important, non-obvious Linux networking fact: an
explicit firewall `REJECT` using an ICMP host-prohibited response
produces the *client-side* message `No route to host`, even though
routing itself is completely fine — the message describes the *symptom
the client observes*, not literally "no route exists." Taking this
message at face value and debugging routing tables would be chasing the
wrong layer entirely.

### 2.5 TCP connectivity vs. HTTP connectivity — `000` vs. `403` mean fundamentally different things

```bash
curl -sS --connect-timeout 3 -o /dev/null -w "HTTP=%{http_code}\n" http://stapp01:5002
```
```text
stapp01 → HTTP=000     (curl never got an HTTP response at all)
stapp02 → HTTP=403     (TCP worked, Apache responded, just denied the request)
stapp03 → HTTP=403
```

```text
HTTP=403  →  TCP ✓ → Apache received the request ✓ → Apache responded ✓ → (access denied)
HTTP=000  →  curl never got far enough to receive ANY HTTP response
```

`403` is actually **good news** for a connectivity diagnosis — it proves
the entire chain up through "Apache answered" works. `000` proves the
opposite: something below HTTP is still broken. Confusing these two
(treating `403` as "still broken") would send you chasing an application
config problem that doesn't exist.

### 2.6 Find who's actually listening — don't assume Apache itself is the cause of its own failure

```bash
systemctl is-active httpd
```
```text
failed
```
```bash
systemctl status httpd --no-pager -l
```
```text
AH00072: make_sock: could not bind to address [::]:5002
(98)Address already in use: AH00072: could not bind to address 0.0.0.0:5002
```

"Apache failed to start" is a symptom — `Address already in use` tells
you the actual question to ask isn't *"why is Apache broken"* but
*"who already owns port 5002?"*:

```bash
ss -lntp | grep ':5002'
```
```text
LISTEN 0 10 127.0.0.1:5002 ... users:(("sendmail"...))
```

Sendmail, not Apache, is the actual conflict — a completely different
service that happened to already be bound to the exact port Apache
needed.

### 2.7 Why a process bound only to `127.0.0.1` still blocks `0.0.0.0`

```text
0.0.0.0:5002   →  "all IPv4 interfaces" — INCLUDES 127.0.0.1:5002
127.0.0.1:5002 →  loopback-only, a SUBSET of what 0.0.0.0 covers
```

Apache can't claim the entire port across all interfaces
(`0.0.0.0:5002`) while sendmail already owns a subset of that same port
space (`127.0.0.1:5002`) — the two binding scopes genuinely overlap,
even though they look like "different addresses" at a glance. This is
worth understanding generally: `0.0.0.0` is not a *different* address
from `127.0.0.1`, it's a superset that includes it.

### 2.8 Fixing the first problem doesn't mean the second problem doesn't exist

```bash
systemctl stop sendmail
systemctl start httpd
systemctl is-active httpd    # active
ss -lntp | grep ':5002'      # LISTEN *:5002 httpd
```

Apache is now correctly running and listening on all interfaces — but
the jump host *still* gets `No route to host`. This is the lab's central
structural lesson: **there were two independent problems stacked on top
of each other**, and fixing the first (port conflict) only reveals that
the second (firewall) was there all along, previously masked by the
first failure. Stopping at "Apache is active now" would have been a
premature declaration of victory.

### 2.9 `iptables` processes rules top-to-bottom — order is the entire mechanism

```text
Chain INPUT (policy ACCEPT)
1 ACCEPT established connections
2 ACCEPT ICMP
3 ACCEPT loopback
4 ACCEPT TCP 22
5 REJECT all
```

A packet is evaluated against rules **in order**, and the first match
wins — TCP port 22 matches rule 4 and is accepted; TCP port 5002 matches
none of rules 1–4, falls through to rule 5, and is rejected. This
explains precisely why SSH worked throughout while Apache never did:
nothing about Apache's own health mattered, because the firewall never
let the packet reach it.

### 2.10 Inserting an ACCEPT rule in the correct position — why `-I ... 5`, not `-A`

```bash
iptables -I INPUT 5 -p tcp --dport 5002 -m conntrack --ctstate NEW -j ACCEPT
```

```text
iptables -A INPUT ... -j ACCEPT     → appended AFTER rule 5 (REJECT all)
                                        → packets never reach it; REJECT fires first
iptables -I INPUT 5 ... -j ACCEPT   → inserted AT position 5, BEFORE the REJECT
                                        → evaluated before the catch-all REJECT
```

`-A` (append) would have added a rule that's syntactically present but
functionally dead — the existing `REJECT all` would always match first.
`-I INPUT 5` inserts *before* that position, so the new rule is actually
reachable. Rule position is not cosmetic — it's the entire mechanism
that determines whether a rule ever executes.

### 2.11 Why disabling the firewall is the wrong fix, even though it would "work"

```text
systemctl stop firewalld / iptables -F    →  technically restores connectivity
                                             →  but removes ALL filtering, not just the one block
```

The task explicitly requires the fix not compromise security. The
correct approach — add one narrowly-scoped ACCEPT rule for exactly the
port needed — follows **default-deny, explicit-allow**: only the traffic
genuinely required gets a path through, everything else remains
rejected exactly as before. This is the same least-privilege principle
behind Day 10's "don't run the script as root/sudo" and Day 8's "don't
loosen sudo's PATH" — the easy fix that removes a restriction entirely
is almost never the correct one when a narrower fix is available.

### 2.12 A duplicate rule is harmless but worth cleaning up

```text
4 ACCEPT TCP 22
5 ACCEPT TCP 5002
6 ACCEPT TCP 5002   ← duplicate, inserted twice by accident
7 REJECT all
```

```bash
iptables -L INPUT -n -v --line-numbers
iptables -D INPUT <duplicate-rule-number>
```

Not a correctness bug (the first matching ACCEPT still wins), but
untidy and worth removing — and a useful reminder that **rule numbers
shift after any deletion**, so re-list before deleting a second rule
rather than trusting a previously-captured numbering.

### 2.13 Local vs. remote testing — they prove different things

```bash
curl http://localhost:5002    # tests ONLY: loopback → Apache (never leaves the host)
curl http://stapp01:5002      # tests: network → firewall → Apache (the FULL real path)
```

`403` from `localhost` on `stapp01` only proves Apache itself works
locally — it says nothing about whether the jump host can reach it. The
final proof has to come from the same location the original failure was
observed: the jump host, not the affected server itself.

### 2.14 Port numbers don't inherently "belong" to any service

```text
sendmail → 127.0.0.1:5002   (a perfectly valid, if unfortunate, choice)
httpd    → 0.0.0.0:5002      (Apache's config SAYS to use 5002, nothing more)
```

The OS has no concept of "port 5002 is Apache's port" — a port is just a
number any process can request to bind, and conflicts happen purely by
coincidence (or misconfiguration) of which services were told to use
which ports. "Apache's port" is a fact about Apache's *configuration*,
not a property the operating system or the port number itself enforces.

### 2.15 The compressed reasoning chain

```text
Requirement (Apache reachable on :5002 from jump host, firewall intact)
   → Comparative test: loop nc -zv over all 3 servers      → only stapp01 fails
   → nc -zv stapp01 5002                                     → "No route to host" (below-HTTP failure)
   → SSH to stapp01, check httpd                             → failed: "Address already in use"
   → ss -lntp | grep 5002                                    → sendmail already owns 127.0.0.1:5002
   → systemctl stop sendmail; systemctl start httpd           → httpd now active, listening *:5002
   → Re-test from jump host                                  → STILL "No route to host" (2nd problem)
   → iptables -L INPUT -n -v --line-numbers                   → catch-all REJECT after an SSH-only ACCEPT list
   → iptables -I INPUT 5 -p tcp --dport 5002 ... -j ACCEPT    → inserted BEFORE the REJECT, not appended after
   → Re-verify: ss, systemctl is-active, nc -zv, curl -w       → all green, HTTP=403 (connectivity proven)
   → Confirm from the JUMP HOST specifically, not just locally on stapp01
```

---

## 3. Concepts (reference)

### 3.1 Listening vs. reachable
A process can be confirmed `LISTEN`ing on a port (`ss -lntp`) while
still being completely unreachable from outside — listening only means
the OS will accept a local bind; reachability additionally depends on
routing, firewall rules, and network path, none of which `ss` reports.

### 3.2 `0.0.0.0` vs `127.0.0.1` vs a specific IP (recap from Day 6, Cloud-AWS)
`0.0.0.0` = all local interfaces (a superset); `127.0.0.1` = loopback
only (a subset of that superset). A process bound to the subset still
blocks another process from binding the full superset on the same port —
this is why sendmail-on-loopback and Apache-on-all-interfaces genuinely
conflicted.

### 3.3 TCP three-way handshake
`nc -zv` succeeding means a SYN → SYN-ACK → ACK handshake completed —
proof of TCP-level connectivity, independent of and prior to any
application-level (HTTP) exchange.

### 3.4 `iptables` rule evaluation order
Rules in a chain are evaluated top-to-bottom; the first match
determines the packet's fate, and evaluation stops there. A rule placed
after a catch-all `REJECT`/`DROP` is effectively unreachable regardless
of how correct its own match criteria are.

### 3.5 `iptables -I` (insert) vs. `-A` (append)
`-A` always adds at the end of the chain; `-I <chain> <position>` inserts
at a specific position. Given a catch-all rule at the end of a chain,
new permissive rules must be inserted *before* it, never appended after.

### 3.6 Default-deny, explicit-allow
A firewall policy that rejects everything not explicitly permitted,
rather than permitting everything not explicitly blocked — minimizes
attack surface by construction. The correct fix for "a legitimate
service can't get through" is always adding one precise allow rule, not
weakening or removing the default-deny posture.

### 3.7 HTTP status codes as connectivity evidence
`000` (curl never got a response) vs. any real status code (`403`,
`200`, etc. — Apache responded) is one of the clearest, fastest signals
for "is the failure above or below the HTTP layer."

### 3.8 Local test vs. remote test
Testing from the affected server itself (`localhost`) only exercises
the loopback interface and skips the network/firewall path entirely —
always reproduce the *original* failure from the *original* vantage
point (here, the jump host) for the test to actually mean something.

---

## 4. Runbook

### 4.1 Comparative test across all three servers
```bash
for server in stapp01 stapp02 stapp03; do
  echo "===== $server ====="
  nc -zv -w 3 "$server" 5002 2>&1
done
```
```text
===== stapp01 =====
No route to host

===== stapp02 =====
Connected

===== stapp03 =====
Connected
```

### 4.2 Confirm TCP vs. HTTP distinction via status codes
```bash
curl -sS --connect-timeout 3 -o /dev/null -w "HTTP=%{http_code}\n" http://stapp01:5002
curl -sS --connect-timeout 3 -o /dev/null -w "HTTP=%{http_code}\n" http://stapp02:5002
```
```text
stapp01: HTTP=000
stapp02: HTTP=403
```

### 4.3 SSH to the affected server and check Apache
```bash
ssh tony@stapp01
sudo su
systemctl is-active httpd
systemctl status httpd --no-pager -l
```
```text
failed
...
AH00072: make_sock: could not bind to address [::]:5002
(98)Address already in use: AH00072: could not bind to address 0.0.0.0:5002
```

### 4.4 Find who owns port 5002
```bash
ss -lntp | grep ':5002'
```
```text
LISTEN 0 10 127.0.0.1:5002 ... users:(("sendmail"...))
```

### 4.5 Stop the conflicting service, start Apache
```bash
systemctl stop sendmail
ss -lntp | grep ':5002'
```
(no output — port now free)
```bash
systemctl start httpd
systemctl is-active httpd
```
```text
active
```
```bash
ss -lntp | grep ':5002'
```
```text
LISTEN ... *:5002 ... httpd
```

### 4.6 Re-test from the jump host — second problem surfaces
```bash
nc -zv -w 5 stapp01 5002
```
```text
No route to host
```
Apache is confirmed healthy locally, yet the jump host still can't
reach it — a second, independent layer is still broken.

### 4.7 Inspect the firewall
```bash
firewall-cmd --list-all
```
```text
firewall-cmd: command not found
```
```bash
iptables -L -n -v
```
```text
Chain INPUT (policy ACCEPT)
1 ACCEPT established connections
2 ACCEPT ICMP
3 ACCEPT loopback
4 ACCEPT tcp dpt:22
5 REJECT all
```

### 4.8 Insert a precisely-scoped ACCEPT rule before the REJECT
```bash
iptables -I INPUT 5 \
  -p tcp \
  --dport 5002 \
  -m conntrack \
  --ctstate NEW \
  -j ACCEPT
```

### 4.9 Verify the firewall rule ordering
```bash
iptables -L INPUT -n -v --line-numbers
```
```text
4 ACCEPT tcp dpt:22
5 ACCEPT tcp dpt:5002
6 REJECT all
```
(If a duplicate rule exists from a retry, clean it up:)
```bash
iptables -L INPUT -n -v --line-numbers
iptables -D INPUT <duplicate-rule-number>
```

### 4.10 Verify locally on stapp01
```bash
curl -I http://localhost:5002
```
```text
HTTP/1.1 403 Forbidden
```
Expected — `/var/www/html` has no `index.html` and directory listing is
disabled; **do not create/modify it**, the task warns against altering
it, and `403` is sufficient proof of connectivity (§2.5).

### 4.11 Final verification from the jump host
```bash
nc -zv -w 5 stapp01 5002
```
```text
Connected to 10.244.x.x:5002
```
```bash
curl -I http://stapp01:5002
```
```text
HTTP/1.1 403 Forbidden
```

### 4.12 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Comparative diagnosis isolates `stapp01` | ✅ |
| Port-conflict root cause identified (`sendmail`) | ✅ |
| Sendmail stopped, Apache started and listening on `*:5002` | ✅ |
| Firewall root cause identified (catch-all REJECT) | ✅ |
| Precise ACCEPT rule inserted before the REJECT, nothing else weakened | ✅ |
| Reachable from the jump host (`nc` connects, `curl` returns `403`) | ✅ |
| Existing security posture (SSH-only + default-deny) preserved | ✅ |

```text
Jump Host
    │  nc -zv / curl
    ▼
iptables INPUT
    │  4 ACCEPT :22
    │  5 ACCEPT :5002   ← added
    │  6 REJECT all
    ▼
stapp01 : httpd
    │  (sendmail no longer on :5002)
    ▼
*:5002  LISTEN
    │
    ▼
HTTP/1.1 403 Forbidden   ← connectivity proven
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No route to host` from `nc`/`curl` | Could mean an actual routing issue, OR a firewall `REJECT` using `icmp-host-prohibited` | Check `iptables -L -n -v` before assuming it's a routing/DNS problem — the message doesn't distinguish the two |
| `httpd` fails to start: `Address already in use` | Another process already bound the same port (possibly on a narrower address) | `ss -lntp \| grep :<port>` to identify the actual owner before assuming Apache's own config is broken |
| Stopped the conflicting service, started Apache, but the jump host still can't connect | A second, independent layer (firewall) is also blocking it | Don't stop investigating after fixing the first root cause — re-test from the original failing vantage point |
| New `iptables -A` ACCEPT rule doesn't seem to take effect | Appended after an existing catch-all `REJECT`/`DROP`, so it's never reached | Use `-I <chain> <position>` to insert *before* the catch-all rule, not `-A` |
| Tempted to `systemctl stop firewalld` / flush all rules to "just fix it" | Treating the firewall itself as the obstacle instead of the missing one rule | Add exactly the one ACCEPT rule needed; never disable the whole security layer as a shortcut |
| Two identical ACCEPT rules for the same port | Inserted the same rule twice during troubleshooting | `iptables -L INPUT -n -v --line-numbers`, delete the duplicate by its current number (re-list before each delete — numbers shift) |
| `curl http://localhost:5002` works but the jump host still fails | Local test only exercises loopback, never the real network/firewall path | Always confirm from the original failing location, not just locally on the affected host |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.15) out loud
      from "stapp01 unreachable on :5002" to "verified from the jump
      host, firewall intact."
- [ ] Explain, in one sentence, why `No route to host` doesn't always
      mean there's literally no network route.
- [ ] Explain why a process bound to `127.0.0.1:5002` can still block
      another process from binding `0.0.0.0:5002`.
- [ ] Explain why `iptables -A` would have failed to fix this firewall
      issue, while `iptables -I INPUT 5` worked.
- [ ] Explain why `HTTP=403` is actually a *successful* connectivity
      test result for this specific task.
- [ ] Explain why disabling the firewall entirely would have "worked"
      but was the wrong fix.
- [ ] Reproduce both failures on a test system (bind a dummy process to
      a port Apache needs; add a catch-all REJECT rule) and diagnose
      each from memory using only the tools in §3.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: comparative/baseline test → isolate the failing layer →
diagnose each root cause individually (there may be more than one,
stacked) → fix → re-run → verify from the ORIGINAL failing vantage
point. End with a compressed arrow-chain version.>

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
