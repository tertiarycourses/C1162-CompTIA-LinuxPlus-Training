# Lab 2 — Kernel Modules & Device Management

In this lab you will list, inspect, load, and unload Linux kernel modules and use the standard hardware-enumeration tools (`lspci`, `lsusb`, `lscpu`, `lshw`, `lsmem`, `lsblk`, `dmidecode`). By the end you will be able to explain the role of `modprobe` vs `insmod`, where module dependencies are stored, and how to read `dmesg` to confirm a module loaded. Maps to Linux+ XK0-006 Objective 1.2 (kernel modules and device management).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Killercoda shares the host kernel, so module loading sometimes fails with `Operation not permitted` for security-sensitive modules. The `dummy` and `loop` network/block modules are commonly permitted; we use those for the load/unload demo.

---

## Step 1 — Install hardware inspection tools

```bash
apt-get update -qq
apt-get install -y pciutils usbutils hwinfo lshw dmidecode kmod >/dev/null
```

Installs the standard discovery utilities. `kmod` provides `lsmod`, `modprobe`, `modinfo`, `insmod`, `rmmod`, and `depmod`.

---

## Step 2 — List loaded modules and inspect one

```bash
lsmod | head -20
lsmod | wc -l
MOD=$(lsmod | awk 'NR==2 {print $1}')
echo "Inspecting module: $MOD"
modinfo "$MOD" | head -20
```

`lsmod` reads `/proc/modules` and shows currently loaded modules with their size and dependents. `modinfo` shows metadata baked into the `.ko` file (description, author, parameters, dependencies).

---

## Step 3 — Load and unload a module

```bash
modprobe dummy 2>&1 || modprobe loop 2>&1 || echo "module load blocked in this sandbox"
lsmod | grep -E "dummy|loop" | head
ip link add dummy0 type dummy 2>/dev/null && ip link show dummy0
ip link del dummy0 2>/dev/null
modprobe -r dummy 2>&1 || true
```

`modprobe` resolves dependencies and loads the module, where `insmod` would only load the exact `.ko` file you point at. `modprobe -r` removes it; `rmmod` is the lower-level equivalent.

---

## Step 4 — Module dependencies and depmod

```bash
ls /lib/modules/$(uname -r) 2>/dev/null | head
cat /lib/modules/$(uname -r)/modules.dep 2>/dev/null | head -5
depmod -a 2>&1 | head || echo "depmod requires module tree for running kernel"
modprobe --show-depends loop 2>&1 | head
```

`depmod` scans `/lib/modules/<kernel>` and writes `modules.dep`, the dependency map used by `modprobe`. On a containerized host the kernel modules may not be present.

---

## Step 5 — Read the kernel ring buffer

```bash
dmesg | tail -20 2>/dev/null || journalctl -k --no-pager | tail -20
dmesg -T 2>/dev/null | tail -5 || true
```

`dmesg` shows the kernel ring buffer where module load events, hardware probe results, and driver errors appear. `-T` formats timestamps as human-readable dates.

---

## Step 6 — Enumerate hardware

```bash
lscpu
echo "---"
lsmem 2>/dev/null | head || free -h
echo "---"
lspci | head
echo "---"
lsusb 2>/dev/null | head || echo "no usb bus in container"
echo "---"
lsblk
echo "---"
lshw -short 2>/dev/null | head -20
echo "---"
dmidecode -s system-manufacturer 2>/dev/null || echo "dmidecode requires /dev/mem on real hardware"
```

Each `ls*` tool reads a different kernel interface: `lscpu` parses `/proc/cpuinfo`, `lspci` reads `/sys/bus/pci`, `lsblk` reads `/sys/block`. `dmidecode` reads SMBIOS from `/dev/mem` and may be empty in a container.

---

## Step 7 — initramfs builders overview

```bash
which dracut 2>/dev/null || echo "dracut not installed (used on RHEL/Fedora/SUSE)"
which mkinitramfs && mkinitramfs --help 2>&1 | head
which update-initramfs && update-initramfs --help 2>&1 | head
```

Debian/Ubuntu use `mkinitramfs`/`update-initramfs`; RHEL family uses `dracut`. Both rebuild the initramfs after installing modules or kernel updates.

---

## Step 8 — Cleanup

```bash
ip link del dummy0 2>/dev/null || true
modprobe -r dummy 2>/dev/null || true
apt-get purge -y pciutils usbutils hwinfo lshw dmidecode >/dev/null
apt-get autoremove -y >/dev/null
```

Removes added packages and any leftover virtual interface.

---

## What you learned
- The difference between `insmod`, `modprobe`, `rmmod`, and `depmod`.
- How `lsmod`, `modinfo`, and `/proc/modules` expose the running module set.
- Which `ls*` tool to reach for given a hardware-discovery question on the exam.
- That `dracut` and `mkinitramfs` solve the same problem on different distros.

## Free tools used
- kmod (lsmod/modprobe/modinfo) — https://git.kernel.org/pub/scm/utils/kernel/kmod/kmod.git
- pciutils & usbutils — https://mj.ucw.cz/sw/pciutils/
- lshw — https://ezix.org/project/wiki/HardwareLiSter
- dmidecode — https://www.nongnu.org/dmidecode/
