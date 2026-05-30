# DiplomaProject — GitOps repo

This is the GitOps source of truth for my diploma project. Everything that runs in the EKS cluster is described here — applications, monitoring, secrets, autoscaling. ArgoCD watches this repo and reconciles the cluster.

## The three repos

| Repo | What lives there |
|------|------------------|
| [`DiplomaProject-Terraform`](../DiplomaProject-Terraform) | EKS cluster, VPC, IAM/IRSA, ECR, ACM, RDS subnet group, ArgoCD itself (Helm release), AWS Secrets Manager containers |
| [`DiplomaProject-App`](../DiplomaProject-App) | Node.js/TypeScript backend (Express + Kubernetes client) and Next.js frontend |
| **`DiplomaProject-ArgoCD`** *(this one)* | Everything Kubernetes — apps, monitoring, observability, autoscaling, secrets wiring |

The split is deliberate: Terraform owns AWS, ArgoCD owns Kubernetes, and the apps own their own code. Bootstrapping is one-way: `terraform apply` creates the cluster and installs ArgoCD; ArgoCD then picks up everything in this repo.

## What runs in the cluster

```
eu-central-1 EKS
├── argocd                    (managed by Terraform Helm release)
├── external-secrets          → AWS Secrets Manager
├── monitoring                Prometheus, Alertmanager, Grafana, Loki, Tempo, Alloy
├── karpenter                 node autoscaling, spot + on-demand
├── kube-system               ALB controller, EBS CSI, RDS ACK
└── gm-diploma-project-prod   the actual app
    ├── backend  (Node 20, Express, talks to the Kubernetes API)
    └── frontend (Next.js)
```

PR previews run in ephemeral `pr-{N}` namespaces with their own ephemeral Postgres pod and a unique hostname `pr-{N}.elsys.itgix.eu`. Cleanup happens automatically when the PR closes.

## Repo layout

```
.
├── gm-diploma-project-*.yaml          root-level ArgoCD Applications (one per service)
├── eu-central-1/
│   ├── helm/                          base Helm chart used by both prod and preview
│   ├── prod/
│   │   ├── helm/                      prod values (overrides base chart)
│   │   ├── databases/                 RDS instance via ACK controller
│   │   ├── monitoring/                kube-prometheus-stack + custom rules/dashboards
│   │   └── secrets/                   ClusterSecretStore + ExternalSecrets
│   ├── preview/helm/                  preview values (ephemeral DB, latest tags)
│   └── karpenter/                     NodePool + EC2NodeClass
```

## Bootstrap

Assuming Terraform has already brought up the cluster and installed ArgoCD:

```bash
kubectl apply -f gm-diploma-project-external-secrets.yaml   # ESO first — others depend on its CRDs
kubectl apply -f gm-diploma-project-secrets.yaml            # ClusterSecretStore + ExternalSecrets
kubectl apply -f gm-diploma-project-monitoring.yaml         # Prometheus stack + custom rules
kubectl apply -f gm-diploma-project-loki.yaml
kubectl apply -f gm-diploma-project-tempo.yaml
kubectl apply -f gm-diploma-project-alloy.yaml
kubectl apply -f gm-diploma-project-karpenter.yaml
kubectl apply -f gm-diploma-project-db.yaml
kubectl apply -f gm-diploma-project-prd.yaml
kubectl apply -f gm-diploma-project-preview-appset.yaml
```

Order matters only for the first one (ESO must be Healthy before its CRDs are referenced by others). Everything else can land in any order.

## Secrets

