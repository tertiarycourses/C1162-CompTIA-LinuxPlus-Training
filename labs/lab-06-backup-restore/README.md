# Lab 6 — Backup & Restore

In this lab you will create tar, cpio, gzip/bzip2/xz, and dd-based archives; pull a disk image off a loopback device with `dd` and recover a damaged one with `ddrescue`; replicate trees with `rsync`; verify integrity with `sha256sum`; and read compressed files in place with `zcat`/`zless`/`zgrep`. By the end you will know which tool fits which backup shape. Maps to Linux+ XK0-006 Objective 1.6 (backup, restore, and compression).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Real backup media are unavailable; we use loopback files and local directories to simulate source and destination.

---

## Step 1 — Install tooling and create sample data

```bash
apt-get update -qq
apt-get install -y tar cpio gzip bzip2 xz-utils rsync gddrescue coreutils >/dev/null
mkdir -p /lab6/src/{etc,logs,docs}
cp -r /etc/hostname /etc/os-release /lab6/src/etc/
for i in 1 2 3 4 5; do
  dd if=/dev/urandom of=/lab6/src/logs/log${i}.bin bs=1K count=20 status=none
done
echo "important document" > /lab6/src/docs/notes.txt
find /lab6/src -type f
```

Creates a small fake tree with config files, random binary "logs", and a text document — enough to exercise every archiver.

---

## Step 2 — tar with gzip, bzip2, and xz

```bash
cd /lab6
tar -czvf src.tar.gz  src/ 2>&1 | tail -5
tar -cjvf src.tar.bz2 src/ 2>&1 | tail -2
tar -cJvf src.tar.xz  src/ 2>&1 | tail -2
ls -lh src.tar.*
tar -tzf src.tar.gz | head -5
mkdir -p restore && tar -xzvf src.tar.gz -C restore/ 2>&1 | tail -3
#diff -r src restore/src && echo "restore matches source"
diff -r --exclude=os-release src restore/src
```

Flags: `c` create, `x` extract, `t` list; `z`=gzip, `j`=bzip2, `J`=xz, `v` verbose, `f` file. xz wins on ratio; gzip wins on speed.

---

## Step 3 — cpio (the older archiver)

```bash
cd /lab6
find src -depth -print | cpio -ov > src.cpio 2>/dev/null
ls -lh src.cpio
mkdir cpio-restore && cd cpio-restore
cpio -idv < ../src.cpio 2>&1 | tail -3
cd /lab6
```

`cpio` reads filenames on stdin (typically from `find`) and writes the archive on stdout — that's why initramfs images are cpio archives.

---

## Step 4 — dd a "disk" image to a loopback file

```bash
cd /lab6
truncate -s 64M srcdisk.img
mkfs.ext4 -F srcdisk.img >/dev/null
mkdir -p mnt
LOOP=$(losetup -fP --show srcdisk.img)
mount "$LOOP" mnt
echo "payload" > mnt/file.txt
umount mnt
losetup -d "$LOOP"

dd if=srcdisk.img of=backup.img bs=1M status=progress
ls -lh srcdisk.img backup.img
sha256sum srcdisk.img backup.img
```

`dd` does block-level copy. The two `sha256sum` lines must match — that is the canonical bit-for-bit integrity check.

---

## Step 5 — ddrescue from a damaged image

```bash
cd /lab6
cp backup.img damaged.img
# Punch holes to simulate unreadable sectors
dd if=/dev/zero of=damaged.img bs=1 count=1024 seek=10240 conv=notrunc status=none
ddrescue -d -r3 damaged.img recovered.img recovered.log 2>&1 | tail -5
ls -lh damaged.img recovered.img
cat recovered.log | head
```

`ddrescue` (the GNU one, installed as `gddrescue`) reads what it can and logs which sectors failed, then retries on later passes — much friendlier than plain `dd conv=noerror`.

---

## Step 6 — rsync local-to-local with delete

```bash
mkdir -p /lab6/mirror
rsync -avz /lab6/src/ /lab6/mirror/ | tail -5
echo "new" > /lab6/src/docs/new.txt
rm /lab6/src/logs/log1.bin
rsync -avz --delete /lab6/src/ /lab6/mirror/ | tail -8
#diff -r /lab6/src /lab6/mirror && echo "mirror in sync"
diff -r --exclude=os-release /lab6/src /lab6/mirror

```

Trailing slashes matter: `src/` copies the contents, not the directory itself. `--delete` removes files from the destination that no longer exist in the source — required for a true mirror.

---

## Step 7 — Integrity and reading compressed files

```bash
cd /lab6
sha256sum src.tar.gz src.tar.bz2 src.tar.xz > checksums.txt
cat checksums.txt
sha256sum -c checksums.txt
zcat src.tar.gz | tar -t | head -5
zgrep -a "PRETTY_NAME" src.tar.gz || echo "binary tar - search the extracted tree instead"
gzip -l src.tar.gz
```

`sha256sum -c` validates a checksum file. `zcat`/`zgrep`/`zless` operate on gzip without an explicit decompress step; equivalents exist for bzip2 (`bzcat`) and xz (`xzcat`).

---

## Step 8 — Cleanup

```bash
rm -rf /lab6
apt-get purge -y gddrescue >/dev/null
apt-get autoremove -y >/dev/null
losetup -D 2>/dev/null || true
```

Removes the lab directory and the only package not preinstalled, and detaches any stray loop device.

---

## What you learned
- When to choose `tar`, `cpio`, `dd`, `ddrescue`, or `rsync` for a backup task.
- Tradeoffs among gzip, bzip2, and xz compressors.
- Why `sha256sum -c` belongs in every restore drill.
- The semantics of trailing slashes and `--delete` in `rsync`.

## Free tools used
- GNU tar — https://www.gnu.org/software/tar/
- cpio — https://www.gnu.org/software/cpio/
- GNU ddrescue — https://www.gnu.org/software/ddrescue/
- rsync — https://rsync.samba.org/
- xz-utils — https://tukaani.org/xz/
