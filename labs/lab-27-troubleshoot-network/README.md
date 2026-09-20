# Lab 27 — Troubleshooting Network Connectivity

This lab covers CompTIA Linux+ XK0-006 Objective 5.3. You will break and repair DNS resolution, default routes, firewall rules, MTU, IP assignment, link state, and dual-stack behavior. Throughout, you will practice the standard triage toolkit: `ss`, `tcpdump`, `mtr`, `traceroute`, `dig`, `nmap`.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Baseline and install tools

```bash
apt update
apt install -y dnsutils iproute2 iputils-ping mtr-tiny traceroute nmap tcpdump ufw netcat-openbsd
ip -br addr
ip route
cat /etc/resolv.conf
ss -tulpn | head
```

A baseline snapshot of interfaces, routes, resolver, and listening sockets is the single most useful artifact during an incident.

---

## Step 2 — Break: DNS, then fix

```bash
cp /etc/resolv.conf /tmp/resolv.conf.bak
sed -i 's/^nameserver/#nameserver/' /etc/resolv.conf
dig +time=2 +tries=1 example.com || echo "DNS broken as expected"
echo 'nameserver 8.8.8.8' > /etc/resolv.conf
dig +short example.com
cp /tmp/resolv.conf.bak /etc/resolv.conf
```

Commenting nameservers reproduces the most common "internet is down" report; restoring with a public resolver confirms DNS is the only fault.

---

## Step 3 — Break: default gateway, then revert

```bash
ORIG=$(ip route show default)
echo "Saving: $ORIG"
ip route del default 2>/dev/null
ip route add default via 10.10.10.10 2>/dev/null || true
ip route
ping -c 2 -W 2 1.1.1.1 || echo "No route as expected"
ip route del default 2>/dev/null
ip route add $ORIG
ping -c 2 -W 2 1.1.1.1
```

A wrong gateway looks identical to a cable being unplugged from outside the host. Saving the original route string is the safe recovery pattern.

---

## Step 4 — Break: firewall blocks a port

```bash
ufw --force enable
ufw deny 8080/tcp
python3 -m http.server 8080 >/tmp/http.log 2>&1 &
HTTP=$!
sleep 2
IF=$(ip -br link | awk '$1!="lo" && $2=="UP" {print $1; exit}')
curl -s --max-time 3 --interface lo http://127.0.0.1:8080/ | head -c 60
echo
curl -s --max-time 3 --interface "$IF" http://127.0.0.1:8080/ || echo "Blocked on $IF as expected"
ufw delete deny 8080/tcp
ufw --force disable
kill $HTTP 2>/dev/null
```

Loopback bypasses ufw; the physical interface does not. Comparing the two pinpoints firewall rules as the culprit.

---

## Step 5 — Break: MTU mismatch

```bash
IF=$(ip -br link | awk '$1!="lo" && $2=="UP" {print $1; exit}')
ORIG_MTU=$(cat /sys/class/net/$IF/mtu)
ip link set $IF mtu 1300
ping -M do -s 1400 -c 2 -W 2 1.1.1.1 || echo "Fragmentation needed as expected"
ip link set $IF mtu $ORIG_MTU
ping -M do -s 1400 -c 2 -W 2 1.1.1.1
```

`-M do` sets the don't-fragment bit. When payload exceeds MTU, the kernel refuses to send — exactly the symptom of a VPN or tunnel MTU mismatch.

---

## Step 6 — Break: duplicate IP

```bash
IF=$(ip -br link | awk '$1!="lo" && $2=="UP" {print $1; exit}')
PRIMARY=$(ip -4 -br addr show $IF | awk '{print $3}')
ip addr add 10.99.99.99/24 dev $IF
ip -br addr show $IF
arping -c 1 -I $IF 10.99.99.99 2>/dev/null || true
ip addr del 10.99.99.99/24 dev $IF
```

Adding a secondary address lets you observe the multi-IP layout. In production, duplicate IPs surface as intermittent ARP flapping caught by `arping`.

---

## Step 7 — Break: link down on a dummy interface

```bash
modprobe dummy
ip link add lab0 type dummy
ip link set lab0 up
ip addr add 10.55.55.1/24 dev lab0
ip -br link show lab0
ip link set lab0 down
ip -br link show lab0
ip link set lab0 up
ip link del lab0
```

A dummy interface lets you safely practice link toggling. Real interfaces follow the same `ip link set ... up/down` workflow.

---

## Step 8 — IPv4 vs IPv6 and capture

```bash
curl -4 -s --max-time 3 https://ifconfig.co || true
curl -6 -s --max-time 3 https://ifconfig.co || true
ss -tulpn
mtr -rwc 3 1.1.1.1 | head
traceroute -n -m 5 1.1.1.1 | head
timeout 3 tcpdump -i any -nn -c 5 'icmp' &
ping -c 3 1.1.1.1 >/dev/null
wait
nmap -sT -p 22,80,443 127.0.0.1
```

`curl -4`/`-6` forces address family selection — useful when one stack is broken. `tcpdump`, `mtr`, `traceroute`, and `nmap` finish the standard L3/L4 triage toolkit.

---

## Step 9 — Cleanup

```bash
ufw --force disable
ip link del lab0 2>/dev/null
rm -f /tmp/http.log /tmp/resolv.conf.bak
apt purge -y nmap tcpdump mtr-tiny traceroute >/dev/null 2>&1 || true
apt autoremove -y
```

Disables ufw, removes the dummy interface, and drops the diagnostic packages that were added for the lab.

---

## What you learned
- Isolating DNS, routing, firewall, and MTU faults using single-variable changes
- Comparing `--interface lo` vs the physical NIC to identify firewall blocks
- Validating IPv4 vs IPv6 reachability independently with `curl -4`/`-6`
- Standard triage toolkit: `ss`, `tcpdump`, `mtr`, `traceroute`, `nmap`, `dig`

## Free tools used
- iproute2 — https://wiki.linuxfoundation.org/networking/iproute2
- bind9-dnsutils (dig) — https://www.isc.org/bind/
- tcpdump — https://www.tcpdump.org
- mtr — https://www.bitwizard.nl/mtr/
- nmap — https://nmap.org
- ufw — https://launchpad.net/ufw
