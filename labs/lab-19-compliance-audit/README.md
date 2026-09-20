# Lab 19 — Compliance, Auditing, and File Integrity

This lab maps to Linux+ XK0-006 Objective 3.6. You will install AIDE for file-integrity monitoring, scan for rootkits with `rkhunter`, verify package contents with `debsums` (and `rpm -V` in a Rocky container), run a `lynis` audit, perform a basic `nmap` scan, securely delete data, set login banners, and review OpenSCAP and CIS benchmarks.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — AIDE baseline and tamper check

```bash
apt update && apt install -y aide
aideinit -y -f 2>/dev/null || aide --init
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
echo "tampered" >> /etc/hostname
aide --check | head -n 40
```

`aide --init` builds the cryptographic baseline. The subsequent `--check` reports any addition, removal, or content change against that baseline.

---

## Step 2 — Rootkit hunting with rkhunter

```bash
apt install -y rkhunter
rkhunter --update || true
rkhunter --propupd -q
rkhunter --check --sk --rwo
```

`--sk` skips the keypress prompts and `--rwo` shows only warnings. `--propupd` resyncs the package-property database to reduce false positives.

---

## Step 3 — Package verification: debsums and rpm -V

```bash
apt install -y debsums
debsums -c | head -n 20 || echo "No changed files detected"
apt install -y docker.io && systemctl start docker
docker run --rm rockylinux:9 bash -c 'rpm -V coreutils || echo "rpm -V flags S/M/5/D/L/U/G/T/P per file"'
```

`debsums -c` reports modified files in Debian-packaged software. The Rocky one-liner shows the equivalent RPM verification.

---

## Step 4 — System audit with lynis

```bash
apt install -y lynis
lynis audit system --quick --no-colors 2>&1 | tail -n 40
ls /var/log/lynis*.log
```

`lynis` runs hundreds of hardening checks and outputs warnings, suggestions, and a hardening index. The full log is saved under `/var/log/lynis.log`.

---

## Step 5 — nmap service detection

```bash
apt install -y nmap openssh-server
systemctl start ssh
nmap -sV -p22,80 localhost
```

`-sV` probes service banners to identify versions; combined with port and host data it forms the first step of vulnerability assessment.

---

## Step 6 — Secure deletion and wipe

```bash
echo "very sensitive" > /tmp/secret.txt
shred -vn 3 -u /tmp/secret.txt
ls /tmp/secret.txt 2>/dev/null || echo "Shredded"
dd if=/dev/zero of=/tmp/wipe.img bs=1M count=10
LOOP=$(losetup -f --show /tmp/wipe.img)
dd if=/dev/urandom of="$LOOP" bs=1M count=10 status=progress
losetup -d "$LOOP" && rm -f /tmp/wipe.img
```

`shred` overwrites a file three times and unlinks it. On SSDs use full-disk encryption plus discard, since wear leveling can defeat per-file overwrites.

---

## Step 7 — Banners and OpenSCAP / CIS pointers

```bash
echo "Authorized use only. Activity is monitored." > /etc/issue
cp /etc/issue /etc/issue.net
echo "Welcome to Lab Host" > /etc/motd
apt install -y libopenscap8 || apt install -y openscap-scanner
oscap --version | head -n 5
echo "CIS Benchmarks: https://www.cisecurity.org/cis-benchmarks"
echo "OpenSCAP content: https://github.com/ComplianceAsCode/content"
```

`/etc/issue` is shown on local TTY logins, `/etc/issue.net` on network logins, and `/etc/motd` after a successful login. OpenSCAP consumes SCAP/XCCDF content (often shipped by ComplianceAsCode) to evaluate CIS and STIG benchmarks.

---

## Step 8 — Cleanup

```bash
sed -i '/tampered/d' /etc/hostname
rm -f /var/lib/aide/aide.db /var/lib/aide/aide.db.new
echo "" > /etc/issue; echo "" > /etc/issue.net; echo "" > /etc/motd
systemctl stop ssh
apt purge -y aide rkhunter debsums lynis nmap docker.io libopenscap8 openscap-scanner 2>/dev/null
```

All baselines, banners, and scanning tools are removed.

---

## What you learned
- How AIDE detects unauthorized changes to system files
- How to verify package integrity with `debsums` and `rpm -V`
- How to combine `lynis`, `nmap`, and OpenSCAP for compliance assessment

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- AIDE — https://aide.github.io/
- rkhunter — https://rkhunter.sourceforge.net/
- Lynis — https://cisofy.com/lynis/
- Nmap — https://nmap.org/
- OpenSCAP — https://www.open-scap.org/
- CIS Benchmarks — https://www.cisecurity.org/cis-benchmarks
