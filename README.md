# app-of-apps-meetup-demo

A demo of the ArgoCD [App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/).

## Why App of Apps?

In a standard setup you apply each ArgoCD Application manually. With App of Apps, a single parent Application watches a directory in Git and automatically creates/manages all child Applications it finds there. The result is:

- **One bootstrap command** to bring up an entire environment
- **Git is the source of truth** — add a new app by adding a file, remove it by deleting the file
- **Self-healing** — if a child app is deleted from the cluster, the parent recreates it automatically

## Structure

```
apps/
  demo.yaml                        # Parent app — watches envs/demo/
envs/
  demo/
    dummy/application.yaml         # Child app: deploys charts/dummy via Helm
    nginx/application.yaml         # Child app: deploys Bitnami nginx as reverse proxy
charts/
  dummy/                           # Custom Helm chart
    values.yaml                    # Base values (empty — all overrides in values.demo.yaml)
    values.demo.yaml               # Demo environment values
    templates/
      deployment.yaml
      service.yaml
      serviceaccount.yaml
```

## Application hierarchy

```
app-of-apps-meetup-demo            (apps/demo.yaml)
├── dummy                          (envs/demo/dummy/application.yaml)
└── nginx                          (envs/demo/nginx/application.yaml)
```

- **app-of-apps-meetup-demo** — the parent. Watches `envs/demo/` recursively. Any ArgoCD Application manifest dropped in that folder gets picked up automatically.
- **dummy** — deploys `crccheck/hello-world` via the local Helm chart in `charts/dummy`. Runs on port 8000, exposed internally as a ClusterIP service on port 80.
- **nginx** — deploys Bitnami nginx (chart v23.0.0) configured as a reverse proxy. All requests to nginx are forwarded to the dummy service at `dummy.dummy.svc.cluster.local:80`.

## Sync policy

All applications use the same automated sync policy:

| Option | Effect |
|---|---|
| `automated` | ArgoCD syncs automatically on every Git change |
| `selfHeal: true` | If someone manually changes a resource in the cluster, ArgoCD reverts it |
| `prune: true` | Resources removed from Git are deleted from the cluster |
| `CreateNamespace: true` | Namespaces are created automatically if they don't exist |
| `PruneLast: true` | Deletions happen after all other changes are applied |

## Connecting the repository to ArgoCD

Go to **Settings → Repositories → Connect Repo** in the ArgoCD UI and select **HTTPS**.

**Public repo** — only the URL is required, no credentials needed:

| Field | Value |
|---|---|
| Repository URL | `https://github.com/smantzou/app-of-apps-meetup-demo.git` |
| Everything else | leave blank |

**Private repo** — add a GitHub Personal Access Token (PAT):

| Field | Value |
|---|---|
| Repository URL | `https://github.com/smantzou/app-of-apps-meetup-demo.git` |
| Username | your GitHub username |
| Password | your PAT (GitHub → Settings → Developer settings → Personal access tokens → `repo` scope) |

## Setup

**Prerequisites:** minikube, kubectl, helm, argocd CLI

```bash
# Start cluster
minikube start --cpus=4 --memory=4096

# Install ArgoCD (use --server-side to avoid CRD annotation size limit)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts

# Wait for ArgoCD to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Bootstrap the parent app (one-time)
kubectl apply -f apps/demo.yaml
```

ArgoCD will detect `envs/demo/` and automatically create and sync `dummy` and `nginx`.

## Access

```bash
# ArgoCD UI — keep this running in a separate terminal
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080

# ArgoCD CLI login
argocd login localhost:8080 --username admin --password admin --insecure

# Check all apps
argocd app list
```

```bash
# Test the nginx → dummy proxy chain
kubectl port-forward svc/nginx -n nginx 9090:80
curl http://localhost:9090
# Returns the Hello World response from the dummy container
```

## RBAC

The parent app is protected — only `admin` can delete it. The `developer` role can manage child apps but cannot delete the parent.

| User | Username | Password | Can delete parent app |
|---|---|---|---|
| Admin | `admin` | `admin` | Yes |
| Developer | `developer` | `developer` | No |

Policy is defined in the `argocd-rbac-cm` ConfigMap in the `argocd` namespace.

## Demo scenarios

**Self-healing** — delete a child app and watch ArgoCD recreate it:
```bash
kubectl delete application dummy -n argocd
# within seconds ArgoCD recreates it
argocd app list
```

**GitOps add** — add a new Application manifest under `envs/demo/`, push to Git, and watch it appear in ArgoCD automatically without any `kubectl apply`.

**GitOps remove** — delete a manifest from `envs/demo/`, push, and ArgoCD prunes the app and all its resources from the cluster.

## Troubleshooting

**CRD annotation too long error on ArgoCD install**
```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long
```
Use `--server-side --force-conflicts` flag with `kubectl apply` as shown in the setup above.

**nginx stuck in Progressing**
The Bitnami nginx chart defaults to `LoadBalancer` service type. On minikube there is no cloud load balancer so `EXTERNAL-IP` stays `<pending>` indefinitely. This is handled in `envs/demo/nginx/application.yaml` by overriding `service.type: ClusterIP`.
