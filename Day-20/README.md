# Day 20 — Configure Nginx + PHP-FPM Using a Unix Socket

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

Day 19 served static HTML directly from Nginx's document root. This
lab adds the layer Day 19 didn't need: a separate execution runtime
(PHP-FPM) that Nginx must hand requests off to rather than answer
itself — and the handoff happens over a Unix domain socket instead of
a plain TCP port, which is the lab's central new concept.

---

## 1. Scenario

Configure a PHP application on App Server 1 (`stapp01`, CentOS Stream
9) using Nginx and PHP-FPM, communicating over a specific Unix socket.

| Requirement | Target configuration |
|---|---|
| Web server | Nginx, port `8099` |
| Document root | `/var/www/html` |
| PHP runtime | PHP 8.3, via PHP-FPM |
| PHP-FPM socket | `/var/run/php-fpm/default.sock` |
| Entry point | `/var/www/html/index.php` (must not be modified) |

```text
Client ── HTTP :8099 ──▶ Nginx ── FastCGI over Unix socket ──▶ PHP-FPM ── executes index.php
                          │        /var/run/php-fpm/default.sock         │
                          ◀─────────────── response body ────────────────┘
```

Confirmed success:
```text
HTTP/1.1 200 OK
X-Powered-By: PHP/8.3.19

Welcome to xFusionCorp Industries!
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Why two services instead of one

```text
Nginx:    accepts HTTP connections, routes by URL, serves static files
PHP-FPM:  executes PHP source and returns generated output
```

Nginx has no built-in PHP interpreter — it's a web server/reverse
proxy, not a runtime. A `.php` file isn't something Nginx can "serve"
the way it serves static HTML (Day 19); it has to be handed to a
process that actually executes PHP and returns the *result*, not the
source. Splitting these responsibilities means a failure in one layer
(say, PHP-FPM crashing) doesn't necessarily take down the other — the
same separation-of-concerns reasoning as Day 16's Nginx LBR separating
the load balancer from the backend application servers.

### 2.2 TCP socket vs. Unix domain socket — same transport purpose, different addressing

```text
TCP socket:        127.0.0.1:9000        (IP + port)
Unix domain socket: /run/php-fpm/x.sock   (filesystem path)
```

Both move bytes between two local processes; a Unix socket simply uses
a filesystem pathname as its address instead of an IP:port pair. The
practical consequence that matters for this lab: because the address
is a *path*, standard Linux filesystem permissions (owner, group,
mode bits) directly control who can connect to it — no separate
firewall/port-binding layer needed, unlike iptables' port-based rules
in Day 13.

### 2.3 A missing parent directory blocks socket creation before PHP-FPM even starts

```text
ls -ld /var/run/php-fpm   →  No such file or directory
```

A process can't create a file (or socket) at a path whose parent
directory doesn't exist — this is pure filesystem mechanics, not a
PHP-FPM quirk. `/var/run` resolving to `/run` (a symlink) matters here
only because `/run` is a tmpfs/runtime filesystem that gets recreated
on every boot — so manually `mkdir -p`'ing it is a **lab-scoped fix**:
production would need to confirm something (a tmpfiles.d rule, a
systemd unit's `RuntimeDirectory=`) recreates that directory on every
boot, or PHP-FPM would fail identically after the next reboot.

### 2.4 Matching the socket *path*, not just the containing directory

```text
PHP-FPM pool: listen = /var/run/php-fpm/default.sock
Nginx upstream: server unix:/run/php-fpm/www.sock     ← different filename, same directory
```

Both paths resolve into the same directory, but a directory match
isn't an endpoint match — `default.sock` and `www.sock` are two
distinct files PHP-FPM could create. This is the Unix-socket version
of Day 11's ENI/attachment verification principle: *where something
lives* and *what the exact identifier is* are separate checks, and
only the second one determines whether two sides actually connect.

### 2.5 Worker identity vs. socket identity are two independent settings

```text
user/group            → what Linux identity the PHP-FPM WORKER PROCESSES run as
listen.owner/group    → what Linux identity owns the SOCKET FILE itself
```

These answer different questions — "who executes my PHP code" versus
"who's allowed to connect to the endpoint it listens on." Leaving
`user = apache` unchanged while setting `listen.owner = nginx` is
correct precisely because the socket must be writable by whichever
process initiates the connection (Nginx's worker, running as `nginx`),
regardless of which identity later executes the PHP script on the
other end.

### 2.6 Why `listen.acl_users` silently overrides `listen.owner`/`listen.group`

```text
listen.acl_users = apache,nginx   → ACL mode active, listen.owner/group IGNORED
;listen.acl_users = apache,nginx  → ACL mode disabled, listen.owner/group now take effect
```

PHP-FPM treats ACL-based access control and simple owner/group/mode
permissions as mutually exclusive mechanisms for the same socket — if
`acl_users` is set, it wins, and `listen.owner`/`listen.group` are
parsed but never applied. A config that *looks* correct
(`listen.owner = nginx` is right there in the file) can still fail
until you notice a different, earlier directive is quietly taking
priority — the same "read the whole file before concluding a single
directive change is sufficient" discipline as Day 13's iptables rule
*ordering*.

### 2.7 `0660` on a socket: read+write for owner/group, no execute needed

```text
srw-rw---- → owner: rw, group: rw, others: none
```

Execute permission is meaningless for a socket (there's no "running"
a communication endpoint); read+write is what lets a connecting
process actually exchange data over it. Restricting to owner+group
(not world) means only processes running as `nginx` — or root — can
open it, which is the access-control benefit of the socket approach
from §2.2 made concrete.

### 2.8 Validate configuration before starting/reloading — same discipline as every prior service lab

```bash
sudo nginx -t
```

Same principle as Day 19's `httpd -t`: catch a syntax error before
it affects a running service, not after. It only validates *syntax*,
though — a config can be syntactically perfect and still reference a
socket path that doesn't exist yet, which is why validation is one
step in a sequence (§2.9), not a standalone guarantee.

### 2.9 Diagnose by layer, not by guessing — the dependency chain determines where to look

```text
Client curl
   ↓ (connection refused?)        → Nginx not listening on 8099
