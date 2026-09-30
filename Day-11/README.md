# Day 11 — Install & Configure Apache Tomcat Server

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

---

## 1. Scenario

Install Apache Tomcat on **App Server 3** (`stapp03`), configure it to
listen on port `5002`, deploy `ROOT.war`, and verify the application is
reachable at `http://stapp03:5002`.

```text
                    HTTP Request
                         │
                         ▼
              ┌────────────────────┐
              │      stapp03        │
              │   TCP Port 5002     │
              │        │            │
              │        ▼            │
              │     Tomcat          │
              │        │            │
              │        ▼            │
              │     ROOT.war        │
              │        │            │
              │        ▼            │
              │   Web Application    │
              └────────────────────┘
```

---

## 2. Reasoning model — how to *derive* the fix, not memorize it

### 2.1 The request path is the debugging model

```text
DNS/hostname resolution
        ↓
stapp03
        ↓
TCP connection to port 5002
        ↓
Tomcat listening on 5002
        ↓
Tomcat maps "/" to the ROOT application
        ↓
ROOT.war application
        ↓
HTTP 200 response
```

Every layer here is a separate, independently-checkable fact. If
`curl http://stapp03:5002` fails, the useful question isn't "what's
wrong with Tomcat" — it's "*which layer* of this chain is the first one
that's broken" (§2.13 formalizes this as the actual troubleshooting
sequence).

### 2.2 What Tomcat is, specifically

Tomcat is a **servlet container** — it runs Java web applications
packaged as Servlets/JSP/WAR files, distinct from a general-purpose web
server (Nginx, Apache HTTPD) that primarily serves static
HTML/CSS/JS/images. One Tomcat instance can host multiple independent
applications simultaneously:

```text
                 Network
                    │
                    ▼
              TCP :5002
                    │
                    ▼
                Tomcat
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Application A       Application B
          │                   │
       ROOT.war          another.war
```

### 2.3 Ports identify the service, not the machine

```text
10.244.x.x : 5002
     │        │
  machine    service endpoint
```

`curl http://stapp03:5002` means: resolve `stapp03` to an address,
open a TCP connection specifically to port `5002` on that address, then
speak HTTP over it. A machine can run many services on many ports
simultaneously (SSH on 22, HTTP on 80, Tomcat's default on 8080) — the
port is what disambiguates which one you're actually talking to.

### 2.4 Why Tomcat defaults to 8080, and why we change it in config, not code

```xml
<Connector port="8080" protocol="HTTP/1.1" ... />
```

The task requires `5002`, not `8080`. This is a **runtime
configuration** change, not an application code change — the general
pattern behind almost every "make this service listen on a different
port" task: locate the config file that declares the binding, edit that,
restart the service. You never modify Tomcat's own source to change a
port.

### 2.5 What a WAR file is

```text
ROOT.war
│
├── HTML/static resources
├── WEB-INF/
│   ├── classes/
│   └── ...
└── application files
```

**WAR = Web Application Archive** — a packaged Java web application.
Tomcat watches its deployment directory and automatically expands/
deploys any WAR file dropped into it.

### 2.6 Why the filename `ROOT.war` specifically matters

Tomcat derives a deployed application's **context path** directly from
the WAR's filename:

```text
ROOT.war     →  context path  /
shop.war     →  context path  /shop
ecommerce.war → context path  /ecommerce
```

This is the entire reason the task specifies `ROOT.war` by name — it's
what makes `curl http://stapp03:5002` (no path suffix) reach the
application, instead of requiring `http://stapp03:5002/ROOT`. Naming the
WAR correctly *is* the configuration step for the URL path, not a
separate setting somewhere else.

### 2.7 The webapps directory is often a symlink — know the real path

```bash
ls -ld /usr/share/tomcat/webapps
```
```text
lrwxrwxrwx ... /usr/share/tomcat/webapps -> /var/lib/tomcat/webapps
```

On this RPM-based install, `/usr/share/tomcat/webapps` and
`/var/lib/tomcat/webapps` are the *same* location — a common Linux
packaging pattern (keep the "canonical" software tree under
`/usr/share`, symlink to the actual variable/writable data under
`/var/lib`). Worth confirming explicitly rather than assuming, since
copying a WAR to the wrong one of two seemingly-different paths is an
easy, silent mistake if you don't know they're linked.

### 2.8 Establish the baseline before installing anything

```bash
hostname
java -version
rpm -qa | grep -i tomcat
systemctl status tomcat --no-pager
```

Confirm you're on `stapp03`, confirm Java exists (Tomcat needs a JVM),
and confirm Tomcat *isn't* already installed/running before assuming a
clean install is the right move — same "inspect before act" discipline
as every prior lab in this series.

### 2.9 `dnf info` before `dnf install` — know what you're about to install

```bash
dnf search tomcat
dnf info tomcat
```

Checking the package's name, version, repo, and description before
installing is the same "don't blindly install" discipline as Day 8's
Ansible lab — cheap insurance against installing the wrong package
variant, especially on a system where multiple similarly-named packages
might exist.

### 2.10 Installed ≠ running — a service has multiple independent states

```text
Package installed
       ↓
Service configured
       ↓
Service started
       ↓
Service healthy
```

Immediately after `dnf install -y tomcat`, `systemctl status tomcat`
correctly shows `inactive (dead)` — installing a package never
implies the service is running, exactly the `installed`/`enabled`/
`active` distinction from Day 9's MariaDB lab, applied here to a fresh
install rather than a broken existing one.

### 2.11 Find configuration by querying the package, not guessing

```bash
rpm -ql tomcat | grep 'server.xml'
```
```text
/etc/tomcat/server.xml
```

`rpm -ql <package>` lists every file a package actually installed —
using it to *find* the config file is more reliable than guessing a
path from memory or documentation that may not match this exact
package's layout.

### 2.12 A configuration match in `grep` doesn't mean it's the active setting

```bash
grep -n '8080\|5002' /etc/tomcat/server.xml
```

can surface a match that's actually inside an XML comment block:

```xml
<!--
    <Connector executor="tomcatThreadPool"
               port="5002"
    ...
-->
```

**Text appearing in a config file is not proof the application is using
it.** You have to understand the file's comment syntax (`<!-- -->` for
XML) to tell an active directive from a disabled/example one — the same
"read the actual semantics, not just grep for a string" discipline as
recognizing a Makefile recipe line needs a real tab (Day 5, MLOps).

