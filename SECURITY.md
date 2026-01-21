# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| main    | :white_check_mark: |

## Reporting a Vulnerability

We take security vulnerabilities seriously. If you discover a security issue, please report it responsibly.

### How to Report

1. **Do NOT** create a public GitHub issue for security vulnerabilities
2. Email security concerns to the maintainers directly
3. Include as much detail as possible:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

### What to Expect

- **Acknowledgment**: We will acknowledge receipt within 48 hours
- **Assessment**: We will assess the vulnerability and determine its severity
- **Resolution**: We will work on a fix and coordinate disclosure
- **Credit**: We will credit reporters in the release notes (unless anonymity is requested)

## Security Considerations

### Development Environment

This repository is designed for **local development environments** using Kind clusters. It is NOT intended for production use without significant modifications.

### Default Configurations

The following default configurations prioritize developer experience over security:

| Component | Configuration | Note |
|-----------|---------------|------|
| Aspire Dashboard | Anonymous access | No authentication by default |
| OTEL Collector | Unsecured endpoints | No TLS or auth |
| Grafana | Default credentials | admin/prom-operator |
| Gateway | Self-signed certificates | dev-tls secret |

### Production Considerations

If adapting this configuration for production:

1. **Enable authentication** on all dashboards and APIs
2. **Use proper TLS certificates** from a trusted CA
3. **Implement network policies** to restrict pod-to-pod communication
4. **Enable RBAC** and follow least-privilege principles
5. **Secure secrets** using external secret management
6. **Enable audit logging** for compliance
7. **Configure resource limits** on all workloads

### Secrets Management

- TLS certificates are expected to be pre-created by the VDK CLI
- No secrets are stored in this repository
- Sensitive values should be managed externally

### Network Security

The default configuration:
- Exposes services only through the gateway on ports 30080/30443
- Uses ClusterIP services internally
- Does not configure NetworkPolicies (add as needed)

### Dependencies

We regularly update dependencies via automated workflows. Security patches are applied as they become available through upstream Helm charts.

## Security Best Practices

When using VDK Flux:

1. **Keep dependencies updated** - Review and merge automated update PRs promptly
2. **Use namespace isolation** - Deploy applications in separate namespaces
3. **Limit RBAC permissions** - Grant only necessary permissions to service accounts
4. **Monitor telemetry** - Use the observability stack to detect anomalies
5. **Review before deployment** - Always review manifests before applying

## Vulnerability Disclosure

We follow coordinated vulnerability disclosure:

1. Reporter notifies maintainers privately
2. We assess and develop a fix
3. We release the fix and notify users
4. Public disclosure after users have time to update

## Contact

For security-related inquiries, please reach out through:
- GitHub Security Advisories (for this repository)
- Direct communication with maintainers
