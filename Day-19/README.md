# Day 19 — Install and Configure Apache for Two Static Websites

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This lab reuses the service-management triad from Day 17 (PostgreSQL)
and Day 18 (MariaDB) — install → configure → start/verify — but applies
it to a web server instead of a database, and adds a new wrinkle: the
content to serve doesn't originate on the target server, so file
transfer between hosts becomes part of the runbook.

---

## 1. Scenario

Install Apache (`httpd`) on App Server 1 (`stapp01`), configure it to
listen on port `8086`, and serve two independent static websites from
backups staged on the jump host.

| Website | Required URL                  | Document path                   |
| ------- | ------------------------------ | -------------------------------- |
| News    | `http://localhost:8086/news/`  | `/var/www/html/news/index.html` |
| Apps    | `http://localhost:8086/apps/`  | `/var/www/html/apps/index.html` |

```text
jump_host                              stapp01
/home/thor/news  ──(scp)──┐
/home/thor/apps  ──(scp)──┤
                           ▼
                     /tmp/{news,apps}
                           │ (sudo cp -r .../.)
                           ▼
              /var/www/html/{news,apps}/index.html
                           │
                           ▼
                  Apache (httpd), Listen 8086
                           │
         curl http://localhost:8086/news/  → News HTML
         curl http://localhost:8086/apps/  → Apps HTML
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Serving a file over HTTP needs exactly three coordinates

```text
HTTP request → http://localhost:8086/news/
                         │        │    │
                         │        │    └── URL path
                         │        └─────── listening port
                         └──────────────── host
```

Apache resolves a request by combining its **document root** (a base
directory on disk) with the **URL path**: `/var/www/html` + `/news/` →
`/var/www/html/news/index.html`. The **port** is a separate, unrelated
coordinate — it only decides where Apache accepts the connection in the
first place. Changing one doesn't move or affect the other; that's why
this lab has two independent fixes to make (move the listener to
`8086`, place files under two new subdirectories), not one.

Because the site is static HTML, Apache can serve it directly — no
application runtime or database layer sits between the file on disk and
the HTTP response, unlike the PostgreSQL/MariaDB labs (Days 17-18) where
the *service itself* held the data.

### 2.2 Inspect before you edit — same discipline as every prior config lab

```text
Day 13 (iptables): list existing rules before inserting a new one
Day 16 (Nginx LBR): read upstream{}/server{} before adding backends
Day 19 (Apache):    grep Listen/DocumentRoot before touching either
```

```bash
sudo grep -nE '^[[:space:]]*Listen|^[[:space:]]*DocumentRoot' /etc/httpd/conf/httpd.conf
```

Reading the current state first avoids two failure modes: overwriting
an unrelated existing setting, and — specific to `Listen` — ending up
with *two* `Listen` directives if you append instead of replace, which
would leave Apache listening on both the old and new port rather than
just the required one.

### 2.3 Why replace the `Listen` directive with `sed`, not append a second one

```bash
sudo sed -i 's/^Listen 80$/Listen 8086/' /etc/httpd/conf/httpd.conf
```

`s/^Listen 80$/Listen 8086/` anchors both ends of the line (`^...$`) so
it matches the exact existing directive and nothing else, then
substitutes it in place (`-i`). The task says Apache must listen on
`8086` — not "listen on `8086` in addition to `80`" — so a targeted
replace is the correct operation, the same reasoning as Day 13's
iptables rule *ordering*: the file should end up describing the one
intended state, not an accumulation of every state it ever passed
through.

### 2.4 The content doesn't live where the web server runs — staging is required

```text
jump_host:/home/thor/{news,apps}   →   stapp01:/tmp/{news,apps}   →   stapp01:/var/www/html/{news,apps}
         (scp over SSH)                      (sudo cp -r)
