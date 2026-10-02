# Day 13 — iptables Installation & Configuration

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the firewall
policy yourself*.

---

## 1. Scenario

```text
                    ┌─────────────────┐
                    │    Jump Host    │
                    │      thor       │
                    └────────┬────────┘
                             │  (should be REJECTED on :6300)
              ┌──────────────┴───────────────┐
              │                              │
              ▼                              ▼
       ┌──────────────┐              ┌──────────────┐
       │   stapp01    │              │   stapp02    │
       │ Apache :6300 │              │ Apache :6300 │
       └──────────────┘              └──────────────┘
              ▲                              ▲
              │        (should be ACCEPTED)  │
              └──────────────┬───────────────┘
                             │
                      ┌──────┴──────┐
                      │   stlb01    │
                      │Load Balancer│
                      └─────────────┘
```

Apache listens on TCP `6300` on `stapp01`/`stapp02`/`stapp03`. Task:
configure `iptables` on each app server so that **only the load
balancer** (`stlb01`) can reach port `6300`; every other source (the
jump host included) must be rejected — SSH and already-established
connections must keep working, and the policy must survive a reboot.

---

## 2. Reasoning model — how to *derive* the firewall policy, not memorize the commands

### 2.1 Listening is not the same thing as reachable

```bash
ss -lntp | grep ':6300'
```
```text
LISTEN 0 511 *:6300 *:* users:(("httpd"...))
```

`LISTEN` only means Apache is *ready* to accept connections — it says
nothing about whether the network actually lets any particular client
get there. **Application availability ≠ network reachability** —
exactly the lesson from Day 12's Apache-on-5002 lab, now applied at the
firewall layer specifically instead of a service-startup layer.

### 2.2 Test before changing anything — establish the "before" baseline

```bash
for server in stapp01 stapp02 stapp03; do
  nc -zv -w 3 "$server" 6300
done
```
```text
Connected
Connected
Connected
```

Before adding any firewall rule, everyone (including the jump host)
could reach port 6300 — confirming this baseline first is what makes
"after configuration, only the LBR can connect" a provable before/after
change, rather than an unverified assumption.

### 2.3 A packet carries source, destination, protocol, and port — the firewall's entire vocabulary

```text
┌──────────────────────────────┐
│ Source IP       10.244.164.117│
│ Destination IP  10.244.195.53 │
│ Protocol        TCP           │
│ Destination Port 6300         │
│ Connection state (conntrack)  │
└──────────────────────────────┘
```

Every `iptables` rule is a boolean predicate over exactly these fields
— reduce the task's requirement to this vocabulary *before* writing any
command:

```text
WHO (source)   → 10.244.164.117 (the LBR) → ACCEPT; anyone else → REJECT
WHAT (proto)    → TCP
WHERE (port)    → 6300
WHEN (state)    → applies to NEW connection attempts
```

### 2.4 `iptables` is user-space configuration for a kernel mechanism (netfilter)

```text
iptables   → the CLI tool you type commands into
netfilter  → the actual kernel packet-filtering framework iptables configures
```

Running `iptables -A INPUT ...` doesn't start some separate firewall
daemon — it's configuring behavior the Linux kernel's own networking
stack already implements. This matters because the rules take effect
immediately, in the kernel, for every packet — there's no separate
"firewall service" to restart for the *filtering* to apply (though a
systemd unit does matter for *persistence*, §2.11).

### 2.5 `INPUT` is the chain that matters here — traffic arriving at this host

```text
INPUT   → traffic destined FOR this machine
FORWARD → traffic passing THROUGH this machine (routing/NAT scenarios)
OUTPUT  → traffic originating FROM this machine
```

Since the app servers are the final destination of the LBR's and jump
host's connections (not routing traffic onward to somewhere else),
`INPUT` is the only chain this task needs to touch.

### 2.6 Rules evaluate top-to-bottom; first match wins — the entire mechanism this task exploits

```text
1 ACCEPT  ESTABLISHED,RELATED
2 ACCEPT  TCP port 22
3 ACCEPT  TCP, source 10.244.164.117, dport 6300
4 REJECT  TCP, dport 6300  (no source restriction — matches EVERYONE ELSE)
```

