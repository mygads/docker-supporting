# Supporting Services

Monitoring, logging, dan message broker untuk infrastructure.

## 🚀 Quick Start

```bash
# Copy environment
cp .env.example .env

# Start all services
docker compose up -d

# Check status
docker compose ps
```

## 📁 Structure

```
supporting/
├── .github/workflows/ci-cd.yml  # CI/CD Pipeline
├── docker-compose.yml           # Development
├── docker-compose.swarm.yml     # Production (Swarm)
├── .env.example                 # Environment template
├── rabbitmq/
│   ├── rabbitmq.conf           # RabbitMQ configuration
│   └── definitions.json        # Users, vhosts, exchanges
├── prometheus/
│   └── prometheus.yml          # Prometheus configuration
├── grafana/
│   └── provisioning/           # Dashboards & datasources
├── loki/
│   └── loki-config.yml         # Loki configuration
├── alertmanager/
│   └── alertmanager.yml        # Alert rules
└── promtail/
    └── promtail-config.yml     # Log collection config
```

## 🔧 Services

| Service | Port | URL | Description |
|---------|------|-----|-------------|
| RabbitMQ | 5672, 15672 | rabbitmq.govconnect.my.id | Message broker |
| Prometheus | 9090 | prometheus.govconnect.my.id | Metrics collection |
| Grafana | 3000 | grafana.govconnect.my.id | Dashboards |
| Loki | 3100 | - | Log aggregation |
| Alertmanager | 9093 | - | Alert management |
| Node Exporter | 9100 | - | Host metrics |
| cAdvisor | 8081 | - | Container metrics |

## 🐰 RabbitMQ

### Default Credentials
- Username: `admin`
- Password: Set in `.env` (`RABBITMQ_PASSWORD`)
- VHost: `govconnect`

### Adding New VHost/User

Edit `rabbitmq/definitions.json`:

```json
{
  "vhosts": [
    {"name": "govconnect"},
    {"name": "new_vhost"}  // Add new vhost
  ],
  "users": [
    {"name": "admin", ...},
    {"name": "new_user", "password": "...", "tags": ""}  // Add new user
  ]
}
```

## 📊 Prometheus

### Adding New Scrape Target

Edit `prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'new-service'
    static_configs:
      - targets: ['new-service:9090']
```

## 📈 Grafana

### Default Credentials
- Username: `admin`
- Password: Set in `.env` (`GRAFANA_PASSWORD`)

### Adding Dashboards

Place JSON files in `grafana/provisioning/dashboards/`

## 🔐 GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `SERVER_HOST` | Server IP/hostname |
| `SSH_USERNAME` | SSH username |
| `SSH_PRIVATE_KEY` | SSH private key |
| `ENV_PRODUCTION` | Production .env content |

## 🔄 CI/CD Features

- Auto-deploy on push to main
- Detects which service changed
- Only restarts affected services
- Manual restart via workflow_dispatch
