# Day 17 — Install and Configure PostgreSQL

A KodeKloud "100 Days DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each command exists and *how you'd derive it yourself*.

This is the second database lab in this series — Day 9 diagnosed a
*down* MariaDB service; today's task is configuring a *running*
PostgreSQL instance's users/databases/privileges, explicitly without
restarting it. Different database engine, but several of Day 9's
layering distinctions apply directly.

---

## 1. Scenario

On `stdb01`, PostgreSQL is already installed and running. Task: create
a PostgreSQL role `kodekloud_gem`, create a database `kodekloud_db8`,
grant database-level privileges from the role to the database, and
verify — **without restarting PostgreSQL**.

```text
Linux Server
   │
   ├── PostgreSQL Service (systemd)
   │       │
   │       ├── Roles → kodekloud_gem
   │       └── Databases → kodekloud_db8
   │
   └── Linux Users → peter, postgres
```

---

## 2. Reasoning model — how to *derive* the commands, not memorize them

### 2.1 Two separate authentication/permission worlds — Linux and PostgreSQL

```text
Linux               PostgreSQL
──────              ──────────
users: peter,        roles: postgres, kodekloud_gem
       postgres
controls: SSH,       controls: databases, schemas,
  files, sudo           tables, SQL permissions
```

The Linux `postgres` user and the PostgreSQL `postgres` role share a
name and are commonly used together for local administration, but
they are genuinely different systems with different permission models
— this is the same "two independent control planes" distinction that
made Day 5's SELinux lab (DAC vs. MAC) and every "which layer is
actually failing" lab in this series non-trivial.

### 2.2 Installed and running are two different claims — verify, don't assume

```bash
sudo systemctl is-active postgresql
```
```text
active
```

The task states PostgreSQL is "already installed" — that's a claim
about the *software*, not about whether the *service* is currently
running. Same `installed`/`active` distinction from Day 9's MariaDB
lab and Day 11's Tomcat install — never assume "installed" implies
"running," check explicitly.

### 2.3 Why the explicit "don't restart" constraint matters here

```text
Role creation, database creation, and GRANT all take effect
IMMEDIATELY — none of them require a service restart.
```

This constraint isn't arbitrary friction — it's testing whether you
know that PostgreSQL's catalog-level changes (new roles, new
databases, privilege grants) are live operations, not configuration-
file edits requiring a reload. Reaching for `systemctl restart
postgresql` here would indicate a misunderstanding of *which* kinds of
PostgreSQL changes need a restart (connection/memory/auth-config
changes in `postgresql.conf`) versus which don't (anything done
through `psql`/SQL, including this entire lab).

### 2.4 Peer authentication — why becoming the Linux `postgres` user matters

```bash
sudo -iu postgres
whoami
```
```text
postgres
```

PostgreSQL's default local authentication method (`peer`) maps the
connecting **Linux** user directly to a **PostgreSQL** role of the same
name, for local (Unix-socket) connections — becoming Linux user
`postgres` is what lets `psql` connect as the PostgreSQL superuser role
`postgres` without a password prompt. This is why the task's `peter`
login has to switch Linux users before running `psql`, rather than
`psql` simply working from any account.

### 2.5 `psql`'s prompt — reading the two pieces of state it encodes

```text
postgres=#
│        │
│        └── # = superuser (a regular, non-superuser role would show =>)
└────────── current database name
```

The prompt alone tells you which database you're connected to and
whether your current role has superuser privileges — useful to read at
a glance before running anything privileged.

### 2.6 SQL vs. `psql` meta-commands — two different things being parsed

```text
SELECT current_user;    → SQL, sent to and executed BY the PostgreSQL server
\du                       → a psql CLIENT command, never sent to the server at all
```

