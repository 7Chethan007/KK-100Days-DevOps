# Day 2 - Create a Temporary User with Expiry Date

## Task

As part of the temporary assignment to the Nautilus project, a developer named **mariyam** requires access for a limited duration.

### Requirements

* Create a user named `mariyam` on **App Server 2**.
* Ensure the username is in lowercase.
* Set the account expiry date to **2027-03-28**.

---

# Understanding the Environment

Before performing the task, we clicked **"Details of all Users and Servers"** in the KodeKloud Engineer lab.

From the Infrastructure Details page, we identified:

| Server               | Hostname  | User  | Password   |
| -------------------- | --------- | ----- | ---------- |
| Application Server 2 | stapp02   | steve | Am3ric@    |
| Jump Host            | jump-host | thor  | mjolnir123 |

Since the task specifically mentioned **App Server 2**, we needed to connect to the server with hostname **stapp02** using the provided credentials.

---

# Steps Performed

## 1. Connect to App Server 2

From the Jump Host:

```bash
ssh steve@stapp02
```

### First Connection Prompt

```text
The authenticity of host 'stapp02 (10.244.240.174)' can't be established.
ED25519 key fingerprint is SHA256:XSQhw1/nHOvFEE+NbJ/R0DLXHcjz9TOseAa1KrGMVW4.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

### Why This Appears

This message appears because this is the first SSH connection to the server. SSH has not seen this host before and asks for confirmation.

We typed:

```text
yes
```

SSH then stored the server fingerprint in the `known_hosts` file for future connections.

---

## 2. Authenticate as steve

```bash
steve@stapp02's password:
```

Entered password:

```text
Am3ric@
```

Successfully logged into App Server 2.

---

## 3. Switch to Root User

```bash
sudo su -
```

### Explanation

* `sudo` = Execute command with elevated privileges.
* `su` = Switch user.
* `-` = Load root user's environment.

When prompted, entered Steve's password:

```text
Am3ric@
```

After successful authentication:

```bash
[root@stapp02 ~]#
```

This indicates we now have root access.

---

## 4. Create the User with Expiry Date

Command:

```bash
useradd -e 2027-03-28 mariyam
```

### Explanation

* `useradd` = Creates a new Linux user.
* `-e` = Specifies the account expiration date.
* `2027-03-28` = Expiry date.
* `mariyam` = Username to create.

Result:

A new user named `mariyam` was created and configured to expire on March 28, 2027.

---

## 5. Verify Expiry Date

Command:

```bash
chage -l mariyam
```

### Explanation

* `chage` = Change user password expiry information.
* `-l` = List account aging information.

Output:

```text
Account expires : Mar 28, 2027
```

This confirms the expiry date was configured correctly.

---

## 6. Verify User Creation

Command:

```bash
id mariyam
```

### Explanation

The `id` command displays:

* User ID (UID)
* Group ID (GID)
* Group memberships

Output:

```text
uid=1001(mariyam) gid=1001(mariyam) groups=1001(mariyam)
```

This confirms that the user was successfully created.

---

# Commands Used

```bash
ssh steve@stapp02

sudo su -

useradd -e 2027-03-28 mariyam

chage -l mariyam

id mariyam
```

---

# Validation

### Verify User Exists

```bash
id mariyam
```

Expected Output:

```text
uid=1001(mariyam)
```

### Verify Expiry Date

```bash
chage -l mariyam
```

Expected Output:

```text
Account expires : Mar 28, 2027
```

---

# Key Learnings

* How to identify target servers using KodeKloud Infrastructure Details.
* How SSH host key verification works.
* How to connect to remote Linux servers.
* How to obtain root privileges using `sudo`.
* How to create Linux users using `useradd`.
* How to set account expiration dates using the `-e` option.
* How to verify user account details using `chage` and `id`.

---

## Outcome

Successfully connected to **App Server 2 (stapp02)**, created the user **mariyam**, configured the account to expire on **2027-03-28**, and verified the configuration.
