# Day 16 — Install and Configure Nginx as a Load Balancer

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the config
yourself*.

This is Nginx's second distinct role in this series — Day 15 used it
for TLS termination on a single backend; today it's a reverse proxy
distributing traffic across three backends. Same tool, same
`nginx -t`-before-`restart` discipline, a different `server {}`/context
structure.

---

## 1. Scenario

Three Apache application servers (`stapp01`–`stapp03`, each on port
`3001`) need traffic distributed across them via an Nginx load balancer
on `stlb01`, configured in `/etc/nginx/nginx.conf`. Constraint: do
**not** change Apache's port on any app server.

```text
                     ┌── stapp01:3001 ── Apache
                     │
Client ── HTTP ──> stlb01:80
                     │ Nginx
                     ├── stapp02:3001 ── Apache
                     │
                     └── stapp03:3001 ── Apache
```

---

## 2. Reasoning model — how to *derive* the config, not memorize it

### 2.1 What a load balancer actually buys you

```text
No LBR:                          With LBR:
Client ──────> stapp01               Client ──> LBR ──┬── stapp01
               (single point of                        ├── stapp02
                bottleneck)                             └── stapp03
```

Without a distribution layer, every client request lands on one
server, which becomes a bottleneck as traffic grows. A load balancer
sits in front of several identical backends and spreads requests across
them — the same horizontal-scaling motivation behind the entire reason
this lab exists.

### 2.2 Confirm the existing backend state before touching anything — don't assume

```bash
sshpass -p 'Ir0nM@n' ssh -o StrictHostKeyChecking=no tony@stapp01 \
  "sudo -S ss -lntp <<< 'Ir0nM@n' | grep httpd"
```
```text
*:3001 ... httpd
```
(repeated for `stapp02`/`steve`/`Am3ric@` and `stapp03`/`banner`/`BigGr33n`)

Confirming all three backends are genuinely listening on `3001` and
`active` *before* writing any Nginx config is the same "inspect before
act" discipline running through this entire series — it also directly
establishes the exact port number the `upstream` block needs to
reference, rather than assuming it from the task description alone.

### 2.3 Why "don't change the Apache port" is the actual design constraint here

```text
Apache stays on :3001 (unchanged)
          │
          ▼
Nginx's upstream block points AT :3001 — the backend's port is Nginx's
problem to know about, not Apache's problem to change
```

Modifying three already-working Apache configurations to match some
new convention would be unnecessary risk for zero benefit — the whole
point of fronting them with a reverse proxy is that the backend's
existing port becomes an implementation detail the *proxy* knows about,
invisible to the client entirely. This is a direct, concrete instance
of the general principle: prefer changing the new component you're
introducing over modifying already-working, unrelated components.

### 2.4 Nginx's configuration hierarchy — context inside context

```nginx
http {
    upstream nautilus_backend { ... }   # defines a backend POOL
    server {                             # defines a LISTENER + rules
        listen 80;
        location / { proxy_pass http://nautilus_backend; }
    }
}
```

```text
http       → top-level context for all HTTP-related config
  upstream → names a group of backend servers (a POOL)
  server   → defines what Nginx itself listens on, and how it handles requests
    location → path-matching rules within a server block
```

Same nested-block structure as Day 15's SSL config, extended here with
one new context (`upstream`) that Day 15 never needed, because Day 15
proxied to nothing — it served content directly from a local `root`.

### 2.5 `upstream` — naming a pool, independent of how it's later used

```nginx
upstream nautilus_backend {
    server stapp01:3001;
    server stapp02:3001;
    server stapp03:3001;
}
```

This block does nothing by itself — it just gives a name
(`nautilus_backend`) to a set of backend addresses. The actual routing
decision (send traffic *to* this pool) happens separately, in a
`server {}` block's `location`. Separating "what backends exist" from
"when do I use them" is what makes the same upstream pool reusable
across multiple `server` blocks if needed later (e.g. different ports,
different `server_name`s all routing to the same backend pool).

### 2.6 `proxy_pass` — the actual routing decision

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://nautilus_backend;
    }
}
```

`proxy_pass` is the line that actually connects "requests arriving
here" to "the named upstream pool" — without it, the `upstream` block
from §2.5 would be fully valid, named, and completely unused. The
client talking to `stlb01:80` never needs to know Apache exists on port
`3001` at all; Nginx performs that translation entirely on the server
side.

### 2.7 Default load balancing is round-robin — no extra config required for this lab

```text
Request 1 → stapp01
Request 2 → stapp02
Request 3 → stapp03
Request 4 → stapp01
...
```

Listing multiple `server` lines inside one `upstream` block is already
sufficient to get round-robin distribution — this is Nginx's default
behavior with no additional directive needed. More advanced strategies
(`least_conn`, `ip_hash`) exist but weren't required here; don't add
complexity the task doesn't ask for.

### 2.8 Why identical responses from every backend don't disprove load balancing

```bash
for i in {1..10}; do curl -s http://stlb01:80; echo; done
```
```text
Welcome to xFusionCorp Industries!   (x10, identical each time)
```

All three backends serve the exact same static content, so round-robin
distribution is completely invisible from the response body alone —
getting the same text back ten times is the *expected* result, not
evidence the load balancer isn't actually distributing requests. To
actually observe which backend served which request, you'd need either
distinguishable backend content (a hostname baked into each server's
response) or to inspect Nginx's own access logs (`/var/log/nginx/
access.log`) alongside each backend's own logs — a genuinely useful
distinction to internalize, since "the output looks identical" is an
easy, wrong signal to read as "this isn't working."

### 2.9 Backing up before editing the main config — same discipline as Day 15

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```

