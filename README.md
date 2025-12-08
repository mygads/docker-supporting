# Supporting Services - Optimized Stack

Infrastructure services untuk GovConnect dengan fokus pada **lightweight** dan **production-ready**.

## 🎯 Stack Overview

| Service | Port | Purpose | Memory Limit |
|---------|------|---------|--------------|
| **RabbitMQ** | 5672, 15672 | Message broker | 512 MB |
| **Prometheus** | 9090 | Metrics collection | 512 MB |
| **Grafana** | 3300 | Visualization | 512 MB |
| **Loki** | 3101 | Log aggregation | 512 MB |
| **Promtail** | - | Log collector | 256 MB |

**Total Memory:** ~2.3 GB (vs ELK Stack ~4-6 GB) ✅

---

## 🚀 Quick Start

### 1. Setup Environment

```bash
cd supporting
cp .env.example .env
# Edit .env dengan credentials
```

### 2. Start Services

```bash
# Start all services
docker compose up -d

# Start specific services
docker compose up -d rabbitmq grafana

# Check status
docker compose ps

# View logs
docker compose logs -f loki
```

### 3. Access UIs

- **RabbitMQ Management:** http://localhost:15672
- **Prometheus:** http://localhost:9090
- **Grafana:** http://localhost:3300
- **Loki:** http://localhost:3101 (no UI, use Grafana)

---

## 📊 Service Details

### 1. RabbitMQ (Message Broker)

**Purpose:** Event-driven communication antar services

**Features:**
- Virtual host: `govconnect`
- Management UI enabled
- Persistent storage
- Health checks

**Usage:**
```typescript
// Connect from service
const connection = await amqp.connect(process.env.RABBITMQ_URL);
// RABBITMQ_URL=amqp://user:pass@rabbitmq:5672/govconnect
```

**Monitoring:**
- Management UI: http://localhost:15672
- Check queues, exchanges, connections
- Monitor message rates

---

### 2. Prometheus (Metrics)

**Purpose:** Time-series database untuk metrics

**Features:**
- 7 days retention (optimized)
- Auto-scrape GovConnect services
- Metrics endpoint: `/metrics`

**Scrape Targets:**
- Prometheus itself
- GovConnect services (channel, ai, case, notification, dashboard)
- RabbitMQ
- Traefik
- Loki

**Query Examples:**
```promql
# HTTP request rate
rate(http_requests_total[5m])

# Circuit breaker state
circuit_breaker_state{service="case-service"}

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
```

**Configuration:** `prometheus/prometheus.yml`

---

### 3. Grafana (Visualization)

**Purpose:** Dashboard untuk metrics dan logs

**Features:**
- Pre-configured data sources (Prometheus + Loki)
- Dashboard provisioning
- Alert support

**Setup:**

1. Login: http://localhost:3300
   - Username: dari `GRAFANA_USER` di .env
   - Password: dari `GRAFANA_PASSWORD` di .env

2. Add Data Sources:
   - **Prometheus:** http://prometheus:9090
   - **Loki:** http://loki:3100

3. Import Dashboards:
   - Node.js Application Monitoring
   - RabbitMQ Overview
   - Loki Logs Explorer

**Dashboard Locations:** `grafana/dashboards/`

---

### 4. Loki (Centralized Logging)

**Purpose:** Log aggregation (lightweight alternative to ELK)

**Features:**
- 7 days retention (optimized)
- JSON log parsing
- Label-based indexing
- Grafana integration

**Log Format:**
```json
{
  "timestamp": "2024-12-08T10:30:00.000Z",
  "level": "info",
  "service": "channel-service",
  "message": "Request completed",
  "correlationId": "abc123",
  "duration": 45
}
```

**Query in Grafana:**
```logql
# All logs from service
{service="channel-service"}

# Error logs
{service="channel-service"} |= "error"

# Parse JSON and filter
{service="channel-service"} | json | duration > 1000

# Correlation ID tracking
{service="channel-service"} | json | correlationId="abc123"
```

**Configuration:** `loki/loki-config.yml`

---

### 5. Promtail (Log Collector)

**Purpose:** Collect logs dari Docker containers dan kirim ke Loki

**Features:**
- Auto-discover Docker containers
- Label extraction
- JSON parsing
- System log collection