The LBR's traffic matches rule 3 and stops there — never reaching rule
4. The jump host's traffic matches none of rules 1–3, falls through to
rule 4, and gets rejected. **The same packet shape (`TCP → :6300`)
produces different outcomes purely because of which source IP it
carries and where in the ordered list that distinction gets evaluated.**
This is the exact same "order determines outcome" mechanism as Day 12's
firewall fix, now used *proactively* to build a policy rather than
reactively to unblock one.

### 2.7 Rule 1 — `ESTABLISHED,RELATED`: don't re-evaluate traffic you've already approved

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

TCP connections have state, and Linux tracks it (`conntrack`). Once a
connection has been accepted once, its subsequent packets are
`ESTABLISHED` — accepting those immediately, first in the list, avoids
re-running every subsequent packet of an already-approved connection
through the full rule list on every single packet. `RELATED` covers
traffic legitimately associated with an existing connection (certain
protocols open secondary connections tied to an original one).

### 2.8 Rule 2 — SSH first: preserve your own management access before adding restrictions

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

The single most important operational sequencing rule in this entire
lab: **add the rule that keeps you able to manage the machine *before*
adding anything restrictive.** Reversing the order — restricting first,
then trying to add an SSH exception — risks locking yourself out of a
remote machine with no other access path. This generalizes far beyond
`iptables`: any time you're remotely tightening access control, secure
your own path first.

### 2.9 Rule 3 — the actual access-control decision: source IP, not just port

```bash
iptables -A INPUT -p tcp -s 10.244.164.117 --dport 6300 -j ACCEPT
```

```text
-p tcp            → protocol must be TCP
-s 10.244.164.117 → packet must ORIGINATE from the load balancer specifically
--dport 6300       → destination port must be 6300
-j ACCEPT           → action: allow it through
```

This is the rule that actually encodes "only the load balancer" — every
other field (`-p`, `--dport`) also appears in rule 4, so `-s` is the
*only* thing distinguishing "LBR traffic" from "everyone else's
traffic" to this port.

### 2.10 Rule 4 — reject everyone else, with no source restriction at all

```bash
iptables -A INPUT -p tcp --dport 6300 -j REJECT --reject-with icmp-port-unreachable
```

Omitting `-s` here is deliberate — this rule matches *any* source,
which is exactly "everyone who wasn't already accepted by an earlier,
more specific rule." Because rule 3 already caught the LBR's traffic
and stopped evaluation there, this catch-all only ever actually fires
for traffic from sources other than the LBR.

### 2.11 `DROP` vs. `REJECT` — a deliberate choice, not an arbitrary one

```text
DROP    → packet silently discarded; client eventually times out
REJECT  → packet explicitly refused; client gets an immediate error
          (--reject-with icmp-port-unreachable here)
```

`nc` reporting `Connection refused` (not a timeout) confirms `REJECT`
was used and configured correctly — a `DROP` policy would have produced
a hang/timeout instead. Neither is universally "more secure"; `REJECT`
gives faster, clearer feedback (useful for a lab/diagnostics context),
while `DROP` can be preferred in some security postures specifically
*because* it gives an attacker less information.

### 2.12 Runtime rules vs. persisted rules — two entirely separate concerns

```text
iptables -A INPUT ...   →  modifies the kernel's ACTIVE rule set RIGHT NOW
                            (gone on reboot unless explicitly saved)

service iptables save   →  writes the current rules to /etc/sysconfig/iptables
systemctl enable iptables → ensures the SAVED rules are reloaded on every future boot
```

This is the exact same "runtime state vs. boot-time configuration"
distinction as Day 5's SELinux lab (`getenforce` vs.
`/etc/selinux/config`) — a firewall rule you typed is real and active
immediately, but invisible to a fresh boot unless it's been explicitly
persisted through a save step. `systemctl is-active` answers "is it
enforcing right now"; `systemctl is-enabled` answers "will the saved
rules reload on the next boot" — both must be true for the policy to
actually be durable.

### 2.13 Why a package being installed didn't mean the command existed

```text
iptables-services  →  installed  (the systemd persistence integration)
iptables             →  NOT installed  (the actual CLI binary)
```

