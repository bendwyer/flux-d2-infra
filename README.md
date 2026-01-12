# Flux D2 Infrastructure Repository

This repository contains infrastructure components (controllers, operators, cluster-wide configs) deployed via Flux.

## Architecture

- **Per-component OCI artifacts**: Each component publishes to `ghcr.io/bendwyer/d2-infra/<component>`
- **Environment-based overlays**: `omni/` and `home/` directories for cluster-specific configs
- **Separation of controllers and configs**:
  - **Controllers**: Helm releases, operators, and their deployment configs
  - **Configs**: Custom Resources that depend on controllers (ClusterIssuers, Certificates, OnePasswordItems)

## Repository Structure

```
flux-d2-infra/
├── components/                         # Infrastructure components
│   └── {component}/
│       ├── controllers/               # Helm releases and operators
│       │   ├── base/                  # Shared configuration
│       │   ├── omni/                  # Omni cluster-specific values
│       │   └── home/                  # Home cluster-specific values
│       └── configs/                   # Custom Resources (depends on controllers)
│           ├── base/                  # ClusterIssuers, OnePasswordItems, etc.
│           ├── omni/                  # Omni cluster-specific configs
│           └── home/                  # Home cluster-specific configs
└── .github/workflows/
    └── publish-oci.yml                # Per-component OCI publishing
```

## Current Components

### 1Password Operator
- **Purpose**: Secret management via 1Password Connect
- **Bootstrap**: Credentials injected via Talos inline manifests
- **Dependencies**: Must deploy first (other components depend on it)
- **Configs**: None (configured via HelmRelease values)

### cert-manager
- **Purpose**: Automated certificate management
- **Dependencies**: Depends on 1Password Operator (for DNS-01 challenge secrets)
- **Configs**: ClusterIssuers, OnePasswordItem for Cloudflare API token

## CI/CD Workflow

### On Pull Request
- Validates OCI artifact builds for all components
- Does not publish artifacts

### On Merge to `main`
GitHub Actions:
1. Builds OCI artifact for each component in matrix
2. Only publishes if different from `:latest` (diff detection)
3. Signs with cosign
4. Pushes to `ghcr.io/bendwyer/d2-infra/<component>:latest`

## Secrets Management

All secrets are managed via **1Password Operator** using `OnePasswordItem` CRDs:

**Example:**
```yaml
apiVersion: onepassword.com/v1
kind: OnePasswordItem
metadata:
  name: cloudflare
  namespace: cert-manager
spec:
  itemPath: "vaults/<vault-id>/items/<item-id>"
```

The operator automatically creates a Kubernetes Secret with the same name.

## Adding Components

### 1. Create component structure
```bash
mkdir -p components/my-component/{controllers,configs}/{base,omni,home}
```

### 2. Add to workflow matrix
Edit `.github/workflows/publish-oci.yml`:
```yaml
matrix:
  component:
    - onepassword
    - cert-manager
    - my-component  # Add here
```

### 3. Add to fleet ResourceSet
Edit `flux-d2-fleet/tenants/infra.yaml`:
```yaml
spec:
  inputs:
    - component: "my-component"
      tag: "latest"
      environment: "omni"
```

## References

- [ControlPlane D2 Infrastructure Reference](https://github.com/controlplaneio-fluxcd/d2-infra)
- [1Password Operator](https://github.com/1Password/onepassword-operator)
