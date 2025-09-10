# Observability Stack with ArgoCD, Fluent Bit, and OpenTelemetry Collector

This repository provides a comprehensive observability solution using ArgoCD for GitOps deployment, Fluent Bit for log processing, and OpenTelemetry Collector for trace and metrics collection.

## Architecture Overview

```
[Akamai DataStream] → [Fluent Bit] → [OpenTelemetry Collector] → [Observability Backends]
                           ↓                    ↓
                    (Log Processing)     (Trace/Metrics Export)
                           ↓                    ↓
                    (OTLP/HTTP:4318)    (Honeycomb/Splunk/Grafana)
```

## Components

### 1. ArgoCD Applications

The stack deploys two main ArgoCD applications:

- **Fluent Bit** - Log processor and forwarder
- **OpenTelemetry Collector** - Trace and metrics collector/exporter

### 2. Fluent Bit Configuration

Fluent Bit is deployed as a **Deployment** (single replica with HPA enabled) and configured to:

- **Receive logs** via HTTP on port 8888 from Akamai DataStream
- **Process logs** using custom Lua scripts to:
  - Filter logs based on allowed hostnames
  - Extract trace/span IDs from custom fields
  - Extract GRN (Global Reference Number) values
  - Normalize URL paths
  - Generate OTLP-compliant resource spans
  - Buffer and batch traces for efficient processing
- **Forward traces** to OpenTelemetry Collector via OTLP/HTTP on port 4318

#### Key Features:
- Horizontal Pod Autoscaling (1-3 replicas based on CPU usage)
- Custom Lua processing for Akamai log transformation
- Converts Akamai DataStream logs to OpenTelemetry traces
- Calculates RTT and origin turnaround times from breadcrumb data

### 3. OpenTelemetry Collector Configuration

The OpenTelemetry Collector is deployed as a **DaemonSet** and configured to:

- **Receive traces** via:
  - OTLP (gRPC: 4317, HTTP: 4318)
  - Jaeger (gRPC: 14250, Thrift HTTP: 14268, Thrift Compact: 6831)
- **Collect metrics** from:
  - Kubelet stats (enabled)
  - Prometheus scraping (Fluent Bit metrics)
- **Export to multiple backends**:
  - Honeycomb (traces)
  - Splunk/SignalFx (traces)
  - Grafana Cloud (traces/metrics)

#### Key Features:
- Memory limiter processor (80% limit, 25% spike)
- Batch processor for efficient exports
- Health check endpoint on port 13133
- Support for multiple observability backends

## Prerequisites

- Kubernetes cluster (tested on Linode Kubernetes Engine)
- ArgoCD installed and configured
- Traefik ingress controller (optional, for Fluent Bit ingress)
- Sealed Secrets controller (for secret management)

## Installation

### 1. Deploy ArgoCD Applications

Apply the ArgoCD application manifests:

```bash
kubectl apply -f argocd/applications/fluentbit-app.yaml
kubectl apply -f argocd/applications/otel-collector-app.yaml
```

### 2. Configure Secrets

The stack uses Sealed Secrets for secure credential management. You'll need to create sealed secrets for:

- **Honeycomb API Key**: `apps/otelcollector/secrets/otel-api-key-sealed.yaml`
- **Splunk Access Token**: `apps/otelcollector/secrets/signalfx-api-secret-sealed.yaml`
- **Grafana Cloud Auth**: `apps/otelcollector/secrets/grafana-cloud-basic-auth.yaml`
- **Fluent Bit Basic Auth**: `apps/fluentbit/ingress/sealedsecret.yaml`

Example of creating a sealed secret:

```bash
echo -n "your-api-key" | kubectl create secret generic otel-api-key \
  --dry-run=client \
  --from-file=API_KEY=/dev/stdin \
  -o yaml | kubeseal -o yaml > apps/otelcollector/secrets/otel-api-key-sealed.yaml
```

### 3. Configure Akamai DataStream

Configure your Akamai DataStream to send logs to Fluent Bit's HTTP endpoint. The custom fields should include:

