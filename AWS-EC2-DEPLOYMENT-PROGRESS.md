# AWS EC2 Deployment Progress - Continuation Runbook

Last updated: 2026-02-12

This document is now focused on the only remaining objective: connect the **staging Laravel EC2** to the already-running **monitoring hub EC2**.

Reference: monitoring hub setup and health checks are documented in `README-AWS-MONITORING-HUB.md`.

## 1) Current state (what is already done)

### Monitoring hub EC2

- Monitoring instance is launched and configured:
  - Instance ID: `i-01442ed1d6c38591d`
  - Name: `LGTM-Monitoring`
  - Type: `t3.small`
  - AZ: `ap-southeast-1b`
- Security group (`LGTM-group`) is in place with inbound rules already set:
  - `22/tcp` from `211.24.84.192/32`
  - `3000/tcp` from `211.24.84.192/32`
  - `3100/tcp` from `13.251.83.83/32`
  - `4317/tcp` from `13.251.83.83/32`
  - `4318/tcp` from `13.251.83.83/32`
  - `9009/tcp` from `13.251.83.83/32`
- IAM role and policy prepared:
  - Role: `LGTM-Monitoring-Role`
  - Policy: `LGTM-Monitoring-Policy`
  - S3 actions: `s3:ListBucket`, `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`
  - Scope: `arn:aws:s3:::my-lgtm-*` and `arn:aws:s3:::my-lgtm-*/*`

### Repository and stack readiness

- Architecture is finalized as `LGTM + OpenTelemetry`.
- AWS-backed storage config is already wired:
  - `config/loki-config.yaml`
  - `config/tempo-config.yaml`
  - `config/mimir-config.yaml`
- Bucket targets already defined:
  - `my-lgtm-loki`
  - `my-lgtm-tempo`
  - `my-lgtm-mimir`
  - `my-lgtm-mimir-ruler`
- Tempo stability pin already applied: `grafana/tempo:2.6.1`.

## 2) Remaining objective

Send logs from **staging Laravel EC2** to the monitoring hub OTLP endpoint:

- gRPC: `http://<MONITORING_EC2_PRIVATE_IP_OR_DNS>:4317`
- HTTP: `http://<MONITORING_EC2_PRIVATE_IP_OR_DNS>:4318`

Recommended path: install **OpenTelemetry Collector agent** on staging EC2 from scratch and forward logs via OTLP gRPC (`4317`).

Assumption for this runbook: staging EC2 has **no OTel setup yet**.

## 3) Staging EC2 implementation steps from scratch (Laravel + LGTM smoke tests)

Your observed app path on staging is `/home/theone/kol`, so this runbook uses that path directly.

### Step A - Collect values and define variables

Run these on staging EC2 first:

```bash
hostname
hostname -I
```

Set the monitoring hub endpoint once in shell:

```bash
export MONITORING_HOST="<MONITORING_EC2_PRIVATE_IP_OR_DNS>"
echo "$MONITORING_HOST"
```

### Step B - Baseline package setup on staging EC2

```bash
sudo apt-get update
sudo apt-get install -y curl wget tar ca-certificates netcat-openbsd jq
```

### Step C - Verify network connectivity to hub OTLP ports

```bash
nc -zv "$MONITORING_HOST" 4317
nc -zv "$MONITORING_HOST" 4318
```

Both commands must show success before continuing.

If failed:
- add inbound allow rule on monitoring SG for staging EC2 source (IP or SG) on `4317` and `4318`
- verify route tables and NACLs between subnets/VPC

