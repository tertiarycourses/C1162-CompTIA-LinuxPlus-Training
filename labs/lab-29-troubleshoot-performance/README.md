# Lab 29 — Troubleshooting Performance Issues

This lab covers CompTIA Linux+ XK0-006 Objective 5.5. You will deliberately stress CPU, memory, disk I/O, and the network, then walk through the universal performance loop: capture a baseline, generate a controlled spike, identify the bottleneck with the right counter, and resolve it. Tools introduced include `stress-ng`, `fio`, `sysstat` (sar/iostat/mpstat/pidstat), and the classic interactive triumvirate `top`/`htop`/`iotop`.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install the performance toolkit and baseline

```bash
apt update
apt install -y stress-ng fio sysstat htop iotop iftop dstat procps
uptime
free -m
vmstat 1 3
mpstat 1 2
df -h /
```

Always record uptime/load, memory, vmstat, and mpstat before an experiment. Without a baseline, "high CPU" has no meaning.

---

## Step 2 — Generate and observe a CPU spike

```bash
stress-ng --cpu 4 --timeout 30 &
SPIKE=$!
sleep 5
uptime
mpstat -P ALL 1 3
top -bn1 | head -n 15
wait $SPIKE
uptime
```

Load average (1/5/15 min) crosses your CPU count when work piles up faster than cores can drain it. `mpstat -P ALL` confirms whether the load is one runaway thread or evenly distributed.

---

## Step 3 — Memory pressure and swap behavior

```bash
free -m
stress-ng --vm 2 --vm-bytes 256M --timeout 20 &
sleep 5
free -m
vmstat 1 5
wait
free -m
```

`vmstat` columns `si`/`so` show pages moving in and out of swap — the unambiguous sign of memory pressure. If they stay zero, RAM is fine and the symptom is elsewhere.

---

## Step 4 — Disk I/O storm

```bash
fio --name=lab --rw=randwrite --bs=4k --size=128M --runtime=20 --time_based \
    --filename=/tmp/fio.dat --ioengine=sync >/tmp/fio.log 2>&1 &
sleep 3
iostat -x 1 3
pidstat -d 1 3
wait
tail -n 15 /tmp/fio.log
rm -f /tmp/fio.dat
```

`iostat -x` shows per-device `%util`, `await`, and queue depth. `pidstat -d` attributes IOPS to processes — together they answer "what disk, which process".

---

## Step 5 — Network jitter and blocked processes

```bash
ping -i 0.2 -c 20 8.8.8.8 | tail -n 5
dd if=/dev/zero of=/tmp/big bs=1M count=200 oflag=direct &
DDPID=$!
sleep 1
ps -o pid,stat,wchan,cmd -p $DDPID
ps -eo stat,pid,cmd | awk '$1 ~ /^D/' | head
wait $DDPID
rm -f /tmp/big
```

`ping` summary (`rtt min/avg/max/mdev`) quantifies jitter directly. A process in state `D` is uninterruptible-sleep — almost always waiting on disk or NFS.

---

## Step 6 — Context switches and interactive tools

```bash
pidstat -w 1 5
timeout 3 dstat -tcnd 1 2>/dev/null || timeout 3 vmstat 1
timeout 3 htop -d 10 || true
which perf >/dev/null && perf top -d 5 || echo "perf not installed; skip"
```

`pidstat -w` separates voluntary from involuntary context switches; high involuntary switches mean the CPU is oversubscribed. `dstat` and `htop` are friendlier baselines for live observation.

---

## Step 7 — Walk the full loop on a synthetic incident

```bash
echo "BASELINE:"; uptime; free -m | head -n 2
stress-ng --cpu 2 --vm 1 --vm-bytes 128M --io 1 --timeout 25 &
sleep 6
echo "SPIKE:"; uptime
echo "TOP CPU:"; ps -eo pid,pcpu,pmem,cmd --sort=-pcpu | head -n 5
echo "TOP MEM:"; ps -eo pid,pcpu,pmem,cmd --sort=-pmem | head -n 5
echo "IO WAIT:"; iostat -c 1 1 | tail -n 2
wait
echo "RECOVERED:"; uptime; free -m | head -n 2
```

This is the universal triage script: baseline, spike, top consumers by CPU and memory, IO wait, recovery. Memorize the shape — it works for any incident.

---

## Step 8 — Cleanup

```bash
rm -f /tmp/fio.dat /tmp/fio.log /tmp/big
apt purge -y stress-ng fio sysstat htop iotop iftop dstat
apt autoremove -y
```

Removes the scratch files and the diagnostic packages installed for the lab.

---

## What you learned
- The four-stage performance loop: baseline, spike, identify, resolve
- Mapping symptoms to counters: load avg vs `mpstat`, `si`/`so` vs `free`, `%util` vs `await`
- Attributing resource use to specific PIDs with `pidstat` and `ps --sort`
- Recognizing D-state (uninterruptible sleep) as a disk/wait indicator
- Quantifying network jitter from ping mdev rather than averages

## Free tools used
- stress-ng — https://github.com/ColinIanKing/stress-ng
- fio — https://fio.readthedocs.io
- sysstat (sar/iostat/mpstat/pidstat) — https://github.com/sysstat/sysstat
- htop — https://htop.dev
- iotop — http://guichaz.free.fr/iotop/
- dstat — https://github.com/dstat-real/dstat
