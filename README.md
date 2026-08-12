# mongo-gitops

Argo CD GitOps repository for the MongoDB-on-EKS platform. This repo contains no application code — it's a tree of Helm "wrapper" charts and `ApplicationSet` templates that Argo CD reads directly to install and configure every cluster-wide component (ingress, TLS, secrets, storage, MongoDB, and monitoring) across `dev`, `staging`, and `prod`.

It's the GitOps counterpart to the [infrastructure Terraform repo](../mongodb-eks-infra): the Terraform side provisions the VPC/EKS/IAM plumbing and bootstraps a root Argo CD `ApplicationSet` pointed at this repo; everything from that point on is reconciled from here.

## How it fits together

```
Terraform (bastion.tf)
   └─ installs Argo CD into EKS
   └─ creates a root ApplicationSet → points at bootstrap/ in this repo, per environment

bootstrap/  (Helm chart, values-<env>.yaml selects the environment)
   └─ templates/*.yaml — one ApplicationSet per platform component
        each ApplicationSet renders one Argo CD Application per environment,
        pointing at either an upstream Helm repo or a local chart in platform/

platform/<component>/  (local "wrapper" charts)
   ├── Chart.yaml       — upstream chart dependency (if any)
   ├── values.yaml       — shared values
   ├── values-<env>.yaml — per-environment overrides
   └── templates/         — raw manifests (Gateways, Secrets, IRSA bindings, HTTPRoutes)
```

Every component under `platform/` follows the same layout:

| Directory | Upstream chart | Local `templates/` content |
|---|---|---|
| `platform/mongodb/` | `bitnami/mongodb` v19.1.21 | `service-account.yaml`, `secret-store.yaml`, `external-secret.yaml`, `storageclass.yaml` |
| `platform/monitoring/` | `prometheus-community/kube-prometheus-stack` v61.3.1 | `grafana-routes.yaml` |
| `platform/gateways/` | none (local chart) | `GatewayClass.yaml`, `Gateway.yaml` |
| `platform/aws-load-balancer-controller/` | `eks-charts/aws-load-balancer-controller` v3.4.3 (pulled by the bootstrap `ApplicationSet`, not a local chart) | — |
| `platform/cert-manager/` | `jetstack/cert-manager` v1.21.1 (pulled by the bootstrap `ApplicationSet`) | — |
| `platform/external-secrets/` | `external-secrets/external-secrets` v2.8.0 (pulled by the bootstrap `ApplicationSet`) | — |

## Sync-wave ordering

Argo CD applies everything from `bootstrap/templates/` in this order, driven by `argocd.argoproj.io/sync-wave` annotations:

| Wave | Component | Why it goes there |
|---|---|---|
| `-2` | API Gateway CRDs (`gateway-api` upstream) | CRDs must exist before any `Gateway`/`GatewayClass`/`HTTPRoute` objects |
| `-1` | cert-manager, External Secrets Operator | Controllers + CRDs need to exist before anything that creates `Certificate`, `ClusterIssuer`, `SecretStore`, or `ExternalSecret` objects |
| `0` | AWS Load Balancer Controller, Gateway/GatewayClass, kube-prometheus-stack | Depends on the CRDs from wave `-1`; provisions the actual ALB and dashboards |
| `1` | Argo CD's own `HTTPRoute`, Grafana's `HTTPRoute` | Needs the `Gateway` from wave 0 |
| `2` | (within `platform/mongodb`) `ExternalSecret` | Needs the `SecretStore` (wave 1, defined inside the same chart) |
| `3` | MongoDB (Bitnami chart) | Needs the `StorageClass`, `ServiceAccount`, and synced `ExternalSecret` to exist first |

The comment block in `bootstrap/Chart.yaml` documents the same rule of thumb: **namespace → controllers/CRDs → dependent resources (issuers, secret stores) → applications that consume them.**

## What each component does