Same reasoning as Day 15's certificate/config backup — a one-line
safety net before editing a file an active, already-working service
depends on.

### 2.10 Validate before restarting — the same non-negotiable gate as Day 15

```bash
sudo nginx -t
```
```text
syntax is ok
test is successful
```

Identical discipline to Day 15: never restart a running web/proxy
server on unvalidated configuration. The consequence of skipping this
step here is arguably worse than Day 15's single-backend case — a typo
in the `upstream` block could take down traffic to *all three* backend
servers simultaneously, not just one.

### 2.11 Verify at the correct layer — backend connectivity, then the proxy, then end-to-end

```text
Layer 1 (Apache itself):     curl http://stapp01:3001  (from stlb01 directly)
Layer 2 (Nginx config):       nginx -t
Layer 3 (Nginx service):      systemctl is-active nginx
Layer 4 (end-to-end, from the ACTUAL client vantage point):
                               curl http://stlb01:80     (from the jump host)
```

Same layered-verification, "test from the real client vantage point"
discipline as Day 15's SSL lab and Day 13's iptables lab — testing
`curl http://stlb01:80` from `stlb01` itself would only prove the proxy
layer works locally; testing from the jump host proves the entire
network path a real client would actually take.

### 2.12 The compressed reasoning chain

```text
Requirement (Nginx LBR on stlb01 → 3 Apache backends on :3001, Apache unchanged)
   → Confirm all 3 Apache backends: active, listening on :3001 (don't assume)
   → Install + enable Nginx on stlb01
   → Back up /etc/nginx/nginx.conf
   → Add upstream nautilus_backend { server stapp0N:3001; x3 }
   → Add server { listen 80; location / { proxy_pass http://nautilus_backend; } }
   → nginx -t                                   → validate BEFORE restarting
   → systemctl restart nginx; is-active           → active
   → curl http://stapp0N:3001 from stlb01 (optional) → confirm each backend directly reachable
   → curl http://stlb01:80 FROM THE JUMP HOST       → the actual end-to-end proof
   → Repeated curls returning identical content     → EXPECTED, not evidence LB isn't working (§2.8)
```

---

## 3. Concepts (reference)

### 3.1 Load balancer / reverse proxy
A component sitting between clients and backend servers that accepts
client requests and forwards them on the client's behalf — the client
never directly communicates with the backend, and has no knowledge of
how many backends exist or which one actually served a given request.

### 3.2 `upstream` block
Names a pool of backend addresses, independent of how/when that pool
gets used. Multiple `server {}` blocks could reference the same
`upstream` name if a more complex routing setup required it.

### 3.3 `proxy_pass`
The directive that actually forwards a matched request to a named
upstream (or a direct address). Without it, an `upstream` block is
inert — defined but never invoked.

### 3.4 Round-robin as Nginx's default load-balancing algorithm
Simply listing multiple `server` lines in an `upstream` block is
sufficient for round-robin distribution with no extra directive —
`least_conn`/`ip_hash`/etc. are opt-in alternatives for more specific
distribution needs.

### 3.5 Frontend port vs. backend port — decoupled by design
The port a client connects to (`stlb01:80`) and the port a backend
actually listens on (`stapp0N:3001`) don't need to match, and
deliberately shouldn't be conflated — the proxy is exactly the
component responsible for translating between the two.

### 3.6 Why identical backend responses don't disprove load balancing (recap)
Static, identical content across all backends makes round-robin
distribution invisible at the response-body level — verifying actual
distribution requires either distinguishable backend content or
inspecting logs, not just repeating the same `curl`.

### 3.7 `nginx -t` as a mandatory pre-restart gate (recap from Day 15)
Validates configuration syntax/sanity without touching the running
process — essential before any `restart`/`reload`, and higher-stakes
here than in Day 15 since a mistake could affect traffic to every
backend simultaneously rather than just one.

---

## 4. Runbook

### 4.1 Confirm all three Apache backends
```bash
sshpass -p 'Ir0nM@n' ssh -o StrictHostKeyChecking=no tony@stapp01 \
  "sudo -S ss -lntp <<< 'Ir0nM@n' | grep httpd"
```
```text
*:3001 ... httpd
```
(repeat for `steve@stapp02` / `Am3ric@`, `banner@stapp03` / `BigGr33n`)