This distinction matters practically: a `\`-prefixed command typed into
a script expecting plain SQL (or vice versa) simply won't work, because
they're handled by entirely different layers — the client program
versus the server it's connected to.

### 2.7 "User" is PostgreSQL's informal name for a login-capable role

```sql
CREATE USER kodekloud_gem WITH PASSWORD 'ksH85UJjhb';
```
```text
CREATE ROLE
```

Notice the server echoes back `CREATE ROLE`, not `CREATE USER` — in
PostgreSQL, `CREATE USER` is literally syntactic sugar for `CREATE
ROLE ... WITH LOGIN`. A "user" is just a role with login capability;
there's no separate underlying object type. This is worth knowing
before being confused by `CREATE ROLE` appearing in output for a
command you typed as `CREATE USER`.

### 2.8 Verifying the role has no excess privileges — a good sign, not a gap

```text
\du kodekloud_gem
```
```text
Role name       | Attributes | Member of
kodekloud_gem   |            | {}
```

An empty `Attributes` column means `kodekloud_gem` has none of
`Superuser`/`Createrole`/`Createdb`/`Replication` — exactly what a
least-privilege role creation should look like, since nothing in the
task asked for any of those broader capabilities. Reading this as
"something's missing" would be the wrong instinct — the task only
asked for a login-capable role with specific *database-level*
privileges, granted separately (§2.9).

### 2.9 `GRANT ... ON DATABASE` grants database-level privileges specifically — not table access

```sql
GRANT ALL PRIVILEGES ON DATABASE kodekloud_db8 TO kodekloud_gem;
```

```text
Database privileges   → CONNECT, CREATE, TEMPORARY  (what this grants)
Schema privileges      → separate layer
Table privileges        → separate layer, further down
```

"ALL PRIVILEGES ON DATABASE" is scoped entirely to the *database*
level — it does not cascade into automatic `SELECT`/`INSERT`/`UPDATE`/
`DELETE` on tables that exist or will exist inside it. The task only
required database-level privileges, so this grant alone fully satisfies
the requirement — no table exists yet, and none was asked for (§2.10).

### 2.10 The task never asked for a table — don't add scope beyond what's required

```text
PostgreSQL hierarchy:  Database → Schema → Table → Rows
Task's actual scope:                Database  (stop here)
```

Creating a table would be solving a problem the task never posed —
worth noting explicitly, since "configure a database" can tempt an
unrequested `CREATE TABLE` out of habit. The lab's acceptance criteria
stop at the database level.

### 2.11 Reading `\l`'s privilege notation

```text
kodekloud_gem=CTc/postgres
```
```text
C → CONNECT
T → TEMPORARY
c → CREATE
/postgres → the role that GRANTED these privileges
```

A compact encoding worth being able to read on sight — three letters
for three database-level privileges, plus who granted them.

### 2.12 Verifying with `has_database_privilege` — asking PostgreSQL directly, not inferring from config

```sql
SELECT has_database_privilege('kodekloud_gem', 'kodekloud_db8', 'CONNECT');
```
```text
t
```

This is categorically stronger evidence than reading `\l`'s notation
and mentally decoding it — you're asking PostgreSQL's own privilege
engine the exact yes/no question the task cares about, and getting an
authoritative `t`/`f` back. Same "verify against the actual system's
own answer, not your interpretation of its displayed state" discipline
running through this entire series.

### 2.13 The compressed reasoning chain

```text
Requirement (role kodekloud_gem, db kodekloud_db8, DB-level grant, NO restart)
   → Confirm PostgreSQL active (don't assume from "already installed")
   → SSH as peter; switch to Linux postgres user (peer auth, §2.4)
   → psql → confirm prompt (postgres=#) and SELECT current_user
   → CREATE USER kodekloud_gem WITH PASSWORD '...'    → server echoes CREATE ROLE (same thing)
   → \du kodekloud_gem                                   → confirm NO extra privileges (correct, least-privilege)
   → CREATE DATABASE kodekloud_db8
   → \l                                                    → confirm database exists
   → GRANT ALL PRIVILEGES ON DATABASE kodekloud_db8 TO kodekloud_gem   (database-level ONLY — §2.9)
   → \l kodekloud_db8                                        → read the CTc/postgres notation
   → SELECT has_database_privilege(...)                        → authoritative t/f confirmation, both CONNECT and CREATE
   → Confirm throughout: NEVER ran systemctl restart/stop/start postgresql
```

---

## 3. Concepts (reference)

### 3.1 Linux users vs. PostgreSQL roles
Two independent identity/permission systems that happen to share names
for convenience (`postgres`/`postgres`). Linux governs OS-level access;
PostgreSQL roles govern database-level access — changing one has no
direct effect on the other.

### 3.2 Peer authentication
PostgreSQL's common default for local (Unix-socket) connections: the
connecting OS user's name must match the PostgreSQL role being
connected as, with no password required. This is why administrative
`psql` access commonly goes through `sudo -iu postgres` first.

### 3.3 `CREATE USER` is `CREATE ROLE ... WITH LOGIN`
PostgreSQL has one underlying object type (`ROLE`); "user" is purely a
naming convenience for a role that can log in. Expect server responses
to say `CREATE ROLE` even when you typed `CREATE USER`.

### 3.4 Database-level vs. schema-level vs. table-level privileges
Three independent, stacked privilege layers. Granting at the database
level (`CONNECT`/`CREATE`/`TEMPORARY`) says nothing about what a role
can do to any specific schema or table inside that database — those
require their own, separate `GRANT` statements.

### 3.5 Which PostgreSQL changes need a restart, and which don't
Role/database/privilege changes via SQL (`CREATE ROLE`, `CREATE
DATABASE`, `GRANT`) take effect immediately — no restart needed.
Changes to `postgresql.conf` (memory settings, connection limits,
listen addresses) typically require a restart or at least a config
reload (`pg_ctl reload` / `SIGHUP`). Knowing which category a change
falls into prevents both unnecessary restarts and missed reloads.

### 3.6 `has_database_privilege()` and similar `has_*_privilege()` functions
PostgreSQL's own built-in functions for authoritatively answering "does
role X have privilege Y on object Z" — stronger verification than
visually parsing `\l`/`\du` output, since you're querying the actual
privilege engine directly.

### 3.7 `systemctl` (the Linux service) vs. `psql`/SQL (the PostgreSQL server)
Same category of separation as Day 9's MariaDB lab — `systemctl`
controls whether the PostgreSQL *process* runs at all; `psql`/SQL
operates entirely within an already-running server's own catalog and
data. Neither substitutes for the other, and most configuration tasks
(like this one) only ever need the second.

---

## 4. Runbook

### 4.1 Connect and confirm PostgreSQL is running (don't restart it)
```bash
sshpass -p 'Sp!dy' ssh -o StrictHostKeyChecking=no peter@stdb01
sudo systemctl is-active postgresql
```
```text
active
```

### 4.2 Confirm client version and the Linux postgres account
```bash
psql --version
```
```text
psql (PostgreSQL) 13.23
```
```bash
id postgres
```
```text
uid=26(postgres) gid=26(postgres) groups=26(postgres)
```

### 4.3 Switch to the Linux postgres user (peer auth) and enter psql
```bash
sudo -iu postgres
whoami
```
```text
postgres
```
```bash
psql
```
```text
postgres=#
```
```sql
SELECT current_user;
```
```text
postgres
```

### 4.4 Create the role
```sql
CREATE USER kodekloud_gem WITH PASSWORD 'ksH85UJjhb';
```
```text
CREATE ROLE
```

### 4.5 Verify the role has no excess privileges
```text
\du kodekloud_gem
```
```text
Role name       | Attributes | Member of
kodekloud_gem   |            | {}
```

### 4.6 Create the database
```sql
CREATE DATABASE kodekloud_db8;
```
```bash
\l
```
Confirm `kodekloud_db8` appears in the list.

### 4.7 Grant database-level privileges
```sql
GRANT ALL PRIVILEGES ON DATABASE kodekloud_db8 TO kodekloud_gem;
```
```text
GRANT
```

### 4.8 Verify via `\l` notation
```text
\l kodekloud_db8
```
```text
kodekloud_gem=CTc/postgres
```

### 4.9 Verify authoritatively via SQL
```sql
SELECT has_database_privilege('kodekloud_gem', 'kodekloud_db8', 'CONNECT');
```
```text
t
```
```sql
SELECT has_database_privilege('kodekloud_gem', 'kodekloud_db8', 'CREATE');
```
```text
t
```

### 4.10 Confirm PostgreSQL was never restarted
```bash
exit    -- back to peter
systemctl is-active postgresql
```
```text
active   (same service instance, never restarted)
```

### 4.11 Click "Check" in the lab UI to validate.

---

## 5. Final state

| Requirement | Result |
|---|---|
| PostgreSQL confirmed running (not assumed from "installed") | ✅ |
| Role `kodekloud_gem` created with password | ✅ |
| Role has no excess privileges | ✅ (empty `Attributes`) |
| Database `kodekloud_db8` created | ✅ |
| Database-level privileges granted (`CONNECT`, `CREATE`, `TEMPORARY`) | ✅ |
| `CONNECT` and `CREATE` verified via `has_database_privilege` | ✅ `t`, `t` |
| PostgreSQL never restarted | ✅ |

```text
PostgreSQL Server (never restarted)
│
├── Role: kodekloud_gem (login, password set, no extra attributes)
│
└── Database: kodekloud_db8
       │
       └── Privileges for kodekloud_gem: CONNECT, CREATE, TEMPORARY
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `psql: FATAL: Peer authentication failed` | Connected as a Linux user whose name doesn't match the intended PostgreSQL role | `sudo -iu postgres` first, then `psql`, for local peer-authenticated administrative access |
| `CREATE USER` succeeds but output says `CREATE ROLE` | Expected — `CREATE USER` is sugar for `CREATE ROLE ... WITH LOGIN` (§2.7) | Not an error; verify with `\du <name>` instead of worrying about the echoed statement name |
| `GRANT ALL PRIVILEGES ON DATABASE` granted, but the role still can't query a table | Database-level privileges don't cascade to schema/table level | Grant schema/table privileges separately if table access is actually required (not needed for this lab) |
| Tempted to `systemctl restart postgresql` after making changes | Misjudging which PostgreSQL changes need a restart | SQL-level changes (roles, databases, grants) take effect immediately; restart is never needed for this lab's scope |
| `\l` output's privilege letters are hard to interpret | Compact notation (`CTc/postgres`) not immediately readable | Decode: `C`=CONNECT, `T`=TEMPORARY, `c`=CREATE, `/grantor`; or just use `has_database_privilege()` for an unambiguous yes/no |
| Unsure whether a privilege grant actually "took" | Only looked at `\l`'s notation, didn't query the privilege engine directly | `SELECT has_database_privilege(...)` for an authoritative answer |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.13) out loud
      from "configure PostgreSQL role/database/grant, no restart" to
      "verified via `has_database_privilege`."
- [ ] Explain, in one sentence, why `sudo -iu postgres` is necessary
      before running `psql` for administrative access, using the term
      "peer authentication."
- [ ] Explain why `CREATE USER` and `CREATE ROLE ... WITH LOGIN` are
      effectively the same operation in PostgreSQL.
- [ ] Explain why granting `ALL PRIVILEGES ON DATABASE` doesn't give a
      role automatic access to query any table inside that database.
- [ ] Explain which category of PostgreSQL change requires a restart,
      and which doesn't — and why this lab's changes fall into the
      "no restart needed" category.
- [ ] Explain why `has_database_privilege()` is stronger verification
      than reading `\l`'s privilege notation by eye.
- [ ] Create a second role and database from memory, grant only
      `CONNECT` (not `CREATE`), and verify both privileges
      independently return the correct `t`/`f`.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, what was broken/required, constraints>

## 2. Reasoning model — how to derive the fix/commands
<walk the requirement down step by step, in the order you'd actually
discover it: confirm running state (don't assume from "installed") →
understand the authentication/permission model connecting the two
layers involved → make the minimal required changes at the correct
privilege scope → verify authoritatively against the system's own
answer, not visual inspection → confirm any explicit constraints (e.g.
no restart) were honored throughout. End with a compressed arrow-chain
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
