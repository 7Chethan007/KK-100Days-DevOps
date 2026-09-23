# Linux File Permissions — A Hands-On Tutorial

This is a **do-it-yourself** tutorial, not just a read. Every section has
commands you actually run in a terminal (any Linux machine, VM, container,
or the KodeKloud jump host works). By the end, `chmod 755 file.sh` should
stop looking like a magic incantation and start looking like arithmetic.

Companion doc: `README.md` in this folder applies these concepts to the
actual KodeKloud lab task.

---

## 0. Set up a sandbox (30 seconds)

Run this once, anywhere you have a shell:

```bash
mkdir -p ~/perms-lab && cd ~/perms-lab
touch demo.sh
ls -l demo.sh
```

You should see something like:

```text
-rw-r--r-- 1 yourname yourname 0 Sep 23 20:00 demo.sh
```

Keep this terminal open — every example below is meant to be typed, not
just read.

---

## 1. The core idea

Linux treats almost everything as a **file**: scripts, configs, logs,
directories, even devices and sockets. Every file has an **owner**, a
**group**, and a set of **permissions** — an access-control label:

```text
                 WHO?
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      owner      group     others
        │         │         │
        └─────────┼─────────┘
                  │
                WHAT?
                  │
             read/write/execute
```

---

## 2. Reading `ls -l` — try it now

```bash
ls -l demo.sh
```

```text
-rw-r--r-- 1 yourname yourname 0 Sep 23 20:00 demo.sh
```

Break apart the first column, character by character:

```text
-  rw-  r--  r--
│   │    │    │
│   │    │    └── others
│   │    └─────── group
│   └──────────── owner
└──────────────── file type (- = regular file, d = directory)
```

So a freshly created file is `rw-r--r--`:

```text
owner   = rw-   (read, write, no execute)
group   = r--   (read only)
others  = r--   (read only)
```

**Try it:** create a directory and `ls -ld` it — notice the first
character changes to `d`:

```bash
mkdir demo-dir
ls -ld demo-dir
```

```text
drwxr-xr-x 2 yourname yourname 4096 Sep 23 20:00 demo-dir
```

---

## 3. The three permissions: `r`, `w`, `x`

| Permission | Symbol | Meaning for a **file** | Meaning for a **directory** |
|---|---|---|---|
| Read    | `r` | view contents          | list contents (`ls`)          |
| Write   | `w` | modify contents        | create/delete/rename entries  |
| Execute | `x` | run it as a program    | enter it (`cd`)                |

**Try it — prove the directory rule to yourself:**

```bash
mkdir locked-dir
chmod 000 locked-dir
cd locked-dir
```

```text
bash: cd: locked-dir: Permission denied
```

That's because `x` (execute on a directory = "permission to enter") was
removed. Restore it:

```bash
chmod 755 locked-dir
cd locked-dir && cd ..
```

Now it works.

---

## 4. The three "who" groups

Every file has exactly three permission slots — **owner**, **group**,
**others** — never more, never fewer.

```bash
ls -l demo.sh
```

```text
-rw-r--r-- 1 yourname yourname 0 Sep 23 20:00 demo.sh
```

```text
             demo.sh

       ┌─────────┬─────────┬─────────┐
       │  owner  │  group  │ others  │
       ├─────────┼─────────┼─────────┤
       │  rw-    │  r--    │  r--    │
       └─────────┴─────────┴─────────┘
```

- **Owner** — the user who created/owns the file (you, right now).
- **Group** — the Unix group tied to the file (your default group).
- **Others** — literally everyone else on the system.

---

## 5. From letters to numbers — derive it yourself

Each permission has a fixed value:

```text
r = 4
w = 2
x = 1
```

Add the values that are "on" in each triplet:

```text
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
-wx = 0+2+1 = 3
-w- = 0+2+0 = 2
--x = 0+0+1 = 1
--- = 0+0+0 = 0
```

**Try it — don't trust the table, derive your own file's number:**

```bash
ls -l demo.sh          # -rw-r--r--
```

