# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a comprehensive observability stack for monitoring Claude Code usage, performance, and costs. It implements OpenTelemetry (OTel) standards to collect metrics and events from Claude Code and visualizes them through Grafana dashboards.

**Architecture**: Claude Code → OpenTelemetry Collector → Prometheus (metrics) + Loki (logs) → Grafana (dashboards)

## Core Commands

### Stack Management
```bash
make up                 # Start all services (Collector, Prometheus, Loki, Grafana)
make down              # Stop all services
make restart           # Restart services
make clean             # Clean up containers and volumes
make status            # Show service status and URLs
```

### Monitoring & Debugging
```bash
make logs              # View all service logs
make logs-collector    # View OpenTelemetry collector logs only
make logs-prometheus   # View Prometheus logs
make logs-grafana      # View Grafana logs
```

### Validation & Setup
```bash
make validate-config   # Validate docker-compose.yml and collector-config.yaml
make setup-claude      # Display Claude Code telemetry setup instructions
```

## Architecture Components

| Service | Purpose | Port(s) | Key File |
|---------|---------|---------|----------|
| **OpenTelemetry Collector** | Ingests metrics/logs from Claude Code | 4317 (gRPC), 4318 (HTTP) | `collector-config.yaml` |
| **Prometheus** | Time-series metrics storage & querying | 9090 | `prometheus.yml` |
| **Loki** | Log aggregation & storage | 3100 | (embedded config) |
| **Grafana** | Visualization & dashboards | 3000 | `grafana-datasources.yml`, `grafana-dashboards.yml` |

### Service Dependencies
- **Grafana** depends on Prometheus and Loki (configured in `docker-compose.yml`)
- **Prometheus** depends on OpenTelemetry Collector
- **OpenTelemetry Collector** is the entry point for Claude Code telemetry

## Configuration Files

### `collector-config.yaml`
Configures OpenTelemetry Collector:
- **Receivers**: OTLP gRPC (4317) and HTTP (4318) endpoints
- **Processors**: Resource attributes enrichment
- **Exporters**: Prometheus (metrics) and HTTP (logs to Loki)
- **Pipelines**: Separate routes for metrics and logs

Key configuration points:
- Receiver endpoints must match OTLP_EXPORTER_OTLP_ENDPOINT set in Claude Code
- Resource processor adds environment attributes to all telemetry
- Prometheus exporter listens on 8889 (scraped by Prometheus)

### `prometheus.yml`
- Defines Prometheus scrape targets
- Collector metrics scraped from `otel-collector:8889`

### `loki-config.yaml`
- Configures Loki log storage and ingestion pipeline
- Defines storage backend and retention settings
- Enables scraping of logs from OpenTelemetry Collector

### `docker-compose.yml`
- Defines all services, volumes, ports, and network configuration
- Uses custom bridge network `otel-network` for service communication
- Uses `otel/opentelemetry-collector-contrib` (includes extra receivers/processors)
- Includes named Docker volumes for data persistence (more portable than bind mounts):
  - **prometheus-data**: Persists Prometheus time-series metrics
  - **loki-data**: Persists Loki log data
  - **grafana-data**: Persists Grafana dashboards and data sources

### `grafana-datasources.yml` & `grafana-dashboards.yml`
- Auto-provisioning configuration for Grafana
- Loads Prometheus and Loki as data sources
- Provisions `claude-code-dashboard.json`

### `claude-code-dashboard.json`
Pre-configured Grafana dashboard with panels organized into sections:
- **Overview**: Active sessions, cost, token usage, LOC changes
- **Cost & Usage Analysis**: Cost trends, API requests, token breakdown
- **Tool Usage & Performance**: Tool frequency, success rates
- **Performance & Errors**: API latency, error rates
- **User Activity & Productivity**: Code changes, commits, PRs
- **Event Logs**: Tool execution events and API errors

## Claude Code Telemetry Configuration

To send metrics to this observability stack, configure Claude Code with:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Optional: For faster feedback during development
export OTEL_METRIC_EXPORT_INTERVAL=10000
export OTEL_LOGS_EXPORT_INTERVAL=5000
```

For more options (custom endpoints, authentication headers, mTLS), see `CLAUDE_OBSERVABILITY.md`.

## Key Metrics Collected

**Core Metrics** (counter/gauge data):
- `claude_code.session.count` - CLI sessions started
- `claude_code.lines_of_code.count` - Lines modified (added/removed)
- `claude_code.commit.count`, `claude_code.pull_request.count` - Git operations
- `claude_code.cost.usage` - Cost by model
- `claude_code.token.usage` - Token usage breakdown (input/output/cache)

**Event Data** (structured logs):
- `claude_code.user_prompt` - User prompts with content (if enabled)
- `claude_code.tool_result` - Tool execution with timing
- `claude_code.api_request` - API calls with duration and token counts
- `claude_code.api_error` - API errors with status codes
- `claude_code.tool_decision` - Tool permission decisions

## Development & Customization

### Adding New Panels to Dashboard
The dashboard (`claude-code-dashboard.json`) is a standard Grafana dashboard JSON. To add panels:
1. Create/edit panels in Grafana UI (http://localhost:3000)
2. Export the dashboard JSON
3. Replace `claude-code-dashboard.json`

### Modifying Collector Pipeline
To change how metrics/logs are processed:
1. Edit `collector-config.yaml`
2. Run `make validate-config` to check syntax
3. Run `make restart` to apply changes
4. Check logs: `make logs-collector`

### Retention & Performance Tuning
- **Prometheus**: Configure retention in `prometheus.yml` (default: 15 days)
- **Loki**: Adjust retention in docker-compose.yml environment variables
- **Metrics cardinality**: Claude Code supports `OTEL_METRICS_INCLUDE_*` variables to control attribute inclusion

## Troubleshooting

### Services won't start
```bash
make validate-config    # Check configuration syntax
make logs              # View error logs
docker system prune -f  # Clear unused Docker resources if storage is an issue
```

### Resetting data and starting fresh
```bash
make clean                           # Stops containers and removes volumes
docker volume rm prometheus-data    # Remove Prometheus data volume
docker volume rm loki-data          # Remove Loki data volume
docker volume rm grafana-data       # Remove Grafana data volume
make up                              # Start services with clean state
```

### Metrics not appearing in Grafana
1. Verify Claude Code is running with telemetry enabled
2. Check collector logs: `make logs-collector`
3. Verify Prometheus scrape: http://localhost:9090/targets
4. Confirm data source connectivity in Grafana UI

### Collector connection errors
- Verify `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317` matches collector receiver
- Check firewall/network: `curl http://localhost:4317` (may not respond, but connection should succeed)
- Collector logs will show rejected connection attempts

## Documentation References

- **CLAUDE_OBSERVABILITY.md**: Complete Claude Code OpenTelemetry configuration reference
- **CONTRIBUTING.md**: Contribution guidelines
- **README.md**: Feature overview and use cases
- Official OpenTelemetry: https://opentelemetry.io/docs/
- Grafana Documentation: https://grafana.com/docs/
