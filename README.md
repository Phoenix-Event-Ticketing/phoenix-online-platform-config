# Phoenix Platform Config

This repository stores Kubernetes deployment manifests for the Phoenix platform. It is a GitOps-focused configuration repository that is separate from the individual microservice application repositories.

## Environment Model

There is a single Kubernetes cluster. Environment separation is handled with two namespaces in that same cluster:

- `phoenix-dev`
- `phoenix-prod`

This repository does not use separate Git branches for environments. Instead, it uses folder-based overlays with Kustomize.

## Repository Layout

```text
platform-config/
├── apps/
│   ├── inventory-service/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       └── prod/
│   ├── user-service/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       └── prod/
│   ├── booking-service/
│   ├── payment-service/
│   └── notification-service/
├── environments/
│   ├── shared/          # ClusterSecretStore + cert-manager ClusterIssuers (LE + Cloudflare DNS)
│   ├── dev/
│   └── prod/
└── argocd/
    ├── dev-app.yaml
    └── prod-app.yaml
```

## Kustomize Structure

Each service is expected to follow a `base` + `overlays` model:

- `apps/<service>/base` contains reusable manifests such as `Deployment`, `Service`, and `ConfigMap`
- `apps/<service>/overlays/dev` applies development-specific settings such as namespace, image tag, replica count, and resource sizing
- `apps/<service>/overlays/prod` applies production-specific settings

The `environments/dev` and `environments/prod` folders aggregate:

- the target namespace manifest
- shared cluster resources from `environments/shared` (External Secrets store, Let’s Encrypt issuers)
- **Gateway API** (`Gateway`, `HTTPRoute`) and **cert-manager** `Certificate` (Let’s Encrypt via **DNS-01** with **Cloudflare**)
- `ExternalSecret` for the Cloudflare DNS API token (synced from GCP Secret Manager)
- `ResourceQuota` and `LimitRange`
- the service overlays for that environment

This starter currently includes working manifests for:

- `user-service`
- `inventory-service`
- `booking-service`
- `payment-service`

## TLS and ingress (Gateway API + Cloudflare DNS)

Public clients hit **Cloudflare** first; origin TLS on the GKE load balancer is provided by **Let’s Encrypt** certificates obtained with **cert-manager** using **ACME DNS-01** against **Cloudflare** (Cloudflare API token with permission to create `_acme-challenge` TXT records in your zone).

Requirements on the cluster (install outside this repo or via a platform Helm release):

- **GKE Gateway** (or compatible Gateway API implementation) and a valid `GatewayClass` (update `gatewayClassName` in [environments/dev/gateway.yaml](environments/dev/gateway.yaml) / [environments/prod/gateway.yaml](environments/prod/gateway.yaml) to match `kubectl get gatewayclass`)
- **cert-manager** with Gateway API support (Certificate `spec.targetRef` → `Gateway`)
- **External Secrets Operator** and GCP **Workload Identity** (or equivalent auth) for [ClusterSecretStore](environments/shared/cluster-secret-store.yaml)

Operational checklist:

1. Create a **Cloudflare API token** with DNS edit rights for the zone that contains `api-dev.example.com` / `api.example.com`.
2. Store that token in **GCP Secret Manager** and set `remoteRef.key` in [environments/dev/cloudflare-api-token-externalsecret.yaml](environments/dev/cloudflare-api-token-externalsecret.yaml) (and prod) to the GSM secret id.
3. Point DNS for those hostnames at Cloudflare and route to the GKE Gateway load balancer (orange-cloud or grey-cloud per your design; DNS-01 only needs Cloudflare to serve the zone).
4. Replace `REPLACE_ACME_EMAIL` in [environments/shared/cluster-issuer-letsencrypt-staging.yaml](environments/shared/cluster-issuer-letsencrypt-staging.yaml) and [cluster-issuer-letsencrypt-prod.yaml](environments/shared/cluster-issuer-letsencrypt-prod.yaml).
5. Dev uses **Let’s Encrypt staging** (`letsencrypt-staging`); prod uses **production** (`letsencrypt-prod`).

## Argo CD Flow

Argo CD watches this repository and deploys manifests from:

- `environments/dev` into `phoenix-dev`
- `environments/prod` into `phoenix-prod`

The `argocd/` folder contains one `Application` per environment. Each application points to the appropriate environment folder and enables automated sync with pruning and self-healing.

Each environment build includes `environments/shared`, so **ClusterSecretStore** and **ClusterIssuer** resources are applied from both apps (idempotent).

## Image Update and Promotion Flow

The intended workflow is:

1. Developers push application code to individual service repositories.
2. CI builds container images and pushes them to the registry.
3. CI updates image tags in this config repository.
4. Argo CD detects the change and syncs it to the cluster.

In this starter repo, image tags are set in the overlay `kustomization.yaml` files, which makes them easy for CI to update without modifying the base manifests.

## Secrets

Secret values are not committed to Git.

- **Application secrets** (`DATABASE_URL`, `JWT_SECRET`, `MONGODB_URI`, booking/payment `MONGO_URI` + `JWT_SECRET`, Mongo root passwords for in-cluster DBs): [ExternalSecret](https://external-secrets.io/) manifests in each service overlay sync from **GCP Secret Manager** via `ClusterSecretStore` `phoenix-gcp`. Replace `REPLACE_GSM_*` placeholders in those files with your GSM secret names (use separate secrets per environment in practice). For booking, set `MONGODB_URI` to `booking-service-mongodb`; for payment, set `MONGO_URI` to `payment-service-mongodb` (e.g. `mongodb://root:...@payment-service-mongodb:27017/phoenix_payment_service?authSource=admin`) matching `REPLACE_GSM_PAYMENT_MONGODB_ROOT_PASSWORD`.
- **Cloudflare DNS API token**: synced by [environments/dev/cloudflare-api-token-externalsecret.yaml](environments/dev/cloudflare-api-token-externalsecret.yaml) (and prod) for cert-manager DNS-01.

Replace placeholders in [environments/shared/cluster-secret-store.yaml](environments/shared/cluster-secret-store.yaml) for GCP project, region, and cluster name.

## Local Validation

Render an environment locally with:

```bash
kustomize build environments/dev
kustomize build environments/prod
```

If you use `kubectl`, the equivalent commands are:

```bash
kubectl kustomize environments/dev
kubectl kustomize environments/prod
```

You need cluster CRDs for Gateway API, cert-manager, and External Secrets if you validate with `kubectl apply --dry-run=server`.