- **`bootstrap/`** — the root Helm chart Argo CD renders per environment. `values-<env>.yaml` sets `environment` and `domainName` (e.g. `argo-dev.aks.com`), which every `ApplicationSet` template consumes via `{{ .Values.environment }}` / `{{ .Values.domainName }}`.
- **API Gateway (`platform-api-gateway.yaml`)** — installs the upstream Gateway API CRDs (`gateway-api` v1.1.0) that everything downstream (`Gateway`, `GatewayClass`, `HTTPRoute`) depends on.
- **cert-manager** — installs CRDs (`Certificate`, `Issuer`) and issues/renews TLS certs; the AWS Load Balancer Controller is configured to delegate webhook cert management to it (`enableCertManager: true`) instead of self-signing.
- **External Secrets Operator** — installs the `SecretStore`/`ExternalSecret` CRDs used by `platform/mongodb` to pull credentials out of AWS SSM Parameter Store.
- **AWS Load Balancer Controller** — provisions an internet-facing ALB in `ip` target-type mode, with Gateway API support enabled (`enableGatewayAPI: true`). Its service account is bound via IRSA to the `<env>-aws-load-balancer-controller` role created in Terraform.
- **Gateways (`platform/gateways`)** — defines the `aws-alb` `GatewayClass` and the cluster's single `main-gateway` (HTTP :80), which every `HTTPRoute` in the cluster (Argo CD UI, Grafana) attaches to.
- **MongoDB (`platform/mongodb`)** — a wrapper chart around `bitnami/mongodb`:
  - `service-account.yaml` creates `eso-parameter-store-sa`, annotated with the `<env>-external-secrets-irsa` role ARN from Terraform.
  - `secret-store.yaml` defines a namespaced `SecretStore` that authenticates to AWS SSM Parameter Store using that service account.
  - `external-secret.yaml` pulls `/mongo-app/<env>/mongodb/admin` and `/mongo-app/<env>/mongodb/user` from SSM into a local `mongo-credentials` Kubernetes Secret, refreshed hourly.
  - `storageclass.yaml` registers `gp3` (via the EBS CSI driver) as the cluster's default `StorageClass`.
  - `values.yaml` points MongoDB's `auth.existingSecret` at `mongo-credentials` and defines the `appuser` / `appdb` application identity, with 2Gi of `gp3` persistent storage.
- **Monitoring (`platform/monitoring`)** — a wrapper chart around `kube-prometheus-stack`, with EKS-managed control-plane scraping disabled (no access to etcd/scheduler/controller-manager on managed nodes), resource requests/limits set for Prometheus, 10Gi of `gp3` storage, and a `grafana-routes.yaml` `HTTPRoute` exposing Grafana on the shared gateway.

## Environments

Every component that varies per environment does so through a `values-<env>.yaml` file layered on top of its base `values.yaml`:

| Environment | Argo CD hostname | Grafana hostname | LBC / ESO IAM roles |
|---|---|---|---|
| `dev` | `argo-dev.aks.com` | *(see note below)* | `dev-aws-load-balancer-controller`, `dev-external-secrets-irsa` |
| `staging` | `argo-staging.aks.com` | `dev-grafana.aks.com` (default, see note) | `staging-aws-load-balancer-controller`, `staging-external-secrets-irsa` |
| `prod` | `argo-prod.aks.com` | `dev-grafana.aks.com` (default, see note) | `prod-aws-load-balancer-controller`, `prod-external-secrets-irsa` |

IAM role ARNs are AWS account `043272859093`, region `eu-central-1`.

## How this repo gets updated

Per the (currently commented-out) Jenkins stage in the infra repo, the pipeline is expected to:
1. Read `external_secrets_irsa_arn`, `lbc_role_arn`, and `eks_cluster_name` from Terraform outputs after `apply`.
2. Clone this repo and `sed`-patch the relevant `role-arn` / `clusterName` / `domainName` fields in `platform/aws-load-balancer-controller/values.yaml` and `platform/monitoring/values.yaml`.
3. Commit and push, which Argo CD then picks up automatically (`syncPolicy.automated` is set on every `ApplicationSet`).

Until that stage is re-enabled, the `*-<env>.yaml` files above must be kept in sync with Terraform outputs by hand.

## Known issues to fix

- **`platform/mongodb/values-prod.yaml`** has a malformed line — `esenvironment: "staging"oRoleArn: "..."` — which should be split into `environment: "prod"` and `esoRoleArn: "..."`. As written, Helm will fail to parse this file.
- **`platform/monitoring/values-prod.yaml`** sets `environment: "dev"` instead of `"prod"`; likely a copy-paste bug that will misroute the prod Grafana `ApplicationSet` name.
- **`platform/monitoring/values.yaml`** hardcodes `domainName: dev-grafana.aks.com` as the base value with no `values-staging.yaml`/`values-prod.yaml` override for `domainName`, so staging and prod Grafana currently resolve to the dev hostname.
- Several `ApplicationSet` templates (`platform-aws-elb.yaml`, `platform-cert-manager.yaml`, `platform-external-secrets.yaml`) reference this repo by a different name — `https://github.com/dragan-actions-course/mongo-app-gitops-repo.git` — rather than `mongo-gitops`. Confirm which URL is canonical and update whichever side is stale.
- IAM role ARNs and the AWS account ID are hardcoded per environment rather than templated/generated — fine for a course/demo project, but worth turning into pipeline-injected values (as the commented-out Jenkins stage intends) before treating this as production-grade.

## Related repository

- **Infrastructure (Terraform, VPC/EKS/IAM, Jenkins pipeline):** the companion repo that provisions the AWS resources this GitOps repo assumes already exist.
