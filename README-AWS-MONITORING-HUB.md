# AWS Monitoring Hub Quick Start

This repository is prepared for AWS monitoring hub deployment.

## 1) Prerequisites on EC2

- Docker Engine + Docker Compose plugin installed
- IAM role attached to EC2 instance profile with S3 access to:
  - `my-lgtm-loki`
  - `my-lgtm-tempo`
  - `my-lgtm-mimir`
  - `my-lgtm-mimir-ruler`
- Security group allows:
  - `3000` (Grafana, restricted admin IP)
  - `3100` (Loki API, restricted)
  - `3200` (Tempo API, restricted)
  - `9009` (Mimir API, restricted)
  - `4317`, `4318` (OTLP ingest from app/staging EC2)

## 2) Deploy

```bash
cd /home/ubuntu
git clone <YOUR_REPO_URL> centralized-log-monitoring
cd centralized-log-monitoring

export AWS_REGION=ap-southeast-1
docker compose up -d
docker compose ps
```

## 3) Health checks

```bash
curl -sf http://localhost:3100/ready && echo "loki ok"
curl -sf http://localhost:3200/ready && echo "tempo ok"
curl -sf http://localhost:9009/ready && echo "mimir ok"
curl -sf http://localhost:3000/api/health && echo "grafana ok"
```

## 4) Staging Laravel endpoint

Point staging OTel exporter/agent to monitoring EC2 OTLP endpoint:

- gRPC: `http://<MONITORING_EC2_PRIVATE_IP>:4317`
- HTTP: `http://<MONITORING_EC2_PRIVATE_IP>:4318`

The hub receives OTLP via `poc-otel-collector` and forwards to Loki/Mimir/Tempo.


