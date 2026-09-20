# Lab 3 — Storage with LVM, Partitions & Filesystems

In this lab you will build a complete LVM stack on top of three loopback-backed "disks", partition them, create physical volumes, a volume group, and logical volumes, then format with ext4, xfs, and btrfs, mount with security-relevant options, and grow a filesystem online. By the end you will be able to walk from raw block device to a mounted, expandable filesystem. Maps to Linux+ XK0-006 Objective 1.3 (storage, partitions, LVM, filesystems, fstab).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Killercoda has no spare physical disks, so we simulate three 512 MB disks using loopback devices backed by sparse files.

---

## Step 1 — Install tooling and create three loopback "disks"

```bash
apt-get update -qq
apt-get install -y lvm2 xfsprogs btrfs-progs parted util-linux >/dev/null
mkdir -p /lab/disks && cd /lab/disks
for i in 1 2 3; do
  truncate -s 512M disk${i}.img
  losetup -fP disk${i}.img
done
losetup -a
lsblk | grep loop
```

`truncate -s` creates a sparse file consuming no real space until written. `losetup -fP` attaches the next free `/dev/loopN` device and scans for partitions.

---

## Step 2 — Partition the first loopback disk

```bash
LOOP1=$(losetup -j /lab/disks/disk1.img | cut -d: -f1)
echo "Disk 1 -> $LOOP1"
parted -s "$LOOP1" mklabel gpt
parted -s "$LOOP1" mkpart primary 1MiB 100%
parted -s "$LOOP1" set 1 lvm on
partprobe "$LOOP1"
lsblk "$LOOP1"
```

`parted` writes a GPT label and one partition spanning the disk, then flags it as LVM. `partprobe` re-reads the partition table so the kernel exposes `${LOOP1}p1`.

---

## Step 3 — Create PVs, a VG, and LVs

```bash
LOOP2=$(losetup -j /lab/disks/disk2.img | cut -d: -f1)
LOOP3=$(losetup -j /lab/disks/disk3.img | cut -d: -f1)
pvcreate "${LOOP1}p1" "$LOOP2" "$LOOP3"
pvs
vgcreate labvg "${LOOP1}p1" "$LOOP2" "$LOOP3"
vgs
lvcreate -L 200M -n lv_ext4 labvg
lvcreate -L 200M -n lv_xfs  labvg
lvcreate -L 200M -n lv_btrfs labvg
lvs
```

`pvcreate` writes LVM metadata onto each block device, `vgcreate` pools them into one volume group, and `lvcreate` carves logical volumes out of that pool.

---

## Step 4 — Make filesystems

```bash
mkfs.ext4  -F /dev/labvg/lv_ext4
mkfs.xfs   -f /dev/labvg/lv_xfs
mkfs.btrfs -f /dev/labvg/lv_btrfs
blkid /dev/labvg/lv_ext4 /dev/labvg/lv_xfs /dev/labvg/lv_btrfs
```

`mkfs.<type>` writes a superblock. `blkid` reports UUID and TYPE — these UUIDs are what you reference in `/etc/fstab`.

---

## Step 5 — Mount with hardening options and persist in fstab

```bash
mkdir -p /mnt/ext4 /mnt/xfs /mnt/btrfs
mount -o noatime,nodev,nosuid,noexec /dev/labvg/lv_ext4  /mnt/ext4
mount -o noatime,nodev,nosuid         /dev/labvg/lv_xfs   /mnt/xfs
mount -o noatime,compress=zstd        /dev/labvg/lv_btrfs /mnt/btrfs
mount | grep labvg
UUID=$(blkid -s UUID -o value /dev/labvg/lv_ext4)
echo "UUID=$UUID  /mnt/ext4  ext4  noatime,nodev,nosuid,noexec  0 2" >> /etc/fstab
tail -1 /etc/fstab
df -hT /mnt/ext4 /mnt/xfs /mnt/btrfs
du -sh /mnt/*
```

`noatime` skips access-time writes, `nodev`/`nosuid`/`noexec` defang the filesystem against device nodes, setuid binaries, and executables — appropriate hardening for data-only mounts.

---

## Step 6 — Grow a logical volume and its filesystem online

```bash
lvextend -L +100M /dev/labvg/lv_ext4
resize2fs /dev/labvg/lv_ext4
df -hT /mnt/ext4

lvextend -L +100M /dev/labvg/lv_xfs
xfs_growfs /mnt/xfs
df -hT /mnt/xfs

# lvresize is the unified verb; -r auto-runs the filesystem resize
lvresize -L +50M -r /dev/labvg/lv_btrfs
df -hT /mnt/btrfs
```

`resize2fs` grows ext4; `xfs_growfs` grows XFS (XFS cannot shrink). `lvresize -r` does the LV and FS in one shot.

---

## Step 7 — Cleanup

```bash
umount /mnt/ext4 /mnt/xfs /mnt/btrfs
sed -i '/labvg/d' /etc/fstab
lvremove -y labvg
vgremove -y labvg
pvremove -y "${LOOP1}p1" "$LOOP2" "$LOOP3" 2>/dev/null
for L in $(losetup -j /lab/disks/disk1.img /lab/disks/disk2.img /lab/disks/disk3.img | cut -d: -f1); do
  losetup -d "$L"
done
rm -rf /lab /mnt/ext4 /mnt/xfs /mnt/btrfs
apt-get purge -y lvm2 xfsprogs btrfs-progs parted >/dev/null
apt-get autoremove -y >/dev/null
```

Reverses every operation: unmount, remove LVs/VG/PVs, detach loop devices, delete files, purge packages.

---

## What you learned
- The PV-VG-LV layering that lets storage be pooled and resliced without downtime.
- How to create ext4, XFS, and btrfs filesystems and reference them by UUID in `/etc/fstab`.
- Which mount options harden a filesystem and what each one disables.
- That `xfs_growfs` only grows, while `resize2fs` and `btrfs filesystem resize` can grow or shrink (XFS cannot shrink).

## Free tools used
- LVM2 — https://sourceware.org/lvm2/
- xfsprogs — https://xfs.org/
- btrfs-progs — https://btrfs.readthedocs.io/
- parted — https://www.gnu.org/software/parted/
