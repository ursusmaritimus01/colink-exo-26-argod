# ArgoCD GitOps Apps

Deze repository bevat de ArgoCD Application en ApplicationSet definities voor het **eXo Platform Breda** cluster.

## Structuur

```
argocd-apps/
├── kustomization.yaml                      # Bundelt alle resources
├── bootstrap-application.yaml             # Bootstrap: ArgoCD beheert deze repo zichzelf
├── github-argocd-apps-repo-secret.yaml    # SSH-toegang tot DEZE repo voor ArgoCD
├── applicationset-exo-platform.yaml       # ApplicationSet voor de eXo Platform applicatie
├── application-argocd.yaml                # Application voor ArgoCD zelf (self-managed)
└── application-headlamp.yaml              # Application voor Headlamp (Kubernetes dashboard)
```

## Overzicht

| Resource | Naam | Namespace | Source repo / pad |
|---|---|---|---|
| Application | `argocd-apps-bootstrap` | `argocd` | **deze repo** → `argocd-apps/` |
| ApplicationSet | `exo-platform` | `argocd` | deploy-repo → `exo-platform-breda/k8s/overlays/` |
| Application | `infra-argocd` | `argocd` | deploy-repo → `exo-platform-breda/k8s/overlays/infra-argocd/` |
| Application | `infra-headlamp` | `kube-system` | deploy-repo → `exo-platform-breda/k8s/base/infra-headlamp/` |

**Deploy-repo:** `git@github.com:ursusmaritimus01/colink-exo-26-deploy.git`
**Deze repo:** `git@github.com:ursusmaritimus01/colink-exo-26-argod.git`

## Deployment volgorde

### 1. ArgoCD installeren (eenmalig, handmatig)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. GitHub SSH repo secret handmatig toepassen

De SSH-sleutel voor toegang tot de private repo moet **vóór** alles worden aangemaakt:

```bash
kubectl apply -f \
  exo-platform-breda/k8s/overlays/infra-argocd/github-repo-secret.yaml
```

> ⚠️ **Waarschuwing:** De private key in `github-repo-secret.yaml` staat momenteel plaintext.
> Overweeg gebruik van [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
> of External Secrets Operator voor productie.

### 3. ArgoCD Applications toepassen

Alle Applications in één keer via Kustomize:

```bash
kubectl apply -k argocd-apps/
```

Of individueel:

```bash
kubectl apply -f argocd-apps/application-argocd.yaml
kubectl apply -f argocd-apps/application-headlamp.yaml
kubectl apply -f argocd-apps/applicationset-exo-platform.yaml
```

ArgoCD synchroniseert daarna automatisch alle resources vanuit de deploy-repo.

## Sync beleid

Alle Applications zijn geconfigureerd met:
- `automated.prune: true` — resources die uit de repo verdwijnen worden ook van het cluster verwijderd
- `automated.selfHeal: true` — handmatige wijzigingen op het cluster worden automatisch teruggedraaid
- `ServerSideApply: true` — gebruikt Kubernetes Server-Side Apply voor betere conflictafhandeling

## Cluster URL

De huidige configuratie gebruikt `https://kubernetes.default.svc` (in-cluster ArgoCD).
Bij gebruik van een remote cluster: vervang deze URL door de API-server URL van het doelcluster.
