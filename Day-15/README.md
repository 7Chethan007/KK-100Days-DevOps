# Day 15 — Setup SSL for Nginx

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive the config
yourself*.

---

## 1. Scenario

Make Nginx on `stapp02` serve HTTPS, using a supplied certificate and
key, reachable and verified from the jump host.

```text
Jump Host
   │ curl https://stapp02/
   ▼
stapp02
   │
   ▼
Nginx
   ├── Port 443, TLS
   │     /etc/nginx/ssl/nautilus.crt
   │     /etc/nginx/ssl/nautilus.key
   └── /usr/share/nginx/html/index.html → "Welcome!"
```

Four independent pieces: install Nginx, place the cert/key correctly,
configure an HTTPS server block, and verify the whole path from a
different machine (the jump host, not `stapp02` itself).

---

## 2. Reasoning model — how to *derive* the config, not memorize it

### 2.1 HTTP vs. HTTPS — one extra layer, one extra port

```text
HTTP   → port 80,  plaintext
HTTPS  → port 443, TLS-wrapped HTTP
```

HTTPS doesn't replace HTTP — it wraps the same request/response model
inside an encrypted TLS channel. The server needs a certificate and a
matching private key specifically to establish that TLS channel before
any HTTP traffic flows through it — which is exactly why the lab
supplies `nautilus.crt`/`nautilus.key` as prerequisites rather than
something Nginx can generate on its own.

### 2.2 Certificate vs. private key — public identity vs. protected secret

```text
nautilus.crt  → the server's PUBLIC identity (subject, issuer, validity,
                 public key) — safe for clients to receive
nautilus.key  → the SECRET that proves the server actually owns that
                 certificate — must never leave the server, must be
                 tightly permissioned
```

```bash
chmod 600 /etc/nginx/ssl/nautilus.key   # owner rw, nobody else — a SECRET
chmod 644 /etc/nginx/ssl/nautilus.crt   # world-readable — fine, it's PUBLIC
```

Same "credential file gets the tightest permission that still lets the
owner use it" principle as `.pem` files (Cloud-AWS Day 6) and `.env`
files (MLOps Day 4) — the certificate and the key are not symmetric in
sensitivity, and their permissions should reflect that asymmetry
exactly.

### 2.3 Inspect what's actually provided before moving anything

```bash
sudo ls -l /tmp/nautilus.crt /tmp/nautilus.key
```

Confirms the lab's prerequisite material genuinely exists before
building any configuration around it — same "inspect before act"
discipline as every lab in this series.

### 2.4 `/tmp` is the wrong permanent home — move to a stable, conventional path

```bash
sudo mkdir -p /etc/nginx/ssl
sudo mv /tmp/nautilus.crt /tmp/nautilus.key /etc/nginx/ssl/
```

`/tmp` can be cleared on reboot on some systems, and more importantly
isn't where configuration should reference long-lived material from —
Nginx's `ssl_certificate`/`ssl_certificate_key` directives should point
at a stable path that won't disappear or change out from under the
running config.

### 2.5 `systemctl enable` vs. `start` — boot-time vs. right-now (recap)

```text
enable  → start automatically on future boots
start   → start right now, this session
```

Same `enable`/`active` distinction as Day 9's MariaDB lab and Day 5's
SELinux lab — `enable`d alone doesn't mean it's currently running, and
`start`ed alone doesn't survive a reboot. A production service
generally wants both.

