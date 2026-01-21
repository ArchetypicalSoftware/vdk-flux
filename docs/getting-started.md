# Getting Started with VDK Flux

This guide will help you set up a local Kubernetes development environment using the Vega Development Kit (VDK).

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [VDK CLI](https://github.com/ArchetypicalSoftware/VDK-Template) installed

## Quick Start

### 1. Create a Kind Cluster

Use the VDK CLI to create a new Kind cluster:

```bash
vdk cluster create
```

This will:
- Create a Kind cluster with the control plane node labeled `ingress-ready=true`
- Generate a wildcard TLS certificate stored in `vega-system/dev-tls`
- Bootstrap Flux and point it to this repository

### 2. Verify Installation

Check that all components are running:

```bash
# Check Flux status
kubectl get kustomizations -n flux-system

# Check all pods
kubectl get pods -A
```

### 3. Access the Aspire Dashboard

The Aspire Dashboard provides a unified view of traces, logs, and metrics:

```bash
# Open in browser
https://localhost:30443/observability
```

If using a custom hostname:
```bash
https://app.local.vega.dev:30443/observability
```

## Sending Telemetry

Configure your applications to send OpenTelemetry data to the collector.

### OTEL Collector Endpoints

| Protocol | Endpoint |
|----------|----------|
| gRPC | `otel-collector.opentel.svc.cluster.local:4317` |
| HTTP | `otel-collector.opentel.svc.cluster.local:4318` |

### Example: .NET Application

Add the OpenTelemetry packages:

```bash
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

Configure in your application:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://otel-collector.opentel.svc.cluster.local:4317");
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://otel-collector.opentel.svc.cluster.local:4317");
        }));
```

### Example: Environment Variables

For containerized applications, set these environment variables:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentel.svc.cluster.local:4317"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
```

## Accessing Services

### Grafana

Access Grafana for additional dashboards:

```bash
kubectl port-forward -n prometheus svc/kube-prometheus-stack-grafana 3000:80
```

Then open `http://localhost:3000` (default credentials: admin/prom-operator)

### Prometheus

Access Prometheus directly:

```bash
kubectl port-forward -n prometheus svc/kube-prometheus-stack-prometheus 9090:9090
```

Then open `http://localhost:9090`

## Deploying Applications

### Creating an HTTPRoute

To expose your application through the gateway, create an HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-namespace
spec:
  parentRefs:
    - name: vega-gateway
      namespace: vega-system
  hostnames:
    - "my-app.local.vega.dev"
  rules:
    - backendRefs:
        - name: my-app-service
          port: 80
```

### Cross-Namespace References

If your service is in a different namespace than the gateway, create a ReferenceGrant:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-vega-gateway
  namespace: my-namespace
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: vega-system
  to:
    - group: ""
      kind: Service
      name: my-app-service
```

## Troubleshooting

### Check Gateway Status

```bash
kubectl get gateway -n vega-system
kubectl describe gateway vega-gateway -n vega-system
```

### Check HTTPRoute Status

```bash
kubectl get httproute -A
kubectl describe httproute <name> -n <namespace>
```

### View Collector Logs

```bash
kubectl logs -n opentel -l app.kubernetes.io/component=opentelemetry-collector
```

### Check Flux Reconciliation

```bash
flux get kustomizations
flux logs --level=error
```

## Next Steps

- Read the [Architecture Overview](./architecture.md) to understand the system design
- Configure your applications to use OpenTelemetry
- Create HTTPRoutes to expose your services
- Explore the Aspire Dashboard to analyze traces and logs