### 2.13 Backup before editing, then verify the edit landed correctly

```bash
cp /etc/tomcat/server.xml /etc/tomcat/server.xml.bak
sed -i 's/port="8080"/port="5002"/' /etc/tomcat/server.xml
grep -n '5002\|8080' /etc/tomcat/server.xml
```

A backup before a config edit is a cheap, reversible safety net — if the
`sed` substitution does something unexpected (matches more than
intended, or nothing at all), you have an immediate rollback path rather
than trying to reconstruct the original file from memory.

### 2.14 Logs are stronger evidence than a green systemd status alone

```bash
systemctl status tomcat --no-pager -l
```
```text
Active: active (running)
```

is real evidence, but the **logs** prove something more specific:

```text
Initializing ProtocolHandler ["http-nio-5002"]
Starting ProtocolHandler ["http-nio-5002"]
```

`active (running)` confirms the Java *process* is alive; the log lines
confirm Tomcat's HTTP connector actually bound to port `5002`
specifically. A process can be running while its application-level
configuration is broken — checking progressively deeper (process → port
→ application → response) is more reliable than stopping at the first
green signal.

### 2.15 Deployment is observable in logs too, not just "the file exists"

```bash
journalctl -u tomcat -n 30 --no-pager
```
```text
Deploying web application archive [/var/lib/tomcat/webapps/ROOT.war]
Deployment of web application archive [/var/lib/tomcat/webapps/ROOT.war] has finished
```

Seeing a `ROOT/` directory appear alongside `ROOT.war` confirms Tomcat
expanded the archive; the log lines confirm it did so *without error*.
Copying the WAR file is necessary but not sufficient evidence of
successful deployment — Tomcat still has to process it.

### 2.16 Three levels of verification, each proving something different

```bash
curl -i http://localhost:5002              # Tomcat itself is serving HTTP
curl -i http://stapp03:5002                # ...AND reachable by hostname/network
systemctl is-active tomcat                  # ...AND the service layer agrees
```

```text
Level 1 (localhost)   proves: Tomcat + application work, LOCALLY
Level 2 (hostname)     proves: the FULL network path also works
Level 3 (systemd)       proves: the service layer's own view agrees
```

If localhost works but the hostname doesn't, the problem is narrowed to
something between "Tomcat itself" and "the network path to it" (DNS,
firewall, binding address) — exactly the kind of narrowing a single
"does curl work" test can't give you.

### 2.17 HTTP 200 + real content is stronger proof than "the port is open"

```text
HTTP/1.1 200
<h2>Welcome to xFusionCorp Industries!</h2>
```

A listening port alone only proves *something* is accepting TCP
connections — it says nothing about whether that something is Tomcat,
whether the application deployed correctly, or whether it's serving the
right content. Getting back the actual expected HTML proves the entire
chain in §2.1 end-to-end, not just its first link.

