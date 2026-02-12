# LGTM Stack on AWS EC2 - Deployment Progress Handoff

Last updated: 2026-02-12

This note captures what is already done in this repository for AWS EC2 deployment, what is still pending, and how to continue quickly next time.

## 0) Confirmed AWS progress (from current implementation)

- Monitoring EC2 is launched and running:
  - Instance ID: `i-01442ed1d6c38591d`
  - Name: `LGTM-Monitoring`
  - Instance type: `t3.small`
  - AZ: `ap-southeast-1b`
- Security group in place (`LGTM-group`) with currently visible inbound rules:
  - `22/tcp` from `211.24.84.192/32`
  - `3000/tcp` from `211.24.84.192/32`
  - `3100/tcp` from `13.251.83.83/32`
  - `4317/tcp` from `13.251.83.83/32`
  - `4318/tcp` from `13.251.83.83/32`
  - `9009/tcp` from `13.251.83.83/32`
- IAM role/policy has been prepared:
  - Role: `LGTM-Monitoring-Role`
  - Policy: `LGTM-Monitoring-Policy`
  - S3 actions allowed: `s3:ListBucket`, `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`
  - Scope: `arn:aws:s3:::my-lgtm-*` and `arn:aws:s3:::my-lgtm-*/*`

## 1) What has been completed

- Architecture direction is finalized to `LGTM + OpenTelemetry`:
  - `Loki` for logs
  - `Tempo` for traces
  - `Mimir` for metrics
  - `Grafana` for visualization
  - `OpenTelemetry Collector` for telemetry pipeline
- A monitoring-only compose profile exists in `docker-compose.monitoring.yaml` for cloud-style deployment (Loki, Tempo, Mimir, Grafana).
- AWS region support is wired into compose and config files using `AWS_REGION` (default `ap-southeast-1` in current files).
- Storage backend for Loki/Tempo/Mimir is already set to AWS S3 in:
  - `config/loki-config.yaml`
  - `config/tempo-config.yaml`
  - `config/mimir-config.yaml`
- S3 bucket targets are already defined in config:
  - `my-lgtm-loki`
  - `my-lgtm-tempo`
  - `my-lgtm-mimir`
  - `my-lgtm-mimir-ruler`
- Known stability fix already applied:
  - Tempo pinned to `grafana/tempo:2.6.1` to avoid `empty ring` issue.
- Prior AWS deployment guide was previously created in commit `dc4555b` (`LGPO-Stack-AWS.md`), covering:
  - EC2 monitoring hub setup
  - IAM + SG recommendations
  - OTel agent setup on app nodes
  - Grafana + Prometheus operational flow

## 2) Current status snapshot

- Monitoring EC2 infra base is already in place (instance + SG + IAM policy baseline).
- Repo is ready for the **monitoring hub container deployment** on EC2.
- Repo already reflects **S3-backed LGTM configs** (not local MinIO-only).
- Existing markdown docs focus mostly on local/WSL workflow; this file now serves as explicit AWS EC2 continuation note.
- Main remaining work: connect staging Laravel EC2 logs to the monitoring hub via OpenTelemetry Collector.

## 3) What is not yet in repo (pending)

- No Infrastructure-as-Code yet (Terraform/CloudFormation not found).
- No committed EC2 bootstrap automation (cloud-init/user-data scripts not found).
- No committed systemd installer scripts for OTel agent on application EC2 nodes.
- No dedicated `env` template for AWS deployment values (region/buckets/role assumptions).
- No committed final record of actual AWS resource IDs from current deployment (instance IDs, SG IDs, bucket names per environment, DNS names).

## 4) Recommended resume plan (next session)

1. Prepare/verify AWS resources:
   - EC2 monitoring instance (Ubuntu 24.04+)
   - IAM role with S3 access for Loki/Tempo/Mimir buckets
   - Security groups for `22`, `3000`, `3100`, `4317`, `4318`, `9009`, `3200` (restricted to trusted CIDR/SG)
2. On monitoring EC2:
   - Install Docker + Docker Compose plugin
   - Clone this repo
   - Start stack with:
     - `docker compose -f docker-compose.monitoring.yaml up -d`
3. Validate services:
   - Grafana `:3000`
   - Loki `/ready` on `:3100`
   - Mimir `/ready` on `:9009`
   - Tempo `/ready` on `:3200`
4. Configure telemetry ingestion from app nodes:
   - Point OTel agents/SDKs to monitoring hub (`4317` or `4318`)
   - Confirm logs/metrics/traces appear in Grafana
5. Capture real deployed values back into this file:
   - Public/private IP or DNS
   - SG and IAM role names
   - Final bucket names and lifecycle policy status

## 5) Staging EC2 (Laravel) - next implementation checklist

Target: ship Laravel logs from staging EC2 to monitoring hub OTLP endpoint (`4317`/`4318`).

1. Install OTel collector on staging EC2:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin otelcol || true
cd /tmp
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.112.0/otelcol-contrib_0.112.0_linux_amd64.tar.gz
tar -xzf otelcol-contrib_0.112.0_linux_amd64.tar.gz
sudo mv otelcol-contrib /usr/local/bin/otelcol-contrib
sudo chmod +x /usr/local/bin/otelcol-contrib
sudo mkdir -p /etc/otelcol /var/lib/otelcol
sudo chown -R otelcol:otelcol /etc/otelcol /var/lib/otelcol
```

2. Create collector config (`/etc/otelcol/config.yaml`) and point exporter to monitoring hub private endpoint:

```yaml
receivers:
  filelog:
    include:
      - /var/www/**/storage/logs/*.log
      - /var/log/nginx/*.log
      - /var/log/syslog
    start_at: end

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
        value: staging-laravel-ec2
        action: upsert
      - key: env
        value: staging
        action: upsert
      - key: service.name
        value: laravel-staging
        action: upsert

exporters:
  otlp:
    endpoint: <MONITORING_EC2_PRIVATE_IP_OR_DNS>:4317
    tls:
      insecure: true

service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [memory_limiter, resource, batch]
      exporters: [otlp]
```

3. Create and start systemd service:

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
```

4. Validate end-to-end:
   - On staging EC2, generate Laravel logs (hit app routes, trigger sample app log).
   - On monitoring EC2, verify collector/hub receive traffic.
   - In Grafana Explore (Loki), query:
     - `{env="staging"}`
     - `{service_name="laravel-staging"}`
     - `{server_name="staging-laravel-ec2"}`

## 6) Fill-in checklist for your real deployment

Use this section as your continuation tracker.

- [x] Monitoring EC2 launched
- [ ] IAM role attached to monitoring EC2 (re-verify attachment on instance profile)
- [ ] S3 buckets created and accessible from EC2 role
- [x] Security groups applied and verified
- [ ] Stack started with `docker-compose.monitoring.yaml`
- [ ] Grafana reachable from admin network
- [ ] Loki/Mimir/Tempo healthy
- [ ] Staging Laravel EC2 sending OTLP telemetry
- [ ] Dashboards/Explore show real data
- [ ] Lifecycle policy (Intelligent-Tiering + retention) confirmed
- [ ] Production hardening (TLS/auth/restricted ingress) completed

## 7) Notes for future cleanup

- Consider restoring the old detailed AWS guide from commit `dc4555b` as `docs/aws/LGPO-Stack-AWS.md` for full runbook reference.
- Add IaC and bootstrap scripts so EC2 setup is reproducible.
- Add an `aws.env.example` file to avoid manual variable drift between environments.


