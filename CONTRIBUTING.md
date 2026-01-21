# Contributing to VDK Flux

Thank you for your interest in contributing to the Vega Development Kit (VDK) Flux repository! This document provides guidelines for contributing.

## Code of Conduct

By participating in this project, you agree to maintain a respectful and inclusive environment for everyone.

## How to Contribute

### Reporting Issues

Before creating an issue, please:

1. Search existing issues to avoid duplicates
2. Use the appropriate issue template
3. Provide as much detail as possible

### Submitting Changes

1. **Fork the repository** and create your branch from `main`
2. **Make your changes** following the guidelines below
3. **Test your changes** by deploying to a local Kind cluster
4. **Submit a pull request** with a clear description

### Branch Naming

Use descriptive branch names:
- `feature/add-new-component`
- `fix/gateway-routing-issue`
- `docs/update-architecture`

### Commit Messages

Follow conventional commit format:

```
type(scope): description

[optional body]

[optional footer]
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `chore`: Maintenance tasks
- `refactor`: Code refactoring

Examples:
```
feat(gateway): add support for gRPC routes
fix(otel): correct collector endpoint configuration
docs(readme): update installation instructions
```

## Development Guidelines

### Project Structure

```
vdk-flux/
├── base/              # Flux Kustomization definitions
├── clusters/          # Cluster-specific configurations
├── docs/              # Documentation
└── modules/           # Component manifests
    ├── aspire-dashboard/
    ├── cert-manager/
    ├── flux/
    ├── gateway-api/
    ├── kgateway/
    ├── opentel/
    ├── prometheus/
    ├── vega-gateway/
    └── vega-system/
```

### Adding a New Module

1. Create a new directory under `modules/`
2. Add the necessary Kubernetes manifests
3. Create a `kustomization.yaml` file
4. Add a Flux Kustomization in `base/`
5. Update dependencies if needed
6. Update documentation

### Modifying Helm Releases

When updating Helm chart versions:

1. Test the new version locally first
2. Update the version in the HelmRelease manifest
3. Document any breaking changes or new configurations
4. Consider backwards compatibility

### Testing Changes

Before submitting a PR:

1. Create a test Kind cluster:
   ```bash
   vdk cluster create --name test
   ```

2. Apply your changes:
   ```bash
   kubectl apply -k ./base
   ```

3. Verify all resources are healthy:
   ```bash
   flux get kustomizations
   kubectl get pods -A
   ```

4. Test affected functionality

### Documentation

- Update relevant documentation for any user-facing changes
- Keep the architecture diagram current
- Add examples for new features

## Review Process

1. All PRs require at least one approval
2. CI checks must pass
3. Documentation must be updated for user-facing changes
4. Breaking changes require discussion in an issue first

## Automated Updates

This repository uses automated workflows:

- **flux-updater**: Weekly updates to Flux manifests and Helm charts
- PRs are automatically created for review

When reviewing automated PRs:
1. Check the changelog for breaking changes
2. Verify compatibility with existing configurations
3. Test in a development cluster if significant changes

## Questions?

- Open a [Discussion](https://github.com/ArchetypicalSoftware/vdk-flux/discussions) for questions
- Create an [Issue](https://github.com/ArchetypicalSoftware/vdk-flux/issues) for bugs or feature requests

## License

By contributing, you agree that your contributions will be licensed under the same license as the project.