No secret values live in this repo. They are stored in AWS Secrets Manager and pulled into the cluster by [External Secrets Operator](https://external-secrets.io). The Terraform side creates the empty secret containers; you populate them in the AWS console (one-time per environment).

| Path | Used by |
|------|---------|
| `gm-diploma-project/grafana-admin` | Grafana login (`username`, `password`) |
| `gm-diploma-project/alertmanager-config` | Discord webhook URL (`webhook-url`) |
| `gm-diploma-project/preview-db` | ephemeral Postgres password for PR previews (`password`) |
| `gm-diploma-project/github-token` | GitHub PAT (`token`) — read by the preview ApplicationSet's `pullRequest` generator and by the `preview-cleanup` CronJob to list / poll PR state |

The IAM role `gm-diploma-external-secrets-role` (created in Terraform with `use_name_prefix = false`) is what ESO assumes via IRSA. The trust policy is bound to the `external-secrets:external-secrets` service account.

## Monitoring

Open Grafana at `https://grafana-gmdiplomaproject.elsys.itgix.eu`. Default landing page is the **GM Diploma — Application Overview** dashboard.

What's covered:

- **Alerts** — 35 rules across SLO availability, pod health, resource pressure, infra. Split by file under `eu-central-1/prod/monitoring/rules/`. All firing alerts go to Discord; `severity=critical` uses a dedicated receiver with shorter `repeat_interval`.
- **Dashboards** — provisioned via ConfigMaps (label `grafana_dashboard: "1"`). The custom ones live in `eu-central-1/prod/monitoring/dashboards/`. The kube-prometheus-stack ships dozens of standard ones too.
- **Metrics** — Prometheus scrapes everything via ServiceMonitors. Custom ones for Karpenter, Loki, Loki canary, Tempo, Alloy, ESO under `monitoring/servicemonitors/`.
- **Logs** — Alloy DaemonSet collects pod logs and ships to Loki. Trace IDs in logs (pattern `traceID=...`) auto-link to Tempo via Grafana derived fields.
- **Traces** — Tempo single-binary, OTLP/Jaeger/Zipkin receivers exposed. The app isn't instrumented yet, so traces are empty for now.

To test the alert pipeline manually:

```bash
kubectl create namespace alert-test
kubectl run crash-test -n alert-test --image=busybox --restart=Always \
  --command -- /bin/sh -c "exit 1"
# wait ~1 minute, watch Discord
kubectl delete namespace alert-test
```

## Production app

Production runs in `gm-diploma-project-prod`:

- **backend**: 2 replicas (HPA min 2, max 10, target 50% CPU), pinned to a SHA tag, uses the in-cluster Kubernetes API (via its ServiceAccount) to list namespaces and deployments for the dashboard.
- **frontend**: 2 replicas (HPA min 2, max 5, target 80% CPU), Next.js standalone, talks to backend via `/api`.
- Behind an AWS ALB, hostname `gmdiplomaproject.elsys.itgix.eu`, TLS terminated by ACM.

An RDS Postgres instance (provisioned via the ACK controller in `eu-central-1/prod/databases/`) and its `gm-diploma-prod-db-secret` exist as scaffolding for future persistence work — the current backend does not use them.

## PR preview environments

Driven by the `gm-diploma-preview-appset` ApplicationSet. It polls the GitHub `DiplomaProject-App` repo for open PRs every 60 seconds. For each open PR:

- A namespace `pr-{number}` is created
- The chart in `eu-central-1/preview/helm/` is deployed with overrides for image tags (`backend-pr-{number}`, `frontend-pr-{number}`), DB name, hostname, and CORS origin
- Ephemeral Postgres runs in-cluster (not RDS — keeps cost down)
- Hostname `pr-{number}.elsys.itgix.eu`

When the PR closes the namespace and everything in it is pruned automatically.

## Notes & gotchas

- **CRDs and the 256KiB annotation limit** — ESO has very large CRDs. ArgoCD's classic client-side apply can't handle them. The `gm-diploma-project-external-secrets` Application has `ServerSideApply=true` in `syncOptions` so this works. ArgoCD ≥ v3.3 is required.
- **Server-side diff** — enabled globally in ArgoCD via `controller.diff.server.side=true` (set in the Terraform Helm values). Without it, charts that don't explicitly set `protocol: TCP` on container ports show as out-of-sync after deployment.
- **Loki and Tempo storage** — currently filesystem-backed. Logs and traces are lost on pod restart. Migrating to S3 is on the to-do list.
- **The backend image runs as root** — the in-cluster pod has no security context yet. Image needs a non-root USER directive in the Dockerfile before pod-level `runAsNonRoot: true` can be enforced.
- **Secrets in old git history** — the original commits contained plaintext Grafana/Discord/preview-DB credentials. They've since been rotated and moved to Secrets Manager, but `git log -p` still shows them. Purging requires a `git filter-repo` rewrite.

## Useful commands

```bash
# what's in the cluster from this repo
kubectl get applications -n argocd

# inspect the alert pipeline
kubectl exec -n monitoring alertmanager-monitoring-kube-prometheus-alertmanager-0 \
  -c alertmanager -- wget -qO- http://localhost:9093/api/v2/alerts | jq .

# force a sync without pushing a commit
kubectl annotate application <name> -n argocd argocd.argoproj.io/refresh=hard --overwrite
```
