# Filebeat -> Logstash -> Elasticsearch Monitoring Project

A Docker-based observability stack demonstrating log aggregation and monitoring using Filebeat, Logstash with load balancing, and Elasticsearch Cloud.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Environment                        │
│                                                                  │
│  ┌──────────────┐                                               │
│  │  post-log1   │──► logs1:/var/log/mylogs/abc.log              │
│  └──────────────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐          ┌──────────────────┐                │
│  │  filebeat1   │─────────►│                  │                │
│  └──────────────┘          │  logstash-proxy  │                │
│                            │  (nginx:5044)    │                │
│  ┌──────────────┐          │                  │                │
│  │  post-log2   │──► logs2:/var/log/mylogs/abc.log             │
│  └──────────────┘          │  Round-robin LB  │                │
│         │                  └──────────────────┘                │
│         ▼                      │            │                   │
│  ┌──────────────┐              │            │                   │
│  │  filebeat2   │──────────────┘            │                   │
│  └──────────────┘                           │                   │
│                                              ▼                   │
│                                    ┌──────────────┐             │
│                                    │  logstash1   │─────┐       │
│                                    └──────────────┘     │       │
│                                                          │       │
│                                    ┌──────────────┐     │       │
│                                    │  logstash2   │─────┤       │
│                                    └──────────────┘     │       │
│                                                          │       │
└──────────────────────────────────────────────────────────┼───────┘
                                                           │
                                                           ▼
                                              ┌────────────────────┐
                                              │  Elastic Cloud     │
                                              │                    │
                                              │  • Elasticsearch   │
                                              │  • Kibana          │
                                              │  • Monitoring      │
                                              └────────────────────┘
```

## Features

- **Dual Filebeat Instances**: Two independent Filebeat agents collecting logs from separate sources
- **Load Balanced Logstash**: Nginx-based TCP load balancer distributing traffic across two Logstash instances
- **Automatic Log Generation**: Busybox containers generating test logs at different intervals (10s and 5s)
- **Full Stack Monitoring**: Built-in monitoring for Filebeat and Logstash sent to Elastic Cloud
- **Pre-configured Dashboards**: Kibana dashboards for visualizing the complete data pipeline

## Prerequisites

- Docker and Docker Compose installed
- **Elastic Cloud account** (this must be set up separately)
  - Create a deployment at [cloud.elastic.co](https://cloud.elastic.co)
  - Note your Cloud ID and credentials
  - Obtain your Cluster UUID from the deployment

## Setup Instructions

### 1. Prepare Elastic Cloud

Before running this project, you need to set up Elastic Cloud:

1. Sign up or log in to [Elastic Cloud](https://cloud.elastic.co)
2. Create a new deployment (Elasticsearch + Kibana)
3. Save the following information:
   - **Cloud ID**: Found in deployment details
   - **Cloud Auth**: `elastic:your_password` format
   - **Cluster UUID**: Found in deployment overview

### 2. Configure Environment Variables

Copy the example environment file and fill in your Elastic Cloud credentials:

```bash
cp .env.example .env
```

Edit `.env` and add your credentials:

```bash
STACK_VERSION=8.18.2
PROJECT=dc-filebeat-logstash-monitoring-v8
ELASTIC_PASSWORD=your_password_here
CLOUD_ID=your_cloud_id_here
CLOUD_AUTH=elastic:your_password_here
CLUSTER_UUID=your_cluster_uuid_here
```

### 3. Create Docker Network

Create the external network required by the stack:

```bash
docker network create elastic
```

### 4. Start the Stack

Launch all services:

```bash
docker-compose up -d
```

Verify all containers are running:

```bash
docker-compose ps
```

### 5. Import Kibana Dashboards

To visualize the monitoring data:

1. Log in to your Kibana instance in Elastic Cloud
2. Navigate to **Stack Management** → **Saved Objects**
3. Click **Import**
4. Select the `kibana-saved-objects.ndjson` file from this repository
5. Click **Import**

This will install:
- **Data Views**: `.monitoring-beats*`, `.monitoring-logstash*`, `test-filebeat-logstash`
- **Dashboard**: "filebeat -> logstash -> elasticsearch observability dashboard"
- Pre-configured visualizations for monitoring the complete pipeline

### 6. View the Dashboard

1. Go to **Analytics** → **Dashboard** in Kibana
2. Open "filebeat -> logstash -> elasticsearch observability dashboard"
3. You should see real-time metrics for:
   - Filebeat events published and acknowledged
   - Logstash events in/out
   - Elasticsearch document count
   - Pipeline health metrics

## Components

### Log Generators

- **post-log1**: Generates log entries every 10 seconds
- **post-log2**: Generates log entries every 5 seconds

### Filebeat Instances

- **filebeat1**: Monitors logs1 volume
- **filebeat2**: Monitors logs2 volume
- Both send to Logstash via the nginx proxy
- Monitoring data sent directly to Elastic Cloud

### Load Balancer

- **logstash-proxy**: Nginx-based TCP load balancer
- Round-robin distribution to Logstash instances
- Health checking with automatic failover
- Port: 5044

### Logstash Instances

- **logstash1** and **logstash2**: Process incoming Beats events
- Auto-reload pipeline configuration
- Send processed logs to Elasticsearch index: `test-filebeat-logstash`
- Monitoring data sent to Elastic Cloud

## Configuration Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main orchestration file |
| `filebeat1.yml` / `filebeat2.yml` | Filebeat configurations |
| `logstash.yml` | Logstash settings |
| `logstash/pipeline/syslog-pipeline.conf` | Logstash pipeline |
| `nginx.conf` | Load balancer configuration |
| `log4j2.properties` | Logstash logging (optional debug) |

## Monitoring Metrics

The dashboard displays:

### Filebeat Metrics
- Events published (input rate)
- Events acknowledged (output rate)
- Active events in pipeline
- Retry, dropped, and failed event counts

### Logstash Metrics
- Events in (received from Filebeat)
- Events out (sent to Elasticsearch)
- Queued events
- Per-instance breakdown

### Elasticsearch Metrics
- Document count in `test-filebeat-logstash` index
- Ingestion rate over time

## Troubleshooting

### Check Container Logs

```bash
# Filebeat logs
docker-compose logs filebeat1
docker-compose logs filebeat2