```text
owner rw- = 4+2+0 = 6
group r-- = 4+0+0 = 4
others r-- = 4+0+0 = 4

→ 644
```

Check yourself against the machine:

```bash
stat -c '%a %n' demo.sh
```

```text
644 demo.sh
```

If your hand-derived number matches `stat`'s output, you've got it.

---

## 6. `chmod` — change the mode, prove it every time

`chmod` = **change mode**. Syntax:

```bash
chmod <mode> <file>
```

**Try the full cycle yourself:**

```bash
chmod 755 demo.sh
ls -l demo.sh
```

```text
-rwxr-xr-x 1 yourname yourname 0 ... demo.sh
```

Derive `755` before checking:

```text
7 = rwx  (owner)
5 = r-x  (group)
5 = r-x  (others)
```

Now try a few more and predict the output **before** running `ls -l`:

```bash
chmod 700 demo.sh   # predict: owner rwx, group ---, others ---
ls -l demo.sh

chmod 644 demo.sh   # predict: owner rw-, group r--, others r--
ls -l demo.sh

chmod 600 demo.sh   # predict: owner rw-, group ---, others ---
ls -l demo.sh
```

If your prediction matches the real output every time, you've internalized
the arithmetic — that's the actual goal of this tutorial.

---

## 7. Why not just always use `777`?

```bash
chmod 777 demo.sh
ls -l demo.sh
```

```text
-rwxrwxrwx 1 yourname yourname 0 ... demo.sh
```

`777` means **everyone can read, write, and execute** — including
strangers on a shared system. That's almost always more access than a
task actually needs. The habit to build: ask "what does the *requirement*
actually say?" — not "what number makes the error go away?"

- Requirement: *"everyone can run this script"* → needs execute for all →
  `755` (owner keeps write too, since they maintain it).
- Requirement: *"only I can read this password file"* → `600`.
- Requirement: *"everyone can read and write"* → still almost never `777`
  unless execute is genuinely needed too.

---

## 8. The trap: `+x` is not the same as `755`

This is the single most common mistake in permission tasks — reproduce it
yourself so you never fall for it again.

```bash
chmod 000 demo.sh
ls -l demo.sh
```

```text
---------- 1 yourname yourname 0 ... demo.sh
```

Now add execute the "obvious" way:

```bash
chmod a+x demo.sh
ls -l demo.sh
```

```text
---x--x--x 1 yourname yourname 0 ... demo.sh
```

That's `111`, **not** `755`. `a+x` means *"add execute on top of
whatever is already set"* — it does not reset the file to a sane,
conventional mode. Starting from `000`, adding `x` only ever gets you
`111`.

If a task (or an automated checker) expects the conventional executable
mode, set it explicitly instead of incrementally:

```bash
chmod 755 demo.sh
ls -l demo.sh
```

```text
-rwxr-xr-x 1 yourname yourname 0 ... demo.sh
```

**Rule of thumb:** use `+x`/`-x` shorthand when you're *tweaking* an
already-reasonable mode. Use the explicit numeric mode (`755`, `644`,
`600`, ...) when you need a **specific, known-correct end state** —
which is exactly what most lab checkers and production scripts expect.

---

## 9. Symbolic mode — the other syntax

Numeric mode (`755`) sets the whole triplet at once. Symbolic mode adds or
removes individual bits without touching the rest:

```bash
chmod 000 demo.sh          # reset

chmod u+x demo.sh          # u = user/owner
ls -l demo.sh              # ---x------

chmod g+x demo.sh          # g = group
ls -l demo.sh              # ---x--x---

chmod o+x demo.sh          # o = others
ls -l demo.sh              # ---x--x--x

chmod a+x demo.sh          # a = all (same as u+x g+x o+x combined)
```

**Try removing instead of adding:**

```bash
chmod 777 demo.sh
chmod o-w demo.sh          # remove write from others only
ls -l demo.sh
```

```text
-rwxrwxr-x
```

---

