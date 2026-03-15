# ArgoCD GitOps Repository

An ArgoCD setup implementing the **App-of-Apps pattern** with a 3-tier bootstrap hierarchy. Built and tested locally on **Rancher Desktop**.

---

## What is this?

This repository is the single source of truth for deploying and managing Kubernetes workloads using [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). Every application, cluster add-on, and deployment configuration is declared as code here — no manual `kubectl apply` required after the initial bootstrap.

It separates **cluster-level infrastructure** (metrics, policy engines, ingress controllers) from **developer application workloads**, giving platform and dev teams independent ownership of their respective concerns.

---

## Repository Structure

```
.
├── bootstrap-app.yaml                          # The only file you ever apply manually
│
├── bootstrap/                                  # Tier 1 — meta-root manages this
│   ├── cluster-management-root/
│   │   ├── project.yaml                        # AppProject for cluster add-ons
│   │   └── app-of-apps.yaml                    # sync-wave: "1"
│   └── application-management-root/
│       ├── project.yaml                        # AppProject for developer apps
│       └── app-of-apps.yaml                    # sync-wave: "2"
│
├── cluster-management/                         # Tier 2 — cluster infrastructure
│   ├── metrics-server/
│   │   └── app.yaml                            # ApplicationSet — metrics-server
│   ├── kyverno-policies/
│   │   └── kyverno-policies.yaml               # ApplicationSet — Kyverno engine
│   └── aws-load-balancer-controller/
│       └── app.yaml                            # ApplicationSet — AWS LBC
│
└── application-management/                     # Tier 2 — developer workloads
    ├── fibo-app/
    │   └── app.yaml                            # ApplicationSet — fapp-dev, fapp-qa
    └── fibo-web/
        └── app.yaml                            # ApplicationSet — fweb-dev, fweb-qa
```

---

## How It Works — 3-Tier Bootstrap

```
bootstrap-app.yaml          ← kubectl apply this ONCE
        │
        ▼
    meta-root                watches: bootstrap/
        │
        ├── cluster-management-root   (sync-wave: "1")
        │       └── deploys everything in cluster-management/
        │
        └── application-management-root   (sync-wave: "2")
                └── deploys everything in application-management/
```

**Sync waves** ensure cluster infrastructure (metrics-server, Kyverno) is always deployed and healthy before developer applications attempt to start.

Adding a new application is as simple as dropping an `Application` or `ApplicationSet` manifest into the appropriate directory and pushing to `main`. ArgoCD picks it up automatically.

---

## How This Helps SREs

**Single bootstrap command.** One `kubectl apply` sets up the entire platform. ArgoCD self-manages everything after that via `selfHeal` and `automated` sync.

**Clear ownership boundaries.** The `cluster-management` tree is owned by the platform team. The `application-management` tree is owned by dev/SRE teams. Each has its own ArgoCD `AppProject` with scoped RBAC and allowed source repositories.

**Multi-environment support out of the box.** `ApplicationSet` list generators create per-environment Applications (dev, qa, prod) from a single manifest. Adding a new environment means adding one element to the list.

**Consistent sync policy everywhere.** All Applications use the same `prune`, `selfHeal`, `retry`, and `syncOptions` configuration, reducing drift and making behaviour predictable.

**Ordered deployments.** Sync waves (`-90` for infra add-ons, `1` for cluster root, `2` for app root) guarantee dependency order without manual coordination.

---

## Local Setup — Rancher Desktop

This repository is built and tested on [Rancher Desktop](https://rancherdesktop.io/), which provides a local Kubernetes cluster with `containerd` and a built-in `kubectl`.

### Prerequisites

- [Rancher Desktop](https://rancherdesktop.io/) installed and running
- Kubernetes enabled in Rancher Desktop preferences
- `kubectl` configured (Rancher Desktop sets this up automatically)
- `helm` (optional, for manual inspection)

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  --server-side \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD to be ready:

```bash
kubectl wait --for=condition=available deployment -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s
```

### 2. Access the ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open [https://localhost:8080](https://localhost:8080) in your browser.

Retrieve the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with username `admin` and the password above.

### 3. Bootstrap the Entire Platform

```bash
kubectl apply -f https://raw.githubusercontent.com/ssah-blr/argocd/refs/heads/main/bootstrap-app.yaml
```

That's it. ArgoCD will now reconcile the full hierarchy — cluster add-ons first, then developer applications.

### 4. Watch the Sync

In the ArgoCD UI, you will see the following Applications appear in order:

| Application | Wave | What it deploys |
|---|---|---|
| `bootstrap` | — | The meta-root itself |
| `cluster-management-root` | 1 | Cluster add-on tree |
| `metrics-server-in-cluster` | -90 | Kubernetes Metrics Server |
| `kyverno-policies-in-cluster` | -90 | Kyverno policy engine |
| `application-management-root` | 2 | Developer app tree |
| `fapp-dev` / `fapp-qa` | — | Fibonacci app (dev + qa) |
| `fweb-dev` / `fweb-qa` | — | Fibonacci web (dev + qa) |

### 5. Teardown

To remove everything ArgoCD manages, delete the bootstrap app with cascading finalizers:

```bash
kubectl delete -f bootstrap-app.yaml
```

To remove ArgoCD itself:

```bash
kubectl delete namespace argocd
```

---

## Adding a New Application

1. Create a directory under `application-management/your-app/`
2. Add an `app.yaml` with an `Application` or `ApplicationSet` manifest
3. Set `project: application-management-root`
4. Push to `main`

ArgoCD will detect the new file within the default reconciliation interval (3 minutes) and deploy it automatically.

For a new cluster add-on, follow the same steps under `cluster-management/` and use `project: cluster-management-root`.

---

## Contributing

All changes should be made via pull request to `main`. ArgoCD's `selfHeal: true` means any manual change to cluster resources will be reverted — the repository is always the source of truth.
