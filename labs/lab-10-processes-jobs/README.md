# Lab 10 — Processes, Jobs & Scheduling

This lab covers CompTIA Linux+ XK0-006 Objective 2.3 (Given a scenario, manage processes, jobs, and scheduling). You will inspect processes, manipulate jobs, adjust priorities, send signals, and schedule tasks with `at`, `cron`, and `anacron`.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Inspect running processes

```bash
ps auxf | head -20
pstree -p | head -20
apt update -y && apt install -y htop sysstat lsof strace at
pidstat 1 3
mpstat 1 2
```

`ps auxf` shows a forest of all processes with CPU/memory; `pstree` displays the parent-child hierarchy. `pidstat` and `mpstat` (from sysstat) sample per-process and per-CPU statistics.

---

## Step 2 — Explore /proc and lsof

```bash
sleep 600 &
SPID=$!
ls /proc/$SPID/
cat /proc/$SPID/status | head
cat /proc/$SPID/cmdline; echo
lsof -p $SPID | head
```

Every process exposes a `/proc/<PID>` directory containing `status`, `cmdline`, `fd/`, `maps`, and more. `lsof -p` lists every file descriptor the process holds.

---

## Step 3 — Process states and strace

```bash
ps -o pid,stat,cmd -p $SPID
strace -p $SPID -e trace=nanosleep -o /tmp/strace.out &
sleep 2
kill %2 2>/dev/null
head /tmp/strace.out
```

State codes: R=running, S=interruptible sleep, D=uninterruptible (usually I/O), Z=zombie, T=stopped. `strace -p` attaches to a live PID to trace its system calls.

---

## Step 4 — Background, foreground, jobs

```bash
sleep 300 &
sleep 400 &
jobs
fg %1
```

Press Ctrl+Z to suspend, then continue with the next block:

```bash
bg %1
jobs
nohup sleep 500 > /tmp/nohup.out 2>&1 &
disown
jobs
```

`&` runs in background, Ctrl+Z suspends to a stopped job, `bg`/`fg` resume in background/foreground, and `nohup` plus `disown` detach a job from the shell so it survives logout.

---

## Step 5 — Priority with nice and renice

```bash
nice -n 10 sleep 200 &
NPID=$!
ps -o pid,ni,cmd -p $NPID
renice -n 5 -p $NPID
ps -o pid,ni,cmd -p $NPID
```

Nice values range from -20 (highest priority) to 19 (lowest). Only root can lower the nice value (raise priority).

---

## Step 6 — Signals: kill, pkill, killall

```bash
sleep 1000 &
TPID=$!
kill -HUP $TPID 2>/dev/null; echo "HUP sent"
sleep 1000 &
kill -15 $!
sleep 1000 &
kill -9 $!
sleep 1000 & sleep 1000 &
pkill -f "sleep 1000"
killall sleep 2>/dev/null
```

`-15` (SIGTERM) requests graceful exit, `-9` (SIGKILL) forces termination, `-1` (SIGHUP) often triggers config reload. `pkill` matches by pattern; `killall` matches by exact command name.

---

## Step 7 — Scheduling with at and cron

```bash
//vi storage.sh 
#!/bin/bash
echo "=== Disk Usage Report - $(date '+%Y-%m-%d %H:%M:%S') ==="
df -k -h | column -t
echo "----------------------------------------"

//crontab -e
choose 1

*/5 * * * * /root/storage.sh >> /var/log/disk_usage.log 2>&1

cntl+x +Y

//crontab -l

#set execute persmisison to script to run 
chmod +x storage.sh
touch /var/log/disk_usage.log 2>&1

```

`at` schedules a one-shot future job; `cron` runs recurring jobs via the five-field schedule (minute hour day month weekday). `anacron` runs jobs that may have been missed (good for laptops).

---

## Step 8 — Cleanup

```bash
atrm $(atq | awk '{print $1}') 2>/dev/null
crontab -r
killall sleep strace 2>/dev/null
rm -f /tmp/at-fired.txt /tmp/cron.log /tmp/strace.out /tmp/nohup.out
apt remove -y htop sysstat at
```

All test jobs, cron entries, background processes, and installed packages are removed.

---

## What you learned
- Inspecting processes via `ps`, `pstree`, `/proc`, and `strace`
- Managing jobs with `&`, `bg`, `fg`, `nohup`, and adjusting priority with `nice`/`renice`
- Scheduling work with `at`, `cron`, and understanding `anacron`

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- htop — https://htop.dev/
- sysstat (pidstat/mpstat) — https://github.com/sysstat/sysstat
- strace — https://strace.io/
