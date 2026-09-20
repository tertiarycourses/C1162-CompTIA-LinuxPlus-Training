# Lab 14 — AAA: sudo, PAM, and Polkit

This lab maps to Linux+ XK0-006 Objective 3.1 (authentication, authorization, accounting). You will install and configure `sudo`, create a privileged user, inspect PAM stacks for SSH and local auth, review auth logs via journald and rsyslog, configure `logrotate`, and turn on basic `auditd` rules. We finish with a conceptual look at centralized identity (LDAP, Kerberos, SSSD, Samba).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install sudo and create a user

```bash
apt update && apt install -y sudo
useradd -m -s /bin/bash alice
echo "alice:Passw0rd!" | chpasswd
id alice
```

The user `alice` now exists with a home directory and bash shell. We will grant her elevated rights in the next step.

---

## Step 2 — Grant sudo via the sudo group and a drop-in file

```bash
usermod -aG sudo alice
EDITOR=tee visudo -f /etc/sudoers.d/lab <<'EOF'
alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *, /usr/bin/apt update
EOF
chmod 440 /etc/sudoers.d/lab
visudo -c
sudo -l -U alice
```

`visudo` validates syntax before saving. The drop-in grants `alice` two NOPASSWD commands without editing the master `/etc/sudoers`.

---

## Step 3 — Switch identity with sudo and su

```bash
su - alice -c 'sudo -n systemctl status ssh || true'
su - alice -c 'whoami; sudo -i whoami'
```

`sudo -i` runs an interactive root login shell; `su -` becomes another user with a full login environment. Both are common ways to escalate.

---

## Step 4 — Inspect PAM stacks

```bash
ls /etc/pam.d/
grep -vE '^\s*#|^$' /etc/pam.d/sshd
grep -vE '^\s*#|^$' /etc/pam.d/common-auth
apt show sssd 2>/dev/null | head -n 20
```

PAM (Pluggable Authentication Modules) chains `auth`, `account`, `password`, and `session` modules. `sssd` is the daemon that integrates Linux with LDAP/Kerberos/AD; `apt show` previews the package without installing.

---

## Step 5 — Read auth logs via journald and rsyslog

```bash
apt install -y rsyslog
systemctl enable --now rsyslog
journalctl _COMM=sshd --no-pager | tail -n 20 || true
tail -n 20 /var/log/auth.log 2>/dev/null || journalctl -u ssh --no-pager | tail
logrotate -d /etc/logrotate.conf 2>&1 | tail -n 20
```

`journalctl _COMM=sshd` filters by process name. `logrotate -d` performs a dry-run showing what would rotate without changing any files.

---

## Step 6 — Enable auditd and watch /etc/passwd

```bash
apt install -y auditd audispd-plugins
systemctl enable --now auditd
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -l
echo "# test comment" >> /etc/passwd
ausearch -k passwd_changes | tail -n 20
```

The watch records write (`w`) and attribute (`a`) changes to `/etc/passwd`. `ausearch -k` queries events by key.

---

## Step 7 — Persistent audit rules and Polkit note

```bash
cat >/etc/audit/rules.d/lab.rules <<'EOF'
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
EOF
augenrules --load
auditctl -l
which pkexec && pkexec --version
```

Rules in `/etc/audit/rules.d/` survive reboot. `pkexec` is the Polkit equivalent of sudo for desktop and D-Bus services.

---

## Step 8 — Cleanup

```bash
auditctl -D
rm -f /etc/audit/rules.d/lab.rules /etc/sudoers.d/lab
sed -i '/# test comment/d' /etc/passwd
systemctl stop auditd rsyslog
userdel -r alice 2>/dev/null
apt purge -y auditd audispd-plugins rsyslog sudo
```

All lab artifacts are removed and the auditing stack is torn down.

---

## What you learned
- How sudo, sudoers drop-ins, and group membership grant authorization
- How PAM stacks shape authentication for SSH and local logins
- How to enable auditing and read security events via `ausearch` and `journalctl`

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- sudo — https://www.sudo.ws/
- Linux PAM — https://github.com/linux-pam/linux-pam
- auditd (Linux Audit) — https://github.com/linux-audit/audit-userspace
- SSSD — https://sssd.io/