```

Two different problems, two different tools:

- **Getting the bytes onto the right machine** is a network-transfer
  problem → `scp` (recursive, over SSH).
- **Getting the bytes into Apache's protected document root** is a
  *permissions* problem → the lab user (`tony`) can plausibly write to
  `/tmp`, but `/var/www/html` is owned by root/managed by the package,
  so placing files there needs `sudo`.

Staging in `/tmp` first and deploying with a separate `sudo cp` keeps
those two concerns separate — if the transfer step fails, you find out
before ever touching the privileged destination.

### 2.5 `cp -r source/. dest/` vs. `cp -r source dest/` — the trailing `/.` is load-bearing

```text
cp -r /tmp/news/.  /var/www/html/news/   →  /var/www/html/news/index.html   ✅ correct
cp -r /tmp/news    /var/www/html/news/   →  /var/www/html/news/news/index.html  ✘ wrong
```

`cp -r` copies a directory *as an entry* by default — copying `news`
into `news/` creates a nested `news/news/`. Appending `/.` tells `cp`
to copy the *contents* of the source directory instead of the directory
itself, which is what lands `index.html` directly where the required
document path expects it. This is a pure filesystem mechanic, not an
Apache behavior — get the destination path wrong here and Apache will
return a 404 for a perfectly running server, which is exactly the
"wrong page / 404" failure class called out in §6.

### 2.6 A typo'd directory name fails silently until you request the URL

While creating the destination directories, an `apps` → `appsv` typo
produced a valid, successfully-created directory that nothing in
`mkdir`'s output flags as wrong — `mkdir -p` has no way to know
`appsv` wasn't intended. The mismatch only surfaces downstream, as a
404 on `/apps/`, because Apache was never told to look in `appsv` in
the first place. The fix is the same category of lesson as Day 11
(ENI) and Day 12 (EBS) — *verify by requesting the actual decisive
endpoint*, not by trusting that a command which "succeeded" did what
you intended.

### 2.7 Validate configuration before starting/reloading the service

```bash
sudo httpd -t
```

`httpd -t` parses the configuration without starting anything.
Checking syntax before `systemctl start` catches a malformed `Listen`
line or stray edit immediately, rather than discovering it later as a
service that silently refuses to start — the same "validate before
you act" instinct as `terraform plan`/`aws ec2 describe-*` in the
Cloud-AWS series before any mutating call.

The `AH00558` "could not reliably determine the server's fully
qualified domain name" message is a warning about a missing global
`ServerName`, unrelated to serving two local sites on a custom port —
it does not block anything this lab requires.

### 2.8 `start` establishes current state; it says nothing about boot persistence

```text
systemctl start httpd      → running now
systemctl enable httpd     → will start automatically on next boot
systemctl is-active httpd  → query: is it running right now?
```

Same `installed ≠ active ≠ enabled` triad as Day 9 (MariaDB), Day 11
(Tomcat), and Day 17/18 (Postgres/MariaDB) — `dnf install` only gets
you "installed." This lab's requirement is satisfied by "running now,"
so `is-active` is the correct check; `enable` wasn't asked for and
wasn't run.

### 2.9 A missing tool (`ss`) is an environment fact, not an Apache failure

```text
sudo ss -lntp | grep ':8086'  →  sudo: ss: command not found
```

`command not found` means the *utility* isn't installed on this
minimal lab image — it says nothing about Apache's state one way or
the other. Rather than installing an extra package just to satisfy a
convenience check, the lab's actual success condition — a correct HTTP
response — can be verified more directly with a tool that already
works: `curl`.

### 2.10 `curl` verifies the whole chain, not just one layer

```text
file exists on disk      → confirms deployment only
httpd -t passes          → confirms config syntax only
systemctl is-active      → confirms the process is running only
curl http://localhost:8086/news/  → confirms port + routing + document root + file, end to end
```

Each earlier check validates one layer in isolation; none of them
alone proves a client can actually get the News/Apps content back.
`curl` is the one check that exercises the full request path the same
way the lab's grader will, which is why it's the final verification
step rather than an optional nice-to-have.

### 2.11 The compressed reasoning chain

```text
Requirement: serve two static sites on stapp01:8086
   → inspect current Listen/DocumentRoot (§2.2)
   → sed -i replace (not append) Listen 80 → Listen 8086 (§2.3)
   → mkdir -p the two document subdirectories, verify the exact names (§2.6)
   → scp content from jump_host → /tmp staging on stapp01 (§2.4)
   → sudo cp -r /tmp/<site>/. /var/www/html/<site>/  (trailing /. matters, §2.5)
   → httpd -t to validate syntax before starting (§2.7)
   → systemctl start httpd; is-active to confirm running (§2.8)
   → curl each required URL — the end-to-end check (§2.10)