## 10. `chown` — a different problem entirely

`chmod` changes **what actions are allowed**. `chown` changes **who owns
the file** — a completely separate axis.

```bash
ls -l demo.sh
```

```text
-rwxr-xr-x 1 yourname yourname 0 ... demo.sh
              │          │
              owner      group
```

Changing ownership (needs root/sudo on most systems):

```bash
sudo chown root demo.sh        # change owner only
sudo chown root:root demo.sh   # change owner AND group
```

Don't reach for `chown` when the actual problem is permissions, and vice
versa — a very common source of wasted time on "access denied" errors is
fixing the wrong axis (e.g. `chmod 777`-ing a file when the real issue was
that the wrong user owned it).

---

## 11. Users and groups — the other half of "owner" and "group"

`chown` lets you assign an owner/group to a file, but that only makes
sense once you understand where users and groups themselves come from.
This section is the missing piece: creating them, and reasoning about who
belongs where.

### Where user/group identity actually lives

```bash
cat /etc/passwd | grep "$(whoami)"
```

```text
yourname:x:1000:1000:yourname:/home/yourname:/bin/bash
```

```text
yourname : x : 1000 : 1000 : yourname : /home/yourname : /bin/bash
   │        │    │      │        │            │               │
 username  pw   UID    GID    comment      home dir         shell
          (in
        /etc/shadow)
```

Every user has a numeric **UID**, and a **primary GID** pointing at their
default group. Groups themselves live in a separate file:

```bash
cat /etc/group | grep "$(whoami)"
```

```text
yourname:x:1000:
```

`id` is the fast way to see both at once:

```bash
id
```

```text
uid=1000(yourname) gid=1000(yourname) groups=1000(yourname),27(sudo),999(docker)
```

Notice **one primary group** (`gid=1000`) but **multiple supplementary
groups** (`groups=...`) — this is the key idea the rest of this section
builds on.

### Creating users and groups yourself

```bash
sudo groupadd devteam
sudo useradd -m -G devteam alice
sudo passwd alice
```

```text
useradd -m         → create the user, with a home directory
        -G devteam → add alice to the supplementary group "devteam"
groupadd devteam    → create the group first (must exist before -G can use it)
```

Verify:

```bash
id alice
```

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),1002(devteam)
```

### Primary group vs. supplementary groups — why the distinction matters

```text
Primary group  → the group stamped on every NEW file alice creates,
                 unless something else overrides it (§ setgid, above)
Supplementary  → additional groups alice belongs to, used for permission
groups           checks against files owned by OTHER groups
```

**Try it — prove primary group determines new-file group ownership:**

```bash
su - alice
touch myfile.txt
ls -l myfile.txt
```

```text
-rw-r--r-- 1 alice alice ... myfile.txt
```

The group is `alice` (her primary group), **not** `devteam`, even though
she's a member of `devteam` too — new files always take the *primary*
group unless a setgid directory (§17) overrides it.

### Adding an existing user to an existing group (the command you'll use most)

```bash
sudo usermod -aG devteam bob
```

**The `-a` is not optional in practice** — it means *append*. Omitting it
replaces bob's entire supplementary group list with just `devteam`,
silently removing him from every other group he was in:

```bash
sudo usermod -G devteam bob      # DANGEROUS: wipes bob's other groups
sudo usermod -aG devteam bob     # SAFE: adds devteam, keeps the rest
```

**Try it — see the difference for yourself:**

```bash
sudo usermod -aG sudo alice
id alice
```

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),1002(devteam),27(sudo)
```

`devteam` membership survived because `-a` was used.

### How group membership decides access — the full evaluation order

```bash
sudo groupadd finance
sudo mkdir /srv/finance-reports
sudo chown root:finance /srv/finance-reports
sudo chmod 770 /srv/finance-reports
```

```text
drwxrwx--- 2 root finance ... /srv/finance-reports
```

The kernel checks access in a strict order, stopping at the first match:

