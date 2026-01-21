# VDK Flux

GitOps repository for the [Vega Development Kit (VDK)](https://github.com/ArchetypicalSoftware/VDK-Template) - a CLI tool that creates opinionated Kind clusters for local Kubernetes development.

## Overview

VDK Flux provides a complete local development environment with:

- **Gateway API with KGateway**: Modern ingress using the Kubernetes Gateway API standard
- **Aspire Dashboard**: Unified observability for traces, logs, and metrics
- **OpenTelemetry**: Industry-standard telemetry collection and routing
- **Prometheus + Grafana**: Metrics storage and visualization
- **Cert Manager**: Automated TLS certificate management

## Quick Start

1. Install the [VDK CLI](https://github.com/ArchetypicalSoftware/VDK-Template)

2. Create a cluster:
   ```bash
   vdk cluster create
   ```

3. Access the Aspire Dashboard:
   ```
   https://localhost:30443/observability
   ```

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        Kind Cluster                            │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            Control Plane (ingress-ready=true)            │ │
│  │                                                          │ │
│  │   Port 30443 ────► Vega Gateway ────► Your Apps          │
│  │        │                  │                               │
│  │        │                  └────► /observability           │
│  │        │                              │                   │
│  │        │                              ▼                   │
│  │        │                    Aspire Dashboard              │
│  │        │                              ▲                   │
│  │        │                              │                   │
│  │   OTEL Collector ◄──── Your Apps ─────┘                   │
│  │        │                                                  │
│  │        └────► Prometheus ────► Grafana                    │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Version | Purpose |
|-----------|---------|---------|
| [Flux](https://fluxcd.io/) | v2.6.2 | GitOps automation |
| [Gateway API](https://gateway-api.sigs.k8s.io/) | v1.4.0 | Kubernetes standard for ingress |
| [KGateway](https://kgateway.dev/) | v2.1.0 | Envoy-based Gateway implementation |
| [Aspire Dashboard](https://aspire.dev/) | 9.1 | Unified observability UI |
| [OpenTelemetry](https://opentelemetry.io/) | v0.102.0 | Telemetry collection |
| [Prometheus](https://prometheus.io/) | v81.2.0 | Metrics storage |
| [Cert Manager](https://cert-manager.io/) | v1.19.2 | TLS management |

## Documentation

- [Getting Started](docs/getting-started.md) - Set up your development environment
- [Architecture](docs/architecture.md) - Understand the system design
- [Observability Guide](docs/observability.md) - Use the telemetry stack

## Sending Telemetry

Configure your applications to send OpenTelemetry data:

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentel.svc.cluster.local:4317"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
```

## Exposing Services

Create an HTTPRoute to expose your service:

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
  rules:
    - backendRefs:
        - name: my-service
          port: 80
```

## Repository Structure

```
vdk-flux/
├── base/                  # Flux Kustomization definitions
├── clusters/default/      # Cluster configuration
├── docs/                  # Documentation
├── modules/               # Component manifests
│   ├── aspire-dashboard/  # Aspire Dashboard + OTEL Collector
│   ├── cert-manager/      # Certificate management
│   ├── flux/              # Flux controllers
│   ├── gateway-api/       # Gateway API CRDs
│   ├── kgateway/          # KGateway Helm releases
│   ├── opentel/           # OpenTelemetry Operator
│   ├── prometheus/        # Prometheus + Grafana
│   ├── vega-gateway/      # Main gateway configuration
│   └── vega-system/       # VDK Operator
├── CONTRIBUTING.md        # Contribution guidelines
├── SECURITY.md            # Security policy
└── README.md
```

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Security

For security concerns, please see [SECURITY.md](SECURITY.md).

## License

This project is part of the Vega Development Kit ecosystem by [Archetypical Software](https://github.com/ArchetypicalSoftware).
