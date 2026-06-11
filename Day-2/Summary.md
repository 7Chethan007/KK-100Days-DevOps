# Deep Dive: Understanding SSH, Users, useradd, and chage

## The Bigger Picture

When we worked on the Nautilus task, we weren't simply creating a user.

What we actually did was:

1. Connected to a remote machine.
2. Authenticated ourselves.
3. Became an administrator.
4. Created a new identity inside Linux.
5. Configured when that identity should stop existing.
6. Verified the configuration.

Understanding these concepts is more important than remembering the commands.

---

# Understanding Host, Username, and Password

Consider this command:

```bash
ssh steve@stapp02
```

Most beginners memorize it.

Instead, let's understand what is happening.

Think about a company office building.

```text
Building = Server
Employee = User
Security Desk = SSH Service
```

Suppose you want to enter a company building.

The security guard asks:

```text
Who are you?
```

You reply:

```text
Steve
```

The guard then asks:

```text
Prove it.
```

You provide your badge or password.

Linux works exactly the same way.

---

# What is a Host?

A host is simply a machine connected to a network.

Examples:

```text
Laptop
Server
Database Server
Virtual Machine
EC2 Instance
```

In our lab:

```text
Hostname:
stapp02
```

This is simply the machine's name.

Think:

```text
Hostname = House Address
```

---

# What is a User?

Linux is a multi-user operating system.

Multiple people can use the same server.

Examples:

```text
root
steve
mariyam
ubuntu
ec2-user
```

Each user gets:

* Identity
* Permissions
* Home directory
* Password

Think:

```text
User = Employee Badge
```

---

# Understanding SSH

SSH stands for:

```text
Secure Shell
```

Purpose:

```text
Connect securely to another machine.
```

Without SSH:

```text
You sit physically near server.
```

With SSH:

```text
You manage servers from anywhere.
```

---

# What Happens Internally During SSH?

When you type:

```bash
ssh steve@stapp02
```

SSH performs roughly these steps:

### Step 1

Find server.

```text
Where is stapp02?
```

DNS or host configuration resolves it to an IP.

Example:

```text
10.244.240.174
```

---

### Step 2

Connect to SSH service.

Default SSH port:

```text
22
```

Think:

```text
Knocking on door number 22.
```

---

### Step 3

Verify host identity.

SSH asks:

```text
Can I trust this server?
```

That's why you saw:

```text
The authenticity of host can't be established
```

SSH is preventing man-in-the-middle attacks.

---

### Step 4

Authentication begins.

SSH asks:

```text
Are you really Steve?
```

You provide:

```text
Am3ric@
```

---

### Step 5

Server checks.

Linux looks inside:

```text
/etc/shadow
```

Password hashes are stored there.

If password matches:

```text
Access Granted
```

---

# Why ssh user@host?

Because Linux needs two things.

### Who?

```text
steve
```

### Where?

```text
stapp02
```

Without both pieces of information SSH cannot work.

Think:

```text
Deliver Package To:
Steve
Building 2
```

---

# Understanding sudo su -

Command:

```bash
sudo su -
```

Actually combines two commands.

### sudo

Means:

```text
Execute as administrator.
```

### su

Means:

```text
Switch User.
```

### -

Means:

```text
Load target user's environment.
```

So:

```bash
sudo su -
```

really means:

```text
Switch completely to root user.
```

---

# Understanding useradd

Command:

```bash
useradd mariyam
```

Looks simple.

But Linux does a lot internally.

---

# What Happens Internally?

Linux updates:

```text
/etc/passwd
```

Adds:

```text
mariyam:x:1001:1001::/home/mariyam:/bin/bash
```

---

Creates:

```text
UID
```

Example:

```text
1001
```

Unique User ID.

---

Creates:

```text
Primary Group
```

Example:

```text
mariyam
```

---

Creates:

```text
Home Directory
```

Typically:

```text
/home/mariyam
```

(if configured to do so)

---

# Understanding useradd -e

Command:

```bash
useradd -e 2027-03-28 mariyam
```

Option:

```text
-e
```

Means:

```text
Expiry Date
```

Think:

```text
Temporary Employee
```

After:

```text
2027-03-28
```

Linux automatically disables account access.

---

# Other Useful useradd Variations

## Create User with Home Directory

```bash
useradd -m mariyam
```

Creates:

```text
/home/mariyam
```

---

## Create User with Custom Home

```bash
useradd -d /data/mariyam mariyam
```

---

## Create User with Specific Shell

```bash
useradd -s /bin/bash mariyam
```

---

## Create User with UID

```bash
useradd -u 2001 mariyam
```

---

## Create User with Group

```bash
useradd -g developers mariyam
```

---

## Create User and Expiry Together

```bash
useradd -m -s /bin/bash -e 2027-03-28 mariyam
```

Production environments often use combinations like this.

---

# Understanding chage

Command:

```bash
chage -l mariyam
```

Many people think:

```text
Change Age?
```

Which is actually close.

It stands for:

```text
Change Aging Information
```

---

# What Does Aging Mean?

Linux tracks:

```text
Password age
Password expiry
Account expiry
Warning period
```

---

# Why Does Linux Need This?

Imagine a company.

Someone leaves.

Should their account remain active forever?

No.

Linux therefore tracks:

```text
When account expires.
```

and

```text
When password expires.
```

---

# Understanding chage -l

Command:

```bash
chage -l mariyam
```

Option:

```text
-l
```

Means:

```text
List Information
```

Output:

```text
Password expires
Account expires
Warning period
```

Think:

```text
Employee Account Report
```

---

# Useful chage Commands

## Set Password Expiry

```bash
chage -M 90 mariyam
```

Password expires after:

```text
90 days
```

---

## Force Password Change

```bash
chage -d 0 mariyam
```

User must change password on next login.

---

## Set Account Expiry

```bash
chage -E 2027-03-28 mariyam
```

Same idea as:

```bash
useradd -e
```

but modifies an existing user.

---

## View Aging Information

```bash
chage -l mariyam
```

Most common verification command.

---

# Why id mariyam Works

Command:

```bash
id mariyam
```

Linux asks:

```text
Does user exist?
```

If yes:

```text
Show UID
Show GID
Show Groups
```

Output:

```text
uid=1001(mariyam)
gid=1001(mariyam)
groups=1001(mariyam)
```

Think:

```text
Employee Identity Card
```

---

# The Mental Model You'll Remember

Whenever you see:

```bash
ssh steve@stapp02
```

Think:

```text
Connect as Steve to Building stapp02
```

Whenever you see:

```bash
useradd
```

Think:

```text
Create a new employee identity inside Linux
```

Whenever you see:

```bash
useradd -e
```

Think:

```text
Create a temporary employee
```

Whenever you see:

```bash
chage -l
```

Think:

```text
Show employee account lifecycle information
```

Whenever you see:

```bash
id mariyam
```

Think:

```text
Show employee identity card
```

Once you see Linux as a company managing employees and buildings, these commands stop being things to memorize and start becoming things that make sense.