```text
1. Are you the owner (root)?          → apply owner bits
2. Are you in the file's group?       → apply group bits
3. Neither?                           → apply other bits
```

This means: if you're the file's **owner** but somehow have a *weaker*
owner permission than the group permission, you still only get the owner
bits — being the owner never "falls through" to a more generous group or
other permission. Membership is evaluated top-down, first match wins.

**Try it end-to-end:**

```bash
sudo usermod -aG finance carol
su - carol
cd /srv/finance-reports && touch q3.csv     # works: carol is in "finance", dir is 770
```

```bash
su - alice
cd /srv/finance-reports                      # Permission denied: alice isn't in "finance"
```

### Cheat-sheet for this section

```text
groupadd <name>              create a group
useradd -m -G <group> <user> create a user, home dir, add to group
usermod -aG <group> <user>   ADD user to group (never omit -a)
gpasswd -d <user> <group>    remove user from group
id <user>                    show UID, primary GID, all groups
groups <user>                shorter: just list the groups
cat /etc/passwd              user:UID:primaryGID:home:shell
cat /etc/group               group:GID:member,member,...
```

---

## 12. `stat` — remove all ambiguity

`ls -l` is for humans; `stat` gives you the exact number, which is what
you should actually compare against a requirement:

```bash
stat -c '%a %n' demo.sh
```

```text
755 demo.sh
```

**Try it as a self-check habit:** after any `chmod`, immediately run
`stat -c '%a %n'` and compare the number to what you intended — don't
eyeball `rwxr-xr-x` and hope you read it right.

---

## 13. Common modes worth memorizing

| Mode | Permissions   | Typical use |
|---:|---|---|
| `600` | `rw-------` | Private file (SSH private keys, secrets) |
| `644` | `rw-r--r--` | Normal file, world-readable |
| `700` | `rwx------` | Private executable/script |
| `755` | `rwxr-xr-x` | Public executable script/program |
| `777` | `rwxrwxrwx` | Almost always too permissive — avoid |

---

## 14. `umask` — why new files aren't born at `777`

You never see a brand-new file created as `rwxrwxrwx`, even though that's
the "everything on" value. Something is subtracting from it. That's the
**umask**.

```bash
umask
```

```text
0022
```

The umask is a **mask that gets subtracted** from a maximum starting
point:

```text
files start from a max of      666  (rw-rw-rw-, never x by default)
directories start from a max of 777  (rwxrwxrwx)
```

Subtract the umask's bits:

```text
666
-022
----
644   → rw-r--r--   ← matches the demo.sh you created in §0
```

```text
777
-022
----
755   → rwxr-xr-x   ← matches demo-dir from §2
```

**Try it — change the umask and watch new files inherit it:**

```bash
umask 077
touch strict.txt
ls -l strict.txt
```

```text
-rw------- 1 yourname yourname 0 ... strict.txt
```

```text
666 - 077 = 600   → rw-------
```

Reset it back for the rest of this session:

```bash
umask 022
```

**Why this matters:** umask explains *default* permissions — the
starting point before anyone runs `chmod`. Servers, containers, and CI
runners often set a strict umask (`027` or `077`) precisely so that newly
created files/logs aren't accidentally world-readable.

---

## 15. Recursive `chmod` — and the trap inside it

Real tasks rarely involve one file — usually a whole directory tree.

```bash
mkdir -p project/{bin,src}
touch project/src/app.py project/bin/run.sh
chmod -R 755 project
```

`-R` applies the mode to the directory and **everything inside it,
recursively**. This works, but creates a subtle problem:

```bash
ls -l project/src/app.py
```

```text
-rwxr-xr-x 1 yourname yourname 0 ... app.py
```

`app.py` is a plain Python source file — it now has execute permission it
almost certainly doesn't need, just because it happened to live inside a
directory that needed `755`. Blindly recursive numeric `chmod` conflates
"directories need `x` to be enterable" with "files need `x` to be
executable" — they're different requirements.

**The fix — apply execute only where it already made sense, using
`find`:**

