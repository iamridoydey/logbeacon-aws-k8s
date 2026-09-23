# LogBeacon — Kubernetes and GitOps

This repository defines how LogBeacon runs on Amazon EKS. Argo CD reads the repository and reconciles platform services, application configuration, networking, TLS certificates, and frontend/backend workloads. Helm installs the platform components, Kustomize assembles application manifests, and Argo Rollouts controls canary traffic through Traefik and the Kubernetes Gateway API.

Argo CD runs in a **management cluster** and deploys LogBeacon into a separate **workload cluster**. AWS resources and application container builds are maintained in the companion repositories.

## The LogBeacon repositories

| Repository | Responsibility |
| --- | --- |
| [logbeacon-app](https://github.com/iamridoydey/logbeacon-app) | Application source, migrations, tests, and frontend/backend images published to ECR. |
| [logbeacon-aws-infra](https://github.com/iamridoydey/logbeacon-aws-infra) | AWS VPC, private EKS clusters, IAM/Pod Identity, ECR, secrets, and admin EC2. |
| **[logbeacon-aws-k8s](https://github.com/iamridoydey/logbeacon-aws-k8s)** | Desired Kubernetes state and GitOps deployment. |

For local application development without AWS, use the application README. This repository expects the AWS infrastructure to exist first.

## Contents

- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [CI Pipeline Architecture](#ci-pipeline-architecture)
- [Prerequisites](#prerequisites)
- [Configure your deployment](#configure-your-deployment)
- [Required first-deployment corrections](#required-first-deployment-corrections)
- [Bootstrap and verify](#bootstrap-and-verify)
- [Access the services](#access-the-services)
- [Release and operate the application](#release-and-operate-the-application)
- [CI and local validation](#ci-and-local-validation)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

## Architecture

```mermaid
flowchart TD
    Git[Git repository] --> Argo[Argo CD in management EKS]
    ECR[Amazon ECR] --> Updater[Image Updater]
    Updater --> PR[Image update pull request]
    PR --> Git
    Argo --> Platform[Workload platform services]
    Argo --> Rollouts[Frontend and backend Rollouts]
    Browser[Browser or API client] --> Gateway[Traefik Gateway API]
    Gateway --> Rollouts
    Rollouts --> DB[PostgreSQL]
    Secrets[AWS Secrets Manager] --> ESO[External Secrets]
    ESO --> Platform
    ESO --> Rollouts
```

### Cluster responsibilities

| Management cluster | Workload cluster |
| --- | --- |
| Argo CD and its `Application` resources | Frontend/backend canary Rollouts |
| Argo CD Image Updater | Argo Rollouts controller and dashboard |
| External Secrets Operator | External Secrets Operator |
| cert-manager and Cloudflare DNS-01 | cert-manager and Cloudflare DNS-01 |
| Traefik and Gateway API CRDs | Traefik and Gateway API CRDs |
| ExternalDNS for the Argo CD hostname | ExternalDNS for workload hostnames |
| Workload-cluster registration secret | PostgreSQL, Redis, SonarQube, and EBS-backed PVCs |

**All Argo CD `Application` objects belong in the management cluster's `argocd` namespace**, including the objects under `argocd/workload/`. Their `spec.destination` tells Argo CD where the managed resources should run.

The Argo CD destination name `workload-cluster` is an internal registration name. It is distinct from the AWS EKS cluster name `logbeacon-workload-cluster`. External Secrets creates the registration from `workload-eks-cred` in AWS Secrets Manager. Argo CD authenticates using EKS Pod Identity and an assumed workload role provisioned by Terraform.

The root and child applications use automated sync, pruning, and self-healing. A committed change to a watched branch can therefore affect the cluster automatically.

## Repository layout

| Path | Purpose |
| --- | --- |
| `argocd/argocd-values.yaml` | Values for the initial Argo CD Helm installation. |
| `argocd/management-root.yaml` | Initial root application; points at `argocd/management`. |
| `argocd/management/` | Management child applications and the workload root. |
| `argocd/workload/` | Child applications targeting the workload cluster. |
| `infra/` | Helm values for platform charts. These are not standalone Kubernetes manifests. |
| `infra-resources/` | Namespaces, service accounts, ExternalSecrets, certificates, storage, jobs, and ImageUpdater configuration. |
| `apps/management/` | Argo CD Gateway and HTTPRoute. |
| `apps/workload/logbeacon/` | ConfigMaps, frontend/backend Rollouts, Services, Gateway, HTTPRoutes, and image tags. |
| `apps/workload/sonarqube/` | SonarQube HTTPRoute. |
| `.github/workflows/k8s-ci.yaml` | PR validation. |

Application definitions under `argocd/` point either to a Helm chart plus values or to a directory of manifests. The application image mappings are in `apps/workload/logbeacon/kustomization.yaml`.

## CI Pipeline Architecture
![Ci Pipeline on pull request](diagrams/ci/logbeacon-k8s-ci-pr.png)

## Prerequisites

1. Provision both EKS clusters, the admin host, IAM roles, Pod Identity associations, ECR repositories, and Secrets Manager entries using the infrastructure repository.
2. Ensure the admin EC2 instance is reachable through SSM and has AWS CLI v2, kubectl, Helm, and Git installed. Both EKS APIs are private.
3. Have working Cloudflare credentials and control of the DNS zone you configure below.
4. Publish at least one frontend and backend image to ECR. The application repository's main release workflow can be run after ECR and its CI role exist; it builds/scans/pushes images without depending on the SonarQube PR job.
5. Set the manifests to tags that actually exist in your registries. A new ECR repository will not contain the checked-in `099e213` tag automatically.
6. Use GitHub credentials that permit the image updater's repository writes and pull-request creation when enabling that automation.

Kubernetes is configured for version `1.33`. CI uses Kustomize `5.8.1` and kubeconform `0.8.0`. The kubectl Argo Rollouts plugin is optional for deployment inspection and promotion commands shown below.

## Configure your deployment

Clone the repository locally to review and edit it before bootstrap:

```bash
git clone https://github.com/iamridoydey/logbeacon-aws-k8s.git
cd logbeacon-aws-k8s
```

### Account, Git, region, and DNS settings

| Setting | Files to update |
| --- | --- |
| GitHub owner/repository | `repoURL` values throughout `argocd/`; Git writeback repository in `infra-resources/image-updater/image-updater.yaml`. |
| ECR account/region and image tags | `apps/workload/logbeacon/kustomization.yaml`, `infra/image-updater-values.yaml`, `infra-resources/image-updater/image-updater.yaml`, and the migration Job. |
| AWS region | `infra-resources/cluster-secret-stores/external-secrets.yaml`, the image-updater auth script, and `infra-resources/external-secrets/workload/sonarqube-job.yaml`. |
| DNS zone filter | Both `infra/*-external-dns-values.yaml` files. |
| Public hostnames | Gateways/HTTPRoutes under `apps/`, certificate files under `infra-resources/cert-manager/`, and `infra-resources/argo-rollouts/httproute.yaml`. |
| Backend URL and CORS | `apps/workload/logbeacon/frontend-configmap.yaml` and `backend-configmap.yaml`. |
| ACME contact email | Both `infra-resources/cert-manager/*/cluster-issuer.yaml` files. |
| SonarQube public URL | `infra-resources/external-secrets/workload/sonarqube-job.yaml`. |

Use a search to find project-specific references before committing:

```bash
rg -n '869719105525|iamridoydey|us-east-1|099e213' argocd infra infra-resources apps
```

If changing namespaces or service-account names, update the matching Terraform Pod Identity associations and trust conditions too. Keep separate ExternalDNS `txtOwnerId` values for the two clusters.

### Application and database configuration

| File | Settings |
| --- | --- |
| `apps/workload/logbeacon/backend-configmap.yaml` | Groq model, retention, input limit, estimated token rate, SMTP host/port, allowed origins. |
| `apps/workload/logbeacon/frontend-configmap.yaml` | `FLASK_API_URL` used by Express. |
| `infra/postgres-values.yaml` | PostgreSQL user `logbeacon`, database `logbeacon_db`, 8Gi gp3 storage, existing secret. |
| `infra/redis-values.yaml` | Standalone Redis, authentication disabled, 4Gi gp3 storage. |
| `infra/sonarqube-values.yaml` | Community mode, resources, 10Gi persistence, monitoring passcode secret. |

Replace `smtp.example.com` before enabling email. Rename `CHAT_RETENTION_DAYS` in the backend ConfigMap to **`LOG_RETENTION_DAYS`** to match the application code. The current `PRICE_PER_MILLION_TOKENS` is a configured estimate, not verified provider pricing.

The PostgreSQL chart hard-codes the user/database independently of the secret values. Keep them consistent with `POSTGRES_USER` and `POSTGRES_DB` in AWS. Credentials interpolated into `DATABASE_URL` must be URL-safe or correctly encoded by a revised connection-string implementation.

The backend builds its database URL using `workload-postgresql:5432` and uses `redis://workload-redis-master:6379/0`. The frontend currently calls the public API hostname; changing it to the stable ClusterIP service would bypass the backend HTTPRoute's canary traffic split.

### Secrets

External Secrets Operator reads AWS Secrets Manager using the identity of its `external-secrets` service account. No static AWS access keys are embedded in these manifests.

| Source in AWS | Kubernetes target / purpose |
| --- | --- |
| `logbeacon/app`, `logbeacon/database`, `logbeacon/smtp` | Combined into `logbeacon/logbeacon-app-secret`. |
| `logbeacon/cloudflare` | Cloudflare token secrets in the cert-manager and external-dns namespaces of both clusters. |
| `github-secret` | `argocd/github-secret`, with `username` and `password` keys for Git writeback. |
| `workload-eks-cred` | `argocd/workload-cluster-secret`, labeled as an Argo CD cluster registration. |
| `logbeacon/sonarqube-passcode` | SonarQube monitoring passcode secret. |
| `sonarqube-admin-password` | Read directly from AWS by the SonarQube bootstrap Job. |
| `sonarqube-ci-cred` | Written by that Job; read by application CI. |

Several ExternalSecrets refresh hourly. Updating a Kubernetes Secret used through `envFrom` does not restart an existing application container; plan a controlled pod rollout after changing its credentials/configuration.

## Required first-deployment corrections

This snapshot contains unfinished migration and bootstrap wiring. Resolve the following before treating a fresh installation as ready for traffic.

### Database migration application

In `argocd/workload/resources/db-migration.yaml`, change `spec.source.path` to the actual **directory**:

```yaml
path: infra-resources/db-migration
```

The checked-in path `infra-resources/external-secrets/db-migration/db-migration.yaml` does not exist and points to a file rather than a source directory.

In `infra-resources/db-migration/db-migration.yaml`:

1. Correct the Secret reference `ogbeacon-app-secret` to `logbeacon-app-secret`.
2. Replace `image: logbeacon/backend` with the fully qualified ECR backend image and the release tag you are deploying. This standalone Job is outside the application's Kustomize image transformation.
3. Add the same database environment construction used by the backend Rollout. Neither the ConfigMap nor the combined Secret supplies `DATABASE_URL` directly. The Job container needs:

```yaml
env:
  - name: POSTGRES_USER
    valueFrom:
      secretKeyRef:
        name: logbeacon-app-secret
        key: POSTGRES_USER
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: logbeacon-app-secret
        key: POSTGRES_PASSWORD
  - name: POSTGRES_DB
    valueFrom:
      secretKeyRef:
        name: logbeacon-app-secret
        key: POSTGRES_DB
  - name: DATABASE_URL
    value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@workload-postgresql:5432/$(POSTGRES_DB)"
```

The Job runs `python -m alembic -c /app/alembic.ini upgrade head`. Keep its image synchronized with the backend release whenever migrations change.

The migration application is currently annotated wave `65`, after the application at wave `60`; the Job also depends on a ConfigMap owned by the application. For first installation, wait for its Secret/ConfigMap and PostgreSQL, complete the corrected migration, and verify the schema before sending application traffic. For future releases, arrange explicit migration dependencies and backward-compatible schema changes; the current structure does not guarantee migrations finish before new application pods serve requests.

### SonarQube first-run bootstrap

Terraform creates `sonarqube-ci-cred` without a secret version. The Job currently calls `get-secret-value` under `set -e`; a missing initial version can exit the script before token generation. Update the Job to treat **only a confirmed missing initial version** as an empty credential and continue; authentication, KMS, and network errors must remain failures.

Also review the Job placement. It is a `PostSync` hook of the early `workload-secrets` application, but it requires the SonarQube service account, running server, route, DNS, and TLS. Run/retry it after those dependencies are healthy, or move it into a dedicated later bootstrap application. Its current timeout is 900 seconds.

### Controller ordering and resource prerequisites

- Create the `cert-manager`, `external-dns`, and `sonarqube` namespaces before manually applying resources into them. `CreateNamespace=true` covers an application's destination namespace; it is not a general guarantee for every namespace referenced by a multi-namespace directory.
- The workload ExternalDNS application actually has sync-wave `5`, despite a `35` comment in the workload kustomization. Adjust the annotation if later ordering is intended.
- Parent sync waves order child `Application` resources. They alone do not prove all child controllers, CRDs, webhooks, or jobs are ready. Inspect child health during first bootstrap.
- There is no RQ worker Deployment or recurring cleanup CronJob in this repository. Add those if automatic expiry/email is required; setting a retention variable alone does not enable cleanup.

## Bootstrap and verify

The infrastructure repository's `main-ci-merge.yaml` already installs Argo CD and applies `argocd/management-root.yaml` through the admin EC2. If that workflow has completed successfully, start at verification. The manual equivalent follows.

### 1. Connect to the admin host

From your workstation, using an AWS identity allowed to start SSM sessions:

```bash
aws ssm start-session --target YOUR_ADMIN_INSTANCE_ID --region us-east-1
```

Inside that session:

```bash
aws eks update-kubeconfig --region us-east-1 \
  --name logbeacon-management-cluster --alias management
aws eks update-kubeconfig --region us-east-1 \
  --name logbeacon-workload-cluster --alias workload
kubectl --context management get nodes
kubectl --context workload get nodes
```

Use your actual cluster names if customized. The instance role has EKS access entries from Terraform. A successful local `update-kubeconfig` does not make a private API reachable from your laptop.

### 2. Install Argo CD in management

On the admin host, clone the reviewed repository/fork and enter it:

```bash
git clone https://github.com/iamridoydey/logbeacon-aws-k8s.git
cd logbeacon-aws-k8s
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  --kube-context management \
  --namespace argocd --create-namespace --wait \
  -f argocd/argocd-values.yaml
kubectl --context management wait \
  --for=condition=Established crd/applications.argoproj.io --timeout=120s
kubectl --context management apply -f argocd/management-root.yaml
```

The current bootstrap command does not pin Argo CD's chart version. Select and pin a reviewed version with `--version` when standardizing your environment. Do not apply `argocd/workload/` to the workload cluster: its files define management-cluster Application objects.

The root creates management applications, including workload-cluster registration, and then the workload root. The workload root creates the applications that install workload resources.

### 3. Inspect reconciliation

```bash
kubectl --context management -n argocd get applications
kubectl --context management -n argocd describe application management-root
kubectl --context management -n argocd get externalsecret workload-cluster
kubectl --context management -n argocd get secret workload-cluster-secret

kubectl --context workload get clustersecretstore aws-secrets-manager
kubectl --context workload get externalsecrets -A
kubectl --context workload get pods -A
kubectl --context workload -n logbeacon get pvc
kubectl --context workload -n logbeacon get rollouts
```

If a first sync races a CRD, webhook, namespace, or chart dependency, resolve the dependency and resync the affected child application. Check actual resource conditions rather than relying only on the root application's sync status.

### 4. Confirm migrations, certificates, and routes

After correcting the migration application and Job:

```bash
kubectl --context workload -n logbeacon get jobs
kubectl --context workload -n logbeacon logs job/logbeacon-db-migrate
kubectl --context workload -n logbeacon wait \
  --for=condition=complete job/logbeacon-db-migrate --timeout=600s

kubectl --context management get certificates -A
kubectl --context workload get certificates -A
kubectl --context workload get gateways,httproutes -A
kubectl --context workload -n traefik get services
```

Verify that the migration exits successfully, PVCs bind, application pods become ready, certificates report `Ready=True`, and Gateway/HTTPRoute conditions show acceptance and resolved references. Finally register/sign in through the UI, submit an analysis, and confirm it appears in the dashboard. `/health` alone only proves database connectivity, not table creation or a functioning Groq request.

## Access the services

These are configured hostnames, **not a claim that a deployment is currently online**. Replace the domain consistently when using your own environment.

| Service | Configured URL |
| --- | --- |
| LogBeacon UI | `https://aws-logbeacon.iamridoydey.me` |
| Flask API | `https://api.aws-logbeacon.iamridoydey.me` |
| Argo CD | `https://argo.aws-logbeacon.iamridoydey.me` |
| SonarQube | `https://sonarqube.aws-logbeacon.iamridoydey.me` |

Traefik terminates TLS using cert-manager certificates. ExternalDNS observes HTTPRoutes and manages Cloudflare records. The Gateway listener port `8443` is Traefik's internal HTTPS entrypoint; check the LoadBalancer Service's external port mapping when diagnosing connectivity.

### Argo CD administrator

The initial username is `admin`. On the admin host, retrieve the initial password if the secret still exists:

```bash
kubectl --context management -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 --decode
printf '\n'
```

This is the initial password, not necessarily the current password after a reset. See [Argo CD's getting-started guide](https://argo-cd.readthedocs.io/en/latest/getting_started/) for changing it and removing the initial-password secret.

Before DNS/TLS is ready, run this on the admin host in one session:

```bash
kubectl --context management -n argocd port-forward service/argocd-server 8080:80
```

That listener is on the admin instance, not your workstation. To reach it from your browser, open another local terminal and start an SSM tunnel using an identity permitted to use this document:

```bash
aws ssm start-session --target YOUR_ADMIN_INSTANCE_ID --region us-east-1 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8080"],"localPortNumber":["8080"]}'
```

Then open [http://localhost:8080](http://localhost:8080). HTTP here matches the repository's Argo CD `server.insecure` setting; public TLS termination is handled by Traefik.

### SonarQube administrator

The configured administrator login is `admin`. After a successful bootstrap Job, its password is the `ADMIN_PASSWORD` value in AWS Secrets Manager. Retrieve it only from a workstation/profile authorized for the secret and its KMS key:

```bash
aws secretsmanager get-secret-value \
  --secret-id sonarqube-admin-password --region us-east-1 \
  --query SecretString --output text | jq -r '.ADMIN_PASSWORD'
```

The admin EC2 role is not automatically granted access to this secret. The monitoring passcode and CI token serve different purposes and are not UI login passwords. If the Job failed, the configured AWS password may not yet match SonarQube; inspect its logs before assuming rotation succeeded.

## Release and operate the application

### Image flow

1. Merge reviewed application code into the app repository's `main` branch.
2. Application CI builds/scans the frontend and backend, then pushes immutable short-SHA tags to ECR.
3. Image Updater, configured with `newest-build`, checks both image repositories and is configured to propose a GitHub pull request updating the application Kustomization.
4. Review the tag changes and migration requirements, then merge the Kubernetes PR.
5. Argo CD reconciles the new desired state; Argo Rollouts performs the canary rollout.

Image Updater configuration lives in `infra-resources/image-updater/image-updater.yaml`; its registry authentication lives in `infra/image-updater-values.yaml`. Verify the installed ImageUpdater CRD accepts the configured `pullRequest` fields and that repository permissions allow writeback. Configuration alone does not prove that a PR was created.

For a manual release, edit `newName`/`newTag` in `apps/workload/logbeacon/kustomization.yaml` and open a PR. Update the separate migration Job image too when applicable. The updater selects the newest build for each image independently; review the pair if frontend/backend compatibility requires matching release tags.

### Canary promotion

Both frontend and backend request three replicas. Their steps are:

| Step | Traffic / action |
| --- | --- |
| 1 | Send 20% to canary. |
| 2 | Pause indefinitely for manual review. |
| 3 | After promotion, send 50% to canary. |
| 4 | Pause for 30 seconds, then continue to completion. |

With the Argo Rollouts kubectl plugin installed on a host with workload-cluster access:

```bash
kubectl --context workload argo rollouts get rollout logbeacon-frontend -n logbeacon
kubectl --context workload argo rollouts get rollout logbeacon-backend -n logbeacon

# Run after reviewing the respective canary:
kubectl --context workload argo rollouts promote logbeacon-frontend -n logbeacon
kubectl --context workload argo rollouts promote logbeacon-backend -n logbeacon
```

The first deployment may not have a previous stable ReplicaSet to split traffic against. No automated AnalysisTemplate/metric gate is configured in these manifests; the indefinite pause is the review gate.

To abort a problematic rollout:

```bash
kubectl --context workload argo rollouts abort logbeacon-backend -n logbeacon
```

Also restore the intended previous image tags through Git so desired state agrees with the recovery. Consider the image updater's selection before it proposes the same image again. Database changes may need separate recovery; an image rollback does not reverse migrations.

### Useful diagnostics

```bash
kubectl --context workload -n logbeacon logs -l app=logbeacon-backend --tail=100
kubectl --context workload -n logbeacon logs -l app=logbeacon-frontend --tail=100
kubectl --context workload -n logbeacon get events --sort-by=.lastTimestamp
kubectl --context management -n argocd describe application workload-logbeacon-apps
kubectl --context workload describe clusterissuer letsencrypt-cloudflare
```

## CI and local validation

The current workflow runs on pull requests to `main`. It does not apply resources directly to a cluster.

| Check | Scope |
| --- | --- |
| yamllint | Repository YAML using `.yamllint.yaml`. |
| Kustomize | Builds `apps/workload` and uploads the rendered result. |
| Checkov | Scans the rendered workload manifest; `soft_fail: false`. |
| kubeconform | Strict validation against Kubernetes `1.33.0`, with missing schemas ignored. |

From the repository root, with those tools installed:

```bash
yamllint -c .yamllint.yaml .
mkdir -p rendered
kustomize build apps/workload > rendered/workload-rendered.yaml
checkov -f rendered/workload-rendered.yaml --framework kubernetes
kubeconform -strict -summary -ignore-missing-schemas \
  -kubernetes-version 1.33.0 rendered/workload-rendered.yaml

# Additional useful rendering checks beyond the current CI workload build:
kustomize build argocd/management > rendered/management-applications.yaml
kustomize build argocd/workload > rendered/workload-applications.yaml
```

The last two commands render Argo CD Application definitions; they do not download/render every referenced Helm chart. Missing custom-resource schemas are skipped, so a passing kubeconform run does not validate all Rollout, Gateway, or other CRD-specific fields. The current CI workload build also excludes the separate migration Job and many platform resources. Do not infer full deployment validity from those checks alone.

## Troubleshooting

| Symptom | Where to look |
| --- | --- |
| `Application` resource is unknown | Install Argo CD and wait for its CRD in the management cluster before applying the root. |
| ExternalSecret is not Ready | Inspect `aws-secrets-manager`, AWS secret names/properties, service-account Pod Identity, and KMS decrypt access. |
| Resource namespace is missing | Ensure the required chart/namespace exists; inspect first-sync ordering. |
| PVC is Pending | Check EBS CSI pods, Pod Identity, `ebs-gp3`, pod scheduling, and node capacity. `WaitForFirstConsumer` intentionally delays binding until a consumer can be scheduled. |
| ImagePullBackOff | Confirm the full ECR repository/tag exists, image architecture matches the nodes, and node image-pull permissions/networking work. |
| Migration application cannot load | Correct its source directory; fix the Secret typo, image, and `DATABASE_URL` wiring described above. |
| Rollout stays Paused | Review it and explicitly promote; the first canary pause has no timeout. |
| Gateway is unprogrammed | Check Traefik, the experimental Gateway API CRDs installed by its application, TLS Secret, and listener configuration. |
| TLS issuance fails | Check cert-manager, ClusterIssuer, Cloudflare token scope, DNS zone, and Challenge/Order resources. |
| DNS record does not appear | Check ExternalDNS logs, domain filters, distinct TXT ownership IDs, and Gateway address availability. |
| SonarQube CI secret has no value | Inspect bootstrap Job dependencies and its handling of the first empty secret version. |
| Expired logs are never removed | No worker/CronJob is deployed; add them and correct the retention variable. |
| Users lose sessions between replicas | The application currently uses Express's in-memory session store; configure shared sessions and production cookie handling in the application. |

The application also needs its documented logging and Markdown-rendering issues addressed before public use. Redis has authentication disabled in the current values, and PostgreSQL uses a legacy image with `allowInsecureImages: true`; review these concrete settings before expanding the deployment's exposure or reliability requirements.

## Cleanup

Use the infrastructure repository's documented teardown sequence. Stop reconciliation before removing resources so Argo CD does not recreate them. Remove Kubernetes-managed load balancers and storage while the clusters/controllers still operate, then destroy Terraform infrastructure.

The `ebs-gp3` StorageClass has `reclaimPolicy: Delete`: deleting its PVCs can delete the underlying volumes. Take backups before removing PostgreSQL or SonarQube storage. Deleting an Argo CD Application is not a substitute for checking the actual workload, cloud load balancers, volumes, and DNS records left behind.

## Contributing and documentation basis

Create a feature branch, validate the changed manifests, and open a PR to `main`. Explain changes to resource ownership, routing, image tags, secrets, or migration order. Prefer Git changes for lasting fixes because self-healing can overwrite manual cluster edits.

Reviewed against Kubernetes commit [`cce50be`](https://github.com/iamridoydey/logbeacon-aws-k8s/tree/cce50bedb0c238df8eb2dc0c9fa1be2f651d3117). Paths, settings, and commands were checked against source; no live cluster was deployed for this documentation review.