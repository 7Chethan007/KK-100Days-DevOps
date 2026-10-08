# Day 18 — Install and Configure MariaDB Database Server

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the third database-administration lab in this series — Day 9
diagnosed a *down* MariaDB on a different server (ownership/
`ExecStartPre` failure); Day 17 configured a *running* PostgreSQL
instance (roles, database, `GRANT`). Today combines both: install
MariaDB from scratch, bring it up correctly, then apply the exact same
user/database/privilege pattern as Day 17, just in MariaDB's syntax.

---

## 1. Scenario

On `stdb01`: install MariaDB, get it running and boot-persistent,
create database `kodekloud_db2`, create user `kodekloud_sam` (password
`TmPcZjtRQx`), and grant that user full privileges on
`kodekloud_db2`.

```text
stdb01
├── MariaDB Server (mariadb.service)
└── MariaDB Data
    ├── Database: kodekloud_db2
    └── User: kodekloud_sam@localhost → ALL PRIVILEGES ON kodekloud_db2.*
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Two independent identity systems — recap, now for MariaDB specifically

```text
Linux            MariaDB
peter      ≠      kodekloud_sam@localhost
```

Same "two separate control planes" distinction as Day 17's PostgreSQL
lab (and Day 5's SELinux DAC/MAC split) — the Linux account used to
SSH in has no inherent relationship to any MariaDB account. Creating
`kodekloud_sam` as a MariaDB user never creates a corresponding Linux
user, and vice versa.

### 2.2 "Installed" is a claim about software existing, not about anything running

```bash
sudo systemctl is-active mariadb
```
```text
inactive
```
```bash
mariadb --version
```
```text
mariadb: command not found
```

Both checks came back negative — not because one implies the other,
but because this lab genuinely started from "nothing installed yet."
Checking *both* the systemd state and the actual client binary before
installing is worth doing as a pair, since a service showing
`inactive` could mean "installed but stopped" (Day 9's actual
scenario) or "never installed at all" (today's) — two very different
starting points requiring different first moves.

### 2.3 `dnf install` only gets you to "exists on disk" — start and enable are separate, deliberate steps

```text
dnf install mariadb-server   → software now exists
systemctl start mariadb       → process now running (THIS session)
systemctl enable mariadb      → process will run again after a REBOOT
```

Identical three-step progression to Day 9's MariaDB lab and Day 17's
PostgreSQL lab (`installed` → `active` → `enabled` as three
independent facts) — a freshly-installed MariaDB is `inactive` and
`disabled` by default, and both states need to be explicitly changed,
not assumed to follow from installation alone.

### 2.4 Administrative access via `sudo mariadb` — the MariaDB analog of PostgreSQL's peer auth

```bash
sudo mariadb
```
```text
MariaDB [(none)]>
```

Running `mariadb` under `sudo` (as root) gets you connected as the
MariaDB `root@localhost` account without a separate password prompt —
conceptually parallel to Day 17's PostgreSQL peer authentication
(`sudo -iu postgres` then `psql`), just implemented through MariaDB's
own default root-auth mechanism rather than PostgreSQL's peer-auth
system. Different mechanism, same practical shape: elevated Linux
access is what gets you administrative database access locally.

### 2.5 `USER()` vs. `CURRENT_USER()` — MariaDB's version of a distinction worth knowing generally

```sql
SELECT USER(), CURRENT_USER();
```
```text
root@localhost   root@localhost
```

`USER()` reports the account that authenticated the connection;
`CURRENT_USER()` reports the account whose privileges are actually in
effect. For a direct root login these are identical, but they can
diverge in more complex authentication setups (e.g. proxying,
`SET ROLE`) — worth knowing both exist even though this lab never
needs them to differ.

### 2.6 A MariaDB account's real identity is `'user'@'host'`, not just the username

```text
'kodekloud_sam'@'localhost'   ≠   'kodekloud_sam'@'192.168.1.10'   ≠   'kodekloud_sam'@'%'
```

The host component restricts *where a connection is allowed to
originate from* — these are three genuinely distinct accounts in
MariaDB's privilege system, even though they share a username. Every
`CREATE USER`/`GRANT`/verification statement in this lab explicitly
includes `@'localhost'` for exactly this reason — omitting it or using
a different host string would create or reference a different
account entirely.

### 2.7 `CREATE DATABASE` creates an empty namespace — tables are a separate, unrequested concern

```sql
CREATE DATABASE kodekloud_db2;
```

```text
Database
├── Tables      (none created — not required by this task)
├── Views
├── Procedures
└── Functions
```

Same "don't add scope beyond what's required" discipline as Day 17's
PostgreSQL lab (which also stopped at the database level, no table
creation) — this task only asks for the database and the privilege
grant, not any schema inside it.

### 2.8 `GRANT ALL PRIVILEGES ON kodekloud_db2.*` — reading the scope precisely

```sql
GRANT ALL PRIVILEGES ON kodekloud_db2.* TO 'kodekloud_sam'@'localhost';
```

```text
kodekloud_db2.*
     │      │
     │      └── every object (table, view, etc.) inside this database
     └── scoped to exactly this one database, not every database on the server