```bash
# Directories need x to be traversable — give all dirs 755
find project -type d -exec chmod 755 {} \;

# Regular files usually don't need execute — give all files 644
find project -type f -exec chmod 644 {} \;

# Then explicitly re-add execute only to the scripts that need it
chmod 755 project/bin/run.sh
```

**Even simpler — symbolic mode has a dedicated flag for exactly this
problem:**

```bash
chmod -R u+rwX,g+rX,o+rX project
```

The capital **`X`** (different from lowercase `x`!) means *"add execute
only if it's a directory, or if execute is already set for someone on
this file."* This is the standard safe way to recursively fix permissions
without accidentally making every file executable.

---

## 16. Folder-level permissions — apply to everything, or only to some

This is the scenario that trips people up most in real infrastructure
work: a requirement says *"fix permissions on this folder,"* but it never
means literally every file identically. Below are the concrete patterns
for each version of that requirement.

### Scenario A — the folder itself vs. its contents (two different things)

```bash
mkdir -p webroot/{public,private}
touch webroot/public/index.html webroot/private/secrets.env
chmod 700 webroot
ls -ld webroot
```

```text
drwx------ 4 yourname yourname ... webroot
```

This changes **only the folder's own entry** — whether people can `cd`
into it or list it at all. It does **not** touch anything already inside:

```bash
ls -l webroot/public/index.html
```

```text
-rw-r--r-- 1 yourname yourname ... index.html    ← unchanged
```

Locking the folder to `700` still blocks *everyone else* from entering it
at all, regardless of the file's own permissive `644` — a parent
directory's `x` bit is a gate that must be passed before a file's own
permissions are even checked. **Rule:** the most restrictive permission
anywhere along the path wins.

### Scenario B — apply to literally everything inside, recursively

```bash
chmod -R 755 webroot
```

Covered in §15 already — the catch is it doesn't distinguish files from
directories, or file types from each other (see the capital-`X` fix
there).

### Scenario C — apply to only files matching a pattern (e.g. only `.sh` files)

```bash
mkdir -p scripts && touch scripts/deploy.sh scripts/backup.sh scripts/notes.txt
find scripts -type f -name "*.sh" -exec chmod 755 {} \;
ls -l scripts
```

```text
-rwxr-xr-x 1 yourname yourname ... backup.sh
-rwxr-xr-x 1 yourname yourname ... deploy.sh
-rw-r--r-- 1 yourname yourname ... notes.txt   ← untouched, doesn't match *.sh
```

`find <dir> -name "<pattern>"` narrows *which files* get touched before
`chmod` ever runs — this is the standard way to answer *"make only the
scripts executable, leave everything else alone."*

### Scenario D — apply to only some subdirectories, not others

```bash
mkdir -p app/{public,internal}
chmod 755 app/public
chmod 750 app/internal
ls -ld app/public app/internal
```

```text
drwxr-xr-x 2 yourname yourname ... app/public
drwxr-x--- 2 yourname yourname ... app/internal
```

No recursion needed here — just target each subdirectory explicitly. The
mistake to avoid is running one `chmod -R` on `app` and assuming it
handles both cases; a single recursive mode can't express "different
rules for different subfolders."

### Scenario E — new files should *automatically* inherit the folder's group (ongoing, not one-time)

This is the "reflected to all files, including ones that don't exist yet"
version of the requirement — a single `chmod`/`chown` only fixes files
that already exist *right now*.

```bash
sudo groupadd webteam
sudo mkdir -p /srv/shared-site
sudo chown :webteam /srv/shared-site
sudo chmod 2775 /srv/shared-site        # note the leading 2 = setgid, from §17
```

```bash
sudo -u alice touch /srv/shared-site/new-page.html
ls -l /srv/shared-site/new-page.html
```

```text
-rw-r--r-- 1 alice webteam ... new-page.html
```

Every future file created by anyone inherits group `webteam` automatically
— no repeated `chown` needed. This is the setgid directory pattern from
§17, applied to solve a "should apply going forward" requirement rather
than a one-time fix.

