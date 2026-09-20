# Lab 16 — OS Hardening: Permissions, SELinux, SSH, Fail2ban

This lab maps to Linux+ XK0-006 Objective 3.3. You will practice classic permission tools (`chmod`, `chown`, `chgrp`, special bits, `umask`), set immutability with `chattr`, layer POSIX ACLs, demo SELinux inside a Rocky container, harden OpenSSH, install `fail2ban`, hunt SUID binaries, and review legacy services to disable.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — chmod, chown, special bits

```bash
useradd -m alice 2>/dev/null
mkdir -p /srv/lab && cd /srv/lab
cat >run.sh <<'EOF'
#!/bin/bash
id
EOF
chmod 750 run.sh
chmod u+s run.sh        # setuid
chmod g+s .             # setgid on directory
chmod +t /tmp           # sticky bit (already set)
chown root:alice run.sh
ls -l run.sh /tmp -d .
```

`setuid` makes the script run as its owner, `setgid` on a directory makes new files inherit the group, and the sticky bit on `/tmp` prevents users from deleting each other's files.

---

## Step 2 — umask, chattr, lsattr

```bash
umask 027
touch /srv/lab/newfile && ls -l /srv/lab/newfile
chattr +i /etc/resolv.conf(# Do not use Symnolic link)
lsattr /etc/resolv.conf(# Do not use Symnolic link)
chattr -i /etc/resolv.conf(# Do not use Symnolic link)
```

`umask 027` produces 640 files and 750 directories. `chattr +i` marks a file immutable so even root cannot modify it until the bit is cleared.

---

## Step 3 — POSIX ACLs

```bash
apt install -y acl
touch /tmp/file
setfacl -m u:alice:r /tmp/file
getfacl /tmp/file
ls -l /tmp/file   # the trailing + indicates an ACL
```

ACLs extend the classic owner/group/other model. The `+` in `ls -l` flags files that carry additional ACL entries.

---

## Step 4 — SELinux demo inside Rocky

```bash
apt install -y docker.io && systemctl start docker
docker run --rm -it rockylinux:9 bash -c '
  dnf install -y policycoreutils selinux-policy-targeted policycoreutils-python-utils setools-console && \
  getenforce || echo "SELinux disabled in container"; \
  ls -Z /etc/passwd; \
  echo "Workflow: chcon -t httpd_sys_content_t /web; semanage fcontext -a -t httpd_sys_content_t \"/web(/.*)?\"; restorecon -Rv /web"; \
  echo "Denials: ausearch -m AVC | audit2allow -M mypol"
'
```

Ubuntu uses AppArmor by default, so SELinux is demonstrated in a Rocky container. The displayed workflow shows the standard label-and-restore cycle and the `audit2allow` denial-to-policy path.

---

## Step 5 — Harden OpenSSH

```bash
apt install -y openssh-server
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sshd -t && systemctl restart ssh
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519
mkdir -p /home/alice/.ssh
cat /root/.ssh/id_ed25519.pub >> /home/alice/.ssh/authorized_keys
chown -R alice:alice /home/alice/.ssh && chmod 700 /home/alice/.ssh
chmod 600 /home/alice/.ssh/authorized_keys
```

Disabling root login and password auth pushes administrators toward key-based access. `sshd -t` validates the config before restart.

---

## Step 6 — fail2ban

```bash
apt install -y fail2ban
cat >/etc/fail2ban/jail.local <<'EOF'
[sshd]
enabled = true
maxretry = 5
bantime = 600
EOF
systemctl enable --now fail2ban
fail2ban-client status
fail2ban-client status sshd
```

`fail2ban` tails auth logs and bans IPs that exceed `maxretry` failures within `findtime`. Per-jail status shows banned addresses.

---

## Step 7 — Hunt SUID binaries and review legacy services

```bash
find / -perm -4000 -type f 2>/dev/null | head -n 20
dpkg -l | grep -Ei 'telnet|tftp|vsftpd|rsh' || echo "Clean: no legacy daemons installed"
systemctl list-unit-files | grep -Ei 'telnet|tftp|ftp' || true
```

Telnet, FTP, and TFTP transmit credentials in clear text and should be replaced with SSH/SFTP/HTTPS. SUID binaries are common privilege-escalation targets and worth inventorying.

---

## Step 8 — Cleanup

```bash
systemctl stop fail2ban
apt purge -y fail2ban acl docker.io
rm -rf /srv/lab /tmp/file /root/.ssh/id_ed25519* /home/alice
userdel alice 2>/dev/null
umask 022
```

All lab files, packages, and the alice user are removed.

---

## What you learned
- The meaning and use of setuid, setgid, sticky bit, umask, and immutability
- How SELinux labels files and how `audit2allow` turns denials into policy
- How to harden SSH and add brute-force protection with fail2ban

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- OpenSSH — https://www.openssh.com/
- fail2ban — https://www.fail2ban.org/
- SELinux Project — https://github.com/SELinuxProject
- acl (POSIX ACLs) — https://savannah.nongnu.org/projects/acl
