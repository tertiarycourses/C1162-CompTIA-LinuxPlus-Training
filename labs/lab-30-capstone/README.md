# Lab 30 — Capstone: End-to-End Server Build and Triage

This capstone integrates CompTIA Linux+ XK0-006 Objectives 1 through 5. You will provision a service account, harden SSH, build LVM-backed storage from loopback devices, deploy nginx with a custom systemd override, lock the firewall to web ports, schedule a health-check timer, version `/etc/nginx` with git, and then deliberately inject a fault and triage it using the tools practiced in earlier labs. Treat every step as a checkpoint — re-run the verification command before moving on.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Provision a user and harden SSH

```bash
apt update
apt install -y openssh-server sudo nginx lvm2 ufw git curl
useradd -m -s /bin/bash -G sudo webops
echo 'webops:LabPass!23' | chpasswd
echo 'webops ALL=(ALL) NOPASSWD: /bin/systemctl, /usr/bin/journalctl' >/etc/sudoers.d/webops
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sshd -t && systemctl restart ssh
id webops
```

A non-root admin with narrow sudo plus key-only SSH is the standard hardening baseline.

---

## Step 2 — Build LVM-backed `/srv/web` over loopback

```bash
dd if=/dev/zero of=/var/lab-pv.img bs=1M count=200
LOOP=$(losetup --find --show /var/lab-pv.img)
pvcreate $LOOP
vgcreate labvg $LOOP
lvcreate -n weblv -L 150M labvg
mkfs.ext4 /dev/labvg/weblv
mkdir -p /srv/web
mount /dev/labvg/weblv /srv/web
echo "/dev/labvg/weblv /srv/web ext4 defaults 0 2" >>/etc/fstab
df -h /srv/web
echo '<h1>Capstone OK</h1>' >/srv/web/index.html
```

Loopback PV → VG → LV → ext4 mirrors the real disk pipeline without needing a spare device. Adding to `/etc/fstab` proves the boot-time mount path.

---

## Step 3 — Install nginx with a systemd override binding `/srv/web`

```bash
rm -f /etc/nginx/sites-enabled/default
cat >/etc/nginx/sites-available/capstone <<'EOF'
server {
    listen 80 default_server;
    root /srv/web;
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
EOF
ln -sf /etc/nginx/sites-available/capstone /etc/nginx/sites-enabled/capstone
mkdir -p /etc/systemd/system/nginx.service.d
cat >/etc/systemd/system/nginx.service.d/override.conf <<'EOF'
[Service]
ReadWritePaths=/srv/web
EOF
systemctl daemon-reload
nginx -t
systemctl restart nginx
curl -sf http://localhost/ | head
```

A drop-in override is the canonical way to extend a packaged unit without touching the vendor file.

---

## Step 4 — Firewall: allow only 80/443 and SSH

```bash
ufw --force reset >/dev/null
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
ufw status verbose
```

Default-deny inbound with an explicit allowlist is the only firewall pattern you should put on a real server.

---

## Step 5 — Health-check script with a systemd timer

```bash
cat >/usr/local/bin/web-health.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/)
ts=$(date -Is)
if [[ "$code" != "200" ]]; then
  echo "$ts FAIL code=$code" | tee -a /var/log/web-health.log
  exit 1
fi
echo "$ts OK" >>/var/log/web-health.log
EOF
chmod +x /usr/local/bin/web-health.sh
cat >/etc/systemd/system/web-health.service <<'EOF'
[Unit]
Description=Web health check
[Service]
Type=oneshot
ExecStart=/usr/local/bin/web-health.sh
EOF
cat >/etc/systemd/system/web-health.timer <<'EOF'
[Unit]
Description=Run web health check every minute
[Timer]
OnBootSec=30s
OnUnitActiveSec=1min
Unit=web-health.service
[Install]
WantedBy=timers.target
EOF
systemctl daemon-reload
systemctl enable --now web-health.timer
systemctl list-timers --no-pager | head
```

A timer is the modern cron replacement that also produces journal entries you can correlate with metrics.

---

## Step 6 — Commit `/etc/nginx` to a local git repo

```bash
cd /etc/nginx
git init -q
git config user.email ops@example.local
git config user.name "Lab Ops"
git add -A
git commit -q -m "Initial nginx configuration for capstone"
git log --oneline
```

Versioning `/etc` is the cheap, no-dependency form of configuration management — every change is recoverable.

---

## Step 7 — Inject a fault and triage

```bash
echo "===== FAULT INJECTION ====="
systemctl stop nginx
dd if=/dev/zero of=/srv/web/filler bs=1M count=160 2>/dev/null || true
systemctl start nginx || true
sleep 2

echo "--- service status ---"
systemctl status nginx --no-pager | head -n 12
echo "--- recent journal ---"
journalctl -u nginx -n 20 --no-pager
echo "--- listen sockets ---"
ss -tlnp | grep -E ':80|nginx' || echo "nginx not listening"
echo "--- disk usage ---"
df -h /srv/web
echo "--- health log ---"
tail -n 5 /var/log/web-health.log 2>/dev/null
echo "--- end-to-end probe ---"
curl -v --max-time 3 http://localhost/ 2>&1 | head -n 20

echo "===== REMEDIATION ====="
rm -f /srv/web/filler
df -h /srv/web
systemctl restart nginx
curl -sf http://localhost/ && echo OK
```

This is the consolidated triage flow from labs 25–29: status, journal, sockets, storage, health log, then a live probe. Remediation removes the fault and verifies recovery.

---

## Step 8 — Cleanup

```bash
systemctl disable --now web-health.timer
rm -f /etc/systemd/system/web-health.{service,timer} /usr/local/bin/web-health.sh /var/log/web-health.log
systemctl daemon-reload
systemctl stop nginx
ufw --force disable
sed -i '\|/dev/labvg/weblv|d' /etc/fstab
umount /srv/web
lvremove -f /dev/labvg/weblv
vgremove -f labvg
pvremove -f $(losetup -j /var/lab-pv.img | cut -d: -f1)
losetup -d $(losetup -j /var/lab-pv.img | cut -d: -f1)
rm -f /var/lab-pv.img
rm -rf /srv/web /etc/nginx/sites-enabled/capstone /etc/nginx/sites-available/capstone
rm -rf /etc/systemd/system/nginx.service.d
userdel -r webops 2>/dev/null
rm -f /etc/sudoers.d/webops
apt purge -y nginx lvm2 ufw
apt autoremove -y
```

Tears down every artifact in reverse order — timers, firewall, fstab, LVM stack, loopback, user, packages.

---

## What you learned
- Composing a hardened admin account, SSH policy, LVM storage, nginx, firewall, timers, and git into one coherent server build
- Using systemd drop-in overrides instead of editing vendor unit files
- Versioning `/etc` directories as a zero-dependency change record
- Running the full triage loop end-to-end: status, journal, sockets, storage, probe, fix
- Cleanly reversing every infrastructure change in dependency order

## Free tools used
- OpenSSH — https://www.openssh.com
- nginx — https://nginx.org
- lvm2 — https://sourceware.org/lvm2/
- ufw — https://launchpad.net/ufw
- systemd timers — https://systemd.io
- git — https://git-scm.com
- curl — https://curl.se