### Scenario F — new files should inherit specific *permissions*, not just group (via default ACLs)

Setgid only propagates *group ownership*. To make new files automatically
get a *specific permission mode* too, use a **default ACL**:

```bash
sudo setfacl -d -m u::rwx,g::rwx,o::rx /srv/shared-site
```

```bash
sudo -u alice touch /srv/shared-site/another-file.txt
getfacl /srv/shared-site/another-file.txt
```

```text
# file: another-file.txt
user::rw-
group::rwx
other::r-x
```

`-d` sets a **default** ACL entry on the directory — it doesn't change
the directory's own permissions, it defines what gets stamped onto
anything created inside it from now on. This is the precise answer to
"how do I make sure this rule applies to all files in this folder, even
ones added later" — plain `chmod -R` only ever fixes files that exist at
the moment you run it.

### Decision table — which mechanism answers which requirement

| Requirement phrasing                                              | Mechanism                          |
|---------------------------------------------------------------------|-------------------------------------|
| "Fix this one folder's own access"                                 | `chmod <mode> folder` (no `-R`)     |
| "Fix everything inside, files and dirs the same way"                | `chmod -R <mode> folder`            |
| "Fix everything, but dirs need `x`, files shouldn't"               | `chmod -R u+rwX,g+rX,o+rX folder`   |
| "Only files matching a pattern"                                     | `find folder -name "<pattern>" -exec chmod ...` |
| "Different rules per subfolder"                                     | separate explicit `chmod` calls, no recursion |
| "New files should share this folder's group automatically"         | setgid bit — `chmod g+s folder` (§17) |
| "New files should get a specific permission mode automatically"    | default ACL — `setfacl -d -m ...`   |
| "Specific user needs access beyond owner/group/others"              | ACL — `setfacl -m u:name:perm file` (§19) |

---

## 17. Special permission bits — setuid, setgid, sticky bit

Beyond `rwx`, there's a fourth, less common set of bits. You'll recognize
them because `ls -l` shows an `s` or `t` in place of `x`.

### setuid (`s` in the owner's execute slot) — "run as the file's owner"

```bash
ls -l /usr/bin/passwd
```

```text
-rwsr-xr-x 1 root root ... /usr/bin/passwd
     │
     └── setuid bit
```

Normally, a program runs with *your* privileges. `passwd` needs to write
to `/etc/shadow`, which ordinary users can't touch. **setuid** makes the
program run with the *file owner's* privileges (`root`) instead of the
caller's — that's how a normal user can change their own password without
being root themselves.

Numerically, setuid adds **4000** to the mode:

```bash
chmod 4755 myprogram      # rwsr-xr-x
```

### setgid (`s` in the group's execute slot) — two different effects

On an **executable**: same idea as setuid, but runs as the file's
**group** instead of owner.

On a **directory**: far more commonly useful — new files created inside
inherit the *directory's* group, not the creating user's group.

```bash
mkdir shared-team-dir
chmod 2775 shared-team-dir     # setgid + rwxrwxr-x
ls -ld shared-team-dir
```

```text
drwxrwsr-x 2 yourname yourname ... shared-team-dir
        │
        └── setgid bit on a directory
```

```bash
touch shared-team-dir/newfile.txt
ls -l shared-team-dir/newfile.txt
```

The new file's group will match `shared-team-dir`'s group automatically —
useful for shared project directories where multiple users need
consistent group ownership without everyone remembering to `chgrp`.

### Sticky bit (`t` in others' execute slot) — "you can only delete your own files"

```bash
ls -ld /tmp
```

```text
drwxrwxrwt 12 root root ... /tmp
```

`/tmp` is world-writable (`777`) so any user can create files there — but
without the sticky bit, any user could also **delete or rename anyone
else's** files in `/tmp`, since directory write permission controls entry
deletion (§3). The sticky bit changes that rule: **inside a
sticky-bit directory, you may only delete/rename files you own**,
regardless of directory write permission.

