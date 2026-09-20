# Lab 7 — Virtualization with QEMU/KVM & libvirt

In this lab you will install QEMU and libvirt, manipulate virtual disk images with `qemu-img`, list libvirt domains and networks with `virsh`, and run a tiny guest under `qemu-system-x86_64` in pure-software (TCG) mode. By the end you will be able to discuss hypervisor types, disk-image formats, snapshots, and libvirt's bridged/NAT/host-only network modes. Maps to Linux+ XK0-006 Objective 1.7 (virtualization concepts and tooling).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Killercoda usually does not expose `/dev/kvm`, so KVM acceleration will be unavailable. We will detect that and fall back to TCG (pure software emulation), which is slower but functionally identical for the purposes of this lab.

---

## Step 1 — Install QEMU and libvirt

```bash
apt-get update -qq
apt-get install -y qemu-system-x86 qemu-utils libvirt-clients libvirt-daemon-system bridge-utils >/dev/null
ls -l /dev/kvm 2>/dev/null && echo "KVM available" || echo "KVM NOT available - will use TCG software emulation"
qemu-system-x86_64 --version | head -1
virsh --version
```

`qemu-system-x86` is the emulator binary; `libvirt-daemon-system` is the management daemon that `virsh` talks to. KVM is the in-kernel accelerator; without `/dev/kvm` QEMU falls back to TCG.

---

## Step 2 — Create and inspect disk images

```bash
mkdir -p /lab7 && cd /lab7
qemu-img create -f qcow2 disk.qcow2 1G
qemu-img create -f raw   disk.raw   256M
qemu-img info disk.qcow2
qemu-img info disk.raw
ls -lh disk.*
```

`qcow2` is QEMU's native sparse, snapshottable format; `raw` is a flat byte-for-byte image. `qcow2` files start tiny and grow on write.

---

## Step 3 — Convert and resize images

```bash
cd /lab7
qemu-img convert -O raw disk.qcow2 disk-from-qcow.raw
qemu-img convert -O qcow2 disk.raw disk-from-raw.qcow2
ls -lh disk*
qemu-img resize disk.qcow2 +512M
qemu-img info disk.qcow2 | grep -E "virtual size|disk size"
```

`qemu-img convert -O <fmt>` is the universal disk-format converter — VMDK, VDI, qcow2, raw all interconvert. `resize` only changes the virtual size; the guest must then grow its partition and filesystem.

---

## Step 4 — Snapshots inside a qcow2

```bash
cd /lab7
qemu-img snapshot -c clean disk.qcow2
qemu-img snapshot -l disk.qcow2
qemu-img snapshot -c after-config disk.qcow2
qemu-img snapshot -l disk.qcow2
qemu-img snapshot -d clean disk.qcow2
qemu-img snapshot -l disk.qcow2
```

`qemu-img snapshot` operates on internal qcow2 snapshots: `-c` create, `-l` list, `-a` apply, `-d` delete. libvirt domains also support external snapshots that span multiple files.

---

## Step 5 — libvirt: domains and networks

```bash
systemctl start libvirtd 2>/dev/null || service libvirtd start 2>/dev/null || true
sleep 2
virsh list --all
virsh net-list --all
virsh net-dumpxml default 2>/dev/null | head -20 || echo "default network not autostarted in container"
virsh capabilities 2>&1 | head -20
```

`virsh list --all` shows all defined domains (running or stopped). The `default` libvirt network is a NAT network on `virbr0`; bridged networks tie a guest NIC directly to a physical interface; host-only networks have no upstream gateway.

---

## Step 6 — Boot a minimal guest under QEMU (TCG fallback)

```bash
cd /lab7
apt-get install -y wget >/dev/null
# A tiny ~17MB Linux kernel + initrd from the Buildroot/Alpine ecosystem would normally go here.
# To keep the lab offline-friendly we boot QEMU into its built-in firmware and stop after 8 seconds.
ACCEL="tcg"
[ -e /dev/kvm ] && ACCEL="kvm"
echo "Using accel=$ACCEL"
timeout 8 qemu-system-x86_64 \
  -machine accel=$ACCEL \
  -m 128 \
  -nographic \
  -serial mon:stdio \
  -drive file=disk.qcow2,if=virtio,format=qcow2 \
  -boot order=c 2>&1 | tail -20 || true
echo "qemu exited (timeout expected)"
```

We start QEMU headless with serial-on-stdio. Without a bootable disk it lands in SeaBIOS searching for boot media — enough to prove the emulator works end-to-end. On a real workstation with `/dev/kvm` the `-machine accel=kvm` switch flips on hardware acceleration.

---

## Step 7 — Discuss network modes (concept)

```bash
cat <<'EOF'
Bridged   : guest NIC attached to a software bridge that also contains the host's
            physical NIC. Guest is a peer on the LAN with its own IP from DHCP.
NAT       : guest sits on libvirt's virbr0; host masquerades outbound traffic.
            Inbound requires explicit port forwarding. This is libvirt's default.
Host-only : guest is on an isolated bridge with no upstream. Host can reach guest;
            guest cannot reach the internet. Useful for offline test labs.
EOF
ip link show type bridge 2>/dev/null
brctl show 2>/dev/null || echo "brctl is legacy; 'ip link' is preferred"
```

These three modes map directly to the same concepts in VirtualBox and VMware — exam questions often phrase them as "which mode lets the VM appear as a peer on the office LAN?" (bridged).

---

## Step 8 — Cleanup

```bash
virsh net-destroy default 2>/dev/null || true
systemctl stop libvirtd 2>/dev/null || service libvirtd stop 2>/dev/null || true
rm -rf /lab7
apt-get purge -y qemu-system-x86 qemu-utils libvirt-clients libvirt-daemon-system bridge-utils wget >/dev/null
apt-get autoremove -y >/dev/null
```

Stops libvirt, removes the lab directory, and purges every package added.

---

## What you learned
- The difference between QEMU (emulator) and KVM (accelerator), and how to detect each.
- The role of `qcow2` vs `raw`, and how `qemu-img convert` bridges every common format.
- Internal qcow2 snapshots vs libvirt external snapshots.
- The three classic guest network modes — bridged, NAT, host-only — and when each applies.

## Free tools used
- QEMU — https://www.qemu.org/
- libvirt — https://libvirt.org/
- KVM — https://www.linux-kvm.org/
- bridge-utils / iproute2 — https://wiki.linuxfoundation.org/networking/bridge
