# Observability Guide

This guide covers the observability stack deployed by VDK Flux and how to effectively use it for local development.

## Overview

VDK Flux deploys a comprehensive observability stack that mirrors production environments:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Observability Stack                         │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │ Your Apps   │───►│    OTEL     │───►│  Aspire Dashboard   │ │
│  │             │    │  Collector  │    │  /observability     │ │
│  └─────────────┘    └──────┬──────┘    └─────────────────────┘ │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                          │
│                   │   Prometheus    │◄──── Metrics Storage     │
│                   └────────┬────────┘                          │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                          │
│                   │    Grafana      │◄──── Dashboards          │
│                   └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

## Aspire Dashboard

The [.NET Aspire Dashboard](https://aspire.dev/dashboard/overview/) provides a unified view of:

- **Traces**: Distributed tracing across services
- **Logs**: Structured log aggregation
- **Metrics**: Real-time metrics visualization

### Accessing the Dashboard

The dashboard is available at `/observability` on the main gateway:

```
https://localhost:30443/observability
```

### Features

| Feature | Description |
|---------|-------------|
| Traces | View distributed traces with span details, timing, and attributes |
| Structured Logs | Search and filter logs by service, level, or content |
| Metrics | Real-time charts for application and infrastructure metrics |
| Resources | Overview of all instrumented services |

## OpenTelemetry Collector

The OTEL Collector acts as a telemetry hub, receiving data from applications and routing it to various backends.

### Architecture

```yaml
Receivers:           Processors:         Exporters:
┌──────────┐        ┌────────────┐      ┌─────────────────┐
│ OTLP     │───────►│ Batch      │─────►│ Aspire Dashboard│
│ (gRPC)   │        │            │      │ (traces, logs,  │
│ :4317    │        │ Memory     │      │  metrics)       │
├──────────┤        │ Limiter    │      ├─────────────────┤
│ OTLP     │        │            │      │ Prometheus      │
│ (HTTP)   │        │            │      │ (metrics only)  │
│ :4318    │        └────────────┘      └─────────────────┘
└──────────┘
```

### Endpoints

| Protocol | Port | Endpoint |
|----------|------|----------|
| gRPC | 4317 | `otel-collector.opentel.svc.cluster.local:4317` |
| HTTP | 4318 | `otel-collector.opentel.svc.cluster.local:4318` |

### Configuration

The collector is configured via the `OpenTelemetryCollector` CR in `modules/aspire-dashboard/otel-collector.yaml`.

## Prometheus Stack

The kube-prometheus-stack provides:

- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Alertmanager**: Alert routing (enabled but not configured by default)

### Accessing Prometheus

```bash
kubectl port-forward -n prometheus svc/kube-prometheus-stack-prometheus 9090:9090
```

Open `http://localhost:9090`

### Accessing Grafana

```bash
kubectl port-forward -n prometheus svc/kube-prometheus-stack-grafana 3000:80
```

Open `http://localhost:3000`

Default credentials:
- Username: `admin`
- Password: `prom-operator`

### Pre-installed Dashboards

Grafana comes with dashboards for:
- Kubernetes cluster monitoring
- Node metrics
- Pod metrics
- Workload metrics

## Instrumenting Applications

### .NET Applications

```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("my-service"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MyApp")
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddMeter("MyApp")
        .AddOtlpExporter())
    .WithLogging(logging => logging
        .AddOtlpExporter());
```

```csharp
// appsettings.json
{
  "OTEL_EXPORTER_OTLP_ENDPOINT": "http://otel-collector.opentel.svc.cluster.local:4317"
}
```

### Node.js Applications

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'grpc://otel-collector.opentel.svc.cluster.local:4317',
  }),
  metricExporter: new OTLPMetricExporter({
    url: 'grpc://otel-collector.opentel.svc.cluster.local:4317',
  }),
});

sdk.start();
```

### Go Applications

```go
import (
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
)

func initTracer() func() {
    exporter, _ := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector.opentel.svc.cluster.local:4317"),
        otlptracegrpc.WithInsecure(),
    )
    // ... setup provider
}
```

### Environment Variables

For auto-instrumentation agents or SDKs that support environment configuration:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentel.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "grpc"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "deployment.environment=development"
```

## Best Practices

### Trace Context

Ensure trace context is propagated across service boundaries:

```yaml
env:
  - name: OTEL_PROPAGATORS
    value: "tracecontext,baggage"
```

### Resource Attributes

Add meaningful resource attributes for filtering:

```yaml
env:
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "service.namespace=my-team,deployment.environment=development"
```

### Sampling

For high-volume services, consider sampling:

```yaml
env:
  - name: OTEL_TRACES_SAMPLER
    value: "parentbased_traceidratio"
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"  # 10% sampling
```

## Troubleshooting

### No Traces Appearing

1. Verify collector is running:
   ```bash
   kubectl get pods -n opentel -l app.kubernetes.io/component=opentelemetry-collector
   ```

2. Check collector logs:
   ```bash
   kubectl logs -n opentel -l app.kubernetes.io/component=opentelemetry-collector
   ```

3. Verify network connectivity from your app to the collector

### Metrics Not in Prometheus

1. Check OTLP receiver is enabled in Prometheus:
   ```bash
   kubectl get prometheus -n prometheus -o yaml | grep otlp
   ```

2. Verify metrics are being exported:
   ```bash
   kubectl logs -n opentel -l app.kubernetes.io/component=opentelemetry-collector | grep metrics
   ```

### Dashboard Not Loading

1. Check dashboard pod status:
   ```bash
   kubectl get pods -n aspire-dashboard
   ```

2. Check HTTPRoute status:
   ```bash
   kubectl describe httproute aspire-dashboard -n aspire-dashboard
   ```
