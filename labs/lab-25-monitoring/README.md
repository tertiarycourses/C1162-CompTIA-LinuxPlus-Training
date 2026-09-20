# Lab 25 — Monitoring Concepts and Tools

This lab covers CompTIA Linux+ XK0-006 Objective 5.1 (monitoring concepts, agents, metrics endpoints, alerting, webhooks, health checks, log aggregation). Before you touch a tool, internalize the vocabulary: an SLA is the contractual promise to the customer (e.g. 99.9% monthly uptime), an SLO is the internal target you set to stay safely inside the SLA (e.g. 99.95%), and an SLI is the actual measured signal you compare against the SLO (e.g. ratio of successful HTTP requests to total). Monitoring stacks like Prometheus exist to produce SLIs from raw metrics so you can detect SLO burn before the SLA breaks.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install SNMP and walk the system MIB

```bash
apt update
apt install -y snmpd snmp snmp-mibs-downloader
systemctl enable --now snmpd
snmpwalk -v 2c -c public localhost 1.3.6.1.2.1.1
```

SNMP exposes device data through a hierarchical OID tree. The `1.3.6.1.2.1.1` subtree is the standard `system` group: hostname, uptime, contact, location.

---

## Step 2 — Install and probe node_exporter

```bash
apt install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
ss -tlnp | grep 9100
curl -s http://localhost:9100/metrics | head -n 20
```

node_exporter is a Prometheus agent that publishes host metrics (CPU, memory, disk, network) on port 9100 in the text-based exposition format.

---

## Step 3 — Install Prometheus and point it at node_exporter

```bash
apt install -y prometheus
cat >/etc/prometheus/prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
EOF
systemctl restart prometheus
sleep 3
curl -s 'http://localhost:9090/api/v1/targets' | head -c 400
```

Prometheus pulls (scrapes) metrics from each target on the configured interval and stores them in a local time-series database.

---

## Step 4 — Write and validate an alert rule

```bash
cat >/etc/prometheus/rules.yml <<'EOF'
groups:
- name: node-alerts
  rules:
  - alert: NodeDown
    expr: up{job="node"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Node exporter is down"
EOF
apt install -y prometheus-alertmanager
promtool check rules /etc/prometheus/rules.yml
```

`promtool check rules` parses your YAML, validates PromQL expressions, and confirms structure before Prometheus reloads — a critical pre-deployment check.

---

## Step 5 — Simulate a webhook receiver

```bash
(nc -l -p 8000 -q 1 >/tmp/webhook.log 2>&1 &) ; sleep 1
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"alert":"NodeDown","severity":"critical"}' \
  http://localhost:8000/
sleep 1
cat /tmp/webhook.log
```

Alertmanager (and other tools) deliver alerts to external systems by POSTing JSON to a webhook URL. A `nc` listener is the simplest way to verify the payload shape during integration testing.

---

## Step 6 — Health check pattern

```bash
curl -fsS http://localhost:9100/metrics >/dev/null && echo "node_exporter OK"
curl -fsS http://localhost:9090/-/healthy && echo
curl -fsS http://localhost:9090/-/ready && echo
```

The `-f` flag makes curl exit non-zero on HTTP errors, `-sS` silences progress but keeps errors visible — the canonical scriptable health-check idiom.

---

## Step 7 — Log aggregation via journalctl

```bash
journalctl -u prometheus -n 20 --no-pager
journalctl -u prometheus-node-exporter -n 20 --no-pager
journalctl --since "10 min ago" -p err --no-pager
```

Systemd's journal is the local log aggregation layer; forward it to a central collector (Loki, Elastic, Splunk) for fleet-wide search.

---

## Step 8 — Cleanup

```bash
systemctl stop prometheus prometheus-node-exporter snmpd prometheus-alertmanager 2>/dev/null
apt purge -y prometheus prometheus-node-exporter prometheus-alertmanager snmpd snmp snmp-mibs-downloader
apt autoremove -y
rm -f /tmp/webhook.log /etc/prometheus/rules.yml
```

Removes packages and configuration written during the lab.

---

## What you learned
- SLA vs SLO vs SLI and how monitoring tools produce SLIs from raw signals
- Pulling metrics over SNMP and the Prometheus exposition format
- Validating alert rules with `promtool` before deploying them
- Webhook integration testing with `nc` and curl
- Standard health-check and journald log inspection patterns

## Free tools used
- Prometheus — https://prometheus.io
- node_exporter — https://github.com/prometheus/node_exporter
- Alertmanager — https://prometheus.io/docs/alerting/latest/alertmanager/
- Net-SNMP — http://www.net-snmp.org
- curl — https://curl.se
- ncat/netcat — https://nmap.org/ncat/