```bash
chmod 1777 shared-drop-dir      # sticky + rwxrwxrwx
ls -ld shared-drop-dir
```

```text
drwxrwxrwt
```

### The full four-digit picture

```text
chmod 4755 file    → setuid + rwxr-xr-x
chmod 2755 dir     → setgid + rwxr-xr-x
chmod 1777 dir     → sticky + rwxrwxrwx

     4    =  setuid
     2    =  setgid
     1    =  sticky
     0    =  none (the usual case — this is why modes are normally 3 digits, not 4)
```

**Security note:** setuid/setgid on writable, world-executable binaries is
a classic privilege-escalation vector — it's why security audits
specifically hunt for unexpected setuid files:

```bash
find / -perm -4000 -type f 2>/dev/null
```

---

## 18. Symbolic links and permissions — a common source of confusion

```bash
touch target.txt
chmod 600 target.txt
ln -s target.txt link.txt
ls -l link.txt
```

```text
lrwxrwxrwx 1 yourname yourname 10 ... link.txt -> target.txt
```

A symlink's own permissions (`rwxrwxrwx`) are **cosmetic and ignored on
Linux** — the kernel always checks the permissions of the file the link
**points to**, not the link itself. So even though `link.txt` shows
`777`, opening it for writing is still governed by `target.txt`'s `600`:

```bash
echo "test" > link.txt   # works only if target.txt permissions allow it
```

**Practical implication:** `chmod` on a symlink, by default, actually
changes the *target's* permissions, not the link's:

```bash
chmod 644 link.txt
ls -l target.txt
```

```text
-rw-r--r-- 1 yourname yourname ... target.txt   ← changed, not the link
```

To affect the link itself (rare, and not supported on all filesystems),
you'd need `chmod -h`, which most Linux systems actually reject for
symlinks since Linux doesn't store real permissions on links at all.

---

## 19. Access Control Lists (ACLs) — beyond owner/group/others

The classic three permission slots have a hard limit: you can only name
*one* owner and *one* group per file. What if you need "user `alice`
specifically gets read-write, everyone else in the team gets read-only"?
That's what **ACLs** are for.

```bash
touch report.csv
chmod 640 report.csv

# Grant an additional, specific user read+write, beyond owner/group/others:
setfacl -m u:alice:rw report.csv

getfacl report.csv
```

```text
# file: report.csv
# owner: yourname
# group: yourname
user::rw-
user:alice:rw-
group::r--
mask::rw-
other::---
```

Notice `ls -l` now shows a trailing `+`:

```bash
ls -l report.csv
```

```text
-rw-rw----+ 1 yourname yourname ... report.csv
```

That `+` is your signal that there's more to the permission story than
the plain `rwxrwxrwx` string shows — always check `getfacl` when you see
it.

Remove an ACL entry:

```bash
setfacl -x u:alice report.csv
```

**Why this matters beyond the lab:** production Linux servers, NFS
shares, and CI artifact storage often rely on ACLs for fine-grained access
that plain `chmod` can't express — knowing the `+` symbol exists means you
won't be puzzled the first time you see it in the wild.

---

## 20. `chattr` — attributes `chmod` can't touch

There's a permission-adjacent tool that operates *below* the normal
`rwx` model entirely: filesystem attributes.

```bash
sudo touch protected.txt
sudo chattr +i protected.txt      # 'i' = immutable
```

```bash
rm protected.txt
```

```text
rm: cannot remove 'protected.txt': Operation not permitted
```

Even as **root**, with `777` permissions, the file cannot be deleted,
renamed, or modified while the immutable attribute is set — `chattr`
operates at a layer `chmod`/`chown` don't reach. Check it with:

```bash
lsattr protected.txt
```

```text
----i---------e------- protected.txt
```

Undo it:

```bash
sudo chattr -i protected.txt
rm protected.txt
```

**Why this matters:** if you ever hit `Operation not permitted` on a file
where `ls -l` shows permissions that *should* allow the action, and
`chmod`/`chown` don't fix it, `lsattr` is the next thing to check.