### Step D - Install OpenTelemetry Collector on staging EC2

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin otelcol || true
cd /tmp
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.112.0/otelcol-contrib_0.112.0_linux_amd64.tar.gz
tar -xzf otelcol-contrib_0.112.0_linux_amd64.tar.gz
sudo mv otelcol-contrib /usr/local/bin/otelcol-contrib
sudo chmod +x /usr/local/bin/otelcol-contrib
sudo mkdir -p /etc/otelcol /var/lib/otelcol
sudo chown -R otelcol:otelcol /etc/otelcol /var/lib/otelcol
/usr/local/bin/otelcol-contrib --version
```

### Step E - Install telemetry generator for trace smoke test

`telemetrygen` is used only to create a test trace to verify Tempo path end-to-end.

```bash
cd /tmp
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.112.0/telemetrygen_0.112.0_linux_amd64.tar.gz
tar -xzf telemetrygen_0.112.0_linux_amd64.tar.gz
sudo mv telemetrygen /usr/local/bin/telemetrygen
sudo chmod +x /usr/local/bin/telemetrygen
telemetrygen --help | head
```

### Step F - Create collector config on staging (`/etc/otelcol/config.yaml`)

This config does all 3 signals:
- logs: tail Laravel + nginx + syslog and send to Loki via hub collector
- metrics: host metrics from staging and send to Mimir via hub collector
- traces: accepts local OTLP traces (from `telemetrygen`) and sends to Tempo via hub collector

```bash
sudo tee /etc/otelcol/config.yaml > /dev/null <<EOF
receivers:
  filelog:
    include:
      - /home/theone/kol/storage/logs/*.log
      - /var/log/nginx/*.log
      - /var/log/syslog
    start_at: end
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu: {}
      memory: {}
      filesystem: {}
      load: {}
      network: {}
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256
  batch:
    timeout: 5s
    send_batch_size: 512
  resource:
    attributes:
      - key: server_name
        value: $(hostname)
        action: upsert
      - key: deployment.environment
        value: staging
        action: upsert
      - key: service.name
        value: laravel-staging
        action: upsert

exporters:
  otlp:
    endpoint: ${MONITORING_HOST}:4317
    tls:
      insecure: true

service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp]
    metrics:
      receivers: [hostmetrics]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp]
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp]
EOF
```

### Step G - Ensure collector user can read log files

```bash
ls -lah /home/theone/kol/storage/logs /var/log/nginx || true
sudo usermod -aG adm otelcol
sudo usermod -aG www-data otelcol
id otelcol
```

### Step H - Create and start systemd service

```bash
sudo tee /etc/systemd/system/otelcol.service > /dev/null <<'EOF'
[Unit]
Description=OpenTelemetry Collector (Agent)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=otelcol
Group=otelcol
Environment=MONITORING_HOST=<MONITORING_EC2_PRIVATE_IP_OR_DNS>
ExecStart=/usr/local/bin/otelcol-contrib --config=/etc/otelcol/config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now otelcol
sudo systemctl restart otelcol
sudo systemctl status otelcol --no-pager
```

### Step I - Verify collector is forwarding without errors

On staging EC2:

```bash
sudo journalctl -u otelcol -n 100 --no-pager
sudo journalctl -u otelcol -f
```

Look for absence of repeated errors such as:
- `connection refused`
- `context deadline exceeded`
- `Permanent error`

### Step J - Test Loki ingestion (logs)

1. Generate explicit test log line from staging:

```bash
echo "[otel-smoke] $(date -Iseconds) staging log pipeline test" | sudo tee -a /home/theone/kol/storage/logs/laravel.log
```

2. On monitoring hub EC2, verify Loki is healthy:

```bash
curl -sf http://localhost:3100/ready && echo "loki ok"
```

3. In Grafana Explore, datasource = Loki, query:

```text
{service_name="laravel-staging"} |= "otel-smoke"
```

### Step K - Test Mimir ingestion (metrics)

1. Wait 1-2 minutes (hostmetrics collection interval is 30s).
2. On monitoring hub EC2, verify Mimir is healthy:

```bash
curl -sf http://localhost:9009/ready && echo "mimir ok"
```

3. In Grafana Explore, datasource = Mimir (Prometheus), try:

```promql
count({__name__=~".+"})
```

Then filter for staging labels:

```promql
count by (service_name, deployment_environment) ({service_name="laravel-staging"})
```

If metric names differ by version/translation, use Grafana metric browser and search for `system` metrics from hostmetrics.

### Step L - Test Tempo ingestion (traces)

1. Generate traces from staging via local collector receiver:

```bash
telemetrygen traces --otlp-insecure --otlp-endpoint 127.0.0.1:4317 --traces 20
```

2. On monitoring hub EC2, verify Tempo is healthy:

```bash
curl -sf http://localhost:3200/ready && echo "tempo ok"
```

3. In Grafana Explore, datasource = Tempo:
- set service filter to `laravel-staging` (or search all services if not shown immediately)
- look for traces in the last 15 minutes

## 4) Definition of done for this phase

Mark this phase complete when all are true:

- [ ] Staging EC2 can connect to monitoring hub on `4317` and `4318`
- [ ] `otelcol` service is active and enabled at boot
- [ ] Loki test log (`otel-smoke`) is visible in Grafana Explore
- [ ] Host metrics from staging are visible in Mimir datasource
- [ ] Trace smoke test from `telemetrygen` is visible in Tempo datasource
- [ ] No recurring exporter/network errors in `journalctl -u otelcol`

## 5) Quick troubleshooting notes

- `failed to start service`: check YAML syntax in `/etc/otelcol/config.yaml`.
- `permission denied` on Laravel logs:
  - confirm file exists: `/home/theone/kol/storage/logs/laravel.log`
  - ensure `otelcol` is in needed groups (`adm`, `www-data`)
  - restart service after group changes: `sudo systemctl restart otelcol`
- no Loki logs:
  - appending test line must be to exact monitored file path
  - keep `start_at: end` in mind (collector reads new lines after startup)
- no Mimir metrics:
  - wait at least 60-120s for first batches
  - check staging journal for exporter retries
- no Tempo traces:
  - rerun `telemetrygen traces ...`
  - confirm `traces` pipeline exists and local OTLP receiver is listening on `127.0.0.1:4317`

## 6) Follow-up improvements (after staging is stable)

- Add bootstrap script for staging/prod OTel agent install and systemd setup.
- Add `aws.env.example` with `MONITORING_HOST`, region, and standard labels.
- Add IaC for EC2, SG, IAM, and S3 resources for repeatable deployment.


