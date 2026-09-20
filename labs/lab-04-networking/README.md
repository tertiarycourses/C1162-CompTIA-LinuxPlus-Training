# Lab 4 — Network Configuration

In this lab you will inspect interfaces, addresses, and routes with `ip`; resolve names with `/etc/hosts`, `/etc/resolv.conf`, and `/etc/nsswitch.conf`; write a netplan YAML; and use the standard diagnostic toolbox (`dig`, `nslookup`, `curl`, `ping`, `mtr`, `ss`, `traceroute`, `tcpdump`, `nmap`). By the end you will be able to read a Linux network stack top-to-bottom. Maps to Linux+ XK0-006 Objective 1.4 (network configuration and troubleshooting).

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

Note: Killercoda runs systemd-networkd, not NetworkManager, so `nmcli` may report no devices; we still install it and show the syntax. `netplan try` will warn instead of reverting in a container, so we use `netplan generate` to validate without applying.

---

## Step 1 — Install tools

```bash
apt-get update -qq
apt-get install -y iproute2 iputils-ping dnsutils curl mtr-tiny tcpdump traceroute nmap network-manager netplan.io >/dev/null
```

Installs the full networking toolbox plus `netplan.io` (Ubuntu's renderer-agnostic config tool) and NetworkManager (for `nmcli` syntax).

---

## Step 2 — Interfaces, addresses, routes

```bash
ip -br link
ip -br address
ip route
ip -6 route
cat /proc/net/dev | head
```

`ip -br` is the compact view: state, addresses, MAC. `ip route` shows the IPv4 routing table including the default gateway.

---

## Step 3 — Name resolution stack

```bash
cat /etc/hosts
cat /etc/resolv.conf
cat /etc/nsswitch.conf | grep -E "^hosts"
getent hosts localhost
getent hosts killercoda.com
```

The `hosts:` line in `/etc/nsswitch.conf` controls lookup order (typically `files dns`). `getent` is the right way to test resolution because it uses the full NSS stack, unlike `dig`.

---

## Step 4 — DNS diagnostics

```bash
dig +short killercoda.com
dig +short MX gmail.com
dig +trace example.com 2>&1 | head -20
nslookup example.com
```

`dig` queries DNS directly via libresolv. `+trace` walks from the root servers down — useful when local resolution looks fine but upstream is broken.

---

## Step 5 — NetworkManager and netplan syntax

```bash
nmcli device status 2>&1 | head
nmcli connection show 2>&1 | head
ls /etc/netplan/
cat > /etc/netplan/99-lab.yaml <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    lo:
      addresses:
        - 127.0.0.2/32
EOF
chmod 600 /etc/netplan/99-lab.yaml
netplan generate
echo "netplan YAML validated"
# netplan apply would touch the live stack; we only validate inside the container
```

Netplan converts the YAML in `/etc/netplan/*.yaml` into renderer-specific config (systemd-networkd or NetworkManager). `netplan generate` parses and renders without applying.

---

## Step 6 — Sockets and connectivity tests

```bash
ss -tulnp | head
ss -s
ping -c 3 127.0.0.1
curl -sI https://killercoda.com | head -5
mtr -r -c 3 1.1.1.1 2>&1 | head -10 || traceroute -n -m 5 1.1.1.1
```

`ss` replaces `netstat`; `-tulnp` shows TCP/UDP listening sockets with the owning process. `mtr` combines ping and traceroute and is the go-to for intermittent path issues.

---

## Step 7 — Packet capture and port scan

```bash
timeout 3 tcpdump -i any -nn -c 5 'icmp or port 53' 2>&1 &
sleep 1
ping -c 2 1.1.1.1 >/dev/null
dig +short example.com >/dev/null
wait
nmap -sT -p 1-1024 127.0.0.1 | head -20
```

`tcpdump` captures live packets; we ran it for 3 seconds and triggered ICMP and DNS to populate it. `nmap -sT` is a TCP connect scan against localhost — safe and informative.

---

## Step 8 — Cleanup

```bash
rm -f /etc/netplan/99-lab.yaml
netplan generate
apt-get purge -y mtr-tiny tcpdump traceroute nmap network-manager netplan.io dnsutils >/dev/null
apt-get autoremove -y >/dev/null
```

Removes the lab netplan file (regenerating to clear any stale rendered config) and the packages added for the lab.

---

## What you learned
- The modern `ip` command replaces `ifconfig`/`route`/`arp`.
- The full resolution path: `/etc/nsswitch.conf` -> `/etc/hosts` or `/etc/resolv.conf` -> DNS.
- How to validate a netplan YAML safely with `netplan generate`.
- Which diagnostic to reach for: `ss` for sockets, `dig`/`getent` for DNS, `mtr` for path issues, `tcpdump` for packets, `nmap` for ports.

## Free tools used
- iproute2 — https://wiki.linuxfoundation.org/networking/iproute2
- netplan — https://netplan.io/
- BIND dnsutils (dig/nslookup) — https://www.isc.org/bind/
- tcpdump — https://www.tcpdump.org/
- nmap — https://nmap.org/