Two related but separately-packaged pieces — installing one doesn't
install the other. `dnf install -y iptables iptables-services` was
needed for both the executable and the persistence tooling, matching
the "package installed ≠ service configured ≠ service running" chain
from Day 11's Tomcat lab, here split across two separate packages
instead of one package's lifecycle states.

### 2.14 Test both directions — a denied path and an allowed path are two separate claims

```text
From jump host → stapp0N:6300   → expect REJECTED (the negative test)
From stlb01    → stapp0N:6300   → expect CONNECTED (the positive test)
```

Confirming the jump host is now blocked proves *something* is being
filtered — it does **not** prove the load balancer wasn't accidentally
blocked too. A security rule is a claim about *both* "what's denied"
and "what remains allowed" — verifying only one half leaves the other
half completely unverified. This mirrors the "verify from both
directions" discipline used for AWS resource relationships throughout
the Cloud-AWS track (Days 10, 11, 12), here applied to a firewall policy
instead of a resource attachment.

### 2.15 Packet counters as a debugging tool, not just a curiosity

```bash
iptables -L INPUT -n -v --line-numbers
```
```text
3  0  0 ACCEPT tcp ... 10.244.164.117 ... dpt:6300
4  1 60 REJECT tcp ... dpt:6300
```

The `pkts`/`bytes` columns show *which rule actual traffic has hit* —
after testing from the jump host, rule 4's counter incrementing proves
that traffic really did fall through to the reject rule (not stall
somewhere else entirely); after testing from the LBR, rule 3's counter
incrementing proves the accept path is genuinely being exercised. This
turns "I think the rule works" into "I can see exactly which rule
matched, with numbers."

### 2.16 Why NAT matters for any `-s`-based rule, even though this lab didn't hit it

```text
If the LBR's traffic were NAT'd before reaching the app server, the
app server might see a DIFFERENT source IP than the LBR's own address.
```

A source-IP-based firewall rule is only correct if the destination host
actually *sees* that IP as the source — always confirm the real
observed source IP (`hostname -I` on the LBR, or inspecting traffic
directly) rather than assuming a device's "own" IP is what a downstream
host will see, especially anywhere NAT, a proxy, or a load balancer's
own networking sits in between.

### 2.17 The compressed reasoning chain

```text
Requirement (only LBR reaches :6300; SSH/established preserved; persisted)
   → Baseline: nc -zv from jump host to all 3 app servers    → all Connected (before)
   → Confirm httpd is listening                                → ss -lntp shows *:6300
   → Identify the LBR's actual source IP                        → hostname -I on stlb01 → 10.244.164.117
   → Install iptables + iptables-services (both needed, §2.13)
   → iptables -F / -X  (clear any stale state first)
   → Rule 1: ESTABLISHED,RELATED → ACCEPT    (don't re-filter known-good traffic)
   → Rule 2: TCP :22 → ACCEPT                  (preserve SSH BEFORE restricting anything — §2.8)
   → Rule 3: TCP, -s <LBR IP>, :6300 → ACCEPT  (the actual allow decision)
   → Rule 4: TCP, :6300 → REJECT                (catch-all for everyone else)
   → service iptables save; systemctl enable --now iptables   (persist + activate)
   → Verify rule ORDER and packet counters       → iptables -L INPUT -n -v --line-numbers
   → NEGATIVE test: jump host → :6300             → Connection refused
   → POSITIVE test: stlb01 → :6300                 → Connected
   → Confirm httpd/ss still show Apache healthy    → the block is network-layer, not app-layer
```

---

## 3. Concepts (reference)

### 3.1 Application availability vs. network reachability
A service can be fully healthy and `LISTEN`ing while still being
completely unreachable due to a firewall — these are independent facts,
checked by different tools (`systemctl`/`ss` for the first,
`nc`/`iptables -L` for the second).

### 3.2 Chains: `INPUT` / `FORWARD` / `OUTPUT`
`INPUT` = traffic destined for this host; `FORWARD` = traffic passing
through this host to somewhere else (routing scenarios); `OUTPUT` =
traffic originating from this host. This lab only needed `INPUT`.