# Logstash logs
docker-compose logs logstash1
docker-compose logs logstash2

# Nginx logs
docker-compose logs logstash-proxy
```

### Verify Log Generation

```bash
# Check if logs are being generated
docker exec -it $(docker-compose ps -q post-log1) cat /var/log/mylogs/abc.log
```

### Test Elasticsearch Connection

```bash
# Enter Logstash container
docker exec -it logstash1 bash

# Test connection (requires curl)
curl -X GET "${CLOUD_ID}" -u "${CLOUD_AUTH}"
```

### Enable Debug Logging

Uncomment the debug lines in `log4j2.properties` to enable Elasticsearch output debugging:

```properties
logger.elasticsearchoutput.name = logstash.outputs.elasticsearch
logger.elasticsearchoutput.level = debug
```

## Stopping the Stack

```bash
# Stop all containers
docker-compose down

# Stop and remove volumes (WARNING: deletes logs)
docker-compose down -v
```

## Customization

### Adjust Log Generation Frequency

Edit the `sleep` values in `docker-compose.yml`:

```yaml
post-log1:
  command: >
    sh -c '
      while true
      do
        echo this is a message on `date` >> /var/log/mylogs/abc.log
        sleep 10  # Change this value
      done
    '
```

### Modify Logstash Pipeline

Edit `logstash/pipeline/syslog-pipeline.conf` to add filters or change output:

```ruby
filter {
  grok {
    match => { "message" => "%{GREEDYDATA:log_message}" }
  }
}
```

### Change Load Balancing Method

Edit `nginx.conf` to use different algorithms:

```nginx
upstream logstash_backend {
    least_conn;  # Use least connections instead of round-robin
    # OR
    ip_hash;     # Use IP hash for session persistence
    
    server logstash1:5044;
    server logstash2:5044;
}
```

## Project Structure

```
.
├── docker-compose.yml              # Main orchestration
├── .env.example                    # Environment template
├── filebeat1.yml                   # Filebeat 1 config
├── filebeat2.yml                   # Filebeat 2 config
├── logstash.yml                    # Logstash config
├── log4j2.properties               # Logstash logging
├── nginx.conf                      # Load balancer config
├── kibana-saved-objects.ndjson     # Kibana dashboards/views
├── logstash/
│   ├── pipeline/
│   │   └── syslog-pipeline.conf   # Logstash pipeline
│   └── secrets/                    # (Optional) secrets directory
└── README.md                       # This file
```

## License

This project is provided as-is for educational and demonstration purposes.

## Support

For issues related to:
- **Elastic Stack**: See [Elastic documentation](https://www.elastic.co/guide/index.html)
- **Docker**: See [Docker documentation](https://docs.docker.com/)
- **This project**: Open an issue in the repository

## Version Information

- Elastic Stack: 8.18.2
- Docker Compose: v3.x
- Nginx: Alpine (latest)