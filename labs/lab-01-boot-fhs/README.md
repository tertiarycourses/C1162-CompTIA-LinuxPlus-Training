# Lab 1 — Boot Process & Filesystem Hierarchy

In this lab you will inspect a running Linux system's boot configuration, kernel command line, initramfs contents, and the standard Filesystem Hierarchy. By the end you will be able to explain how GRUB hands off to the kernel, what the initramfs is for, where major FHS directories live, and how to identify the running distribution and CPU architecture. Maps to Linux+ XK0-006 Objective 1.1 (system boot, FHS, distribution identification).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Killercoda is a containerized environment, so the bootloader on the host cannot be modified or rebooted. We will inspect files and tools as they exist on the running system to learn the layout and syntax.

---

## Step 1 — Identify distribution and architecture

```bash
cat /etc/os-release
uname -a
uname -m
uname -r
hostnamectl 2>/dev/null || true
apt-get update -qq && apt-get install -y lsb-release >/dev/null
lsb_release -a
```

`/etc/os-release` is the standard distribution identification file across modern Linux. `uname -m` shows the machine architecture (x86_64, aarch64), and `uname -r` the running kernel release.

---

## Step 2 — Inspect GRUB configuration

```bash
apt-get install -y grub2-common >/dev/null
ls -l /etc/default/grub 2>/dev/null || echo "no /etc/default/grub in this container"
cat /etc/default/grub 2>/dev/null || true
ls /boot 2>/dev/null
find / -name "grub.cfg" 2>/dev/null | head
```

`/etc/default/grub` is the human-edited GRUB defaults file; `update-grub` regenerates `/boot/grub/grub.cfg` from it. In this container `/boot` is mostly empty because the host kernel is reused.

---

## Step 3 — Inspect the kernel command line

```bash
cat /proc/cmdline
cat /proc/version
ls /proc/sys/kernel/ | head
cat /proc/sys/kernel/hostname
```

`/proc/cmdline` shows the exact arguments GRUB passed to the running kernel (root device, console, quiet, splash, etc.). On a real host you would see `root=UUID=...` here.

---

## Step 4 — Explore the initramfs

```bash
apt-get install -y initramfs-tools >/dev/null
ls /boot/initrd* 2>/dev/null || echo "no initrd image present in container"
# Pull a sample initrd image from the Ubuntu archive to inspect
apt-get install -y wget binutils cpio >/dev/null
mkdir -p /tmp/initrd && cd /tmp/initrd
apt-get download linux-image-generic 2>/dev/null || true
ls
lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | head -30 || echo "use lsinitramfs <file> on a real host"
```

The initramfs is a small cpio archive the kernel mounts as the first root filesystem; it loads storage and crypto modules so the real root can be mounted. `lsinitramfs` lists its contents without extracting.

---

## Step 5 — Tour the Filesystem Hierarchy (FHS)

```bash
for d in /bin /boot /dev /etc /home /lib /proc /sbin /sys /tmp /usr /var /run /opt /srv; do
  printf "%-8s -> " "$d"
  ls -ld "$d" 2>/dev/null | awk '{print $1, $NF}'
done
ls /usr
ls /var
file /bin /sbin /lib 2>/dev/null
```

`/bin`, `/sbin`, and `/lib` on modern Ubuntu are symlinks into `/usr` (the "usrmerge"). `/proc` and `/sys` are virtual kernel filesystems, `/dev` is device nodes, `/etc` is system configuration, and `/var` holds variable data like logs and spool files.

---

## Step 6 — Walk through key boot-related dirs

```bash
ls /etc/systemd/system/ | head
systemctl list-units --type=target --no-pager 2>/dev/null | head
ls /lib/modules/$(uname -r) 2>/dev/null | head || ls /lib/modules | head
cat /proc/mounts | head
```

`systemd` targets replace classic SysV runlevels; `multi-user.target` and `graphical.target` are the common end states. `/lib/modules/<kernel>` is where loadable kernel modules live.

---

## Step 7 — Cleanup

```bash
rm -rf /tmp/initrd
apt-get purge -y lsb-release grub2-common wget binutils 2>/dev/null
apt-get autoremove -y >/dev/null
```

Removes packages installed for the lab and the temporary download directory.

---

## What you learned
- How GRUB, the kernel command line, and the initramfs cooperate to bring up userspace.
- Where each top-level FHS directory lives and what it holds.
- How to identify a running distribution, kernel, and CPU architecture from standard files and tools.
- Why `/bin`, `/sbin`, and `/lib` are symlinks into `/usr` on modern Ubuntu (usrmerge).

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- GNU coreutils & util-linux — https://www.gnu.org/software/coreutils/
- initramfs-tools — https://packages.ubuntu.com/jammy/initramfs-tools
- systemd — https://systemd.io/
