# Lab 9 — Local Account & Group Management

This lab covers CompTIA Linux+ XK0-006 Objective 2.2 (Given a scenario, manage local user and group accounts). You will create users and groups, modify shells and passwords, inspect identity databases, and differentiate user, system, and service accounts.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Inspect account databases

```bash
head -5 /etc/passwd
head -5 /etc/shadow
head -5 /etc/group
ls /etc/skel -la
head -20 /etc/profile
```

`/etc/passwd` holds account metadata, `/etc/shadow` stores hashed passwords (root-only), and `/etc/skel` provides the template files copied into each new user's home directory.

---

## Step 2 — Create users and groups

```bash
groupadd developers
useradd -m -s /bin/bash -G developers alice
adduser --disabled-password --gecos "" bob
usermod -aG developers bob
id alice
groups bob
```

`useradd` is the low-level tool; `adduser` is the friendlier Debian wrapper. `usermod -aG` appends supplementary groups without dropping existing ones (omit `-a` and you replace them).

---

## Step 3 — Passwords and aging

```bash
echo "alice:Passw0rd!" | chpasswd
passwd -S alice
chage -l alice
chage -M 90 -W 7 alice
chage -l alice
```

`chage` controls password aging policy: `-M` sets maximum age in days and `-W` sets the warning window before expiration.

---

## Step 4 — Change shell and identify accounts

```bash
chsh -s /bin/sh bob
getent passwd alice bob
whoami
id
sudo -u alice id
```

`chsh` updates the login shell field in `/etc/passwd`. `sudo -u alice id` shows the effective UID (EUID) when running as another user — note how UID, GID, and groups change.

---

## Step 5 — User vs system vs service accounts

```bash
awk -F: '$3 >= 1000 && $3 < 65534 {print "USER  ", $1, $3}' /etc/passwd
awk -F: '$3 < 1000 {print "SYSTEM", $1, $3}' /etc/passwd | head
getent passwd www-data
useradd -r -s /usr/sbin/nologin svc_app
getent passwd svc_app
```

Regular users have UID >= 1000, system accounts use UID < 1000, and service accounts (created with `-r`) typically use `nologin` to prevent interactive login.

---

## Step 6 — Login history and active sessions

```bash
last | head
lastlog | head
who
w
```

`last` reads `/var/log/wtmp` for historical logins, `lastlog` shows the most recent login per user, and `w` / `who` display sessions currently attached.

---

## Step 7 — Remove users and groups

```bash
userdel -r bob
userdel -r svc_app
groupdel developers
getent passwd bob || echo "bob removed"
```

`userdel -r` removes the account and its home directory plus mail spool. `groupdel` fails if the group is still a user's primary group, so remove users first.

---

## Step 8 — Cleanup

```bash
userdel -r alice 2>/dev/null
getent passwd alice || echo "alice removed"
```

The remaining test user is removed along with their home directory.

---

## What you learned
- Creating and modifying local users and groups with `useradd`, `usermod`, `chage`
- Reading `/etc/passwd`, `/etc/shadow`, `/etc/group`, and `/etc/skel`
- Differentiating user (UID>=1000), system (UID<1000), and service accounts

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- shadow-utils — https://github.com/shadow-maint/shadow