### 2.18 The compressed reasoning chain

```text
Requirement (Tomcat on stapp03, port 5002, ROOT.war deployed, reachable)
   → Baseline: hostname, java -version, confirm Tomcat not yet installed
   → dnf info tomcat (know what you're installing) → dnf install -y tomcat
   → Confirm installed ≠ running (expected inactive right after install)
   → rpm -ql tomcat | grep server.xml → locate the real config file
   → grep for the active (non-commented) Connector port
   → cp server.xml server.xml.bak → sed 8080 → 5002 → grep to confirm
   → systemctl start tomcat → check BOTH status AND logs for "http-nio-5002"
   → Copy ROOT.war into webapps/ (aware it may be a symlinked path)
   → journalctl: confirm "Deploying..." / "...has finished", not just file presence
   → curl localhost:5002 → curl stapp03:5002 → systemctl is-active tomcat
   → Confirm HTTP 200 AND expected HTML content, not just an open port
```

---

## 3. Concepts (reference)

### 3.1 Servlet container vs. general web server
Tomcat executes Java web application code (Servlets, JSP); Nginx/Apache
HTTPD primarily serve static content and commonly reverse-proxy to
application servers like Tomcat in production. They solve different
problems and are often used together, not as substitutes for each
other.

### 3.2 Context path derivation from WAR filename
Tomcat's default deployer maps a WAR's filename directly to a URL path
segment, with `ROOT.war` as the special case that maps to `/`. This is
Tomcat-specific behavior worth knowing generally, not just for this one
task.

### 3.3 `rpm -ql <package>` for locating installed files
The reliable way to find where a package actually placed its
configuration/binaries/docs on *this* system, rather than trusting a
remembered or documented path that may differ by distro/version.