Nginx server block
   ↓ (502 Bad Gateway?)           → upstream socket path wrong, or PHP-FPM not running
Nginx upstream → socket path
   ↓ (socket missing/wrong perms?) → check PHP-FPM pool's listen/listen.owner/mode
PHP-FPM pool
   ↓ (PHP source returned raw?)   → fastcgi_pass / location ~ \.php$ misconfigured
PHP-FPM worker executes script
```

Each failure symptom (connection refused, 502, raw PHP source, 404)
points at a *specific* link in this chain — changing several files at
once before identifying which link actually broke is how you end up
unable to tell which fix (if any) mattered. This mirrors the Cloud-AWS
series' habit of checking the decisive field rather than every
plausible one.

### 2.10 A 200 response with the expected body is the only end-to-end proof

```text
file exists at /var/www/html/index.php      → proves deployment only
nginx -t passes                              → proves config syntax only
systemctl is-active nginx/php-fpm            → proves both processes running only
socket exists with correct owner/mode        → proves the endpoint is reachable only
curl http://localhost:8099/index.php → 200   → proves the WHOLE chain actually works
```

Same "verify the decisive end-to-end behavior, not each layer in
isolation" principle as Day 19's `curl` test — here the chain is one
link longer (Nginx → socket → PHP-FPM → worker), so there are more
places a single-layer check could pass while the full request still
fails.

### 2.11 The compressed reasoning chain

```text
Requirement: PHP app on stapp01:8099 via Nginx + PHP-FPM over a specific Unix socket
   → install nginx, php-fpm (dnf module enable php:8.3 first)
   → mkdir -p the socket's parent directory if missing (§2.3)
   → set PHP-FPM listen = required socket path exactly (§2.4)
   → set listen.owner/listen.group/listen.mode for the socket; leave user/group (worker identity) alone (§2.5)
   → comment out listen.acl_users — it silently overrides owner/group otherwise (§2.6)
   → start php-fpm, verify the socket file's owner/mode on disk
   → point the Nginx upstream at the SAME exact socket path (§2.4)
   → new server{} block on the required port, reusing existing PHP location rules
   → nginx -t before starting (§2.8)
   → start nginx, then curl the actual URL — the only end-to-end proof (§2.10)
   → on failure, diagnose by which layer the symptom points to (§2.9), not by broad guessing
