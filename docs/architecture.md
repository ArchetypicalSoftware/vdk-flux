# VDK Flux Architecture

This document describes the architecture and components deployed by the VDK Flux GitOps repository.

## Overview

VDK Flux provides a complete Kubernetes development environment using GitOps principles. It deploys infrastructure components to Kind clusters created by the [VDK Template](https://github.com/ArchetypicalSoftware/VDK-Template) CLI.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Kind Cluster                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                     Control Plane Node (ingress-ready=true)            │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │ │
│  │  │ Port 30080   │  │ Port 30443   │  │ Port 4317    │                  │ │
│  │  │ (HTTP)       │  │ (HTTPS)      │  │ (OTLP)       │                  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │ │
│  │         │                 │                 │                          │ │
│  │         └────────┬────────┘                 │                          │ │
│  │                  ▼                          ▼                          │ │
│  │         ┌────────────────┐         ┌────────────────┐                  │ │
│  │         │  Vega Gateway  │         │ OTEL Collector │                  │ │
│  │         │  (KGateway)    │         │                │                  │ │
│  │         └────────┬───────┘         └───────┬────────┘                  │ │
│  │                  │                         │                           │ │
│  │    ┌─────────────┼─────────────┐           │                           │ │
│  │    ▼             ▼             ▼           ▼                           │ │
│  │ ┌──────┐  ┌───────────┐  ┌──────────────────────┐                      │ │
│  │ │ Apps │  │ /observ.  │  │  Aspire Dashboard    │                      │ │
│  │ │      │  │ HTTPRoute │──│  (Traces/Logs/Metrics)│                      │ │
│  │ └──────┘  └───────────┘  └──────────────────────┘                      │ │
│  │                                    │                                   │ │
│  │                          ┌─────────┴─────────┐                         │ │
│  │                          ▼                   ▼                         │ │
│  │                  ┌──────────────┐    ┌──────────────┐                  │ │
│  │                  │  Prometheus  │    │   Grafana    │                  │ │
│  │                  └──────────────┘    └──────────────┘                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Hierarchy

The components are deployed in a specific order using Flux Kustomizations with dependencies:

```
0-flux (Flux Controllers)
    │
    └── 1-base-modules
            │
            ├── cert-manager-flux ──────────────────┐
            │                                       │
            ├── gateway-api-flux ───────────────────┤
            │                                       │
            ├── kgateway-flux (depends on gateway-api, cert-manager)
            │       │
            │       └── vega-gateway-flux (depends on kgateway, vega-system)
            │               │
            │               └── aspire-dashboard-flux (depends on opentel, vega-gateway)
            │
            ├── opentel-flux (depends on cert-manager)
            │
            ├── prometheus-flux
            │
            ├── vega-namespace
            │
            └── vega-system-flux
```

## Components

### Gateway Layer

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| Gateway API CRDs | gateway-system | Kubernetes Gateway API standard resources |
| KGateway | kgateway-system | Envoy-based Gateway API implementation |
| Vega Gateway | vega-system | Main gateway for routing traffic |

### Observability Stack

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| OpenTelemetry Operator | opentel | Manages OTEL collectors |
| VDK Collector | opentel | Routes telemetry to backends |
| Aspire Dashboard | aspire-dashboard | Unified traces, logs, and metrics UI |
| Prometheus | prometheus | Metrics collection and storage |
| Grafana | prometheus | Dashboards and visualization |

### Core Infrastructure

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| Flux Controllers | flux-system | GitOps automation |
| Cert Manager | cert-manager | TLS certificate management |
| Vega Operator | vega-system | VDK custom resources |

## Network Configuration

### Gateway Routing

The Vega Gateway is configured to route traffic from the host ports to internal services:

- **Port 30080**: HTTP traffic (redirects to HTTPS)
- **Port 30443**: HTTPS traffic with TLS termination

TLS is terminated using the `dev-tls` certificate from the `vega-system` namespace.

### HTTPRoutes

| Path | Target Service | Namespace |
|------|----------------|-----------|
| /observability | aspire-dashboard | aspire-dashboard |
| /* | Application services | Various |

### OTEL Collection

Telemetry flows through the following path:

```
Applications ──► OTEL Collector ──┬──► Aspire Dashboard (traces, logs, metrics)
                                  │
                                  └──► Prometheus (metrics via OTLP HTTP)
```

Applications should send telemetry to:
- **gRPC**: `otel-collector.opentel.svc.cluster.local:4317`
- **HTTP**: `otel-collector.opentel.svc.cluster.local:4318`

## Directory Structure

```
vdk-flux/
├── base/                      # Flux Kustomization definitions
│   ├── aspire-dashboard-flux.yaml
│   ├── cert-manager-flux.yaml
│   ├── gateway-api-flux.yaml
│   ├── kgateway-flux.yaml
│   ├── opentel-flux.yaml
│   ├── prometheus-flux.yaml
│   ├── vega-gateway-flux.yaml
│   ├── vega-namespace.yaml
│   └── vega-system-flux.yaml
├── clusters/
│   └── default/               # Default cluster configuration
│       ├── base-modules.yaml
│       └── flux-components.yaml
├── docs/                      # Documentation
├── modules/                   # Component manifests
│   ├── aspire-dashboard/
│   ├── cert-manager/
│   ├── flux/
│   ├── gateway-api/
│   ├── kgateway/
│   ├── opentel/
│   ├── prometheus/
│   ├── vega-gateway/
│   └── vega-system/
└── README.md
```

## TLS Configuration

The cluster expects a TLS secret named `dev-tls` in the `vega-system` namespace. This is created by the VDK CLI during cluster provisioning and contains a wildcard certificate for local development.

## Version Information

| Component | Chart/Version |
|-----------|---------------|
| Flux | v2.6.2 |
| Gateway API | v1.4.0 |
| KGateway | v2.1.0 |
| Cert Manager | v1.19.2 |
| Prometheus Stack | v81.2.0 |
| OpenTelemetry Operator | v0.102.0 |
| Aspire Dashboard | 9.1 |
