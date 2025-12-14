# GitOps with Flux and Kustomize

## Project Structure

This repository is structured to support GitOps deployments using Flux and Kustomize.

```
├── base/                   # Shared manifests (Deployment, Service, Ingress, ConfigMap)
├── overlays/
│   ├── development/        # Dev environment specific configuration (1 replica, Dragonfly DB instance)
│   └── production/         # Prod environment specific configuration (3 replicas, Dragonfly Cluster, HPA)
└── clusters/
    └── my-cluster/         # Flux configurations (GitRepository, Kustomizations)
```

## Environments

### Development
- **Namespace:** `development`
- **Replicas:** 1
- **Database:** Dragonfly (Single Instance)
- **Install:** Automatically managed by Flux (`apps-dev` Kustomization).

### Production
- **Namespace:** `production`
- **Replicas:** 3
- **Database:** Dragonfly (Replica Set, 2 replicas)
- **Scaling:** HPA enabled (scale 3-10 replicas based on CPU)
- **Install:** Automatically managed by Flux (`apps-prod` Kustomization).

## Flux Setup

Flux monitors the `clusters/my-cluster` directory. It automatically applies:

1.  `app-dev.yaml` -> Syncs `overlays/development` to the `development` namespace.
2.  `app-prod.yaml` -> Syncs `overlays/production` to the `production` namespace.

### Prerequisites
- Kubernetes Cluster
- Flux installed and bootstrapped
- Dragonfly Operator installed (manually or via Flux)

### Verification

Check the status of Flux Kustomizations:

```bash
flux get kustomizations
```

Check the application pods:

```bash
kubectl get pods -n development
kubectl get pods -n production
```
