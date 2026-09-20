# Lab 26 — Troubleshooting Hardware, Storage, and OS Issues

This lab maps to CompTIA Linux+ XK0-006 Objective 5.2. You will deliberately introduce seven common failure modes — full filesystem, inode exhaustion, broken GRUB config, runaway process, failed systemd unit, broken PATH, and a memory leak — then diagnose and repair each. Every breakage is contained to a loopback device, a throwaway unit, or a subshell so the host stays clean.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Break: fill a filesystem (ENOSPC)

```bash
apt update && apt install -y python3
mkdir -p /mnt/tinyfs
dd if=/dev/zero of=/tmp/tiny.img bs=1M count=64
mkfs.ext4 -F /tmp/tiny.img
mount -o loop /tmp/tiny.img /mnt/tinyfs
dd if=/dev/zero of=/mnt/tinyfs/fill bs=1M count=200 || echo "ENOSPC as expected"
df -h /mnt/tinyfs
du -shx /mnt/tinyfs/* | sort -h
rm /mnt/tinyfs/fill
df -h /mnt/tinyfs
```

`dd` fails with ENOSPC once the 64 MB loopback fills. `du -shx` finds the biggest consumer and deletion restores free space.

---

## Step 2 — Break: inode exhaustion

```bash
df -i /mnt/tinyfs
mkdir /mnt/tinyfs/many
(cd /mnt/tinyfs/many && for i in $(seq 1 20000); do : > f$i 2>/dev/null || break; done)
df -i /mnt/tinyfs
rm -rf /mnt/tinyfs/many
df -i /mnt/tinyfs
```

A filesystem can be 1% full by bytes yet refuse new files because every inode is used. `df -i` is the only way to spot it.

---

## Step 3 — Validate a GRUB config without committing it

```bash
grub-mkconfig -o /tmp/grub.cfg.new 2>&1 | tail -n 5
diff -u /boot/grub/grub.cfg /tmp/grub.cfg.new | head -n 40 || true
```

Generating to a scratch path lets you preview boot-loader changes before overwriting `/boot/grub/grub.cfg`, which would brick the host on a real machine.

---

## Step 4 — Break: runaway CPU process

```bash
bash -c 'while :; do :; done' &
RUNAWAY=$!
sleep 2
top -bn1 -p $RUNAWAY | tail -n 5
ps -o pid,pcpu,cmd -p $RUNAWAY
kill -TERM $RUNAWAY
wait $RUNAWAY 2>/dev/null
```

A busy-loop pegs one CPU at 100%. `top -bn1 -p PID` confirms the culprit; `kill` resolves it cleanly.

---

## Step 5 — Break: a failing systemd unit

```bash
cat >/etc/systemd/system/lab-broken.service <<'EOF'
[Unit]
Description=Deliberately broken unit
[Service]
ExecStart=/bin/false
EOF
systemctl daemon-reload
systemctl start lab-broken.service || true
systemctl status lab-broken.service --no-pager | head -n 15
journalctl -u lab-broken.service --no-pager | tail -n 10
systemctl disable --now lab-broken.service 2>/dev/null
rm /etc/systemd/system/lab-broken.service
systemctl daemon-reload
```

`/bin/false` exits 1, so systemd marks the unit `failed`. The status and journal output show exactly which command failed and with what code.

---

## Step 6 — Break and recover a broken PATH

```bash
( export PATH=/nonexistent
  ls 2>&1 || echo "ls vanished"
  /bin/ls / | head
  export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  ls / | head
)
echo "Parent shell PATH unchanged: $PATH"
```

Running inside `(...)` confines the broken PATH to a subshell. Absolute paths like `/bin/ls` still work, which is the recovery technique on a real host.

---

## Step 7 — Simulate a memory leak

```bash
cat >/tmp/leak.py <<'EOF'
data = []
i = 0
while True:
    data.append("x" * 1024 * 100)
    i += 1
EOF
python3 /tmp/leak.py &
LEAK=$!
sleep 3
ps -o pid,rss,vsz,cmd -p $LEAK
free -m
kill -KILL $LEAK
wait $LEAK 2>/dev/null
rm /tmp/leak.py
```

`ps -o rss` shows resident memory climbing while `free -m` shows shrinking available memory — the classic leak fingerprint.

---

## Step 8 — Cleanup

```bash
umount /mnt/tinyfs
rmdir /mnt/tinyfs
rm -f /tmp/tiny.img /tmp/grub.cfg.new
```

Detaches the loopback filesystem and removes all scratch files. No package removal needed.

---

## What you learned
- Distinguishing block-full from inode-full filesystems with `df` vs `df -i`
- Previewing GRUB regeneration safely to `/tmp` before committing
- Hunting CPU and memory hogs with `top`, `ps -o`, and `free`
- Reading systemd failure output via `systemctl status` and `journalctl -u`
- Confining environment breakage to subshells so the parent shell survives

## Free tools used
- coreutils (dd, df, du) — https://www.gnu.org/software/coreutils/
- procps (top, ps, free) — https://gitlab.com/procps-ng/procps
- GRUB — https://www.gnu.org/software/grub/
- systemd — https://systemd.io
- python3 — https://www.python.org