---

## 21. Finding files by permission — auditing at scale

Real infrastructure work usually means checking permissions across
thousands of files, not one at a time.

```bash
# Find everything world-writable (a common security red flag)
find / -perm -o+w -type f 2>/dev/null

# Find files with permissions EXACTLY 777
find / -perm 777 2>/dev/null

# Find setuid binaries (privilege-escalation audit, §17)
find / -perm -4000 2>/dev/null

# Find files NOT owned by root in a system directory (suspicious)
find /etc -not -user root 2>/dev/null
```

`-perm -MODE` (with a leading dash) means *"at least these bits are set"*
— useful for "is this too open" checks. `-perm MODE` (no dash) means
*"exactly this mode"* — useful for finding an exact known-bad state.

---

## 22. Permissions vs. security — the full picture

`chmod`/`chown`/ACLs answer "who can do what to this file," but real
access control on a running system involves more layers stacked on top:

```text
Can this request reach the file at all?
        ↓
   Network/firewall rules
        ↓
   Process/user identity (who is running this?)
        ↓
   Filesystem permissions (rwx, ACLs)   ← what this doc covers
        ↓
   Mandatory access control (SELinux / AppArmor)
        ↓
   Application-level authorization (does the app's own logic allow it?)
```

`chmod 777` is sometimes reached for as a shortcut to make an error go
away — but it's rarely the *actual* fix, and it silently disables the
filesystem layer of defense for everyone, forever, until someone notices
and reverts it. The habit worth keeping: when you hit a permission error,
find out **which layer** is actually blocking you (a `chmod`? an
SELinux context? the wrong user entirely?) before reaching for the widest
possible fix.

---

## 23. Self-test — do these without looking above

Work through these in your sandbox (`~/perms-lab`). Predict the answer
first, then run the command to check yourself.

1. What numeric mode is `rw-rw-r--`? Verify with `chmod` + `stat`.
2. Set `demo.sh` so only the owner can read, write, and execute it, and
   nobody else can do anything. What's the number?
3. Starting from `644`, what single symbolic command adds execute for the
   owner only (not group, not others)?
4. Why does `chmod a+x` on a `000` file **not** produce `755`? Explain in
   one sentence.
5. Create a directory, `chmod 000` it, and try `ls` inside it vs `cd`
   into it. Which bit controls which action?
6. What's the difference between what `chmod 755 file` and
   `chmod +x file` will do to a file that starts at `640`? (Hint: they're
   not the same — work out both by hand.)
7. Your umask is `0022`. What permissions will a brand-new file get?
   What about a brand-new directory? Verify both with `touch`/`mkdir`.
8. You run `chmod -R 755` on a directory containing both scripts and
   plain data files. What's the problem with this, and what command
   fixes it without re-auditing every file by hand?
9. What does the setgid bit do differently on a directory versus on an
   executable file?
10. You `chmod 600` a file, then create a symlink to it and `chmod 777`
    the symlink. Can another user now write to the original file through
    the symlink? Why or why not?
11. `ls -l` shows `-rw-r-----+` on a file. What does the trailing `+`
    tell you, and what command shows the full picture?
12. `chmod 777` a file, then find it still can't be deleted even by
    root. Permissions aren't the cause — what tool would you check next,
    and what's the specific attribute to look for?

---

## 24. The habit that matters more than the numbers

For any permissions task, resist jumping straight to `chmod`:

```text
1. Inspect   — ls -l  /  stat -c '%a %n' <file>
2. Reason    — what does the requirement actually say (who needs what)?
3. Act       — chmod <derived-mode> <file>
4. Verify    — ls -l  /  stat -c '%a %n' <file>  — does it match?
```

This four-step rhythm (inspect → reason → act → verify) generalizes far
beyond `chmod` — it's the same shape you'll use for ownership, ACLs,
firewall rules, IAM policies, and most other "current state → desired
state" changes in DevOps.