### 2.6 A `server {}` block is a conditional rule set, not just "the config"

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name stapp02;
    root /usr/share/nginx/html;
    ssl_certificate /etc/nginx/ssl/nautilus.crt;
    ssl_certificate_key /etc/nginx/ssl/nautilus.key;
    location / { try_files $uri $uri/ =404; }
}
```

Read it as: *"for requests matching this listener/hostname, apply these
rules."* One Nginx process can host several `server {}` blocks
simultaneously (different ports, different `server_name`s) — this is
exactly the mechanism behind name-based virtual hosting, where one
machine serves multiple distinct sites.

### 2.7 `listen 443 ssl` and dual-stack `[::]:443`

```text
listen 443 ssl;        → IPv4, HTTPS on port 443
listen [::]:443 ssl;    → IPv6, same port, same TLS requirement
```

The `ssl` parameter is what tells Nginx to perform a TLS handshake on
this listener before treating the connection as HTTP — omitting it
would make port 443 a plain HTTP listener, which no TLS-speaking client
could use correctly. Listening on both address families is what lets
the server accept HTTPS connections regardless of whether a client
reaches it over IPv4 or IPv6.

### 2.8 `root` and `location /` — where content lives, and how a path maps to a file

```text
root /usr/share/nginx/html;   → the filesystem directory requests resolve against
location / { try_files $uri $uri/ =404; }  → for any path starting with "/",
                                              try the exact file, then a
                                              directory index, else 404
```

`GET /` resolves to `root` + the configured `index` file
(`index.html`) — this is why creating
`/usr/share/nginx/html/index.html` with the word `Welcome!` is what
makes `curl https://stapp02/` actually return that text, rather than a
404 or an empty response.

### 2.9 A separate `conf.d/*.conf` file instead of editing the main config directly

```nginx
# in /etc/nginx/nginx.conf:
include /etc/nginx/conf.d/*.conf;
```

Nginx's main config explicitly includes every `.conf` file in
`conf.d/` — creating `nautilus-ssl.conf` there, rather than editing the
large main file in place, is the standard modular-configuration
pattern: each logical piece of config lives in its own file, is easy to
add/remove independently, and doesn't risk corrupting a much larger
file you didn't need to touch.

### 2.10 Validate before restarting — `nginx -t` as the mandatory gate

```bash
sudo nginx -t
```
```text
syntax is ok
test is successful
```

Never restart a running web server on unvalidated config changes —
`nginx -t` checks syntax and basic configuration sanity *without*
affecting the currently-running process, giving you a chance to catch a
typo before it takes down a working server. Skipping this step and
going straight to `restart` risks turning "editing a config file" into
"an actual outage."

```text
Edit config → nginx -t → valid? → restart/reload → check service → test
```

### 2.11 Why verification must happen from the jump host, not from `stapp02` itself

```text
curl https://localhost/  (run ON stapp02)   → only proves stapp02 → Nginx
curl https://stapp02/     (run FROM jump host) → proves jump-host → network → stapp02 → Nginx
```

Same "local test vs. remote test prove different things" lesson as Day
13's iptables lab — testing only from the server you just configured
can't catch a networking/firewall issue sitting between it and the
actual client that will use it. The lab's real requirement is reachable
*from the jump host*, so that's the vantage point that actually matters
for verification.

### 2.12 Why `-k` is required, and what it does and doesn't disable

```bash
curl -k https://stapp02/
```

```text
-k / --insecure  → don't reject the connection just because the
                    certificate isn't in curl's trusted CA store
```

The certificate here is **self-signed** — no public Certificate
Authority vouches for it, so curl's default trust verification would
reject the connection entirely without `-k`. Critically: `-k` does
**not** disable encryption — the TLS handshake, the encrypted channel,
all of it still happens exactly as with a CA-signed certificate; `-k`
only disables curl's *trust verification* of who signed the
certificate. Self-signed certs are expected and fine for a lab; a real
production deployment would use a CA-issued certificate so clients
don't need `-k` at all.

### 2.13 `curl -I` (headers only) before `curl` (full content) — layered verification

```bash
curl -Ik https://stapp02/     # HEAD request — headers only, fast, confirms reachability + status
curl -k https://stapp02/       # full request — confirms the ACTUAL content
```

```text
curl -Ik  →  HTTP/1.1 200 OK, Server: nginx/1.20.1    (reachable, TLS works, server identified)
curl -k   →  Welcome!                                  (the actual expected content)
```

