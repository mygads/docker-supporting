# Supporting Services

Infrastructure services untuk Genfity & GovConnect.

## Stack Overview

| Service | Port | Purpose | Memory Limit |
|---------|------|---------|--------------|
| **RabbitMQ** | 5672, 15672 | Message broker | 512 MB |
| **pgAdmin** | 5050 | Database management UI | 512 MB |
| **Redis** | 6379 | Cache, session store, rate limiting | 512 MB |

---

## Quick Start

```bash
cd supporting
cp .env.example .env
# Edit .env dengan credentials

docker compose up -d
docker compose ps
```

## Access

| Service | URL | Credentials |
|---------|-----|-------------|
| RabbitMQ Management | http://localhost:15672 | admin / (see .env) |
| pgAdmin | http://localhost:5050 | admin@genfity.com / (see .env) |
| Redis | localhost:6379 | password di .env |

---

## Redis

### Database Allocation

| DB | Purpose |
|----|---------|
| 0 | Default / general |
| 1 | Sessions |
| 2 | Cache |
| 3 | Rate limiting |

### Connection URLs

```env
# General
REDIS_URL=redis://:PASSWORD@redis:6379/0

# Per-database
REDIS_SESSION_URL=redis://:PASSWORD@redis:6379/1
REDIS_CACHE_URL=redis://:PASSWORD@redis:6379/2
```

### Verify Redis

```bash
# Ping
docker exec redis redis-cli -a YOUR_PASSWORD ping

# Info
docker exec redis redis-cli -a YOUR_PASSWORD INFO memory

# Monitor real-time
docker exec -it redis redis-cli -a YOUR_PASSWORD MONITOR
```

### Configuration

Redis config: `redis/redis.conf`
- **Persistence**: AOF (primary) + RDB snapshots (backup)
- **Memory limit**: 256MB with LRU eviction
- **Security**: Dangerous commands (FLUSHDB, FLUSHALL, DEBUG) disabled

---

## File Structure

```
supporting/
├── docker-compose.yml
├── .env / .env.example
├── README.md
├── pgadmin/
│   └── servers.json
├── rabbitmq/
│   ├── rabbitmq.conf
│   └── definitions.json
└── redis/
    └── redis.conf
```

## Docker Networks

- **infra-network**: Internal communication between supporting services
- Services yang butuh Redis/RabbitMQ harus join `infra-network` atau di-connect manual

```bash
# Connect Redis ke app networks
docker network connect genfity-network redis
docker network connect govconnect-network redis
```

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
