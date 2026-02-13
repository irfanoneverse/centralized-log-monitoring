# Staging EC2 OpenTelemetry Agent Runbook (Untouched Server Start)

Last updated: 2026-02-13

This runbook is the next phase after monitoring hub is up. It assumes:

- Monitoring hub EC2 is healthy and reachable.
- Staging EC2 has no OpenTelemetry setup yet.
- You will perform rollout using SSH only, with safe rollback points.

Quick SSH references:

- Monitoring hub: `ssh -i ~/.ssh/LGTM-key.pem ubuntu@18.140.5.245`
- Staging app server: `ssh -i ~/.ssh/duadualive-staging.pem ubuntu@13.251.83.83`

## 1) Current status and objective

Current status:

- Monitoring hub (Grafana/Loki/Tempo/Mimir/OTel collector) is running.
- Staging Laravel EC2 is still untouched.

Objective:

- Install OpenTelemetry Collector **agent** on staging EC2.
- Collect staging logs/metrics/traces locally and forward to monitoring hub OTLP (`4317`).
- Verify data in Grafana Explore.

Definition of done:

- `otelcol` service is active and enabled on staging.
- Staging can reach monitoring hub on `4317` and `4318`.
- Log smoke test appears in Loki.
- Host metrics appear in Mimir.
- Trace smoke test appears in Tempo.

## 2) Phase 0 - Zero-touch preparation (before touching staging)

Do this from local machine first.

### 2.1 Validate monitoring hub health

```bash
ssh -i ~/.ssh/LGTM-key.pem ubuntu@18.140.5.245
curl -sf http://localhost:3100/ready && echo "loki ok"
curl -sf http://localhost:3200/ready && echo "tempo ok"
curl -sf http://localhost:9009/ready && echo "mimir ok"
curl -sf http://localhost:3000/api/health && echo "grafana ok"
exit
```

### 2.2 Confirm network path from staging to hub

At minimum, monitoring EC2 security group must allow staging source IP/SG on:

- `4317/tcp` (OTLP gRPC)
- `4318/tcp` (OTLP HTTP, optional but recommended)

### 2.3 Create rollback point (recommended)

If AWS CLI + IAM permissions exist, create AMI/snapshots before installing anything.

```bash
export AWS_REGION=ap-southeast-1
export STAGING_PUBLIC_IP=13.251.83.83
export TS=$(date +%Y%m%d-%H%M%S)

export STAGING_INSTANCE_ID=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=ip-address,Values=$STAGING_PUBLIC_IP" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text)

export AMI_NAME="staging-pre-otel-$TS"
aws ec2 create-image \
  --region "$AWS_REGION" \
  --instance-id "$STAGING_INSTANCE_ID" \
  --name "$AMI_NAME" \
  --description "Pre-OTel backup for staging at $TS" \
  --no-reboot
```

If AMI/snapshot permissions are unavailable, continue with filesystem backups in Phase 1.

## 3) Phase 1 - First touch and baseline capture (staging EC2)

Connect:

```bash
ssh -i ~/.ssh/duadualive-staging.pem ubuntu@13.251.83.83
```

Create transcript + backup directory:

```bash
TS=$(date +%Y%m%d-%H%M%S)
script -a "$HOME/staging-otel-$TS.log"
sudo mkdir -p /root/pre-otel-$TS
echo "Backup folder: /root/pre-otel-$TS"
```

Capture current state:

```bash
hostname
hostname -I
id
systemctl status otelcol --no-pager || true
```

Backup any existing OTel artifacts (safe no-op if none):

```bash
sudo test -f /etc/systemd/system/otelcol.service && sudo cp -a /etc/systemd/system/otelcol.service /root/pre-otel-$TS/
sudo test -d /etc/otelcol && sudo cp -a /etc/otelcol /root/pre-otel-$TS/
id otelcol || true
```

## 4) Phase 2 - Install OpenTelemetry Collector agent

Set monitoring target:

```bash
export MONITORING_HOST=18.140.5.245
echo "$MONITORING_HOST"
```

Install base utilities:

```bash
sudo apt-get update
sudo apt-get install -y curl wget tar ca-certificates netcat-openbsd jq
```