Checking headers first is a fast way to confirm "the whole chain up
through HTTP works" before pulling the full response — useful when
debugging, since a `200` with headers but wrong/missing body would
point you toward the document-root/`index.html` layer specifically,
rather than TLS or networking.

### 2.14 The compressed reasoning chain

```text
Requirement (HTTPS on stapp02, verified from jump host, content "Welcome!")
   → Confirm /tmp/nautilus.crt and .key exist
   → dnf install -y nginx; systemctl enable nginx
   → mkdir /etc/nginx/ssl; mv cert+key there (NOT left in /tmp)
   → chmod 600 the key, 644 the cert (asymmetric sensitivity, §2.2)
   → echo "Welcome!" > /usr/share/nginx/html/index.html
   → Write /etc/nginx/conf.d/nautilus-ssl.conf:
        listen 443 ssl + [::]:443 ssl, server_name, root, ssl_certificate(_key), location /
   → nginx -t                                  → validate BEFORE restarting
   → systemctl restart nginx; systemctl status  → active (running)
   → FROM THE JUMP HOST (not stapp02 itself):
        curl -Ik https://stapp02/                → 200 OK (reachable + TLS works)
        curl -k https://stapp02/                  → "Welcome!" (correct content, full chain proven)
```

---

## 3. Concepts (reference)

### 3.1 TLS/SSL certificate and private key
A certificate asserts a public identity (who this server claims to be);
a private key is the secret that cryptographically proves the server
actually controls that identity. Both are required for TLS; only the
key needs to be kept secret.

### 3.2 `systemctl enable` vs. `start` (recap)
`enable` configures boot-time behavior; `start` affects the current
running state. Production services generally want both `enabled` and
`active` simultaneously — see Days 5 and 9 for the same distinction in
other contexts.

### 3.3 Nginx `server {}` blocks and `server_name`
A `server {}` block defines a conditional rule set applied based on
listener (IP/port) and `server_name` (requested hostname). Multiple
blocks let one Nginx process serve multiple distinct sites —
name-based virtual hosting.

### 3.4 `root` and `location`
`root` sets the filesystem base directory for serving content;
`location` blocks match URL path patterns and define how to handle
matching requests (e.g. `try_files` to serve an existing file, fall
back to a directory index, or return `404`).

### 3.5 Modular configuration via `conf.d/`
Keeping logically separate configuration in individual files under
`conf.d/`, included by the main config, is the standard way to avoid
editing a large shared file directly for every change — easier to add,
remove, or review one piece in isolation.

### 3.6 `nginx -t` as a pre-flight check
Validates configuration syntax/sanity without affecting the running
process — the mandatory gate before any `restart`/`reload`, the same
"verify before you commit to the change" discipline as validating any
config file in this series (SELinux's config, Makefiles, pre-commit
configs).

### 3.7 `-k`/`--insecure` in curl
Disables certificate *trust* verification only — the TLS handshake and
encryption still occur normally. Appropriate for testing against
self-signed certificates in a lab/internal context; not a substitute
for a properly CA-signed certificate in production, where clients
should never need `-k`.

### 3.8 Local test vs. remote test (recap from Day 13)
Testing from the configured server itself only proves the
server-to-application path; testing from the actual client vantage
point (the jump host here) proves the full network path end to end —
always verify from where the real requirement is actually measured.

---

## 4. Runbook

### 4.1 Connect and confirm host
```bash
sshpass -p 'Am3ric@' ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null steve@stapp02
hostname
```
```text
stapp02
```

### 4.2 Verify the supplied certificate material
```bash
sudo ls -l /tmp/nautilus.crt /tmp/nautilus.key
```

### 4.3 Install and enable Nginx
```bash
sudo dnf install -y nginx
nginx -v
```
```text
nginx version: nginx/1.20.1
```
```bash
sudo systemctl enable nginx
```

### 4.4 Move certificates to a stable location and set permissions
```bash
sudo mkdir -p /etc/nginx/ssl
sudo mv /tmp/nautilus.crt /tmp/nautilus.key /etc/nginx/ssl/
sudo chmod 600 /etc/nginx/ssl/nautilus.key
sudo chmod 644 /etc/nginx/ssl/nautilus.crt
```