```

---

## 3. Concepts (reference)

### 3.1 Nginx vs. PHP-FPM responsibilities
Nginx handles HTTP connections, routing, and static content. PHP-FPM
(FastCGI Process Manager) manages PHP worker processes and executes
PHP scripts. Nginx forwards `.php` requests to PHP-FPM and returns
whatever PHP-FPM sends back — never the raw PHP source.

### 3.2 Unix domain socket vs. TCP socket
Both are local IPC mechanisms; a Unix socket is addressed by a
filesystem path rather than an IP:port pair, which lets standard file
permissions (owner/group/mode) gate access instead of a network-layer
firewall rule.

### 3.3 Socket ownership vs. worker process identity
`listen.owner`/`listen.group`/`listen.mode` control who can connect to
the socket file. `user`/`group` in the same pool config control what
Linux identity the PHP worker processes themselves run as. They are
independent settings answering different questions.

### 3.4 `listen.acl_users`/`listen.acl_groups` precedence
When set, PHP-FPM uses ACL-based access control for the socket and
ignores `listen.owner`/`listen.group` entirely — the two mechanisms
are mutually exclusive, not additive.

### 3.5 Nginx `upstream` blocks
An `upstream` block gives a backend (here, PHP-FPM's socket) a name
Nginx can reference via `fastcgi_pass` elsewhere in the configuration,
decoupling "where the backend actually lives" from "which server
block uses it."

### 3.6 `include` directives and configuration reuse
`include /etc/nginx/conf.d/*.conf;` and similar directives let a new
server block (`app.conf`) reuse existing, already-correct routing
logic (`default.d/php.conf`) rather than duplicating FastCGI
parameters in every file that needs PHP handling.

### 3.7 `/run` as a runtime filesystem
Contents under `/run` (and the `/var/run` symlink to it) can be wiped
and recreated on every boot. A manually-created directory there is a
point-in-time fix, not a persistent one, unless something (tmpfiles.d,
a systemd unit) is also configured to recreate it at boot.

---

## 4. Runbook

### 4.1 Install Nginx and PHP 8.3 / PHP-FPM
```bash
sudo dnf install -y nginx
nginx -v
sudo dnf module enable -y php:8.3
sudo dnf install -y php-fpm
php-fpm -v
```

### 4.2 Confirm initial service state
```bash
sudo systemctl status nginx --no-pager
sudo systemctl status php-fpm --no-pager
```
```text
Active: inactive (dead)
```
Expected — installed does not mean started.

### 4.3 Create the missing socket parent directory
```bash
ls -ld /var/run /var/run/php-fpm
```
```text
/var/run -> ../run
/var/run/php-fpm: No such file or directory
```
```bash
sudo mkdir -p /var/run/php-fpm
ls -ld /var/run/php-fpm
```
```text
drwxr-xr-x 2 root root ... /var/run/php-fpm
```

### 4.4 Configure the PHP-FPM pool's socket
```bash
sudo vi /etc/php-fpm.d/www.conf
```
Changed:
```ini
listen = /var/run/php-fpm/default.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
;listen.acl_users = apache,nginx
```
`user = apache` / `group = apache` (worker identity) left unchanged.

Verify:
```bash
sudo grep -E '^(listen =|listen.owner|listen.group|listen.mode|listen.acl_users)' /etc/php-fpm.d/www.conf
```
```text
listen = /var/run/php-fpm/default.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

### 4.5 Start PHP-FPM and verify the socket
```bash
sudo systemctl start php-fpm
sudo systemctl status php-fpm --no-pager
sudo ls -l /var/run/php-fpm/default.sock
```
```text
Active: active (running)
Status: "Ready to handle connections"
srw-rw---- 1 nginx nginx ... /var/run/php-fpm/default.sock
```

### 4.6 Point the Nginx upstream at the same socket
```bash
sudo vi /etc/nginx/conf.d/php-fpm.conf
```
```nginx
upstream php-fpm {
    server unix:/var/run/php-fpm/default.sock;
}
```

### 4.7 Create the application server block on port 8099
```bash
sudo vi /etc/nginx/conf.d/app.conf
```
```nginx
server {
    listen 8099;
    server_name _;
    root /var/www/html;
    index index.php index.html index.htm;

    include /etc/nginx/default.d/php.conf;
}
```
Reuses the existing FastCGI routing rules already defined in
`/etc/nginx/default.d/php.conf` rather than duplicating them.

### 4.8 Validate, start, and verify Nginx
```bash
sudo nginx -t
```
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
```bash
sudo systemctl enable --now nginx
sudo systemctl is-active nginx php-fpm
```
```text
active
active
```

### 4.9 Test end to end
```bash
curl -i http://localhost:8099/index.php
```
```text
HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/html; charset=UTF-8
X-Powered-By: PHP/8.3.19

Welcome to xFusionCorp Industries!
```

**Remote verification status:** the local (`localhost`) test above is
confirmed. A follow-up check from the jump host —
`curl http://stapp01:8099/index.php`, expecting the same
"Welcome to xFusionCorp Industries!" body — was identified as a useful
additional step but its output was **not captured in this lab run**.
Treat it as outstanding until actually executed and observed; a local
loopback success does not by itself confirm hostname resolution or
reachability from another host.

### 4.10 Concise end-to-end command sequence
For re-running from scratch, in order:
```bash
sudo dnf install -y nginx
sudo dnf module enable -y php:8.3
sudo dnf install -y php-fpm
sudo mkdir -p /var/run/php-fpm
sudo vi /etc/php-fpm.d/www.conf   # listen=/var/run/php-fpm/default.sock, listen.owner/group=nginx, listen.mode=0660, comment out listen.acl_users
sudo systemctl start php-fpm
sudo vi /etc/nginx/conf.d/php-fpm.conf   # upstream php-fpm { server unix:/var/run/php-fpm/default.sock; }
sudo vi /etc/nginx/conf.d/app.conf       # server { listen 8099; root /var/www/html; include .../php.conf; }
sudo nginx -t
sudo systemctl enable --now nginx
curl -i http://localhost:8099/index.php
curl http://stapp01:8099/index.php   # from the jump host — run this to close out remote verification
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Nginx + PHP-FPM 8.3 installed | ✅ |
| PHP-FPM listening on `/var/run/php-fpm/default.sock` | ✅ |
| Socket owned by `nginx:nginx`, mode `0660`, ACL override disabled | ✅ |
| Nginx upstream points at the same socket path | ✅ |
| Nginx server block listening on `8099`, root `/var/www/html` | ✅ |
| `nginx -t` passes | ✅ |
| Both services active | ✅ |
| Local `curl` → `200 OK` with expected body | ✅ |
| Remote (jump-host) `curl` → confirmed | ⚠️ not yet executed — see §4.9 |
| `index.php` / `info.php` left unmodified | ✅ |

```text
stapp01
 ├── nginx (active), listen 8099 → root /var/www/html
 │     └── conf.d/app.conf, conf.d/php-fpm.conf, default.d/php.conf
 └── php-fpm (active), pool www
       └── listen /var/run/php-fpm/default.sock (nginx:nginx, 0660)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (7) Connection refused` | Nginx not running, or nothing listening on `8099` | `systemctl status nginx`; `ss -ltnp \| grep ':8099'` |
| `502 Bad Gateway` | Upstream socket path wrong, or PHP-FPM not running | Compare the upstream's socket path against PHP-FPM's `listen` value; check `systemctl status php-fpm` |
| `nginx -t` fails | Syntax error, missing semicolon/brace, bad `include` path | Read the filename+line number in the error; fix that section specifically |
| `404 Not Found` | Document root or `index.php` path mismatch | `ls -la /var/www/html/`; confirm `root` directive matches |
| Raw PHP source returned to the client | `.php` requests not being routed to PHP-FPM at all | Inspect the `location ~ \.php$` block and `fastcgi_pass` — treat as a security issue, not just a bug |
| Socket exists but connections still fail | `listen.acl_users` still active, overriding owner/group | Comment out `listen.acl_users`, restart `php-fpm` |
| Works now, breaks after reboot | `/run/php-fpm` (tmpfs) not recreated at boot | Add a tmpfiles.d rule or confirm the package/unit recreates it |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "PHP app on stapp01:8099" to "verified via `curl`."
- [ ] Explain, in one sentence, why Nginx can't execute a `.php` file
      itself the way it serves a static HTML file.
- [ ] Explain why `listen.owner`/`listen.group` and `user`/`group` in
      the same PHP-FPM pool file control two different things.
- [ ] Explain why `listen.acl_users = apache,nginx` being set would
      make `listen.owner = nginx` silently have no effect.
- [ ] Given a `502 Bad Gateway`, name the two specific files you'd
      compare against each other first, and why.
- [ ] Run the remote jump-host `curl` test from §4.9 that wasn't
      executed in this lab run, and record its actual output.
- [ ] From memory, add a second PHP application on a different port
      (e.g. `8098`) using a *different* Unix socket name, without
      copying any command from this file, then verify both apps
      independently.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
