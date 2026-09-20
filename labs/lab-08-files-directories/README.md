# Lab 8 — File & Directory Management

This lab covers CompTIA Linux+ XK0-006 Objective 2.1 (Given a scenario, manage files and directories). You will practice navigation, creation, copying, searching, linking, and inspecting special device files using core Linux utilities.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Navigate and list

```bash
pwd
cd /tmp
mkdir -p lab08/{src,dst,archive}
cd lab08
ls -la
```

`pwd` prints your current working directory and `ls -la` shows hidden files with long-format metadata (permissions, owner, size, mtime).

---

## Step 2 — Create, copy, move, touch

```bash
touch src/file1.txt src/file2.log
echo "hello" > src/file1.txt
cp -a src/file1.txt dst/
mv src/file2.log archive/
stat dst/file1.txt
file dst/file1.txt
```

`cp -a` preserves attributes (mode, ownership, timestamps, symlinks). `stat` shows inode details and `file` identifies content type by magic bytes.

---

## Step 3 — Find files by name, time, and size

```bash
dd if=/dev/zero of=src/big.bin bs=1M count=5
find /tmp/lab08 -name "*.txt"
find /tmp/lab08 -mtime -1
find /tmp/lab08 -size +1M
find /tmp/lab08 -name "*.bin" -exec ls -lh {} \;
```

`find` walks the directory tree with predicates; `-exec ... {} \;` runs a command per match. Use `-mtime -1` for files modified within the last day.

---

## Step 4 — locate and lsof

```bash
apt update -y && apt install -y mlocate lsof
updatedb
locate file1.txt | head
sleep 300 &
lsof -p $! | head
```

`locate` queries a prebuilt database (`updatedb`) and is much faster than `find` for name lookups. `lsof` lists open files for a process — here we inspect a backgrounded `sleep`.

---

## Step 5 — diff and sdiff

```bash
echo -e "alpha\nbravo\ncharlie" > a.txt
echo -e "alpha\nbravo2\ncharlie\ndelta" > b.txt
diff a.txt b.txt
sdiff a.txt b.txt
```

`diff` shows line-by-line differences; `sdiff` displays a side-by-side view with `|`, `<`, `>` markers indicating where files diverge.

---

## Step 6 — Hard links vs symbolic links

```bash
echo "original" > target.txt
ln target.txt hardlink.txt
ln -s target.txt symlink.txt
ls -li target.txt hardlink.txt symlink.txt
rm target.txt
cat hardlink.txt
cat symlink.txt || echo "symlink broken"
```

Hard links share the same inode and survive deletion of the original name. Symbolic links are pointer files that break when the target disappears.

---

## Step 7 — Device files in /dev

```bash
ls -l /dev/sda /dev/null /dev/tty1 2>/dev/null
echo "discarded" > /dev/null
head -c 16 /dev/urandom | xxd
```

`/dev/sda` is a block device (b), `/dev/tty1` is a character device (c), `/dev/null` discards writes. The first column of `ls -l` reveals the device class.

---

## Step 8 — Cleanup

```bash
cd /tmp
rm -rf /tmp/lab08
kill %1 2>/dev/null
apt remove -y mlocate lsof
```

`rm -rf` recursively removes the lab tree and `rmdir` (not used here) only removes empty directories.

---

## What you learned
- Navigating, creating, copying, and inspecting files with core utilities
- Searching with `find` predicates and accelerating lookups with `locate`
- Distinguishing hard links, symbolic links, and special device files

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- GNU coreutils & findutils — https://www.gnu.org/software/coreutils/
- lsof — https://github.com/lsof-org/lsof