```

---

## 3. Concepts (reference)

### 3.1 Listening port vs. document root
Two independent Apache settings: the port is a network-layer decision
(where connections are accepted); the document root is a filesystem-
layer decision (where content is read from). Changing one has zero
effect on the other.

### 3.2 URL path → filesystem path mapping
Apache appends the requested URL path onto the document root to find
a file: `DocumentRoot + path = /var/www/html + /news/ =
/var/www/html/news/`, which is why directory *names* under the
document root must exactly match the URLs the task requires.

### 3.3 Staging vs. deployment
Transferring files onto a machine (`scp`, a network operation) and
placing them into a service's protected directory (`cp` with `sudo`, a
privileged local operation) are different operations with different
failure modes — separating them makes it possible to diagnose which
one failed.

### 3.4 `cp -r dir/. dest/` vs `cp -r dir dest/`
The trailing `/.` copies a directory's *contents* into the destination;
omitting it copies the directory itself, nesting one extra level. Pure
`cp` semantics, independent of Apache.

### 3.5 Configuration validation before service control
`httpd -t` (or the equivalent `nginx -t`, `postgresql-setup --initdb`
dry-runs, etc.) parses configuration without affecting the running
service — a cheap, safe gate to run before every `start`/`reload`.

### 3.6 `systemctl start` vs. `enable` vs. `is-active`
`start`/`stop`/`restart` change current running state; `enable`/
`disable` change boot-time behavior; `is-active`/`status` only read
state without changing anything — the same "installed ≠ active ≠
enabled" triad as every earlier systemd-managed service lab.

### 3.7 End-to-end verification with `curl`
`curl <url>` is a real HTTP client call — it is the only check in this
runbook that proves the full request path (port → routing → document
root → file → response body) works, as opposed to checking any single
layer of that chain in isolation.

---

## 4. Runbook

### 4.1 Inspect the source backups on the jump host
```bash
ls -ld /home/thor/news /home/thor/apps
find /home/thor/news /home/thor/apps -maxdepth 2 -type f
```
Confirms both backup directories exist and each contains an
`index.html` — Apache needs actual website content, not just empty
destination directories to copy into.

### 4.2 Connect to App Server 1 from the jump host
```bash
sshpass -p 'Ir0nM@n' ssh -o StrictHostKeyChecking=no tony@stapp01
```
`sshpass -p` supplies the lab password non-interactively;
`StrictHostKeyChecking=no` auto-accepts the unknown host key for this
disposable lab environment (not a production-appropriate setting —
production should verify host keys or use SSH key auth instead).

### 4.3 Install Apache
```bash
sudo dnf install -y httpd
```

### 4.4 Inspect the existing configuration
```bash
sudo grep -nE '^[[:space:]]*Listen|^[[:space:]]*DocumentRoot|^[[:space:]]*Include' /etc/httpd/conf/httpd.conf
```
```text
Listen 80
DocumentRoot "/var/www/html"
```

### 4.5 Change the listening port to 8086
```bash
sudo sed -i 's/^Listen 80$/Listen 8086/' /etc/httpd/conf/httpd.conf
sudo grep -nE '^[[:space:]]*Listen' /etc/httpd/conf/httpd.conf
```
```text
47:Listen 8086
```

### 4.6 Create the website directories
```bash
sudo mkdir -p /var/www/html/news /var/www/html/apps
```
```text
/var/www/html/
├── news/
│   └── index.html
└── apps/
    └── index.html