- `traceId`: Trace identifier (hex format)
- `spanId`: Span identifier (hex format)
- `grn`: Global Reference Number

Example custom field format:
```
grn:abc123 traceId:1234567890abcdef spanId:fedcba0987654321
```

## Configuration

### Fluent Bit Values

Key configuration in `apps/fluentbit/values.yaml`:

```yaml
kind: Deployment
replicaCount: 1
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 75
extraPorts:
  - port: 8888
    containerPort: 8888
    protocol: TCP
    name: tcp
```

### OpenTelemetry Collector Values

Key configuration in `apps/otelcollector/values.yaml`:

```yaml
mode: "daemonset"
presets:
  kubeletMetrics:
    enabled: true
exporters:
  otlp/honeycomb:
    endpoint: api.honeycomb.io:443
  otlphttp/splunk:
    traces_endpoint: https://ingest.eu0.signalfx.com:443/v2/trace/otlp
  otlphttp/grafana:
    endpoint: https://otlp-gateway-prod-eu-west-0.grafana.net/otlp
```

## Monitoring

### Metrics
- Fluent Bit exposes metrics on port 2020 at `/api/v1/metrics/prometheus`
- OpenTelemetry Collector health check available at port 13133

### Logs
- Fluent Bit logs available via: `kubectl logs -n observability -l app.kubernetes.io/name=fluent-bit`
- OTEL Collector logs: `kubectl logs -n observability -l app.kubernetes.io/name=opentelemetry-collector`

## Troubleshooting

### Common Issues

1. **Fluent Bit not receiving data**
   - Check ingress/service configuration
   - Verify Akamai DataStream endpoint configuration
   - Check Fluent Bit logs for connection issues

2. **Traces not appearing in backend**
   - Verify API keys/tokens are correctly configured
   - Check OpenTelemetry Collector logs for export errors
   - Ensure network connectivity to observability backends

3. **High memory usage**
   - Adjust memory limits in resource configurations
   - Tune batch processor settings
   - Review Lua script buffer sizes

### Debug Commands

```bash
# Check pod status
kubectl get pods -n observability

# View Fluent Bit logs
kubectl logs -n observability deployment/fluent-bit -f

# View OTEL Collector logs (from any node)
kubectl logs -n observability -l app.kubernetes.io/name=opentelemetry-collector -f

# Check service endpoints
kubectl get svc -n observability

# Test Fluent Bit HTTP endpoint
kubectl port-forward -n observability deployment/fluent-bit 8888:8888
curl -X POST http://localhost:8888 -d '{"test": "data"}'
```

## Architecture Details

### Data Flow

1. **Akamai DataStream** sends HTTP logs to Fluent Bit (port 8888)
2. **Fluent Bit** processes logs through Lua pipeline:
   - Filters based on allowed hostnames
   - Extracts trace context (trace_id, span_id)
   - Transforms to OTLP format
   - Batches traces for efficiency
3. **OpenTelemetry Collector** receives traces via OTLP/HTTP
4. **Exporters** send data to configured backends:
   - Honeycomb for distributed tracing
   - Splunk/SignalFx for APM
   - Grafana Cloud for unified observability

### Lua Processing Pipeline

The Fluent Bit Lua scripts (`dswm.lua`) perform:

1. **Hostname Filtering**: Only processes logs from configured domains
2. **ID Extraction**: Parses trace and span IDs from custom fields
3. **Path Normalization**: Ensures consistent URL path formatting
4. **Span Generation**: Creates OTLP-compliant spans with:
   - Trace context (trace_id, span_id, parent_id)
   - HTTP attributes (method, path, status, etc.)
   - Akamai-specific attributes (GRN, cache status, RTT)
   - Timing events (TLS overhead, TTFB, download time)
5. **Batching**: Collects spans into resource spans for efficient transmission

## Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Create an issue in this repository
- Check the [documentation](https://github.com/jjgrinwis/observability-stack)

![Architecture Diagram](https://github.com/user-attachments/assets/ed9cab5f-b8e7-4c63-89ce-e4d8da7edf60)