Validate connectivity to monitoring OTLP:

```bash
nc -zv "$MONITORING_HOST" 4317
nc -zv "$MONITORING_HOST" 4318
```

Install collector and telemetrygen:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin otelcol || true

cd /tmp
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.112.0/otelcol-contrib_0.112.0_linux_amd64.tar.gz
tar -xzf otelcol-contrib_0.112.0_linux_amd64.tar.gz
sudo mv otelcol-contrib /usr/local/bin/otelcol-contrib
sudo chmod +x /usr/local/bin/otelcol-contrib
/usr/local/bin/otelcol-contrib --version

wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.112.0/telemetrygen_0.112.0_linux_amd64.tar.gz
tar -xzf telemetrygen_0.112.0_linux_amd64.tar.gz
sudo mv telemetrygen /usr/local/bin/telemetrygen
sudo chmod +x /usr/local/bin/telemetrygen
telemetrygen --help | head
```

Create collector config:

```bash
sudo mkdir -p /etc/otelcol /var/lib/otelcol
sudo chown -R otelcol:otelcol /etc/otelcol /var/lib/otelcol

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
        endpoint: 127.0.0.1:4317
      http:
        endpoint: 127.0.0.1:4318

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256
  batch:
    timeout: 5s
    send_batch_size: 512
  resource:
    attributes:
      - key: deployment.environment
        value: staging
        action: upsert
      - key: service.name
        value: laravel-staging
        action: upsert

exporters:
  otlp:
    endpoint: \${MONITORING_HOST}:4317
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

Grant log-read access:

```bash
sudo usermod -aG adm otelcol
sudo usermod -aG www-data otelcol
id otelcol
ls -lah /home/theone/kol/storage/logs /var/log/nginx || true
```

Create and start systemd service:

```bash
sudo tee /etc/systemd/system/otelcol.service > /dev/null <<EOF
[Unit]
Description=OpenTelemetry Collector (Agent)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=otelcol
Group=otelcol
Environment=MONITORING_HOST=$MONITORING_HOST
ExecStart=/usr/local/bin/otelcol-contrib --config=/etc/otelcol/config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now otelcol
sudo systemctl status otelcol --no-pager
sudo journalctl -u otelcol -n 80 --no-pager
```

## 5) Phase 3 - Validate ingestion end-to-end

Generate smoke test data on staging:

```bash
echo "[otel-smoke] $(date -Iseconds) staging log pipeline test" | sudo tee -a /home/theone/kol/storage/logs/laravel.log
telemetrygen traces --otlp-insecure --otlp-endpoint 127.0.0.1:4317 --traces 20
```

Check monitoring hub health:

```bash
ssh -i ~/.ssh/LGTM-key.pem ubuntu@18.140.5.245
curl -sf http://localhost:3100/ready && echo "loki ok"
curl -sf http://localhost:9009/ready && echo "mimir ok"
curl -sf http://localhost:3200/ready && echo "tempo ok"
exit
```

Grafana checks:

- URL: `http://18.140.5.245:3000`
- Loki query: `{service_name="laravel-staging"} |= "otel-smoke"`
- Mimir: wait 1-2 minutes and explore host metrics for `service_name="laravel-staging"`
- Tempo: traces from service `laravel-staging` in last 15 minutes

## 6) Fast rollback (if anything goes wrong)

```bash
sudo systemctl disable --now otelcol || true
sudo rm -f /etc/systemd/system/otelcol.service
sudo rm -rf /etc/otelcol
sudo systemctl daemon-reload
sudo userdel otelcol || true
```

Restore backups if needed:

```bash
sudo cp -a /root/pre-otel-<TIMESTAMP>/otelcol /etc/ || true
sudo cp -a /root/pre-otel-<TIMESTAMP>/otelcol.service /etc/systemd/system/ || true
sudo systemctl daemon-reload
```

## 7) Operator notes

- Keep rollout incremental. Stop if repeated exporter errors appear in `journalctl -u otelcol`.
- `start_at: end` means only new log lines after collector starts are shipped.
- If staging and monitoring are in the same VPC, prefer private IP for `MONITORING_HOST`.
- After staging is stable, turn this into a reusable install script for production rollout.