### 4.5 Create the webpage
```bash
echo 'Welcome!' | sudo tee /usr/share/nginx/html/index.html
```

### 4.6 Create the HTTPS server block
```bash
sudo tee /etc/nginx/conf.d/nautilus-ssl.conf > /dev/null <<'EOF'
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name stapp02;

    root /usr/share/nginx/html;
    index index.html;

    ssl_certificate /etc/nginx/ssl/nautilus.crt;
    ssl_certificate_key /etc/nginx/ssl/nautilus.key;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

### 4.7 Validate before restarting
```bash
sudo nginx -t
```
```text
syntax is ok
test is successful
```

### 4.8 Restart and confirm the service
```bash
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
```
```text
Active: active (running)
```

### 4.9 Verify from the jump host — headers first, then content
```bash
curl -Ik https://stapp02/
```
```text
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/html
```
```bash
curl -k https://stapp02/
```
```text
Welcome!
```

### 4.10 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Nginx installed, enabled, active | ✅ |
| Certificate/key moved to `/etc/nginx/ssl/`, permissions `644`/`600` | ✅ |
| HTTPS server block listening on `443` (IPv4 + IPv6) | ✅ |
| `index.html` serving `Welcome!` | ✅ |
| Config validated (`nginx -t`) before restart | ✅ |
| Verified from the jump host: `200 OK` + correct content | ✅ |

```text
Jump Host
    │ curl -k https://stapp02/
    ▼
TCP :443 ── TLS (nautilus.crt/.key) ── Nginx
                                           │
                                     root: /usr/share/nginx/html
                                           │
                                           ▼
                                      index.html → "Welcome!"
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `nginx: command not found` | Nginx not installed | `dnf install -y nginx` |
| `curl` connection refused on `:443` | Nginx not listening on 443 (wrong `listen` directive, or service not running) | `ss -lntp \| grep :443`; confirm `listen 443 ssl` present and `systemctl is-active nginx` |
| `nginx -t` fails | Syntax error in the new config file | Fix reported line/issue before restarting; never restart on a failed `-t` |
| `curl` fails with a certificate trust error | Self-signed certificate, no `-k` used | Add `-k`/`--insecure` for testing against self-signed certs |
| `curl -k` succeeds but body is empty/404 | `root`/`index` misconfigured, or `index.html` missing/empty | Confirm `index.html` exists at the configured `root` path with the expected content |
| Works when tested on `stapp02` itself, fails from the jump host | Only tested locally; a network/firewall layer between jump host and `stapp02` may be blocking it | Always verify from the actual client vantage point, not just the server (§2.11) |
| Private key readable by other users | Permissions left at a default, non-restrictive mode | `chmod 600` on the key specifically — it's a secret, unlike the certificate |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.14) out loud
      from "make Nginx serve HTTPS with this cert" to "verified 200 +
      Welcome! from the jump host."
- [ ] Explain, in one sentence, the difference in sensitivity between a
      certificate and a private key, and why their permissions differ.
- [ ] Explain why `nginx -t` must run before `systemctl restart`, not
      after.
- [ ] Explain what `-k` in curl actually disables, and what it does
      NOT disable.
- [ ] Explain why testing from `stapp02` itself wasn't sufficient
      verification for this task.
- [ ] Explain the purpose of `/etc/nginx/conf.d/*.conf` and why it's
      preferable to editing `nginx.conf` directly.
- [ ] Rebuild the same HTTPS server block from memory on a different
      hostname/port, validating with `nginx -t` before every restart
      attempt.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: inspect prerequisites → install/prepare → place
config/secrets in their stable, correctly-permissioned locations →
write the service configuration → VALIDATE before applying → apply →
verify locally AND from the real client vantage point, layered from
cheap checks to full content checks. End with a compressed arrow-chain
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
