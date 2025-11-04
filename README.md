# pi-monitoring

A comprehensive monitoring stack for Raspberry Pi, built with Docker Compose. This project provides real-time system metrics monitoring, network flow analysis, and visualization capabilities using industry-standard open-source tools.

## Overview

This monitoring solution combines multiple powerful tools to provide complete observability for your Raspberry Pi:

- **System Metrics Monitoring**: Track CPU, memory, disk, and network usage
- **Network Flow Analysis**: Monitor network traffic patterns using NetFlow v5 protocol
- **Time-Series Database**: Store metrics and network flow data
- **Visualization**: Create beautiful dashboards to visualize your data

## Architecture

The stack consists of six containerized services working together:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│ node-       │────▶│ Prometheus   │────▶│   Grafana    │
│ exporter    │     │              │     │ (Port 3000)  │
└─────────────┘     └──────────────┘     └──────────────┘
                           │
                           ▼
                    System Metrics

┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   fprobe    │────▶│   Telegraf   │────▶│  InfluxDB    │
│ (NetFlow)   │     │  (Port 2055) │     │ (Port 8086)  │
└─────────────┘     └──────────────┘     └──────────────┘
                           │
                           ▼
                    Network Flow Data
```

## Components

### Node Exporter (Port 9100)
- Collects hardware and OS-level metrics from the host system
- Exposes metrics in Prometheus format
- Monitors: CPU, memory, disk I/O, network interfaces, and more
- Runs with read-only access to system directories

### Prometheus (Port 9090)
- Time-series database for storing system metrics
- Scrapes metrics from Node Exporter every 30 seconds
- Configured with 7-day retention period and 4GB size limit
- Can monitor additional targets (configured for 192.168.1.20:9100)
- Web UI available for querying metrics and creating alerts

### Grafana (Port 3000)
- Visualization and analytics platform
- Creates customizable dashboards for metrics
- Supports multiple data sources (Prometheus, InfluxDB)
- Default credentials: `admin` / `secret`

### fprobe
- Network flow probe that captures packet data from network interfaces
- Monitors the `wlan0` interface (WiFi)
- Exports NetFlow v5 format data to Telegraf
- Runs in host network mode for direct interface access

### Telegraf (Port 2055/UDP)
- Receives NetFlow v5 data from fprobe
- Processes and forwards network flow data to InfluxDB
- Acts as a bridge between NetFlow and InfluxDB

### InfluxDB (Port 8086)
- Time-series database optimized for high-write loads
- Stores network flow data from Telegraf
- Configured with 1-week retention policy
- Organization: `home`, Bucket: `netflow`
- Default credentials: `admin` / `secret-password`
- Admin token: `secret-token`

## Prerequisites

- Raspberry Pi (tested on Pi 3/4/5)
- Docker Engine installed
- Docker Compose installed
- Minimum 8GB SD card (16GB+ recommended)
- Network access for pulling Docker images

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/Quarfein/pi-monitoring.git
   cd pi-monitoring
   ```

2. Review and customize configurations (optional):
   - `docker-compose.yml`: Adjust ports, passwords, or resource limits
   - `prometheus/prometheus.yml`: Add additional monitoring targets
   - `telegraf/telegraf.conf`: Modify NetFlow settings or output configuration
   - `fprobe/Dockerfile`: Change network interface from `wlan0` if needed

3. Start the monitoring stack:
   ```bash
   docker-compose up -d
   ```

4. Verify all services are running:
   ```bash
   docker-compose ps
   ```

## Usage

### Accessing the Services

- **Grafana Dashboard**: http://your-pi-ip:3000
  - Username: `admin`
  - Password: `secret`
  - Add Prometheus data source: `http://prometheus:9090`
  - Add InfluxDB data source: `http://influxdb:8086` (use token authentication)

- **Prometheus Web UI**: http://your-pi-ip:9090
  - Query metrics directly
  - View targets and their status
  - Create alerts and recording rules

- **Node Exporter Metrics**: http://your-pi-ip:9100/metrics
  - Raw metrics endpoint (primarily for Prometheus)

- **InfluxDB UI**: http://your-pi-ip:8086
  - Username: `admin`
  - Password: `secret-password`
  - Browse and query network flow data

### Setting Up Dashboards

