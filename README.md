# Blackbox Monitoring & Alerting System

A comprehensive monitoring solution using Prometheus, Blackbox Exporter, and Grafana for blackbox testing of HTTP endpoints, TCP connections, and ICMP ping.

## 🏗️ Architecture

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Grafana   │    │  Prometheus  │    │   Blackbox  │
│   :3000     │◄──►│    :9090     │◄──►│   :9115     │
└─────────────┘    └──────────────┘    └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Mocksite  │
                    │    :8088    │
                    └─────────────┘
```

## 🚀 Quick Start

### Prerequisites
- Docker and Docker Compose installed
- Ports 3000, 9090, 9115, and 8088 available

### 1. Create the shared network
```bash
docker network create monitoring
```

### 2. Start Prometheus and Blackbox Exporter
```bash
cd prometheus
docker-compose up -d
```

### 3. Start Grafana
```bash
cd grafana
docker-compose up -d
```

### 4. Access the services
- **Grafana**: http://localhost:3000 (admin/admin)
- **Prometheus**: http://localhost:9090
- **Blackbox Exporter**: http://localhost:9115
- **Mock Site**: http://localhost:8088

## 📊 What is Blackbox Testing?

Blackbox testing monitors external endpoints from the **outside perspective** - just like your users would experience them. It's perfect for:

- **Website availability** monitoring
- **API endpoint health** checks
- **Network connectivity** testing
- **Response time** measurements
- **SSL certificate** validation

## 🔧 Configuration Details

### Blackbox Exporter (`prometheus/blackbox.yml`)
The blackbox exporter is configured with three probe types:

- **`http_2xx`**: HTTP GET requests with 5s timeout
- **`tcp_connect`**: TCP connection testing with 5s timeout  
- **`icmp_ping`**: ICMP ping testing with 5s timeout

### Prometheus (`prometheus/prometheus.yml`)
- **Scrape interval**: 15 seconds
- **Targets**: 
  - Prometheus itself
  - Mock site via blackbox exporter
- **Relabeling**: Automatically routes targets through blackbox exporter

### Current Monitoring Targets
- **Mock Site**: `http://mocksite:80/` (nginx container for testing)

## 📈 Adding New Targets

### 1. Add HTTP endpoints
Edit `prometheus/prometheus.yml` and add to the `targets` list:
```yaml
- job_name: blackbox-http
  # ... existing config ...
  static_configs:
    - targets:
        - http://mocksite:80/
        - https://google.com/          # Add external sites
        - https://github.com/          # Add more targets
        - http://your-api.com/health   # Add your APIs
```

### 2. Add TCP endpoints
Add a new job for TCP monitoring:
```yaml
- job_name: blackbox-tcp
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
    - targets:
        - "database-server:5432"      # PostgreSQL
        - "redis-server:6379"         # Redis
        - "mail-server:25"            # SMTP
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
```

### 3. Add ICMP ping targets
```yaml
- job_name: blackbox-icmp
  metrics_path: /probe
  params:
    module: [icmp_ping]
  static_configs:
    - targets:
        - "8.8.8.8"                   # Google DNS
        - "1.1.1.1"                   # Cloudflare DNS
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
```

## 📊 Grafana Dashboards

### Recommended Dashboards
1. **Blackbox Exporter Dashboard** (ID: 7587)
2. **Prometheus Stats** (ID: 3662)
3. **Create custom dashboards** using these metrics:
   - `probe_duration_seconds` - Response time
   - `probe_success` - Success/failure status
   - `probe_http_status_code` - HTTP status codes
   - `probe_ssl_earliest_cert_expiry` - SSL certificate expiry

### Adding Prometheus as Data Source
1. Go to Configuration → Data Sources
2. Add Prometheus
3. URL: `http://prometheus:9090`
4. Access: Server (default)

## 🚨 Alerting Examples

### HTTP Down Alert
```yaml
groups:
  - name: blackbox_alerts
    rules:
      - alert: EndpointDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
```

### High Response Time Alert
```yaml
      - alert: HighResponseTime
        expr: probe_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time for {{ $labels.instance }}"
          description: "{{ $labels.instance }} is responding slowly ({{ $value }}s)"
```

## 🧪 Testing Your Setup

### 1. Test the mock site
```bash
curl http://localhost:8088
# Should return nginx welcome page
```

### 2. Test blackbox exporter
```bash
curl "http://localhost:9115/probe?target=http://mocksite:80/&module=http_2xx"
# Should return metrics about the probe
```

### 3. Test Prometheus
```bash
curl http://localhost:9090/api/v1/targets
# Should show all configured targets
```

## 🔍 Troubleshooting

### Common Issues

1. **Services can't communicate**
   - Ensure the `monitoring` network exists: `docker network ls`
   - Recreate it: `docker network create monitoring`

2. **Port conflicts**
   - Check if ports are in use: `lsof -i :3000`
   - Modify ports in docker-compose files

3. **Blackbox targets not showing**
   - Verify target URLs are accessible
   - Check Prometheus logs: `docker logs prometheus`

4. **Grafana can't connect to Prometheus**
   - Use `http://prometheus:9090` (not localhost)
   - Verify both services are on the same network

### Useful Commands
```bash
# View all containers
docker ps

# View logs
docker logs grafana
docker logs prometheus
docker logs blackbox

# Check network
docker network inspect monitoring

# Restart services
docker-compose restart
```

## 📁 Project Structure
```
blackbox-monitoring-alert/
├── README.md
├── grafana/
│   └── docker-compose.yml
└── prometheus/
    ├── docker-compose.yaml
    ├── prometheus.yml
    └── blackbox.yml
```

## 🔄 Updating and Maintenance

### Update Images
```bash
# Update all services
docker-compose pull
docker-compose up -d

# Update specific service
docker-compose pull prometheus
docker-compose up -d prometheus
```

### Backup Data
```bash
# Backup Grafana data
docker run --rm -v grafana_grafana-storage:/data -v $(pwd):/backup alpine tar czf /backup/grafana-backup.tar.gz -C /data .

# Backup Prometheus data
docker run --rm -v prometheus_prom_data:/data -v $(pwd):/backup alpine tar czf /backup/prometheus-backup.tar.gz -C /data .
```

## 📚 Additional Resources

- [Blackbox Exporter Documentation](https://github.com/prometheus/blackbox_exporter)
- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Docker Networking](https://docs.docker.com/network/)

## 🤝 Contributing

Feel free to submit issues, feature requests, or pull requests to improve this monitoring setup.

---

**Happy Monitoring! 🚀**
