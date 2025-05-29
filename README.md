# Thanos Multi-Cluster Monitoring POC

A comprehensive proof-of-concept demonstrating a multi-cluster monitoring setup using Thanos, Prometheus, Grafana, and Alertmanager. This project showcases how to implement a highly available, scalable monitoring solution that can query metrics across multiple Prometheus instances and store data in object storage for long-term retention.

## Architecture Overview

This setup implements a complete Thanos architecture with the following components:

- **4 Prometheus Instances** - Simulating metrics collection from two clusters (chicago and seattle) with replica configurations
- **Thanos Sidecars** - Running alongside each Prometheus instance for gRPC communication and object storage uploads
- **Thanos Query/Query Frontend** - Providing a unified query interface across all Prometheus instances
- **Thanos Store Gateway** - Enabling queries against historical data in object storage
- **Thanos Compactor** - Performing data compaction and downsampling in object storage
- **Thanos Ruler** - Running recording and alerting rules at the global level
- **Thanos Bucket Web** - Web interface for exploring object storage buckets
- **Grafana** - Visualization dashboard with Thanos as data source
- **Alertmanager** - Alert routing and management
- **Node Exporter & cAdvisor** - System and container metrics collection

## Project Structure

```
├── docker-compose.yml              # Main orchestration file
├── prometheus/                     # Prometheus configurations
│   ├── prometheus1.yaml           # Chicago cluster, replica 1
│   ├── prometheus2.yaml           # Chicago cluster, replica 2
│   ├── prometheus3.yaml           # Seattle cluster, replica 1
│   ├── prometheus4.yaml           # Seattle cluster, replica 2
│   └── alert.rules                # Prometheus alerting rules
├── thanos/                        # Thanos configurations
│   ├── bucket_config.yml          # S3 bucket configuration
│   └── alert_down_services.rules.yaml # Thanos ruler alert rules
├── alertmanager/                  # Alertmanager configuration
│   └── config.yml                 # Alert routing configuration
├── grafana/                       # Grafana configurations
│   ├── config.monitoring          # Environment variables
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml     # Thanos datasource configuration
└── images/                        # Documentation images
```

## Key Features

### Multi-Cluster Setup
- **Chicago Cluster**: prometheus-1 (r1) and prometheus-2 (r2)
- **Seattle Cluster**: prometheus-3 (r1) and prometheus-4 (r2)
- Each cluster has replica labels for high availability

### Thanos Components
- **Sidecars**: Attach to each Prometheus instance for real-time querying and data upload
- **Query Frontend**: Provides query acceleration and splitting
- **Store Gateway**: Enables historical data queries from object storage
- **Compactor**: Handles data compaction and retention policies
- **Ruler**: Executes global alerting and recording rules
- **Bucket Web**: Object storage browser interface

### Data Collection
- **Node Exporter**: System-level metrics (CPU, memory, disk, network)
- **cAdvisor**: Container-level metrics
- **Prometheus**: Application and infrastructure metrics
- **Thanos**: Thanos component metrics

### Storage
- **Local Storage**: 30-minute TSDB blocks for fast recent queries
- **Object Storage**: S3-compatible storage for long-term retention
- **Data Retention**: Configurable retention policies via Thanos Compactor

## Getting Started

### Prerequisites
- Docker and Docker Compose
- S3-compatible object storage (AWS S3, MinIO, etc.)
- At least 4GB RAM available for containers
- Proper network configuration (see AWS Security Groups section below)

### Configuration

1. **Update S3 Configuration**
   Edit `thanos/bucket_config.yml` with your S3 credentials:
   ```yaml
   type: S3
   config:
     bucket: "your-bucket-name"
     access_key: "your-access-key"
     secret_key: "your-secret-key"
     endpoint: "s3.your-region.amazonaws.com"
     region: "your-region"
   ```

2. **Configure Alerting (Optional)**
   Update `alertmanager/config.yml` with your notification channels:
   ```yaml
   receivers:
     - name: 'slack'
       slack_configs:
         - send_resolved: true
           username: 'alertmanager'
           channel: '#alerts'
           api_url: 'your-webhook-url'
   ```

### AWS Security Groups Configuration

