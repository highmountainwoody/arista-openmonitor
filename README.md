# Arista Campus Monitoring (Prometheus + Grafana + Loki)

This repository provides a production-oriented starter stack for monitoring Arista 750-series access switches and 7280-series core switches using open source tooling (no CloudVision). It emphasizes a phased rollout: start with SNMP + syslog, then add gNMI streaming telemetry for high-resolution metrics.

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

- The default config uses SNMPv2c community `public`. Update for production:
  - Prefer **SNMPv3** with limited views.
  - Replace the `auth` block accordingly.

Example (SNMPv3 authPriv stub):

```yaml
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
- `config/prometheus/prometheus.yml`: scrape targets and Alertmanager configuration.
- `config/snmp_exporter/snmp.yml`: SNMP module and credentials.
- `config/telegraf/telegraf.conf`: gNMI subscriptions and output to Prometheus.
- `config/promtail/promtail.yml`: syslog ingestion into Loki.
- `config/loki/loki.yml`: Loki storage configuration.
- `config/blackbox_exporter/blackbox.yml`: blackbox probe modules.
- `config/alertmanager/alertmanager.yml`: alert routing stub.
