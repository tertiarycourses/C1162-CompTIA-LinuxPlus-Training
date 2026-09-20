# Lab 15 — Firewalls: ufw, firewalld, iptables, nftables, ipset

This lab maps to Linux+ XK0-006 Objective 3.2. You will configure `ufw` on Ubuntu, exercise `firewall-cmd` zones and rich rules inside a Rocky container, inspect `iptables` and `nftables` chains, build an `ipset`, sketch SNAT and DNAT examples, and toggle IP forwarding. Stateful firewalls track connection state (NEW/ESTABLISHED/RELATED) while stateless ones evaluate each packet independently.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install and configure ufw

```bash
apt update && apt install -y ufw
ufw allow 22/tcp
ufw deny 23/tcp
yes | ufw enable
ufw status verbose
```

`ufw` is a friendly wrapper around netfilter. The verbose status shows defaults, logging level, and rule order.

---

## Step 2 — Explore iptables backing ufw

```bash
iptables -L -n -v
iptables -t nat -L -n -v
```

`ufw` writes its rules into the `filter` table chains (`ufw-*`). The `nat` table is empty until you add NAT or masquerade rules.

---

## Step 3 — firewalld with zones and rich rules in a Rocky container

```bash
apt install -y docker.io
systemctl start docker
docker run --rm --privileged -it rockylinux:9 bash -c '
  dnf install -y firewalld iproute && \
  firewall-offline-cmd --zone=public --add-service=https && \
  firewall-offline-cmd --zone=public --add-rich-rule="rule family=ipv4 source address=10.0.0.0/24 service name=ssh accept" && \
  firewall-offline-cmd --list-all --zone=public
'
```

`firewall-offline-cmd` lets us configure zones without the daemon. Rich rules add fine-grained source-based ACLs.

---

## Step 4 — nftables: table, chain, and a rule

```bash
apt install -y nftables
nft add table inet lab
nft 'add chain inet lab input { type filter hook input priority 0 ; policy accept ; }'
nft add rule inet lab input tcp dport 23 drop
nft list ruleset
```

`nftables` is the modern netfilter front end; one `nft` command replaces several `iptables` calls. The `inet` family covers IPv4 and IPv6.

---

## Step 5 — ipset for high-performance IP lists

```bash
apt install -y ipset
ipset create badguys hash:ip
ipset add badguys 192.0.2.10
ipset add badguys 198.51.100.5
ipset list badguys
iptables -I INPUT -m set --match-set badguys src -j DROP
iptables -L INPUT -n -v | head
```

`ipset` stores thousands of addresses in a hash and is matched in O(1). One `iptables` rule references the whole set.

---

## Step 6 — SNAT vs DNAT and masquerade examples

```bash
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A POSTROUTING -o eth0 -s 10.10.0.0/24 -j MASQUERADE
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 10.10.0.50:80
iptables -t nat -L -n -v
iptables -t nat -F
```

SNAT (and its dynamic cousin MASQUERADE) rewrites source addresses for outbound traffic; DNAT rewrites destination addresses for port forwarding. We flush the NAT table after viewing.

---

## Step 7 — Stateful inspection demo

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
iptables -L INPUT -n -v
```

The first rule short-circuits return traffic, the second permits new SSH connections only. This is the canonical stateful firewall pattern.

---

## Step 8 — Cleanup

```bash
iptables -F; iptables -t nat -F; iptables -X
ipset destroy badguys
nft delete table inet lab
yes | ufw reset
ufw disable
sysctl -w net.ipv4.ip_forward=0
apt purge -y ufw nftables ipset docker.io
```

All firewall state, sets, and packages are removed.

---

## What you learned
- The relationship between `ufw`, `firewalld`, `iptables`, and `nftables`
- How to write stateful rules using conntrack states
- How SNAT, DNAT, and MASQUERADE differ and where each fits

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- ufw — https://launchpad.net/ufw
- firewalld — https://firewalld.org/
- nftables — https://www.netfilter.org/projects/nftables/
- ipset — https://ipset.netfilter.org/