If deploying on AWS EC2, configure the following security group rules to allow access to all services:

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| Custom TCP | TCP | 9090-9093 | 0.0.0.0/0 | Prometheus + Alertmanager |
| Custom TCP | TCP | 9081-9084 | 0.0.0.0/0 | Prometheus replicas |
| Custom TCP | TCP | 10901-10904 | 0.0.0.0/0 | Thanos Query Frontend, Query, Ruler, Bucket Web |
| Custom TCP | TCP | 3000 | 0.0.0.0/0 | Grafana UI |
| Custom TCP | TCP | 9100 | 0.0.0.0/0 | Node Exporter |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | cAdvisor |
| SSH | TCP | 22 | YOUR_IP/32 | SSH access (Personal IP) |

> **Security Note**: The above configuration allows public access (0.0.0.0/0) to monitoring services for demonstration purposes. In production, restrict source IPs to your organization's network ranges or use a VPN/bastion host setup.

### Deployment

1. **Start the entire stack:**
   ```bash
   docker compose up -d
   ```

2. **Verify all services are running:**
   ```bash
   docker compose ps
   ```

### Accessing Services

| Service | URL | Description |
|---------|-----|-------------|
| Grafana | http://localhost:3000 | Dashboards (admin/foobar) |
| Thanos Query Frontend | http://localhost:10901 | Global query interface |
| Thanos Query | http://localhost:10902 | Direct query interface |
| Thanos Ruler | http://localhost:10903 | Global ruler interface |
| Thanos Bucket Web | http://localhost:10904 | Object storage browser |
| Prometheus 1 | http://localhost:9081 | Chicago cluster r1 |
| Prometheus 2 | http://localhost:9082 | Chicago cluster r2 |
| Prometheus 3 | http://localhost:9083 | Seattle cluster r1 |
| Prometheus 4 | http://localhost:9084 | Seattle cluster r2 |
| Alertmanager | http://localhost:9093 | Alert management |
| Node Exporter | http://localhost:9100 | System metrics |
| cAdvisor | http://localhost:8080 | Container metrics |

## Monitoring and Alerting

### Prometheus Alerts
- **service_down**: Triggers when any service is unreachable for >2 minutes
- **high_load**: Triggers when node load exceeds 0.5 for >2 minutes

### Thanos Ruler Alerts
- **PrometheusReplicaDown**: Monitors Prometheus replica availability across clusters

### Alert Flow
1. Prometheus → Alertmanager (local alerts)
2. Thanos Ruler → Alertmanager (global alerts)
3. Alertmanager → Notification channels (Slack, email, etc.)

## Data Flow

```
┌─────────────┐    ┌──────────────┐     ┌─────────────┐
│ Exporters   │───▶│ Prometheus   │───▶│ Thanos      │
│ (metrics)   │    │ (collection) │     │ Sidecar     │
└─────────────┘    └──────────────┘     └─────────────┘
                                              │
                                              ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ Grafana     │◀───│ Thanos Query  │◀───│ Object      │
│ (dashboards)│     │ (federation) │     │ Storage     │
└─────────────┘     └──────────────┘     └─────────────┘
```

## Scaling and Production Considerations

### High Availability
- Multiple Prometheus replicas per cluster
- Thanos Query can handle sidecar failures
- Object storage provides data durability

### Performance Tuning
- Adjust `--storage.tsdb.max-block-duration` and `--storage.tsdb.min-block-duration`
- Configure Thanos Compactor retention policies
- Use Query Frontend for query acceleration

### Security
- Enable authentication in Grafana
- Secure object storage credentials
- Implement network policies for service communication
- Restrict security group access to specific IP ranges in production
- Use VPN or bastion hosts for administrative access

## Troubleshooting

### Common Issues

1. **Services not starting**: Check Docker logs with `docker-compose logs <service>`
2. **No data in Grafana**: Verify Thanos Query can reach sidecars
3. **Object storage errors**: Validate S3 credentials and bucket permissions
4. **High memory usage**: Adjust Prometheus retention settings

### Useful Commands

```bash
# View logs for all services
docker-compose logs -f

# Restart specific service
docker-compose restart thanos-querier

# Check service health
docker-compose exec thanos-querier wget -qO- http://localhost:10902/-/healthy

# Scale Prometheus instances
docker-compose up -d --scale prometheus-1=2
```

## Contributing

This is a POC project demonstrating Thanos capabilities. For production deployments:

1. Use Kubernetes for orchestration
2. Implement proper secret management
3. Add monitoring for the monitoring stack
4. Configure appropriate retention policies
5. Set up backup strategies for configurations

## References

- [Thanos Documentation](https://thanos.io/tip/thanos/getting-started.md/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)