```

The `*` matters precisely because MariaDB's grant syntax is
`database.object` — `kodekloud_db2.*` means "everything inside this
one database," while `*.*` (which shows up separately in `SHOW GRANTS`
as `GRANT USAGE ON *.*`) would mean "every database on the server."
Seeing `GRANT USAGE ON *.*` in the output isn't a sign of broader
access — `USAGE` is MariaDB's way of saying "this account exists and
can connect," distinct from the actual applied privilege
(`ALL PRIVILEGES ON kodekloud_db2.*`) granted separately.

### 2.9 Verify the database and the user as two independent facts

```sql
SHOW DATABASES LIKE 'kodekloud_db2';
SELECT User, Host FROM mysql.user WHERE User = 'kodekloud_sam';
```

Two separate verification queries for two separate things that were
created — confirming the database exists says nothing about whether
the user account exists, and vice versa. Same "verify every
independent fact the requirement specifies, not just one" discipline
running through this entire series.

### 2.10 `SHOW GRANTS` confirms configuration — a real login confirms it actually works

```bash
mariadb -u kodekloud_sam -p'TmPcZjtRQx' -e "SELECT USER(), CURRENT_USER();"
```
```text
kodekloud_sam@localhost   kodekloud_sam@localhost
```

`SHOW GRANTS FOR ...` (as root) proves the *configuration* says what
you expect. Actually logging in *as* `kodekloud_sam`, with the real
password, from the shell, proves the configuration *works end to end*
— authentication succeeds, the account resolves to exactly the
expected `user@host` pair. This is stronger evidence than reading
configuration, the same "test the actual behavior, not just the
declared state" principle as Day 15's SSH lab (`BatchMode=yes` testing
real passwordless auth rather than just checking `authorized_keys`
exists).

### 2.11 An empty table list after `USE kodekloud_db2; SHOW TABLES;` is success, not failure

```bash
mariadb -u kodekloud_sam -p'TmPcZjtRQx' -e "USE kodekloud_db2; SHOW TABLES;"
```
(no output)

No tables is exactly expected — none were created, and none were
required. The command returning *without an access-denied error* is
the actual signal that matters: authentication succeeded and the
granted privilege allowed selecting the database. Reading "no output"
as "something went wrong" would be a misdiagnosis — the absence of an
error is the success condition here, not the absence of rows.

### 2.12 The compressed reasoning chain

```text
Requirement (install+configure MariaDB; db kodekloud_db2; user kodekloud_sam; full grant)
   → Check BOTH systemctl state AND mariadb --version    → neither present: not installed at all
   → dnf install -y mariadb-server
   → systemctl start mariadb; is-active                    → active
   → systemctl enable mariadb; is-enabled                   → enabled
   → sudo mariadb  (administrative access, root@localhost)
   → CREATE DATABASE kodekloud_db2
   → CREATE USER 'kodekloud_sam'@'localhost' IDENTIFIED BY 'TmPcZjtRQx'
   → GRANT ALL PRIVILEGES ON kodekloud_db2.* TO 'kodekloud_sam'@'localhost'
   → Verify DB exists: SHOW DATABASES LIKE '...'
   → Verify user exists: SELECT User,Host FROM mysql.user WHERE ...
   → Verify grant: SHOW GRANTS FOR 'kodekloud_sam'@'localhost'
   → REAL login test: mariadb -u kodekloud_sam -p'...' -e "SELECT USER(),CURRENT_USER();"
   → REAL access test: USE kodekloud_db2; SHOW TABLES;  → empty output = success, not failure
