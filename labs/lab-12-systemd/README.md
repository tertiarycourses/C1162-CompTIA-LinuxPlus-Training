# Lab 12 — Service Management with systemd

This lab covers CompTIA Linux+ XK0-006 Objective 2.5 (Given a scenario, manage services using systemd). You will inspect units, control nginx, author a custom service plus timer, analyze boot performance, and configure hostname/time/DNS via systemd helpers.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — List and inspect units

```bash
systemctl list-units --type=service | head -20
systemctl list-unit-files --type=service | head -20
systemctl --failed
```

`list-units` shows currently loaded units while `list-unit-files` shows every installed unit and its enabled/disabled state.

---

## Step 2 — Control nginx

```bash
apt update -y && apt install -y nginx
systemctl status nginx --no-pager
systemctl stop nginx
systemctl start nginx
systemctl restart nginx
systemctl reload nginx
systemctl disable nginx
systemctl enable nginx
systemctl mask nginx && systemctl start nginx 2>&1 | head
systemctl unmask nginx
```

`enable`/`disable` toggles boot-time autostart, `reload` re-reads config without dropping connections, and `mask` makes a unit unstartable (a stronger disable).

---

## Step 3 — Author a custom service

```bash
cat > /usr/local/bin/lab-hello.sh <<'EOF'
#!/usr/bin/env bash
echo "lab-hello fired at $(date)" >> /var/log/lab-hello.log
EOF
chmod +x /usr/local/bin/lab-hello.sh

cat > /etc/systemd/system/lab-hello.service <<'EOF'
[Unit]
Description=Lab Hello one-shot service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lab-hello.sh
EOF

systemctl daemon-reload
systemctl start lab-hello.service
cat /var/log/lab-hello.log
```

`daemon-reload` re-parses unit files after edits. `Type=oneshot` is appropriate for short scripts that exit after work is done.

---

## Step 4 — Add a timer

```bash
cat > /etc/systemd/system/lab-hello.timer <<'EOF'
[Unit]
Description=Run lab-hello every minute

[Timer]
OnBootSec=30s
OnUnitActiveSec=1min
Unit=lab-hello.service

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now lab-hello.timer
systemctl list-timers --no-pager | head
```

Timer units are the systemd-native replacement for cron entries and benefit from journal integration and dependency handling.

---

## Step 5 — Mount unit example

```bash
mkdir -p /mnt/labdata
cat > /etc/systemd/system/mnt-labdata.mount <<'EOF'
[Unit]
Description=Tmpfs at /mnt/labdata

[Mount]
What=tmpfs
Where=/mnt/labdata
Type=tmpfs
Options=size=16M

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl start mnt-labdata.mount
mount | grep labdata
```

Mount unit filenames must match the mount path with `/` replaced by `-` (so `/mnt/labdata` becomes `mnt-labdata.mount`).

---

## Step 6 — Boot performance

```bash
systemd-analyze
systemd-analyze blame | head
systemd-analyze critical-chain | head -20
```

`blame` ranks units by start time; `critical-chain` traces the longest dependency path that delayed reaching the default target.

---

## Step 7 — Host, time, and DNS

```bash
hostnamectl
hostnamectl set-hostname linuxplus-lab
timedatectl
timedatectl set-timezone Asia/Singapore
resolvectl status | head -20
```

These three tools are the systemd-supplied front ends for `/etc/hostname`, the system clock/timezone, and the stub DNS resolver.

---

## Step 8 — Journal and cleanup

```bash
journalctl -u lab-hello --no-pager | tail
systemctl stop lab-hello.timer mnt-labdata.mount
systemctl disable lab-hello.timer
rm -f /etc/systemd/system/lab-hello.service /etc/systemd/system/lab-hello.timer /etc/systemd/system/mnt-labdata.mount
rm -f /usr/local/bin/lab-hello.sh /var/log/lab-hello.log
systemctl daemon-reload
hostnamectl set-hostname ubuntu
apt remove -y nginx
```

All custom units, scripts, logs, and packages are removed to leave a clean system.

---

## What you learned
- Listing, controlling, enabling, masking, and inspecting systemd services
- Writing custom service, timer, and mount units with `daemon-reload`
- Diagnosing boot time and configuring host, time, and DNS via systemd helpers

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- systemd — https://systemd.io/
- nginx — https://nginx.org/