1. Log in to Grafana
2. Add data sources:
   - **Prometheus**: Configuration → Data Sources → Add Prometheus
     - URL: `http://prometheus:9090`
   - **InfluxDB**: Configuration → Data Sources → Add InfluxDB
     - Query Language: Flux
     - URL: `http://influxdb:8086`
     - Organization: `home`
     - Token: `secret-token`
     - Default Bucket: `netflow`

3. Import or create dashboards:
   - Import pre-built Node Exporter dashboards (Dashboard ID: 1860)
   - Create custom dashboards for network flow analysis

### Monitoring Additional Hosts

To monitor other devices on your network:

1. Install Node Exporter on the target device
2. Edit `prometheus/prometheus.yml` and add the target to the `infra` job:
   ```yaml
   - job_name: 'infra'
     static_configs:
       - targets: 
           - 'node-exporter:9100'
           - '192.168.1.20:9100'
           - 'your-device-ip:9100'  # Add your device here
   ```
3. Reload Prometheus configuration:
   ```bash
   docker-compose restart prometheus
   ```

### Customizing Network Interface Monitoring

By default, fprobe monitors the `wlan0` interface. To change this:

1. Edit `fprobe/Dockerfile` and change the interface name:
   ```dockerfile
   ENTRYPOINT ["fprobe", "-i", "eth0", "-fip", "-l", "2", "localhost:2055"]
   ```
2. Rebuild and restart:
   ```bash
   docker-compose up -d --build fprobe
   ```

## Data Retention

- **Prometheus**: 7 days or 4GB (whichever is reached first)
- **InfluxDB**: 1 week

To modify retention periods, edit the respective settings in `docker-compose.yml`.

## Persistent Data

The following Docker volumes store persistent data:
- `prometheus_data`: Prometheus metrics database
- `grafana_data`: Grafana dashboards and settings
- `influxdb_data`: InfluxDB network flow data

Volumes are preserved even when containers are removed.

## Stopping the Stack

Stop all services:
```bash
docker-compose down
```

Stop and remove all data (destructive):
```bash
docker-compose down -v
```

## Troubleshooting

### Services won't start
- Check Docker service status: `sudo systemctl status docker`
- View service logs: `docker-compose logs <service-name>`
- Ensure sufficient disk space: `df -h`

### Can't access web interfaces
- Verify containers are running: `docker-compose ps`
- Check port conflicts: `sudo netstat -tulpn | grep <port>`
- Ensure firewall allows connections (if enabled)

### Prometheus not scraping targets
- Check Prometheus targets page: http://your-pi-ip:9090/targets
- Verify target IPs are reachable
- Review Prometheus logs: `docker-compose logs prometheus`

### No network flow data in InfluxDB
- Confirm fprobe is running: `docker-compose ps fprobe`
- Check network interface exists: `ip link show wlan0`
- Verify Telegraf is receiving data: `docker-compose logs telegraf`
- Ensure there's actual network traffic on the monitored interface

### High memory usage
- Reduce Prometheus retention: Modify `--storage.tsdb.retention.time` and `--storage.tsdb.retention.size`
- Increase Prometheus scrape interval in `prometheus/prometheus.yml`
- Limit InfluxDB retention period

## Security Considerations

**Important**: This configuration uses default passwords for demonstration purposes. For production use:

1. Change all default passwords in `docker-compose.yml`:
   - Grafana: `GF_SECURITY_ADMIN_PASSWORD`
   - InfluxDB: `DOCKER_INFLUXDB_INIT_PASSWORD` and `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN`

2. Consider using Docker secrets or environment files for sensitive data

3. Implement network isolation if exposing services beyond your local network

4. Enable HTTPS for web interfaces in production environments

## Resource Requirements

Minimum recommended specifications:
- RAM: 1GB minimum, 2GB recommended
- Storage: 8GB free space (more for longer retention)
- Network: 100Mbps (for network flow monitoring)

## License

This project configuration is provided as-is for monitoring Raspberry Pi systems.

## Contributing

Feel free to open issues or submit pull requests for improvements.

## Acknowledgments

This monitoring stack leverages these excellent open-source projects:
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)
- [InfluxDB](https://www.influxdata.com/)
- [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [fprobe](https://github.com/RIPE-NCC/fprobe)
