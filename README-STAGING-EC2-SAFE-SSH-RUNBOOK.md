# Staging EC2 Safe Runbook (SSH-Only, No Console Access)

Last updated: 2026-02-13

This guide is for safely enabling OpenTelemetry on the official staging server when you do not have AWS Console access.

- Monitoring hub SSH: `ssh -i ~/.ssh/LGTM-key.pem ubuntu@18.140.5.245`
- Staging SSH: `ssh -i ~/.ssh/duadualive-staging.pem ubuntu@13.251.83.83`

Primary goal: make changes with rollback points, using only SSH + CLI.

## 1) Can AMI be created from SSH?

Yes, if you can run AWS CLI with IAM permissions (`ec2:CreateImage`, `ec2:CreateSnapshot`, and describe permissions).

Important:
- You do not need Console for this.
- You can run AWS CLI from any machine that has AWS credentials (local laptop, bastion, or server with role creds).
- If permissions are denied, use the fallback rollback method in Section 3 and ask your cloud admin for AMI/snapshot creation.

## 2) Preflight: run from your local machine (recommended)

Check AWS CLI and identity:

```bash
aws --version
aws sts get-caller-identity
```

If this fails, configure credentials/profile first:

```bash
aws configure
# or use: aws sso login --profile <profile-name>
```

Set variables (update region if needed):

```bash
export AWS_REGION=ap-southeast-1
export STAGING_PUBLIC_IP=13.251.83.83
export TS=$(date +%Y%m%d-%H%M%S)
```

Find staging instance ID by public IP:

```bash
export STAGING_INSTANCE_ID=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --filters "Name=ip-address,Values=$STAGING_PUBLIC_IP" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text)

echo "$STAGING_INSTANCE_ID"
```

If instance ID is empty, ask admin for the exact staging instance ID.

## 3) Create rollback points before any change

### A. Preferred rollback (AMI + snapshots via AWS CLI)

Create AMI:

```bash
export AMI_NAME="staging-pre-otel-$TS"
aws ec2 create-image \
  --region "$AWS_REGION" \
  --instance-id "$STAGING_INSTANCE_ID" \
  --name "$AMI_NAME" \
  --description "Pre-OTel backup for staging at $TS" \
  --no-reboot
```

Optional: wait for image completion:

```bash
export AMI_ID=$(aws ec2 describe-images \
  --region "$AWS_REGION" \
  --owners self \
  --filters "Name=name,Values=$AMI_NAME" \
  --query "Images[0].ImageId" \
  --output text)

echo "$AMI_ID"
aws ec2 wait image-available --region "$AWS_REGION" --image-ids "$AMI_ID"
```

Create snapshots for all attached EBS volumes:

```bash
VOLUME_IDS=$(aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids "$STAGING_INSTANCE_ID" \
  --query "Reservations[].Instances[].BlockDeviceMappings[].Ebs.VolumeId" \
  --output text)

for v in $VOLUME_IDS; do
  aws ec2 create-snapshot \
    --region "$AWS_REGION" \
    --volume-id "$v" \
    --description "staging-pre-otel-$TS volume $v"
done
```

### B. Fallback rollback if AMI/snapshot permission is denied

You still can reduce risk significantly:

1. Make local backups on staging for every file/service you touch.
2. Keep a command transcript (`script`) to undo exactly what changed.
3. Apply changes in tiny steps and validate each step.
4. Stop immediately on repeated errors.

## 4) SSH sessions setup

Open 2 terminals.

Terminal A (monitoring):

```bash
ssh -i ~/.ssh/LGTM-key.pem ubuntu@18.140.5.245
```

Terminal B (staging):

```bash
ssh -i ~/.ssh/duadualive-staging.pem ubuntu@13.251.83.83
```

## 5) Staging safety baseline (Terminal B)

Create a transcript and backup folder:

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

Backup pre-existing files:

```bash
sudo test -f /etc/systemd/system/otelcol.service && sudo cp -a /etc/systemd/system/otelcol.service /root/pre-otel-$TS/
sudo test -d /etc/otelcol && sudo cp -a /etc/otelcol /root/pre-otel-$TS/
id otelcol || true
```

## 6) Connectivity + package baseline (Terminal B)

Set monitoring host:

```bash
export MONITORING_HOST=18.140.5.245
echo "$MONITORING_HOST"
```

Install utilities:

```bash
sudo apt-get update
sudo apt-get install -y curl wget tar ca-certificates netcat-openbsd jq
```

Verify OTLP ports:

```bash
nc -zv "$MONITORING_HOST" 4317
nc -zv "$MONITORING_HOST" 4318
```

Do not continue until both are reachable.

## 7) Install OTel binaries (Terminal B)

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

## 8) Create collector config (Terminal B)

```bash
sudo mkdir -p /etc/otelcol /var/lib/otelcol
sudo chown -R otelcol:otelcol /etc/otelcol /var/lib/otelcol
```

Write config:

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

Permissions for log reading:

```bash
sudo usermod -aG adm otelcol
sudo usermod -aG www-data otelcol
id otelcol
ls -lah /home/theone/kol/storage/logs /var/log/nginx || true
```

## 9) Create and start systemd service (Terminal B)

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
```

Start and inspect:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now otelcol
sudo systemctl status otelcol --no-pager
sudo journalctl -u otelcol -n 80 --no-pager
```

If you see repeated `connection refused`, `context deadline exceeded`, or permanent exporter errors, stop and troubleshoot before moving on.

## 10) Health checks and smoke tests

### Monitoring hub checks (Terminal A)

```bash
curl -sf http://localhost:3100/ready && echo "loki ok"
curl -sf http://localhost:9009/ready && echo "mimir ok"
curl -sf http://localhost:3200/ready && echo "tempo ok"
```

### Staging test data (Terminal B)

```bash
echo "[otel-smoke] $(date -Iseconds) staging log pipeline test" | sudo tee -a /home/theone/kol/storage/logs/laravel.log
telemetrygen traces --otlp-insecure --otlp-endpoint 127.0.0.1:4317 --traces 20
```

Grafana Explore checks:
- Loki query: `{service_name="laravel-staging"} |= "otel-smoke"`
- Mimir: wait 1-2 minutes, then inspect host metrics
- Tempo: traces in last 15 minutes, service `laravel-staging`

## 11) Immediate rollback commands (Terminal B)

Stop and remove OTel service/config:

```bash
sudo systemctl disable --now otelcol || true
sudo rm -f /etc/systemd/system/otelcol.service
sudo rm -rf /etc/otelcol
sudo systemctl daemon-reload
```

Optional user cleanup:

```bash
sudo userdel otelcol || true
```

Restore from local backups (if present):

```bash
sudo cp -a /root/pre-otel-<TIMESTAMP>/otelcol /etc/ || true
sudo cp -a /root/pre-otel-<TIMESTAMP>/otelcol.service /etc/systemd/system/ || true
sudo systemctl daemon-reload
```

If AMI/snapshot was created, full rollback is done by launching/recovering from that backup via your cloud admin flow.

## 12) Suggested Git practice (repo only)

Use a branch for runbook/scripts changes:

```bash
git checkout -b chore/staging-otel-safe-rollout
```

Remember: Git reverts repository files, not server OS/package/systemd state.