```

---

## 3. Concepts (reference)

### 3.1 Linux identity vs. database identity (recap from Day 17)
Two independent systems that happen to be used together on the same
machine — a Linux account exists to authenticate OS-level access
(SSH, `sudo`); a database account exists to authenticate to the
database server. Neither implies or creates the other.

### 3.2 `installed` / `active` / `enabled` as three independent facts (recap from Days 5, 9, 17)
Installing a package never implies the service is running; running it
now never implies it survives a reboot. Each state must be checked
and, if needed, explicitly set.

### 3.3 MariaDB's `'user'@'host'` account model
The real identity of a MariaDB account is the pair, not the username
alone. Different host values for the same username are genuinely
different accounts with potentially different privileges.

### 3.4 `GRANT ... ON database.*` scoping
The grant target is always `database.object`; `*` as the object means
"every object in that database," and `*` as the database means "every
database on the server." `GRANT USAGE ON *.*` appearing in `SHOW
GRANTS` output just means "this account is permitted to connect at
all" — it's not evidence of broad, unintended access.

### 3.5 Configuration verification vs. behavioral verification
Reading `SHOW GRANTS` confirms what the server's catalog *says*.
Actually connecting as the target account and running a real query
confirms the configuration *works*. Both matter; the second is
stronger evidence (recap of the same principle from Day 15's SSH lab).

### 3.6 Default TCP port and basic connectivity checks
MariaDB/MySQL conventionally listens on TCP `3306`; `ss -lntp | grep
3306` is the quick way to confirm the server is actually accepting
network connections, separate from confirming the systemd service
state.

### 3.7 Command-line password exposure as a lab-only shortcut
`-p'TmPcZjtRQx'` on the command line is convenient for this disposable
lab but exposes the password via shell history, process listings, and
logs — the same caution as every other lab in this series that used
inline credentials for speed (Days 10, 12, 16) rather than a secrets
manager or credential file, which real production tooling should use
instead.

---

## 4. Runbook

### 4.1 Connect to the database server
```bash
sshpass -p 'Sp!dy' ssh -o StrictHostKeyChecking=no peter@stdb01
```

### 4.2 Check both installation facts before assuming either
```bash
sudo systemctl is-active mariadb
```
```text
inactive
```
```bash
mariadb --version
```
```text
mariadb: command not found
```
Neither present — MariaDB genuinely isn't installed yet.

### 4.3 Install MariaDB
```bash
sudo dnf install -y mariadb-server
mariadb --version
```
```text
mariadb  Ver 15.1 Distrib 10.5.29-MariaDB
```

### 4.4 Start and verify
```bash
sudo systemctl start mariadb
sudo systemctl is-active mariadb
```
```text
active
```

### 4.5 Enable and verify
```bash
sudo systemctl is-enabled mariadb
```
```text
disabled
```
```bash
sudo systemctl enable mariadb
sudo systemctl is-enabled mariadb
```
```text
enabled
```

### 4.6 Connect as administrator
```bash
sudo mariadb
```
```text
MariaDB [(none)]>
```
```sql
SELECT VERSION();
```
```text
10.5.29-MariaDB
```
```sql
SELECT USER(), CURRENT_USER();
```
```text
root@localhost   root@localhost
```

### 4.7 Create the database
```sql
CREATE DATABASE kodekloud_db2;
```
```text
Query OK
```

### 4.8 Create the user
```sql
CREATE USER 'kodekloud_sam'@'localhost' IDENTIFIED BY 'TmPcZjtRQx';
```

### 4.9 Grant full privileges
```sql
GRANT ALL PRIVILEGES ON kodekloud_db2.* TO 'kodekloud_sam'@'localhost';
```

### 4.10 Verify the database exists
```sql
SHOW DATABASES LIKE 'kodekloud_db2';
```
```text
kodekloud_db2
```

### 4.11 Verify the user exists
```sql
SELECT User, Host FROM mysql.user WHERE User = 'kodekloud_sam';
```
```text
User             Host
kodekloud_sam    localhost
```

### 4.12 Verify the grant
```sql
SHOW GRANTS FOR 'kodekloud_sam'@'localhost';
```
```text
GRANT ALL PRIVILEGES ON `kodekloud_db2`.* TO `kodekloud_sam`@`localhost`
GRANT USAGE ON *.* TO `kodekloud_sam`@`localhost`
```

### 4.13 Real login test (as the application user, from the shell)
```bash
mariadb -u kodekloud_sam -p'TmPcZjtRQx' -e "SELECT USER(), CURRENT_USER();"
```
```text
USER()                     CURRENT_USER()
kodekloud_sam@localhost    kodekloud_sam@localhost
```

### 4.14 Real database access test
```bash
mariadb -u kodekloud_sam -p'TmPcZjtRQx' -e "USE kodekloud_db2; SHOW TABLES;"
```
(no output — no tables exist yet, and none were required; no
access-denied error is the success signal)

### 4.15 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| MariaDB installed | ✅ `10.5.29-MariaDB` |
| Service `active` | ✅ |
| Service `enabled` | ✅ |
| Database `kodekloud_db2` created | ✅ |
| User `kodekloud_sam@localhost` created with the specified password | ✅ |
| `ALL PRIVILEGES` granted on `kodekloud_db2.*` | ✅ |
| Real login as `kodekloud_sam` succeeds | ✅ |
| Real `USE kodekloud_db2` succeeds (no access-denied) | ✅ |

```text
stdb01
└── mariadb.service (active, enabled)
    ├── kodekloud_db2                       (empty database, no tables)
    └── kodekloud_sam@localhost
            │
            └── ALL PRIVILEGES ON kodekloud_db2.*
                     │
                     ▼
         verified by a REAL login + REAL USE kodekloud_db2
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `mariadb: command not found` even after checking `systemctl` | MariaDB genuinely never installed, not just stopped | `dnf install -y mariadb-server` |
| `systemctl is-active mariadb` → `inactive` right after install | Installing never starts the service | `systemctl start mariadb` |
| MariaDB running now, but won't survive a reboot | Service installed/started but never enabled | `systemctl enable mariadb` |
| `ERROR 1045 (28000): Access denied` logging in as `kodekloud_sam` | Wrong password, wrong host component on the account, or the account doesn't actually exist | `SELECT User, Host FROM mysql.user WHERE User = 'kodekloud_sam';` to confirm the exact account; re-check the password |
| Confused `GRANT USAGE ON *.*` in `SHOW GRANTS` output as broad access | Misreading `USAGE` (just "can connect") as a real privilege grant | The actual applied privilege is the separate `ALL PRIVILEGES ON kodekloud_db2.*` line |
| `SHOW TABLES` after `USE kodekloud_db2` returns nothing and this looks like a failure | Expected — no tables were created or required | Absence of an access-denied error is the success signal, not the presence of rows |
| Created the user without specifying `@'localhost'` | Account identity in MariaDB is `user@host`, not just the username | Always include the explicit host in `CREATE USER`/`GRANT`/verification statements |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.12) out loud
      from "install MariaDB from scratch" to "verified via a real
      login and a real `USE` statement."
- [ ] Explain, in one sentence, why a MariaDB account's real identity
      includes a host component, not just a username.
- [ ] Explain why `GRANT USAGE ON *.*` showing up in `SHOW GRANTS`
      output isn't evidence of unintended broad access.
- [ ] Explain the difference between verifying a grant via `SHOW
      GRANTS` (as root) and verifying it via an actual login as the
      target user — why is the second stronger evidence?
- [ ] Explain why an empty result from `SHOW TABLES` after a
      successful `USE kodekloud_db2` is the expected outcome, not a
      sign something failed.
- [ ] Compare this lab's structure to Day 17's PostgreSQL lab — what's
      identical in the reasoning, and what's different in the exact
      SQL syntax?
- [ ] Create a second database and user from memory, grant only
      `SELECT` (not `ALL PRIVILEGES`), and verify via a real login that
      the user can query but not modify data.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: check EVERY relevant "does this exist/is this running"
fact independently, don't assume one implies another → install/start/
enable as separate deliberate steps → apply the minimal required
configuration at the correct scope → verify each independent fact
separately → verify the actual real-world behavior, not just the
declared configuration. End with a compressed arrow-chain version.>

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