### 3.3 Rule ordering is the entire access-control mechanism
`iptables` evaluates rules top-to-bottom per chain; the first matching
rule's action applies and evaluation stops. A more specific allow rule
placed before a broader deny rule is what makes "allow X, deny
everyone else" expressible at all.

### 3.4 Connection tracking (`conntrack`) and `ESTABLISHED`/`RELATED`/`NEW`
Linux tracks TCP connection state; `ESTABLISHED` means a packet belongs
to an already-approved connection, `RELATED` means it's legitimately
associated with one, `NEW` means it's the first packet of a connection
attempt. Accepting `ESTABLISHED,RELATED` early is a standard, efficient
baseline rule.

### 3.5 `DROP` vs. `REJECT`
`DROP` silently discards a packet (client times out); `REJECT` sends an
explicit refusal (client fails fast, as this lab's `Connection refused`
demonstrates). The choice is a deliberate security/operational
tradeoff, not a default to leave unexamined.

### 3.6 Runtime state vs. persisted configuration (recap from Day 5)
Firewall rules exist in kernel memory the moment you run `iptables -A`
— they vanish on reboot unless explicitly saved (`service iptables
save`) and the systemd unit is enabled (`systemctl enable iptables`) to
reload them at boot.

### 3.7 `iptables -L -n -v --line-numbers`
`-L` lists rules; `-n` avoids slow/misleading DNS-based name resolution
of IPs; `-v` shows packet/byte counters (debugging gold, §2.15);
`--line-numbers` shows each rule's position (needed to delete/insert
precisely).

### 3.8 The ACL pattern recurring across infrastructure layers
`iptables` rules, AWS Security Groups, Network ACLs, and Kubernetes
NetworkPolicies are all the same underlying idea — WHO/WHAT/WHERE/WHEN
→ ACCEPT or DENY — expressed in different syntax at different layers of
the stack. Traffic generally must satisfy *every* layer it passes
through, not just one.

### 3.9 NAT's effect on source-IP-based rules
If address translation sits between a client and a firewall, the
firewall may see a different source IP than the client's own —
always verify the actually-observed source IP, not an assumed one,
before writing a `-s`-based rule.

---

## 4. Runbook

### 4.1 Discover the relevant IPs
```bash
getent hosts stlb01
for server in stapp01 stapp02 stapp03; do getent hosts "$server"; done
```
```text
LBR: 10.244.164.117
```

### 4.2 Confirm Apache is listening (before touching the firewall)
```bash
systemctl is-active httpd
ss -lntp | grep ':6300'
```
```text
active
LISTEN ... *:6300 ... httpd
```

### 4.3 Establish the "before" baseline from the jump host
```bash
for server in stapp01 stapp02 stapp03; do
  nc -zv -w 3 "$server" 6300
done
```
```text
Connected
Connected
Connected
```

### 4.4 Install iptables and its persistence tooling
```bash
sudo dnf install -y iptables iptables-services
iptables --version
```
```text
iptables v1.8.10 (legacy)
```

### 4.5 Clear any stale rules, then build the policy (on each app server)
```bash
sudo iptables -F
sudo iptables -X

sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp -s 10.244.164.117 --dport 6300 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6300 -j REJECT --reject-with icmp-port-unreachable
```

### 4.6 Persist and enable
```bash
sudo service iptables save
sudo systemctl enable iptables
sudo systemctl start iptables
```

### 4.7 Verify rule order and packet counters
```bash
sudo iptables -L INPUT -n -v --line-numbers
```
```text
Chain INPUT (policy ACCEPT)
num  target   prot  source           destination
1    ACCEPT   all   anywhere         anywhere
2    ACCEPT   tcp   anywhere         anywhere
3    ACCEPT   tcp   10.244.164.117   anywhere
4    REJECT   tcp   anywhere         anywhere
```

### 4.8 Verify the service itself
```bash
systemctl is-active iptables
systemctl is-enabled iptables
```
```text
active
enabled
```

### 4.9 Confirm Apache is still healthy (the block is network-layer, not app-layer)
```bash
systemctl is-active httpd
ss -lntp | grep ':6300'
```

