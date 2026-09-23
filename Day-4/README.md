# Day 4 — Script Execution Permissions

A KodeKloud "100 Days DevOps" lab runbook. For the underlying concept —
`r`/`w`/`x`, why `chmod 755` and not `777`, why `+x` isn't the same as
`755`, hands-on exercises — read **`files-permission.md`** in this folder
first. This file is the applied runbook for the actual task.

---

## 1. Task

> In a bid to automate backup processes, the xFusionCorp Industries
> sysadmin team has developed a new bash script named `xfusioncorp.sh`.
> While the script has been distributed to all necessary servers, it
> lacks executable permissions on **App Server X** within the Stratos
> Datacenter.
>
> Your task is to grant executable permissions to `/tmp/xfusioncorp.sh`
> on App Server X. Additionally, ensure that all users have the
> capability to execute it.

> **Note on the server number:** this lab randomizes which app server is
> targeted per attempt. The first run below was against **App Server 2**
> (`stapp02`). After submitting too early, the lab reset and reopened
> asking for **App Server 1** (`stapp01`) instead — same task, same
> commands, different host. Always re-read the task text for the current
> attempt's server number rather than assuming it matches a previous run.

---

## 2. Inspect — find the target host

From the jump host:

```bash
thor@jump-host ~$ cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
fe00::0 ip6-mcastprefix
fe00::1 ip6-allnodes
fe00::2 ip6-allrouters
10.244.226.145  jump-host

# Entries added by HostAliases.
10.0.15.5       docker-registry-mirror.kodekloud.com

thor@jump-host ~$ grep -i "app" /etc/hosts
(no output)
```

`/etc/hosts` doesn't list app servers by name in this environment — the
inventory (`stapp01`, `stapp02`, `stapp03`, ...) resolves through an
internal mechanism, not static hosts entries. The task text itself names
the target server number — connect directly using the standard KodeKloud
naming convention (`stapp0<N>`) rather than relying on `/etc/hosts`.

---

## 3. Connect to the target server

```bash
thor@jump-host ~$ ssh steve@stapp02
steve@stapp02's password:
[steve@stapp02 ~]$ sudo su
[sudo] password for steve:
[root@stapp02 steve]#
```

### Why?
The jump host is only the entry point. The task targets a specific app
server, so the file must be inspected and modified **on that machine**,
not on the jump host.

---

## 4. Inspect before changing

```bash
[root@stapp02 steve]# ls -l /tmp/xfusioncorp.sh
---------- 1 root root 40 Sep 23 16:03 /tmp/xfusioncorp.sh
```

`----------` → `000`: nobody — not owner, not group, not others — has
read, write, or execute through the normal permission bits.

---

## 5. Reason — determine the required state

The task says:

> ensure that all users have the capability to execute it.

Required:

```text
owner  → rwx   (root still needs full control)
group  → r-x
others → r-x
```

Target mode: `755`.

Do **not** use `chmod a+x` here starting from `000` — that produces `111`
(`---x--x--x`), not `755`. See `files-permission.md` §8 for why.

---

## 6. Implement

```bash
[root@stapp02 steve]# chmod 755 /tmp/xfusioncorp.sh
```

```text
000              ----------
 ↓        i.e.        ↓
755              -rwxr-xr-x
```

---

## 7. Verify

```bash
[root@stapp02 steve]# ls -l /tmp/xfusioncorp.sh
-rwxr-xr-x 1 root root 40 Sep 23 16:13 /tmp/xfusioncorp.sh

[root@stapp02 steve]# stat -c '%a %n' /tmp/xfusioncorp.sh
755 /tmp/xfusioncorp.sh
```

```text
expected → 755
actual   → 755   ✔
```

---

## 8. Final state

```text
stapp0<N> (App Server per current attempt)
  │
  └── /tmp/xfusioncorp.sh
       │
       ├── owner:  rwx
       ├── group:  r-x
       └── others: r-x
```

## 9. Submit the lab
Once `stat -c '%a %n' /tmp/xfusioncorp.sh` reports `755` on the **correct,
currently-assigned** app server, click **Check** in the KodeKloud lab UI —
don't submit before confirming the server number in the task text.

---

## 10. The DevOps habit to build

```text
1. Read the task text for the current server number  → don't assume from a prior run
2. Does the file exist?                               → ls -l /path/file
3. Current permissions?                                → stat -c '%a %n' file
4. What does the task require?                         → owner/group/others needed
5. Smallest correct change                             → chmod <mode> file (explicit, not +x from 000)
6. Verify                                               → ls -l && stat
7. Confirm/submit
```

**Inspect → Reason → Implement → Verify** — see `files-permission.md` for
the full concept tutorial and hands-on exercises.
