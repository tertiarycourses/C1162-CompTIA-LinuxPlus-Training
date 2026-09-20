# Lab 21 — Bash Scripting

This lab covers shell scripting fundamentals for CompTIA Linux+ XK0-006 Objective 4.2 (Given a scenario, perform basic scripting tasks). You will exercise variables, conditionals, loops, functions, options parsing, regex, and file tests, then assemble a small backup script and audit it with `shellcheck`.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Set up a scripts directory

```bash
mkdir -p /root/scripts && cd /root/scripts
apt update && apt install -y shellcheck
bash --version
```

`shellcheck` is the standard static analyser for shell scripts and will catch many subtle quoting bugs.

---

## Step 2 — Variables, command substitution, and parameter expansion

```bash
cat > /root/scripts/basics.sh <<'EOF'
#!/usr/bin/env bash
name="Linux+"
today=$(date +%F)
greeting="${1:-hello}"
echo "$greeting $name on $today"

fruits=(apple banana cherry)
echo "first=${fruits[0]} count=${#fruits[@]}"
echo "all: ${fruits[*]}"
EOF
chmod +x /root/scripts/basics.sh
/root/scripts/basics.sh
/root/scripts/basics.sh "hi"
```

`${1:-hello}` returns `hello` if `$1` is unset or empty. Arrays use `()` and are indexed with `${arr[i]}`.

---

## Step 3 — Conditionals, case, and test operators

```bash
cat > /root/scripts/checks.sh <<'EOF'
#!/usr/bin/env bash
num=${1:-0}
path=${2:-/etc/hostname}

if [[ $num -eq 0 ]]; then
  echo "zero"
elif [[ $num -lt 10 ]]; then
  echo "small"
else
  echo "big"
fi

case "$path" in
  /etc/*) echo "system config" ;;
  /home/*) echo "user data" ;;
  *)      echo "other" ;;
esac

[[ -d $path ]] && echo "$path is a directory"
[[ -f $path ]] && echo "$path is a regular file"
[[ -z "" ]]    && echo "empty string detected"
[[ -n "abc" ]] && echo "non-empty string detected"

s="hello123"
if [[ $s =~ ^[a-z]+[0-9]+$ ]]; then
  echo "matches regex"
fi
EOF
chmod +x /root/scripts/checks.sh
/root/scripts/checks.sh 5 /etc
/root/scripts/checks.sh 42 /etc/hostname
```

Numeric tests use `-eq -ne -lt -gt`; strings use `==`, `!=`, and `=~` (regex) inside `[[ ]]`.

---

## Step 4 — Loops, functions, and exit codes

```bash
cat > /root/scripts/loops.sh <<'EOF'
#!/usr/bin/env bash
log() { echo "[$(date +%T)] $*"; }

for f in /etc/hosts /etc/hostname /etc/nonexistent; do
  if [[ -f $f ]]; then
    log "found $f"
  else
    log "missing $f"
  fi
done

i=0
while [[ $i -lt 3 ]]; do
  log "while i=$i"
  ((i++))
done

j=0
until [[ $j -ge 3 ]]; do
  log "until j=$j"
  ((j++))
done

grep -q root /etc/passwd
echo "grep exit code: $?"
EOF
chmod +x /root/scripts/loops.sh
/root/scripts/loops.sh
```

`$?` holds the exit code of the previous command. Zero means success.

---

## Step 5 — getopts and IFS-based CSV parsing

```bash
cat > /root/scripts/opts.sh <<'EOF'
#!/usr/bin/env bash
verbose=0
target=""
while getopts ":vt:" opt; do
  case $opt in
    v) verbose=1 ;;
    t) target=$OPTARG ;;
    *) echo "usage: $0 [-v] [-t target]"; exit 2 ;;
  esac
done
echo "verbose=$verbose target=$target"

row="alice,30,admin"
IFS=',' read -r user age role <<<"$row"
echo "user=$user age=$age role=$role"
EOF
chmod +x /root/scripts/opts.sh
/root/scripts/opts.sh -v -t web01
```

`getopts` is the POSIX option parser. Temporarily changing `IFS` is a quick way to split CSV without external tools.

---

## Step 6 — Build a real backup script

```bash
cat > /root/scripts/backup-etc.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

SRC="/etc"
DEST="/root/backups"
MAX_AGE_DAYS="${1:-7}"
STAMP=$(date +%Y%m%d-%H%M%S)
ARCHIVE="$DEST/etc-$STAMP.tar.gz"

mkdir -p "$DEST"

latest=$(find "$DEST" -maxdepth 1 -name 'etc-*.tar.gz' -printf '%T@ %p\n' 2>/dev/null \
         | sort -n | tail -1 | awk '{print $2}')

if [[ -n "${latest:-}" ]]; then
  age_days=$(( ( $(date +%s) - $(stat -c %Y "$latest") ) / 86400 ))
  if [[ $age_days -lt $MAX_AGE_DAYS ]]; then
    echo "Recent backup exists ($latest, ${age_days}d old); skipping."
    exit 0
  fi
fi

tar -czf "$ARCHIVE" "$SRC"
echo "Created $ARCHIVE"
EOF
chmod +x /root/scripts/backup-etc.sh
/root/scripts/backup-etc.sh 7
/root/scripts/backup-etc.sh 7
ls -lh /root/backups
```

`set -euo pipefail` makes the script fail fast on errors, undefined variables, and broken pipes. The script only tars `/etc` if the newest archive is older than N days.

---

## Step 7 — Lint with shellcheck

```bash
shellcheck /root/scripts/*.sh || true
shellcheck -s bash /root/scripts/backup-etc.sh
```

Fix any warnings shellcheck reports. Common findings include unquoted variables and using `[ ]` instead of `[[ ]]`.

---

## Step 8 — Cleanup

```bash
rm -rf /root/scripts /root/backups
apt remove -y --purge shellcheck
apt autoremove -y
```

This removes all artifacts so the playground is clean.

---

## What you learned
- How to use shebangs, variables, parameter expansion, arrays, conditionals, loops, and functions in Bash.
- How to parse options with `getopts` and CSV with `IFS`.
- File and string tests, numeric comparisons, and regex matching with `[[ =~ ]]`.
- How to write a defensive script with `set -euo pipefail` and lint it with `shellcheck`.

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- Bash manual — https://www.gnu.org/software/bash/manual/
- ShellCheck — https://www.shellcheck.net/