### 4.10 Negative test — from the jump host
```bash
for server in stapp01 stapp02 stapp03; do
  nc -zv -w 3 "$server" 6300
done
```
```text
stapp01 → Connection refused
stapp02 → Connection refused
stapp03 → Connection refused
```

### 4.11 Positive test — from the load balancer
```bash
ssh <user>@stlb01
hostname -I
```
```text
10.244.164.117
```
```bash
for server in stapp01 stapp02 stapp03; do
  nc -zv -w 3 "$server" 6300
done
```
```text
stapp01 → Connected
stapp02 → Connected
stapp03 → Connected
```

### 4.12 Inspect packet counters after both tests
```bash
sudo iptables -L INPUT -n -v --line-numbers
```
```text
3  1  60 ACCEPT tcp ... 10.244.164.117 ... dpt:6300
4  1  60 REJECT tcp ... dpt:6300
```
Both rules show real hits — the allow path and the deny path were each
genuinely exercised.

### 4.13 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Established/related connections preserved | ✅ |
| SSH (:22) still accessible | ✅ |
| Only `10.244.164.117` (LBR) reaches `:6300` | ✅ |
| All other sources rejected on `:6300` | ✅ |
| Apache itself unaffected (still `active`, `LISTEN`) | ✅ |
| Rules persisted (`iptables save`) and service enabled | ✅ |
| Verified from BOTH an unauthorized and an authorized source | ✅ |

```text
                         Jump Host  ──X── REJECT ──┐
                                                     │
                      Load Balancer ── ACCEPT ──┐    │
                      10.244.164.117            ▼    ▼
                                          ┌──────────────┐
                                          │  iptables    │
                                          │  INPUT chain │
                                          └──────┬───────┘
                                                 │ (only LBR passes)
                                                 ▼
                                          Apache :6300
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `iptables: command not found` despite `iptables-services` being installed | The CLI binary and the systemd persistence package are separate packages | `dnf install -y iptables` (in addition to `iptables-services`) |
| Locked out of SSH after applying firewall rules | Restrictive rule(s) added before an explicit SSH-allow rule | Always add the SSH ACCEPT rule first, before anything restrictive (§2.8) |
| LBR traffic unexpectedly rejected too | The allow rule for the LBR's IP is positioned AFTER the catch-all reject, or the IP used doesn't match what the server actually sees (possible NAT) | Check rule order with `--line-numbers`; confirm the LBR's real observed source IP |
| Rules work now but vanish after a reboot | Rules were never saved/persisted | `service iptables save` + `systemctl enable iptables` |
| `nc` hangs instead of immediately refusing | Using `DROP` instead of `REJECT`, or testing against a different, unconfigured server | Confirm which action the matching rule uses; `REJECT` fails fast, `DROP` times out |
| Unsure whether a specific rule is actually being hit | Only inspected rule text, not traffic evidence | `iptables -L INPUT -n -v` and check the `pkts`/`bytes` counters after generating test traffic (§2.15) |
| Verified the deny path only, declared the task done | Didn't test that the legitimate source (LBR) still works | Always test both directions — denied AND allowed (§2.14) |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.17) out loud
      from "only the LBR may reach :6300" to "verified from both an
      authorized and unauthorized source, persisted across reboot."
- [ ] Explain, in one sentence, why the SSH-allow rule must be added
      before any restrictive rule, not after.
- [ ] Explain why rule 4 (the catch-all reject) never actually blocks
      the load balancer's traffic, even though it has no source
      restriction at all.
- [ ] Explain the difference between `DROP` and `REJECT`, and which one
      this lab's `Connection refused` result proves was used.
- [ ] Explain why testing only "the jump host is now blocked" doesn't
      prove the firewall policy is correct.
- [ ] Explain the difference between `systemctl is-active iptables` and
      `systemctl is-enabled iptables`, and why both need to be true.
- [ ] Rebuild the same four-rule policy from memory on a fresh host,
      verifying with packet counters (not just rule text) that each
      rule is actually being exercised by real test traffic.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: establish a baseline → reduce the requirement to its
underlying mechanism's vocabulary → build the solution rule/step by
rule/step, in the order that avoids self-lockout or other hazards →
persist → verify from BOTH the allowed and denied perspective. End with
a compressed arrow-chain version.>

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
