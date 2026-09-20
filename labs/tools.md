# Tools Index — CompTIA Linux+ XK0-006 Labs

Every free tool used across labs 1–30, grouped by category. Each row is: **name — one-line purpose — install command — homepage**.

All install commands are for Ubuntu 22.04 on the Killercoda Playground (https://killercoda.com/playgrounds/scenario/ubuntu) unless noted otherwise.

---

## Boot, Storage, and Filesystems

- **GRUB** — boot loader for x86/EFI Linux systems — `apt install grub2-common` — https://www.gnu.org/software/grub/
- **lvm2** — logical volume manager (PV/VG/LV) — `apt install lvm2` — https://sourceware.org/lvm2/
- **parted** — script-friendly partition editor — `apt install parted` — https://www.gnu.org/software/parted/
- **blkid / util-linux** — identify block-device UUIDs and types — `apt install util-linux` — https://github.com/util-linux/util-linux
- **rsync** — incremental file/dir sync over local or SSH — `apt install rsync` — https://rsync.samba.org
- **tar** — archive packer used for backups — `apt install tar` — https://www.gnu.org/software/tar/
- **gzip** — fast stream compressor — `apt install gzip` — https://www.gnu.org/software/gzip/
- **xz-utils** — high-ratio LZMA compressor — `apt install xz-utils` — https://tukaani.org/xz/
- **ddrescue** — fault-tolerant disk imager for failing media — `apt install gddrescue` — https://www.gnu.org/software/ddrescue/

## Networking

- **NetworkManager** — desktop/server network config daemon — `apt install network-manager` — https://networkmanager.dev
- **netplan.io** — YAML front-end for systemd-networkd/NM on Ubuntu — `apt install netplan.io` — https://netplan.io
- **iproute2** — modern `ip`, `ss`, `tc` networking suite — `apt install iproute2` — https://wiki.linuxfoundation.org/networking/iproute2
- **dig (dnsutils)** — DNS query and trace tool — `apt install dnsutils` — https://www.isc.org/bind/
- **curl** — scriptable HTTP/S, FTP, and more — `apt install curl` — https://curl.se
- **mtr** — combined ping/traceroute for live path quality — `apt install mtr-tiny` — https://www.bitwizard.nl/mtr/
- **traceroute** — hop-by-hop path discovery — `apt install traceroute` — https://traceroute.sourceforge.net
- **nmap** — port scanner and service detector — `apt install nmap` — https://nmap.org
- **tcpdump** — packet capture from the command line — `apt install tcpdump` — https://www.tcpdump.org

## Virtualization and Containers

- **qemu-kvm** — full-system emulator with KVM acceleration — `apt install qemu-kvm` — https://www.qemu.org
- **libvirt-clients** — `virsh` CLI for managing VMs — `apt install libvirt-clients` — https://libvirt.org
- **qemu-utils** — `qemu-img` for disk image management — `apt install qemu-utils` — https://www.qemu.org
- **docker.io** — container runtime and image tooling — `apt install docker.io` — https://www.docker.com
- **podman** — daemonless OCI container engine — `apt install podman` — https://podman.io

## Security and Firewalling

- **ufw** — uncomplicated firewall front-end for nftables — `apt install ufw` — https://launchpad.net/ufw
- **firewalld** — zone-based firewall (Rocky/RHEL container) — `dnf install firewalld` — https://firewalld.org
- **nftables** — modern Linux packet filter — `apt install nftables` — https://www.nftables.org
- **iptables** — legacy packet filter (nft-backed on Ubuntu) — `apt install iptables` — https://www.netfilter.org
- **ipset** — kernel-level IP/port set matching — `apt install ipset` — https://ipset.netfilter.org
- **OpenSSH** — SSH server and client — `apt install openssh-server openssh-client` — https://www.openssh.com
- **fail2ban** — jails brute-force attackers from logs — `apt install fail2ban` — https://www.fail2ban.org
- **libpam-pwquality** — password complexity for PAM — `apt install libpam-pwquality` — https://github.com/libpwquality/libpwquality
- **libpam-google-authenticator** — TOTP MFA for SSH/login — `apt install libpam-google-authenticator` — https://github.com/google/google-authenticator-libpam
- **gnupg** — OpenPGP keys, signing, encryption — `apt install gnupg` — https://gnupg.org
- **cryptsetup** — LUKS full-disk encryption — `apt install cryptsetup` — https://gitlab.com/cryptsetup/cryptsetup
- **openssl** — TLS, x509, and crypto CLI — `apt install openssl` — https://www.openssl.org
- **wireguard-tools** — modern kernel VPN userspace — `apt install wireguard-tools` — https://www.wireguard.com
- **aide** — file-integrity database — `apt install aide` — https://aide.github.io
- **rkhunter** — rootkit signature scanner — `apt install rkhunter` — https://rkhunter.sourceforge.net
- **debsums** — verify installed deb file checksums — `apt install debsums` — https://manpages.debian.org/debsums
- **lynis** — host security audit framework — `apt install lynis` — https://cisofy.com/lynis/

## Automation, Configuration Management, and Dev Tooling

- **Ansible** — agentless config management — `apt install ansible` — https://www.ansible.com
- **OpenTofu** — open-source Terraform-compatible IaC — `curl https://get.opentofu.org/install-opentofu.sh | sh` — https://opentofu.org
- **Puppet (optional)** — declarative config management — `apt install puppet` — https://www.puppet.com
- **git** — distributed version control — `apt install git` — https://git-scm.com
- **python3** — Python 3 runtime — `apt install python3` — https://www.python.org
- **python3-venv** — isolated Python environments — `apt install python3-venv` — https://docs.python.org/3/library/venv.html
- **pip** — Python package installer — `apt install python3-pip` — https://pip.pypa.io
- **shellcheck** — static analyzer for shell scripts — `apt install shellcheck` — https://www.shellcheck.net
- **black** — opinionated Python formatter — `pip install black` — https://black.readthedocs.io
- **flake8** — Python lint aggregator — `pip install flake8` — https://flake8.pycqa.org

## Monitoring and Performance

- **snmpd / snmp** — SNMP agent and walker — `apt install snmpd snmp` — http://www.net-snmp.org
- **Prometheus** — pull-based time-series monitoring — `apt install prometheus` — https://prometheus.io
- **prometheus-node-exporter** — host metrics exporter — `apt install prometheus-node-exporter` — https://github.com/prometheus/node_exporter
- **sysstat** — sar/iostat/mpstat/pidstat collectors — `apt install sysstat` — https://github.com/sysstat/sysstat
- **stress-ng** — synthetic CPU/mem/IO/network load — `apt install stress-ng` — https://github.com/ColinIanKing/stress-ng
- **fio** — flexible I/O benchmark — `apt install fio` — https://fio.readthedocs.io
- **htop** — interactive process viewer — `apt install htop` — https://htop.dev
- **iotop** — per-process disk I/O viewer — `apt install iotop` — http://guichaz.free.fr/iotop/
- **iftop** — per-connection bandwidth viewer — `apt install iftop` — https://www.ex-parrot.com/pdw/iftop/

## AI and Adjacent

- **python3 + pip** — runtime for local AI tooling (LangChain, etc.) — `apt install python3 python3-pip` — https://www.python.org
- **ollama** — local LLM runtime — `curl -fsSL https://ollama.com/install.sh | sh` — https://ollama.com
- **llama.cpp** — CPU/GPU inference for GGUF models — `pip install llama-cpp-python` — https://github.com/ggerganov/llama.cpp

---

## Notes

- All `apt` commands assume an Ubuntu 22.04 root shell on Killercoda.
- `firewalld` is demonstrated inside a Rocky Linux container (see Lab 28) because Ubuntu ships ufw/nftables by default.
- Tools marked `pip install` should be installed inside a `python3 -m venv` virtualenv on real systems to avoid clobbering the system Python.
