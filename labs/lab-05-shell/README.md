# Lab 5 — Shell Operations & Text Processing

In this lab you will work with shell environment variables, login vs interactive startup files, every common form of redirection, and the classic text-processing pipeline (`grep`, `awk`, `sed`, `cut`, `sort`, `uniq`, `wc`, `head`, `tail`, `tee`, `xargs`, `tr`). By the end you will be able to assemble a non-trivial one-liner and explain what each stage does. Maps to Linux+ XK0-006 Objective 1.5 (shell, scripting basics, text processing).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Environment variables and PATH

```bash
echo "USER=$USER  HOME=$HOME  SHELL=$SHELL"
echo "PS1=$PS1"
echo "PATH=$PATH"
env | head
export LABVAR="hello linux+"
env | grep LABVAR
unset LABVAR
```

`env` prints the exported environment. `export` puts a variable into the environment of subsequently launched processes; `unset` removes it.

---

## Step 2 — Absolute vs relative paths and startup files

```bash
mkdir -p /tmp/lab5 && cd /tmp/lab5
pwd
ls ./
ls ../../tmp
ls /tmp/lab5
ls -la ~/ | head
cat ~/.bashrc 2>/dev/null | head -10 || echo "no .bashrc for root in this image"
#cat ~/.bash_profile 2>/dev/null || echo "no .bash_profile"
cat /etc/profile | head
```

Login shells source `/etc/profile` then `~/.bash_profile` (or `~/.profile`); interactive non-login shells source `~/.bashrc`. Absolute paths begin with `/`; relative paths resolve against the current directory.

---

## Step 3 — Redirection: >, >>, <, <<, |, <<<

```bash
cd /tmp/lab5
echo "first line"  > out.txt
echo "second line" >> out.txt
cat out.txt
wc -l < out.txt
cat <<EOF >> out.txt
line three
line four
EOF
cat out.txt
grep line <<<"line via here-string"
ls /etc | head | tee dirlist.txt | wc -l
cat dirlist.txt
```
```bash
root@ubuntu:/tmp/lab5$ ls -rlt    
total 16
-rw-r--r-- 1 root root    0 Jul  6 05:51 file.txt
-rw-r--r-- 1 root root   44 Jul  6 06:06 out.txt
-rw-r--r-- 1 root root  104 Jul  6 06:08 dirlist.txt
drwxr-xr-x 2 root root 4096 Jul  6 06:09 test1
drwxr-xr-x 2 root root 4096 Jul  6 06:09 test2
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ ls -lrt |grep ^d
drwxr-xr-x 2 root root 4096 Jul  6 06:09 test1
drwxr-xr-x 2 root root 4096 Jul  6 06:09 test2
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ 
root@ubuntu:/tmp/lab5$ ls -lrt |grep ^-
-rw-r--r-- 1 root root    0 Jul  6 05:51 file.txt
-rw-r--r-- 1 root root   44 Jul  6 06:06 out.txt
-rw-r--r-- 1 root root  104 Jul  6 06:08 dirlist.txt
```
`>` truncates and writes, `>>` appends. `<` redirects stdin from a file. `<<EOF` is a here-doc, `<<<` is a here-string. `tee` writes stdout to both a file and the next stage of the pipeline.

---

## Step 4 — grep, cut, sort, uniq, wc

```bash
cd /tmp/lab5
cp /etc/passwd .
wc -l passwd
grep -c bash passwd
cut -d: -f1,7 passwd | head
cut -d: -f7 passwd | sort | uniq -c | sort -rn
```

`cut -d: -f7` pulls the login-shell field; piping through `sort | uniq -c | sort -rn` gives a frequency report — a near-daily one-liner for sysadmins.

---

## Step 5 — sed and awk

```bash
cd /tmp/lab5
sed -n '1,5p' passwd
sed 's|/bin/bash|/bin/zsh|' passwd | head -3
awk -F: '$3 >= 1000 {print $1, $3, $7}' passwd
awk -F: 'BEGIN{n=0} /nologin/{n++} END{print n" nologin accounts"}' passwd
```

`sed` is stream-oriented line editing; `s|x|y|` substitutes (using `|` as the delimiter avoids escaping `/`). `awk` is a full text-processing mini-language with fields, conditions, and BEGIN/END blocks.

---

## Step 6 — head, tail, tr, xargs

```bash
cd /tmp/lab5
head -3 passwd
tail -3 passwd
tail -f /var/log/dpkg.log &
TPID=$!; sleep 1; kill $TPID 2>/dev/null
echo "MIXED case STRING" | tr 'a-z' 'A-Z'
echo "one two three" | tr ' ' '\n'
ls /etc | head -5 | xargs -I{} echo "found: {}"
find /etc -maxdepth 1 -name "*.conf" -print0 | xargs -0 ls -l | head
```

`tr` translates or deletes characters by class. `xargs` turns stdin into command-line arguments — `-I{}` uses a placeholder, `-0` pairs with `find -print0` for filenames with spaces.

---

## Step 7 — Editors: nano and vim, scripted

```bash
apt-get install -y vim nano >/dev/null
cat > script.sh <<'EOF'
#!/bin/bash
echo "hello from $(whoami) on $(hostname)"
EOF
chmod +x script.sh
./script.sh
# Open + save + quit in vim non-interactively
vim -e -s -c 'normal Goappended line' -c 'wq' script.sh
cat script.sh
```

`vim -e -s -c '...'` runs Ex commands silently — useful in automation. In real interactive use you would press `i` to insert, `Esc` to leave insert mode, and `:wq` to write and quit. `nano` saves with `Ctrl-O` and quits with `Ctrl-X`.

---

## Step 8 — Cleanup

```bash
rm -rf /tmp/lab5
apt-get purge -y nano >/dev/null
apt-get autoremove -y >/dev/null
```

Removes the lab directory and the optional `nano` install (vim is preinstalled on most images).

---

## What you learned
- The difference between login and interactive shells and which startup files each sources.
- Every common redirection operator: `>`, `>>`, `<`, `<<`, `<<<`, `|`, `tee`.
- A small but powerful toolkit (`grep`/`awk`/`sed`/`cut`/`sort`/`uniq`/`tr`/`xargs`) that composes into ad-hoc data pipelines.
- How to script `vim` non-interactively, which both demystifies modal editing and pays off in CI.

## Free tools used
- GNU Bash — https://www.gnu.org/software/bash/
- GNU coreutils — https://www.gnu.org/software/coreutils/
- GNU sed & gawk — https://www.gnu.org/software/sed/
- Vim — https://www.vim.org/