### 3.4 XML comments and "text present ≠ text active"
`<!-- ... -->` disables everything between the markers in XML-based
config files (Tomcat's `server.xml`, many others). A `grep` match inside
a comment block is not evidence of active configuration — always confirm
a match isn't inside a comment before trusting it.

### 3.5 `systemctl status` vs. `journalctl -u <service>`
Same distinction as Day 9's MariaDB lab: `status` gives the current
snapshot (and can truncate); `journalctl -u <service>` gives the fuller
log history, including deployment/startup messages a status snapshot
won't show.

### 3.6 Backing up before editing production config
`cp file file.bak` before a `sed`/manual edit is a near-zero-cost safety
net — always cheaper than trying to reconstruct a working config from
memory after a bad edit.

### 3.7 Layered verification over single-signal trust
Checking process state, port binding, logs, and actual HTTP response
content are four independent signals — trusting only the first
(`active (running)`) risks missing a failure at any of the later,
more specific layers.

---

## 4. Runbook

### 4.1 Establish the baseline
```bash
hostname
```
```text
stapp03
```
```bash
java -version
```
```text
openjdk version "11..."
```
```bash
rpm -qa | grep -i tomcat
systemctl status tomcat --no-pager
```
Confirms: correct host, Java present, Tomcat not yet installed.

### 4.2 Discover and inspect the package before installing
```bash
dnf search tomcat
dnf info tomcat
```
```text
Name: tomcat
Version: 9.0.120
```

### 4.3 Install Tomcat
```bash
dnf install -y tomcat
rpm -q tomcat
```
```text
tomcat-9.0.120-1.el9.noarch
```
```bash
systemctl status tomcat --no-pager
```
```text
Active: inactive (dead)
```
Expected — installed, not yet started (§2.10).

### 4.4 Locate the configuration file
```bash
rpm -ql tomcat | grep 'server.xml'
```
```text
/etc/tomcat/server.xml
```
```bash
grep -n '<Connector' /etc/tomcat/server.xml
grep -n '8080\|5002' /etc/tomcat/server.xml
```
Confirm the `8080` match is the **active**, uncommented connector
(§2.12).

### 4.5 Confirm the webapps directory's real location
```bash
ls -ld /usr/share/tomcat/webapps
```
```text
lrwxrwxrwx ... /usr/share/tomcat/webapps -> /var/lib/tomcat/webapps
```

### 4.6 Back up and change the port
```bash
cp /etc/tomcat/server.xml /etc/tomcat/server.xml.bak
sed -i 's/port="8080"/port="5002"/' /etc/tomcat/server.xml
grep -n '5002\|8080' /etc/tomcat/server.xml
```
```text
<Connector port="5002" protocol="HTTP/1.1"
```

### 4.7 Start Tomcat and verify via status AND logs
```bash
systemctl start tomcat
systemctl status tomcat --no-pager -l
```
```text
Active: active (running)
```
```bash
journalctl -u tomcat -n 30 --no-pager
```
```text
Initializing ProtocolHandler ["http-nio-5002"]
Starting ProtocolHandler ["http-nio-5002"]
```

### 4.8 Retrieve and deploy ROOT.war
```bash
ssh thor@jump-host 'ls -lh /tmp/ROOT.war'
scp thor@jump-host:/tmp/ROOT.war /usr/share/tomcat/webapps/
ls -lh /usr/share/tomcat/webapps/ROOT.war
```

### 4.9 Confirm deployment via logs
```bash
journalctl -u tomcat -n 30 --no-pager
```
```text
Deploying web application archive [/var/lib/tomcat/webapps/ROOT.war]
Deployment of web application archive [/var/lib/tomcat/webapps/ROOT.war] has finished
```
```bash
ls -lah /usr/share/tomcat/webapps/
```
```text
ROOT/
ROOT.war
```

### 4.10 Verify at three levels
```bash
curl -i http://localhost:5002
```
```text
HTTP/1.1 200
<h2>Welcome to xFusionCorp Industries!</h2>
```
```bash
curl -i http://stapp03:5002
```
```text
HTTP/1.1 200
```
```bash
systemctl is-active tomcat
```
```text
active
```

### 4.11 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| Tomcat installed | ✅ `9.0.120` |
| Service active | ✅ `active (running)` |
| Connector listening on `5002` | ✅ confirmed via logs (`http-nio-5002`) |
| `ROOT.war` deployed | ✅ expanded to `ROOT/`, logged "has finished" |
| Context path `/` | ✅ (from `ROOT.war` naming) |
| `curl http://localhost:5002` | ✅ `200` |
| `curl http://stapp03:5002` | ✅ `200` |

```text
                    USER
                     │ HTTP
                     ▼
              http://stapp03:5002
                     │
                  TCP :5002
                     │
              ┌──────────────┐
              │    Tomcat    │
              │   (systemd)  │
              └──────┬───────┘
                     │
               server.xml
               Connector port=5002
                     │
                     ▼
             webapps/ROOT.war
                     │
                     ▼
              ROOT application (/)
                     │
                     ▼
            HTTP 200 + HTML response
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `systemctl status tomcat` shows `inactive` right after `dnf install` | Installing a package never starts its service | `systemctl start tomcat`; installed ≠ running (§2.10) |
| `grep` finds `port="5002"` but Tomcat still listens on `8080` | The matched line is inside an XML comment block | Re-check the file manually around the match; confirm it's not between `<!--` and `-->` (§2.12) |
| `curl http://stapp03:5002` returns nothing / connection refused | Tomcat not actually bound to 5002, or not started at all | Check `journalctl -u tomcat` for `Starting ProtocolHandler ["http-nio-5002"]`; re-verify the `server.xml` edit |
| `curl http://localhost:5002` works, but `curl http://stapp03:5002` doesn't | The local application layer is fine; something in the network path (hostname resolution, firewall) is blocking hostname access | Narrow to the network layer specifically — don't re-touch Tomcat config for a networking problem |
| `ROOT.war` present in `webapps/` but app still unreachable | WAR copied but never actually deployed/expanded (e.g. wrong directory due to the symlink, or Tomcat wasn't running when copied) | Confirm `ROOT/` directory appeared and check logs for "Deployment ... has finished" |
| App reachable only at `/ROOT`, not `/` | WAR file wasn't actually named `ROOT.war` (or has an unexpected suffix/typo) | `ls -lh webapps/` to confirm the exact filename; context path is derived directly from it (§2.6) |
| Command fails with a garbled error after copy-pasting multiple commands | Two commands accidentally concatenated with no separator (e.g. `... | grep ...ss ...`) | Re-type the command fresh rather than trying to patch a mis-pasted one |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.18) out loud
      from "install Tomcat and deploy ROOT.war on port 5002" to "HTTP
      200 with real content, verified at three levels."
- [ ] Explain, in one sentence, why `ROOT.war` specifically maps to `/`
      while `shop.war` would map to `/shop`.
- [ ] Explain why a `grep` match for a port number in `server.xml`
      isn't sufficient proof that port is actually active.
- [ ] Explain the difference between `systemctl status` showing
      `active (running)` and the log lines showing
      `Starting ProtocolHandler ["http-nio-5002"]` — why check both?
- [ ] Explain why `curl localhost:5002` succeeding and
      `curl stapp03:5002` failing narrows the problem to a specific
      layer, and which layer that is.
- [ ] Redo the full install-to-verify sequence from memory on a fresh
      host, deliberately backing up `server.xml` before editing it.

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
success state, at multiple layers. End with a compressed arrow-chain
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
