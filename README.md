# gitops-lumieres

Repo GitOps pour le déploiement de la **todo-api** et du **game-2048** via ArgoCD sur un cluster Kubernetes local (k3d).

---

## Architecture

```
gitops-lumieres/
├── apps/
│   ├── base/                            # Manifests K8s de base (Kustomize)
│   │   ├── kustomization.yaml           # Déclare les apps : todos-api + game-2048
│   │   ├── todos-api/                   # Deployment + Service de la todo-api
│   │   └── game-2048/                   # Deployment + Service + Ingress du jeu 2048
│   └── overlays/
│       ├── dev/                         # Env dev  : ns todos-api-dev,  todo-api 1 replica
│       └── prod/                        # Env prod : ns todos-api-prod, todo-api 3 replicas
├── apps-code/
│   ├── docker-2048/                     # Code source + Dockerfile du jeu 2048
│   └── todo-api-python/                 # Code source de la todo-api (FastAPI)
└── argocd/
    └── applications/                    # Application CRDs ArgoCD
        ├── argo-dev.yaml                # App dev  → namespace todos-api-dev
        └── argo-prod.yaml               # App prod → namespace todos-api-prod
```

---

## Applications déployées

| App | Image | Dev | Prod |
|-----|-------|-----|------|
| todo-api | `adame555/todo-api:v1.0.0` | 1 replica | 3 replicas |
| game-2048 | `adame555/docker-2048:v3` | 2 replicas | 2 replicas |

Les deux applications sont déclarées dans `apps/base` et partagent le namespace de chaque environnement (`todos-api-dev` en dev, `todos-api-prod` en prod).

---

## Prérequis

- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optionnel)

---

## Lancer depuis zéro

### 1. Créer le cluster

```bash
k3d cluster create gitops-lab
```

### 2. Installer ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=180s
```

### 3. Déployer les Applications ArgoCD

```bash
kubectl apply -f argocd/applications/argo-dev.yaml
kubectl apply -f argocd/applications/argo-prod.yaml
```

ArgoCD synchronise automatiquement depuis ce repo Git et déploie les ressources dans les namespaces correspondants.

---

## Accéder aux services

### UI ArgoCD

```bash
# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Exposer l'UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Ouvre **https://localhost:8080** (login: `admin`)

### todo-api (dev)

```bash
kubectl port-forward svc/todo-api -n todos-api-dev 8000:80
```

Endpoints disponibles :
- `GET  http://localhost:8000/todos/`
- `POST http://localhost:8000/todos/`
- `GET  http://localhost:8000/health`

### game-2048

Via l'Ingress (host différent par environnement) :
- dev  : **http://game-dev.172-189-159-185.sslip.io**
- prod : **http://game-prod.172-189-159-185.sslip.io**

Ou en port-forward :

```bash
kubectl port-forward svc/game-2048 -n todos-api-dev 8082:80
```

Ouvre **http://localhost:8082**

---

## Choix techniques

### Kustomize plutôt que Helm

Kustomize permet de séparer clairement la configuration de base des surcharges par environnement sans templating complexe. Les deux apps (todo-api + game-2048) sont déclarées une seule fois dans `apps/base`, et chaque overlay (`dev`/`prod`) ne fait qu'« appeler » la base puis patche ce qui diffère (replicas, namespace).

### Apps déclarées dans la base, appelées par les overlays

`apps/base/kustomization.yaml` agrège les manifests des deux applications. Chaque overlay référence `../../base` et applique son `namespace:` global, ce qui place toutes les ressources dans le namespace de l'environnement. game-2048 n'embarque donc plus son propre namespace.

### Un seul cluster, deux namespaces communs

- `todos-api-dev` → environnement de développement (todo-api 1 replica + game-2048)
- `todos-api-prod` → environnement de production (todo-api 3 replicas + game-2048)

---

## Environnements

| Namespace | App | Replicas | Géré par |
|-----------|-----|----------|----------|
| `todos-api-dev` | todo-api | 1 | ArgoCD app `dev` |
| `todos-api-dev` | game-2048 | 2 | ArgoCD app `dev` |
| `todos-api-prod` | todo-api | 3 | ArgoCD app `prod` |
| `todos-api-prod` | game-2048 | 2 | ArgoCD app `prod` |