**Collected Logs:**
- Docker container logs
- GovConnect service logs
- System logs (/var/log/syslog)
- Auth logs (/var/log/auth.log)

**Configuration:** `promtail/promtail-config.yml`

---

## 🔧 Configuration

### Resource Limits

Semua services sudah di-set dengan resource limits untuk prevent memory leaks:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: '0.5'
```

### Retention Periods

**Optimized untuk development/staging:**
- Prometheus: 7 days
- Loki: 7 days

**Production (recommended):**
- Prometheus: 30 days
- Loki: 30 days

Edit di:
- `docker-compose.yml` (Prometheus: `--storage.tsdb.retention.time`)
- `loki/loki-config.yml` (Loki: `retention_period`)

---

## 📈 Monitoring Setup

### 1. Expose Metrics in Services

```typescript
// Add to each service
import { Router } from 'express';
import { register } from 'prom-client';

const router = Router();

router.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.send(await register.metrics());
});

export default router;
```

### 2. Use Structured Logger

```typescript
import { createLogger } from './shared/logger';

const logger = createLogger('my-service', process.env.LOG_LEVEL);
logger.info('Service started', { port: 3001 });
```

### 3. Verify Scraping

Check Prometheus targets: http://localhost:9090/targets

All targets should be **UP** (green).

---

## 🐛 Troubleshooting

### Service Won't Start

```bash
# Check logs
docker compose logs <service-name>

# Check if port is already in use
netstat -tulpn | grep <port>

# Restart service
docker compose restart <service-name>
```

### Prometheus Not Scraping

1. Check service is exposing `/metrics`
2. Verify service name in `prometheus.yml`
3. Check network connectivity: `docker exec prometheus ping <service-name>`

### Loki Not Receiving Logs

1. Check Promtail is running: `docker compose ps promtail`
2. Check Loki health: `curl http://localhost:3101/ready`
3. Verify logs are JSON format
4. Check Promtail logs: `docker compose logs promtail`

### High Memory Usage

1. Reduce retention periods
2. Check for memory leaks in services
3. Restart services: `docker compose restart`

### Grafana Can't Connect to Data Sources

1. Use internal Docker network names:
   - Prometheus: `http://prometheus:9090`
   - Loki: `http://loki:3100`
2. Don't use `localhost` (won't work inside container)

---

## 🔐 Security

### Production Checklist

- [ ] Change default passwords in `.env`
- [ ] Enable Traefik authentication for Prometheus
- [ ] Enable Traefik authentication for Grafana
- [ ] Use HTTPS (Traefik with Let's Encrypt)
- [ ] Restrict RabbitMQ Management UI access
- [ ] Set strong `PROMETHEUS_AUTH` in `.env`

### Generate Basic Auth for Prometheus

```bash
# Install htpasswd
sudo apt-get install apache2-utils

# Generate password
htpasswd -nb admin yourpassword

# Copy output to .env
PROMETHEUS_AUTH=admin:$apr1$...
```

---

## 📊 Resource Usage

### Development (Minimal)

```bash
# Start only required services
docker compose up -d rabbitmq
```

**Memory:** ~500 MB

### Staging (Monitoring)

```bash
# Start all services
docker compose up -d
```

**Memory:** ~2.3 GB

### Production (Full Stack)

Same as staging, but with:
- Increased retention periods
- Higher resource limits
- Backup strategy

---

## 🔄 Maintenance

### Backup

```bash
# Backup volumes
docker run --rm -v rabbitmq-data:/data -v $(pwd):/backup alpine tar czf /backup/rabbitmq-backup.tar.gz /data

# Backup Grafana dashboards
docker run --rm -v grafana-data:/data -v $(pwd):/backup alpine tar czf /backup/grafana-backup.tar.gz /data
```

### Cleanup Old Data

```bash
# Prometheus (automatic based on retention)
# Loki (automatic based on retention)

# Manual cleanup
docker compose down
docker volume rm prometheus-data loki-data
docker compose up -d
```

### Update Services

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d
```

---

## 📚 References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)

---

## 🎯 Next Steps

1. ✅ Services running
2. ⬜ Configure Grafana data sources
3. ⬜ Import dashboards
4. ⬜ Expose `/metrics` in all services
5. ⬜ Implement structured logging
6. ⬜ Setup alerts (optional)
