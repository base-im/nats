# First Init - Required Directories

For the first initialization, create the following directories:

```bash
mkdir -p data/n1 data/n2 data/n3 data/prometheus data/grafana certs
```

These directories are for:
- `data/n1`, `data/n2`, `data/n3` - JetStream storage for each NATS node
- `data/prometheus` - Prometheus time series database storage
- `data/grafana` - Grafana data and configuration persistence
- `certs` - TLS certificates (if enabling TLS)

# Docker Daily Management Commands

## Basic Container Operations

```bash
# Start all services
docker compose up -d

# Stop all services
docker-compose down

# Restart specific service
docker-compose restart nats-1
docker-compose restart grafana

# Stop specific service
docker-compose stop nats-2

# Start specific service
docker-compose start nats-2

# View running containers
docker ps
docker ps -a  # Include stopped containers

# Check container resource usage
docker stats
docker stats --no-stream  # One-time snapshot
```

## Log Management

```bash
# View logs for specific container
docker logs nats-1
docker logs nats-1 -f  # Follow log output
docker logs nats-1 --tail 100  # Last 100 lines
docker logs nats-1 --since 30m  # Last 30 minutes

# View logs for all services
docker-compose logs
docker-compose logs -f  # Follow all logs
docker-compose logs --tail=50 nats-1 nats-2  # Last 50 lines from multiple services

# Check log size
docker inspect nats-1 | grep -A 5 LogConfig
du -sh data/*/  # Check data directory sizes
```

## Container Inspection and Debugging

```bash
# Execute commands in running container
docker exec nats-1 nats-server -v  # Check NATS version
docker exec -it nats-1 /bin/sh  # Interactive shell
docker exec grafana grafana-cli plugins ls  # List Grafana plugins

# Inspect container details
docker inspect nats-1
docker inspect nats-1 | jq '.[0].State'  # Container state
docker inspect nats-1 | jq '.[0].NetworkSettings.Networks'  # Network info

# Check container health
docker ps --format "table {{.Names}}\t{{.Status}}"
docker inspect nats-1 | jq '.[0].State.Health'

# View container processes
docker top nats-1
```

## Network Troubleshooting

```bash
# List docker networks
docker network ls
docker network inspect nats-prod_nnet

# Test connectivity between containers
docker exec nats-1 ping nats-2
docker exec nats-1 wget -qO- http://nats-2:8222/healthz
docker exec prometheus wget -qO- http://nats-exporter:7777/metrics | head -20

# Check exposed ports
docker port nats-1
netstat -tlnp | grep -E "4222|8222|3000|9090"  # On host
```

## Resource Management

```bash
# Clean up unused resources
docker system prune  # Remove stopped containers, unused networks, dangling images
docker system prune -a  # Also remove unused images
docker volume prune  # Remove unused volumes

# Check disk usage
docker system df
df -h data/  # Check host data directory
du -sh data/*  # Size of each data subdirectory

# Update container resource limits (requires recreate)
docker-compose down nats-1
# Edit docker-compose.yml to add limits
docker-compose up -d nats-1
```

## Backup and Restore

```bash
# Backup data directories
tar -czf backup-$(date +%Y%m%d).tar.gz data/

# Backup specific service data
tar -czf nats-data-$(date +%Y%m%d).tar.gz data/n1 data/n2 data/n3
tar -czf grafana-backup-$(date +%Y%m%d).tar.gz data/grafana

# Export container as image
docker commit nats-1 nats-backup:$(date +%Y%m%d)
docker save nats-backup:$(date +%Y%m%d) > nats-backup.tar

# Restore from backup
docker-compose down
tar -xzf backup-20240327.tar.gz
docker-compose up -d
```

## Update and Maintenance

```bash
# Pull latest images
docker-compose pull

# Update single service
docker-compose pull nats-1
docker-compose up -d nats-1

# Rolling update for NATS cluster
for node in nats-3 nats-2 nats-1; do
  docker-compose pull $node
  docker-compose up -d $node
  sleep 30  # Wait for cluster to stabilize
  curl -s http://localhost:8222/healthz | jq
done

# Check image versions
docker images | grep -E "nats|grafana|prometheus"
docker inspect nats:2.11.8 | jq '.[0].Config.Labels'
```

## Emergency Recovery

```bash
# Force remove stuck container
docker rm -f nats-1

# Recreate specific service
docker-compose up -d --force-recreate nats-1

# Reset service completely
docker-compose rm -fsv nats-1
rm -rf data/n1/*
docker-compose up -d nats-1

# Emergency stop all
docker stop $(docker ps -q)

# Clean restart everything
docker-compose down -v  # Warning: removes volumes
rm -rf data/*
mkdir -p data/n1 data/n2 data/n3 data/prometheus data/grafana
docker-compose up -d
```

## Performance Monitoring

```bash
# Real-time performance metrics
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Check container limits vs usage
docker stats --no-stream --format "table {{.Container}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Monitor specific metrics
watch -n 2 'docker stats --no-stream | grep nats'

# Export metrics for analysis
docker stats --no-stream --format json > stats.json
```


ss -ltnp | grep 3000