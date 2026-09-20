# Lab 28 — Troubleshooting Security Issues

This lab covers CompTIA Linux+ XK0-006 Objective 5.4. SELinux ships disabled on Ubuntu, so the SELinux portion runs inside a Rocky Linux container; the remaining issues — ACLs, certificate trust, account lockout, broken repos, weak TLS ciphers, and unpatched packages — run on the Ubuntu host.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Prep host and pull a Rocky container

```bash
apt update
apt install -y docker.io acl openssl libpam-modules ca-certificates
systemctl start docker
docker pull rockylinux:9
docker run -dit --name selab --privileged rockylinux:9 bash
docker exec selab dnf -y install policycoreutils policycoreutils-python-utils selinux-policy-targeted setools-console audit httpd
```

The `--privileged` flag lets the container load SELinux userspace tooling. We will not boot a kernel with SELinux enforcing; we only practice the commands.

---

## Step 2 — Break: SELinux context, then restorecon

```bash
docker exec selab bash -c '
mkdir -p /var/www/html && echo hello > /tmp/index.html
cp /tmp/index.html /var/www/html/index.html
ls -Z /var/www/html/index.html
chcon -t user_home_t /var/www/html/index.html
ls -Z /var/www/html/index.html
restorecon -v /var/www/html/index.html
ls -Z /var/www/html/index.html
setsebool -P httpd_can_network_connect on 2>/dev/null || echo "boolean recorded (no enforcing kernel)"
getsebool httpd_can_network_connect 2>/dev/null || true
'
```

`chcon` shows how a copy can land with the wrong type. `restorecon` resets it from the policy database — the first command to try on any "permission denied" SELinux symptom.

---

## Step 3 — Inspect AVC denials and translate with audit2allow

```bash
docker exec selab bash -c '
ausearch -m avc 2>/dev/null | tail -n 20 || echo "no AVCs recorded"
echo "type=AVC msg=audit(0): avc: denied { read } for pid=1 comm=\"demo\" name=\"x\" dev=\"sda1\" ino=1 scontext=u:r:httpd_t:s0 tcontext=u:object_r:user_home_t:s0 tclass=file" > /tmp/avc.log
audit2allow -i /tmp/avc.log
'
```

`ausearch -m avc` lists policy denials; piping a denial through `audit2allow` generates a candidate policy module — never apply blindly, always read the rule first.

---

## Step 4 — Break: ACL on Ubuntu host

```bash
useradd -m alice 2>/dev/null || true
echo secret > /tmp/payroll.txt
setfacl -m u:alice:--- /tmp/payroll.txt
getfacl /tmp/payroll.txt
sudo -u alice cat /tmp/payroll.txt 2>&1 || echo "ACL denied as expected"
setfacl -x u:alice /tmp/payroll.txt
sudo -u alice cat /tmp/payroll.txt
rm /tmp/payroll.txt
```

POSIX ACLs override standard mode bits. `getfacl` is the only way to see them; `ls -l` will show a `+` but not the rule itself.

---

## Step 5 — Break: certificate trust

```bash
mkdir -p /tmp/ca && cd /tmp/ca
openssl req -x509 -newkey rsa:2048 -nodes -days 1 \
  -keyout ca.key -out ca.crt \
  -subj '/CN=Lab CA' 2>/dev/null
openssl s_client -connect self-signed.badssl.com:443 -servername self-signed.badssl.com </dev/null 2>&1 | grep -E 'verify|Verification' | head
cp ca.crt /usr/local/share/ca-certificates/lab-ca.crt
update-ca-certificates 2>&1 | tail -n 3
ls /etc/ssl/certs/ | grep -i lab-ca || true
rm /usr/local/share/ca-certificates/lab-ca.crt
update-ca-certificates --fresh >/dev/null
cd / && rm -rf /tmp/ca
```

Self-signed certs are untrusted by default. Installing the CA into `/usr/local/share/ca-certificates/` and rerunning `update-ca-certificates` rebuilds the system trust bundle.

---

## Step 6 — Break: account lockout with pam_faillock

```bash
apt install -y libpam-modules
grep -q faillock /etc/pam.d/common-auth || cat >>/etc/pam.d/common-auth <<'EOF'
auth required pam_faillock.so preauth deny=3 unlock_time=120
auth required pam_faillock.so authfail deny=3 unlock_time=120
EOF
for i in 1 2 3 4; do echo "wrongpw" | su - alice -c "true" 2>/dev/null; done
faillock --user alice 2>/dev/null || echo "faillock state recorded"
faillock --user alice --reset 2>/dev/null || true
sed -i '/pam_faillock.so/d' /etc/pam.d/common-auth
```

`pam_faillock` records failed auth attempts in `/var/run/faillock`. The `--reset` flag clears the counter and re-enables login.

---

## Step 7 — Break: misconfigured apt repo and weak TLS

```bash
echo 'deb http://no.such.host.invalid/ubuntu jammy main' > /etc/apt/sources.list.d/broken.list
apt update 2>&1 | tail -n 5 || true
rm /etc/apt/sources.list.d/broken.list
apt update >/dev/null 2>&1
openssl s_client -connect www.google.com:443 -tls1 </dev/null 2>&1 | grep -iE 'protocol|cipher|alert' | head
openssl s_client -connect www.google.com:443 -tls1_2 </dev/null 2>&1 | grep -iE 'protocol|cipher' | head
apt list --upgradable 2>/dev/null | head
apt install -y unattended-upgrades
dpkg-reconfigure -f noninteractive unattended-upgrades >/dev/null 2>&1 || true
systemctl status unattended-upgrades --no-pager | head -n 5
```

A bad sources.list line is a fast simulation of repo outage. `openssl s_client -tls1` is rejected by modern servers; `-tls1_2` succeeds — the standard cipher-suite negotiation test.

---

## Step 8 — Cleanup

```bash
docker rm -f selab
docker rmi rockylinux:9
userdel -r alice 2>/dev/null
apt purge -y docker.io acl unattended-upgrades
apt autoremove -y
rm -f /etc/apt/sources.list.d/broken.list
```

Removes the Rocky container, the alice user, and packages installed only for this lab.

---

## What you learned
- Where SELinux contexts live and how `restorecon` and booleans repair them
- Reading AVC denials with `ausearch` and converting them with `audit2allow`
- Diagnosing access denials caused by ACLs that `ls -l` will not reveal
- Installing a CA into the system trust store and verifying with `s_client`
- Resetting pam_faillock lockouts and detecting weak TLS via openssl

## Free tools used
- SELinux userspace — https://github.com/SELinuxProject/selinux
- acl (setfacl/getfacl) — https://savannah.nongnu.org/projects/acl
- OpenSSL — https://www.openssl.org
- Linux-PAM — https://github.com/linux-pam/linux-pam
- Docker — https://www.docker.com
- unattended-upgrades — https://wiki.debian.org/UnattendedUpgrades
