# Lab 17 — Account Hardening

This lab maps to Linux+ XK0-006 Objective 3.4. You will tune `/etc/login.defs`, enforce password quality via `pam_pwquality`, set password aging with `chage`, lock and unlock accounts, configure brute-force lockouts with `pam_faillock`, convert service accounts to `nologin`, demonstrate a restricted shell, walk through MFA setup with Google Authenticator, and probe Have I Been Pwned with the k-anonymity API.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — login.defs and a test user

```bash
apt update && apt install -y libpam-pwquality
useradd -m -s /bin/bash alice
echo "alice:Passw0rd!" | chpasswd
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
grep -E '^PASS_' /etc/login.defs
```

`/etc/login.defs` provides system-wide defaults applied to new accounts. Existing users keep their settings unless modified with `chage`.

---

## Step 2 — Password quality with pwquality

```bash
cat >/etc/security/pwquality.conf <<'EOF'
minlen = 14
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
retry = 3
EOF
pwscore <<<"short"
pwscore <<<"Tr0ub4dor&3LongerPhrase!"
```

Negative `dcredit`/`ucredit`/`lcredit`/`ocredit` require at least one digit, upper, lower, and other character. `pwscore` rates a candidate from 0–100.

---

## Step 3 — Password aging and lock/unlock

```bash
chage -M 90 -m 1 -W 7 alice
chage -l alice
passwd -l alice
passwd -S alice
passwd -u alice
passwd -S alice
```

`chage -l` lists aging info, `passwd -l` prepends `!` to the hash to lock the account, and `-u` removes the lock.

---

## Step 4 — pam_faillock lockout

```bash
cat >/etc/security/faillock.conf <<'EOF'
deny = 5
unlock_time = 900
fail_interval = 900
EOF
grep -l pam_faillock /etc/pam.d/* 2>/dev/null || echo "Add 'auth required pam_faillock.so preauth' and 'authfail' lines to /etc/pam.d/common-auth"
faillock --user alice
```

`pam_faillock` replaces the older `pam_tally2`. After five failures `alice` is locked for 15 minutes; `faillock --user alice --reset` clears the counter.

---

## Step 5 — Service accounts with nologin and rbash

```bash
useradd -r -s /usr/sbin/nologin svcacct
grep svcacct /etc/passwd
ln -sf /bin/bash /usr/local/bin/rbash
useradd -m -s /usr/local/bin/rbash boxed
echo 'PATH=/usr/local/rbin' >> /home/boxed/.bashrc
mkdir -p /usr/local/rbin && ln -sf /bin/ls /usr/local/rbin/ls
su - boxed -c 'cd /tmp 2>&1 || echo "rbash blocks cd"'
```

`nologin` keeps daemon accounts from being interactively used. `rbash` (bash invoked as `rbash` or via `-r`) blocks `cd`, `PATH` reassignment, and redirection.

---

## Step 6 — MFA with Google Authenticator (walkthrough only)

```bash
apt install -y libpam-google-authenticator
echo "Run as the target user, not root:"
echo "  google-authenticator -t -d -f -r 3 -R 30 -W"
echo "Flags: -t TOTP, -d disallow reuse, -f write config, -r3 -R30 rate-limit, -W window"
echo "Then add 'auth required pam_google_authenticator.so' near the top of /etc/pam.d/sshd"
echo "and set ChallengeResponseAuthentication yes in /etc/ssh/sshd_config."
```

The interactive setup prints a QR code; on Killercoda we just review the procedure. The pairing secret lives in `~/.google_authenticator`.

---

## Step 7 — Pwned Passwords k-anonymity lookup

```bash
apt install -y curl
PW='password123'
HASH=$(printf "%s" "$PW" | sha1sum | awk '{print toupper($1)}')
PREFIX=${HASH:0:5}; SUFFIX=${HASH:5}
echo "Querying HIBP with prefix $PREFIX (suffix never leaves the host)"
curl -s "https://api.pwnedpasswords.com/range/$PREFIX" | grep -i "^$SUFFIX" \
  && echo "BREACHED" || echo "Not found"
```

The k-anonymity API only receives the first five characters of the SHA-1; the client filters the returned list locally. This avoids sending full passwords or hashes.

---

## Step 8 — Cleanup

```bash
userdel -r alice 2>/dev/null
userdel -r boxed 2>/dev/null
userdel svcacct 2>/dev/null
rm -f /etc/security/pwquality.conf /etc/security/faillock.conf
rm -f /usr/local/bin/rbash; rm -rf /usr/local/rbin
apt purge -y libpam-pwquality libpam-google-authenticator
```

All accounts and pwquality/faillock settings are removed.

---

## What you learned
- How to enforce password aging and quality via login.defs and pwquality
- How `pam_faillock` blocks brute-force attempts and how to recover an account
- How TOTP MFA and Pwned Passwords k-anonymity work in practice

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- libpam-pwquality — https://github.com/libpwquality/libpwquality
- pam_faillock — https://github.com/linux-pam/linux-pam
- Google Authenticator PAM — https://github.com/google/google-authenticator-libpam
- Have I Been Pwned (Pwned Passwords) — https://haveibeenpwned.com/Passwords