```

### 4.7 Transfer the backups from the jump host
```bash
sshpass -p 'Ir0nM@n' scp -o StrictHostKeyChecking=no -r /home/thor/news tony@stapp01:/tmp/
sshpass -p 'Ir0nM@n' scp -o StrictHostKeyChecking=no -r /home/thor/apps tony@stapp01:/tmp/
```

### 4.8 Deploy the website files
```bash
sudo cp -r /tmp/news/. /var/www/html/news/
sudo cp -r /tmp/apps/. /var/www/html/apps/
sudo find /var/www/html/news /var/www/html/apps -maxdepth 1 -type f -name 'index.html'
```
```text
/var/www/html/news/index.html
/var/www/html/apps/index.html
```

### 4.9 Validate Apache configuration
```bash
sudo httpd -t
```
```text
AH00558: httpd: Could not reliably determine the server's fully qualified domain name...
Syntax OK
```

### 4.10 Start Apache and check service state
```bash
sudo systemctl start httpd
sudo systemctl is-active httpd
```
```text
active
```

### 4.11 Verify both websites end to end
```bash
curl http://localhost:8086/news/
```
```html
<h1>KodeKloud</h1>
<p>This is a sample page for our news website</p>
```
```bash
curl http://localhost:8086/apps/
```
```html
<h1>KodeKloud</h1>
<p>This is a sample page for our apps website</p>
```

### 4.12 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| `httpd` installed on `stapp01` | ✅ |
| Apache listening on port `8086` (not `80`) | ✅ |
| `/var/www/html/news/index.html` deployed | ✅ |
| `/var/www/html/apps/index.html` deployed | ✅ |
| `httpd -t` reports `Syntax OK` | ✅ |
| `systemctl is-active httpd` → `active` | ✅ |
| `curl http://localhost:8086/news/` returns News content | ✅ |
| `curl http://localhost:8086/apps/` returns Apps content | ✅ |

```text
stapp01
 └── httpd (active), Listen 8086
      ├── /var/www/html/news/index.html  ──→  /news/
      └── /var/www/html/apps/index.html  ──→  /apps/
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Apache won't start | Configuration error or port conflict | `sudo systemctl status httpd`, check journal output |
| `httpd -t` reports a syntax error | Bad edit to `httpd.conf` | Re-check the exact line with `grep`, fix, re-run `httpd -t` before starting |
| Connection refused on `8086` | Service not active, or `Listen` edit didn't take | `sudo systemctl is-active httpd`; re-check `grep Listen httpd.conf` |
| HTTP 404 on `/news/` or `/apps/` | URL path doesn't match the actual directory name (e.g. `appsv` typo) | `sudo find /var/www/html -maxdepth 1 -type d` to see exact directory names |
| Extra nested directory (`news/news/index.html`) | Used `cp -r src dest/` instead of `cp -r src/. dest/` | Re-copy with the trailing `/.`, remove the stray nested directory |
| Files missing after `scp` | Transfer didn't complete, or wrong source path | `ls -l /tmp/news /tmp/apps` on `stapp01` before attempting deployment |
| `sudo: ss: command not found` | Utility not installed on this minimal image | Not an Apache problem — use `curl` for end-to-end verification instead |
| `sudo` prompts for a password mid-script | Privilege cache expired in the current session | Re-authenticate with a plain `sudo` command first |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.11) out loud
      from "serve two static sites on port 8086" to "verified via
      `curl`."
- [ ] Explain, in one sentence, why changing the `Listen` port has no
      effect on where Apache looks for files.
- [ ] Explain why `cp -r /tmp/news/. /var/www/html/news/` is correct
      but `cp -r /tmp/news /var/www/html/news/` produces a nested
      `news/news/` directory.
- [ ] Explain why a typo'd directory name (`appsv` instead of `apps`)
      produces no error at creation time, and where it *does* surface.
- [ ] Explain why `httpd -t` succeeding doesn't by itself prove the
      websites are reachable — what's the one check that does?
- [ ] From memory, add a third static site (`blog`) on a different URL
      path under the same Apache instance, without copying any command
      from this file, then verify it with `curl`.

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
