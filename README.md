# TaskFlow — Infrastructure Repository

Kubernetes manifests + Argo CD Application for TaskFlow, managed GitOps-style:
this repo is the single source of truth for what's running in the cluster.
CI/CD in the application repository updates the image tags here on every
successful build; Argo CD watches this repo and syncs the cluster to match.

```
infra-repo/
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-configmap.yaml
│   ├── 02-secret.yaml        # TEMPLATE - replace values, don't commit real secrets
│   ├── 03-mongo.yaml         # StatefulSet + PVC + headless Service
│   ├── 04-redis.yaml         # StatefulSet + PVC + Service (AOF persistence)
│   ├── 05-backend.yaml       # Deployment + Service
│   ├── 06-worker.yaml        # Deployment
│   ├── 07-worker-hpa.yaml    # HorizontalPodAutoscaler
│   ├── 08-frontend.yaml      # Deployment + Service
│   └── 09-ingress.yaml
├── argocd/
│   └── application.yaml
└── docs/
    └── TaskFlow-Architecture-Document.docx
```

## 1. Get a Kubernetes cluster (k3s)

On any Linux box (a cheap VM is enough):

```bash
curl -sfL https://get.k3s.io | sh -
sudo cat /etc/rancher/k3s/k3s.yaml   # kubeconfig, copy it to ~/.kube/config locally
export KUBECONFIG=~/.kube/config
kubectl get nodes                     # should show your node as Ready
```

k3s ships with Traefik by default; the Ingress manifest here targets
`ingressClassName: nginx`, so either install the ingress-nginx controller,
or change that field to `traefik` if you'd rather use what's built in.

## 2. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# access the UI (quickest way for a demo/local cluster)
kubectl -n argocd port-forward svc/argocd-server 8080:443
# open https://localhost:8080, login as admin / <password above>
```

## 3. Point Argo CD at this repo

Edit `argocd/application.yaml`:

- `spec.source.repoURL` → this repo's real git URL, once you've pushed it
- `spec.source.targetRevision` → the branch you want tracked (`main`)

Then apply it:

```bash
kubectl apply -f argocd/application.yaml
```

Argo CD will:

1. Clone this repo and read everything under `k8s/`
2. Create the `taskflow` namespace (`CreateNamespace=true`)
3. Apply every manifest and report their health/sync status
4. **Auto-sync**: any new commit here is picked up and applied automatically
   (`automated.prune: true` removes resources deleted from git,
   `selfHeal: true` reverts manual `kubectl edit`s back to match git)

Take your dashboard screenshot once the `taskflow` Application shows
**Healthy** and **Synced** — that's the required deliverable screenshot.

## 4. Before you go live — replace the placeholders

- `k8s/02-secret.yaml` — replace `JWT_SECRET` and `ADMIN_SIGNUP_CODE` with
  real random values. Don't commit real secrets in plain text long-term;
  this file is a template. For a real deployment, swap this for Sealed
  Secrets, SOPS, or an External Secrets Operator pointed at a real secret
  manager.
- `k8s/05-backend.yaml`, `06-worker.yaml`, `08-frontend.yaml` — the
  `image:` fields use `yourdockerhubusername/...`. The CI/CD pipeline in
  the application repo rewrites the tag automatically on every build, but
  the username portion needs to be set once, by hand, to match your Docker
  Hub account.
- `k8s/09-ingress.yaml` — swap `taskflow.local` for your real domain, and
  point DNS at your ingress controller's external IP.

## 5. Sanity checks after a sync

```bash
kubectl -n taskflow get pods
kubectl -n taskflow get hpa taskflow-worker-hpa
kubectl -n taskflow logs deploy/taskflow-worker --tail=50
```

## Why these particular design choices

- **Mongo & Redis as StatefulSets with PVCs** — both need stable storage
  across restarts; Redis specifically runs with `--appendonly yes` so a
  restart doesn't drop queued-but-unprocessed tasks.
- **Worker has no Service** — it's a pure queue consumer, nothing ever
  needs to send it traffic directly.
- **HPA on the worker only** — the worker is the component whose load is
  genuinely variable and CPU-bound; frontend/backend run a fixed 2 replicas
  here for basic HA, which is enough at this project's scale.
- **`terminationGracePeriodSeconds: 30`** on the worker — gives an
  in-flight `BRPOP`'d job a chance to finish before the pod is killed during
  a rollout or scale-down.

See `docs/TaskFlow-Architecture-Document.docx` for the full write-up
(worker scaling strategy, handling ~100k tasks/day, MongoDB indexing,
Redis failure/recovery, and the staging vs. production deployment
strategy).
