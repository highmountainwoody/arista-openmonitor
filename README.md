# Arista Campus Monitoring (Prometheus + Grafana + Loki)

This repository provides a production-oriented starter stack for monitoring Arista Campus and DCN switches using open source tooling (no CloudVision). It emphasizes a phased rollout: start with SNMP + syslog, then add gNMI streaming telemetry for high-resolution metrics.

## What you get

- **Prometheus** for metrics scraping and alerting rules.
- **Alertmanager** for alert routing.
- **Grafana** for dashboards and alert UI.
- **SNMP exporter** for baseline health and interface counters.
- **Loki + Promtail** for syslog ingestion and log search.
- **Telegraf** to receive gNMI streaming telemetry and expose it to Prometheus.
- **Blackbox exporter** (optional) for ICMP/TCP probes.

## Quick start

1. **Review and edit configuration files** (see the sections below).
2. **Start the stack**:

   ```bash
   docker compose up -d
   ```

3. **Access UIs**:
   - Grafana: http://localhost:3000 (default user/pass: `admin` / `admin`)
   - Prometheus: http://localhost:9090
   - Alertmanager: http://localhost:9093
   - Loki: http://localhost:3100

> Note: Use `docker-compose` if your environment does not support the `docker compose` plugin.

## Grafana dashboards

Grafana is provisioned with a baseline dashboard at **Dashboards → Arista Campus → Arista Campus Monitoring**. It includes panels for:

- Interface throughput and utilization.
- Interface errors, discards, and drops.
- Interface state and flap detection.
- Queue drops/congestion (expects gNMI/Telegraf metrics; update queries as needed).
- MLAG and LACP health (expects gNMI or custom exporter metrics; update queries as needed).
- CPU and memory utilization (via HOST-RESOURCES-MIB).

If you add or rename metrics, update `grafana/dashboards/arista-campus-monitoring.json`.

## Where to configure switch IPs and credentials

### 1) Prometheus scrape targets (SNMP + blackbox)

Edit `config/prometheus/prometheus.yml`:

- **SNMP targets** (replace placeholder IPs):
  - `scrape_configs.job_name: snmp` -> `static_configs.targets`
- **Blackbox targets** (optional):
  - `scrape_configs.job_name: blackbox` -> `static_configs.targets`

Example:

```yaml
scrape_configs:
  - job_name: snmp
    static_configs:
      - targets:
          - 192.0.2.10
          - 192.0.2.11
```

### 2) SNMP credentials

Edit `config/snmp_exporter/snmp.yml`:

- The default config uses SNMPv2c community `CHANGEME`. Update for production:
  - Prefer **SNMPv3** with limited views (update the `auths` block).
  - Ensure Prometheus `params.auth` matches the auth name you define.


Example (SNMPv3 authPriv stub):

```yaml
auths:
  snmpv3_authpriv:
    username: snmp-user
    security_level: authPriv
    auth_protocol: SHA
    auth_password: "REPLACE_ME"
    priv_protocol: AES
    priv_password: "REPLACE_ME"
modules:
  arista_if_mib:
    auth:
      username: snmp-user
      security_level: authPriv
      auth_protocol: SHA
      auth_password: "REPLACE_ME"
      priv_protocol: AES
      priv_password: "REPLACE_ME"
```

### 3) gNMI targets and credentials (Telegraf)

Edit `config/telegraf/telegraf.conf`:

- **gNMI device addresses**:
  - `[[inputs.gnmi]] addresses = ["192.0.2.10:6030", ...]`
  - `[[inputs.gnmi]] addresses = ["arista-750-1:6030", ...]`
- **gNMI credentials**:
  - `username` / `password`
- **TLS**:
  - `enable_tls = true` and `insecure_skip_verify = true` (replace with real cert validation in production).

Example:

```toml
[[inputs.gnmi]]
  addresses = ["10.10.10.10:6030", "10.10.10.11:6030"]
  username = "gnmi-user"
  password = "REPLACE_ME"
  enable_tls = true
  insecure_skip_verify = false
```

### 4) Syslog ingestion

- **Promtail** listens for syslog on `0.0.0.0:1514` (UDP/TCP).
- Configure your Arista switches to send syslog to the host running Docker Compose on port **1514**.
- Promtail configuration: `config/promtail/promtail.yml`.

## EOS CLI examples (SNMP, syslog, gNMI)

The following EOS CLI snippets show how to point SNMP, syslog, and gNMI telemetry at this stack. Replace `MONITORING_HOST`, usernames, and secrets with your values and align ports with your environment.

### SNMP (SNMPv3 recommended)

```text
! Create an SNMPv3 user with authPriv (example)
snmp-server view MONITORING iso included
snmp-server group MONITORING v3 priv read MONITORING
snmp-server user snmpuser MONITORING v3 auth sha AUTH_PASSWORD priv aes PRIV_PASSWORD

! Optional: restrict to the monitoring host
snmp-server host MONITORING_HOST version 3 priv snmpuser
```

### Syslog (to Promtail)

```text
logging host MONITORING_HOST transport udp port 1514
logging host MONITORING_HOST transport tcp port 1514
logging source-interface Management1
logging level warnings
```

### gNMI (to Telegraf)

```text
management api gnmi
   transport grpc
   provider eos-native
   no shutdown
!
management api gnmi
   transport grpc ssl profile GNMI_PROFILE
!
management security
   ssl profile GNMI_PROFILE
      certificate GNMI_CERT
      key GNMI_KEY
!
username gnmi-user secret GNMI_PASSWORD
```

> Note: gNMI TLS profile and certificate/key handling depend on your EOS version and PKI workflow. Ensure the port (default 6030) matches `config/telegraf/telegraf.conf`.

## Phased rollout guidance

1. **Phase 1 (baseline)**: SNMP + syslog
   - Validate SNMP metrics in Prometheus.
   - Validate syslog in Loki/Grafana.
2. **Phase 2 (gNMI streaming)**:
   - Enable gNMI on EOS and validate subscriptions in Telegraf.
   - Add targeted subscriptions for queues, optics, or interface telemetry.
3. **Phase 3 (optimization)**:
   - Blackbox probes for RTT/availability.
   - sFlow collector/exporter (not included in this stack) for top talkers.

## Important defaults to change

- **Grafana admin password** in `docker-compose.yml` (`GF_SECURITY_ADMIN_PASSWORD`).
- **Alertmanager receiver** in `config/alertmanager/alertmanager.yml`.
- **SNMP community / SNMPv3 credentials** in `config/snmp_exporter/snmp.yml`.
- **gNMI credentials and TLS** in `config/telegraf/telegraf.conf`.

## Stop the stack

```bash
docker compose down
```

## Directory map

- `docker-compose.yml`: service definitions and port mappings.
- `grafana/provisioning`: Grafana data source + dashboard provisioning.
- `grafana/dashboards/arista-campus-monitoring.json`: baseline monitoring dashboard.
- `config/prometheus/prometheus.yml`: scrape targets and Alertmanager configuration.
- `config/snmp_exporter/snmp.yml`: SNMP credentials (auths) and module walk list.
- `config/telegraf/telegraf.conf`: gNMI subscriptions and output to Prometheus.
- `config/promtail/promtail.yml`: syslog ingestion into Loki.
- `config/loki/loki.yml`: Loki storage configuration.
- `config/blackbox_exporter/blackbox.yml`: blackbox probe modules.
- `config/alertmanager/alertmanager.yml`: alert routing stub.