```bash
sshpass -p 'Ir0nM@n' ssh -o StrictHostKeyChecking=no tony@stapp01 \
  "sudo -S systemctl is-active httpd <<< 'Ir0nM@n'"
```
```text
active
```
(repeat for the other two)

### 4.2 Connect to the load balancer and install Nginx
```bash
sshpass -p 'Mischi3f' ssh -o StrictHostKeyChecking=no loki@stlb01
sudo -S dnf install -y nginx <<< 'Mischi3f'
sudo -S systemctl enable nginx <<< 'Mischi3f'
```

### 4.3 Back up the main config
```bash
sudo -S cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak <<< 'Mischi3f'
```

### 4.4 Edit `/etc/nginx/nginx.conf` — add the upstream pool and server block
```nginx
http {
    upstream nautilus_backend {
        server stapp01:3001;
        server stapp02:3001;
        server stapp03:3001;
    }

    server {
        listen 80;
        listen [::]:80;

        server_name _;

        location / {
            proxy_pass http://nautilus_backend;
        }
    }
}
```

### 4.5 Validate before restarting
```bash
sudo -S nginx -t <<< 'Mischi3f'
```
```text
syntax is ok
test is successful
```

### 4.6 Restart and confirm the service
```bash
sudo -S systemctl restart nginx <<< 'Mischi3f'
sudo -S systemctl is-active nginx <<< 'Mischi3f'
```
```text
active
```

### 4.7 Verify from the actual client vantage point — the jump host
```bash
exit
curl http://stlb01:80
```
```text
Welcome to xFusionCorp Industries!
```

### 4.8 Confirm round-robin isn't visibly "broken" by repeated identical content
```bash
for i in {1..10}; do
  echo "Request $i:"
  curl -s http://stlb01:80
  echo
done
```
```text
Welcome to xFusionCorp Industries!   (x10 — expected, §2.8)
```

### 4.9 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Nginx installed, enabled, active on `stlb01` | ✅ |
| All three Apache backends unchanged, still on `:3001` | ✅ |
| `upstream nautilus_backend` defined with all three backends | ✅ |
| `server { listen 80; proxy_pass http://nautilus_backend; }` | ✅ |
| Config validated (`nginx -t`) before restart | ✅ |
| `curl http://stlb01:80` from the jump host succeeds | ✅ |

```text
Client (jump host)
    │ curl http://stlb01:80
    ▼
stlb01 : Nginx (listen 80)
    │ proxy_pass → nautilus_backend
    ▼
┌───────────┬───────────┬───────────┐
stapp01:3001 stapp02:3001 stapp03:3001
   Apache      Apache      Apache
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl http://stlb01:80` fails entirely | Nginx not running, or `listen 80` missing/wrong | `systemctl is-active nginx`; confirm the `server { listen 80; }` block exists |
| `nginx -t` fails after editing | Syntax error in the `upstream`/`server` block (missing semicolon, unmatched brace) | Fix the reported issue; never restart on a failed `-t` |
| `curl` from the jump host times out but `curl` from `stlb01` to a backend directly works | Nginx's proxy config itself is fine; something in the jump-host-to-`stlb01` network path is blocking port 80 | Isolate: test `stlb01` locally first, then the jump-host path, same layered approach as Day 13 |
| All ten repeated `curl`s return identical content | Expected — all backends serve the same static page | Not a bug; see §2.8 for how to actually observe distribution if needed |
| Changed Apache's port on one server "to match" something | Misread the constraint — Apache's port was never supposed to change | Revert; the backend port is a fact Nginx's `upstream` block adapts to, not something Apache should change |
| `proxy_pass` directive present but requests never reach any backend | Typo'd the upstream name, or `upstream` block itself missing/misnamed | Confirm the name in `proxy_pass http://<name>` exactly matches the `upstream <name> { }` block's name |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "distribute traffic across 3 Apache backends via Nginx" to
      "verified end-to-end from the jump host."
- [ ] Explain, in one sentence, why Apache's port didn't need to
      change, even though the client now connects on a completely
      different port (`80` vs `3001`).
- [ ] Explain the difference between an `upstream` block and a
      `proxy_pass` directive — what does each one actually do?
- [ ] Explain why ten identical `curl` responses don't prove the load
      balancer isn't working, and how you'd actually confirm
      round-robin distribution is occurring.
- [ ] Explain why `nginx -t` matters even more here than in a
      single-backend SSL setup (Day 15).
- [ ] Rebuild the same `upstream`/`server`/`proxy_pass` structure from
      memory for a different set of three backend addresses, validating
      with `nginx -t` before every restart attempt.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: confirm existing backend/dependency state → identify the
constraint that shapes the design (what must NOT change) → build the
new component's configuration in the tool's native context hierarchy →
VALIDATE before applying → apply → verify at each layer, from the
actual client vantage point last. End with a compressed arrow-chain
version.>

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
