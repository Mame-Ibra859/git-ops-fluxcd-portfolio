# Dépôt GitOps — FluxCD / Kustomize

Ce dépôt contient la configuration GitOps utilisée pour déployer et maintenir la plateforme du portfolio sur Kubernetes.

Il gère deux catégories de ressources :

- **Application** : le portfolio Next.js ;
- **Data Platform** : PostgreSQL utilisé par la plateforme.

L'objectif est que **Git constitue la source de vérité** du cluster. Flux surveille ce dépôt et réconcilie automatiquement l'état du cluster avec les manifests déclarés ici.

L'architecture repose sur **FluxCD + Kustomize**, avec une séparation `base` / `overlays` permettant de gérer les environnements `dev` et `prod`.

---

## 1. Architecture GitOps

Le fonctionnement général est le suivant :

```text
                         GitHub
                           │
                           │ GitRepository
                           ▼
                        FluxCD
                           │
                           │ Kustomization
                           ▼
                       Kustomize
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        Application                Data Platform
         Portfolio                   PostgreSQL
              │                         │
              └────────────┬────────────┘
                           ▼
                       Kubernetes
```

Une modification du dépôt suit donc le chemin :

```text
Git commit
    │
    ▼
GitRepository
    │
    ▼
Flux Kustomization
    │
    ▼
Kustomize
    │
    ▼
Kubernetes
```

Il n'est plus nécessaire d'appliquer manuellement les manifests avec `kubectl` à chaque modification.

---

# 2. Structure du dépôt

```text
.
├── applications/
│   └── portfolio/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── prod/
│
├── clusters/
│   ├── dev/
│   └── prod/
│
├── data-platform/
│   └── postgres/
│       ├── base/
│       └── overlays/
│           ├── dev/
│           └── prod/
│
└── .sops.yaml
```

La structure sépare les responsabilités :

- `clusters/` définit les environnements Kubernetes ;
- `applications/` contient les applications déployées ;
- `data-platform/` contient les services de données ;
- `base/` contient la configuration commune ;
- `overlays/` contient les différences propres à chaque environnement.

---

# 3. Installation de Flux

Flux est installé une seule fois sur le cluster.

Ce projet utilise l'**opérateur Flux** afin de gérer l'installation et la configuration des composants Flux.

### Installation de l'opérateur

```bash
helm install flux-operator \
  oci://ghcr.io/controlplaneio/fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace
```

L'opérateur permet ensuite de créer une instance Flux à partir d'une ressource `FluxInstance`.

### FluxInstance

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance

metadata:
  name: flux
  namespace: flux-system

spec:
  distribution:
    version: "2.x"
    registry: "ghcr.io/fluxcd"
    artifact: "oci://ghcr.io/controlplaneio-fluxcd/flux-operator-manifests"

  components:
    - source-controller
    - kustomize-controller
    - helm-controller
    - notification-controller
    - image-reflector-controller
    - image-automation-controller

  cluster:
    type: kubernetes
    size: small
    multitenant: false
    networkPolicy: true
    domain: "cluster.local"
```

Cette instance fournit notamment les controllers nécessaires à la synchronisation GitOps et à l'automatisation des images.

---

# 4. Connecter Flux à ce dépôt

Une fois Flux installé, il faut lui indiquer **où se trouve la configuration GitOps**.

C'est le rôle du `GitRepository`.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository

metadata:
  name: portfolio
  namespace: flux-system

spec:
  interval: 1m

  url: ssh://git@github.com/Mame-Ibra859/git-ops-fluxcd-portfolio.git

  ref:
    branch: main

  secretRef:
    name: github-flux-secret-write
```

Flux surveille alors la branche `main` de ce dépôt.

Le flux devient :

```text
GitHub
   │
   │ GitRepository
   ▼
Flux source-controller
   │
   ▼
Repository synchronisé
```

---

# 5. Définir ce qui doit être déployé

Le `GitRepository` permet à Flux de récupérer le dépôt, mais il ne lui indique pas encore **quelle partie du dépôt doit être appliquée**.

C'est le rôle de la `Kustomization` Flux.

Par exemple, pour l'environnement `dev` :

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization

metadata:
  name: portfolio-dev
  namespace: flux-system

spec:
  interval: 5m

  path: ./clusters/dev

  prune: true

  sourceRef:
    kind: GitRepository
    name: portfolio
```

Le lien entre les composants devient donc :

```text
GitRepository
     │
     │ récupère le dépôt
     ▼
clusters/dev
     │
     │ Kustomization
     ▼
Kustomize
     │
     ▼
applications + data-platform
     │
     ▼
Kubernetes
```

Le dossier `clusters/dev` constitue ainsi le **point d'entrée de l'environnement**.

---

# 6. Base et Overlays

Les applications utilisent Kustomize avec une architecture :

```text
base/
overlays/
├── dev/
└── prod/
```

La `base` contient les ressources communes.

Par exemple :

```text
applications/portfolio/base/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
└── ...
```

Les overlays ajoutent les différences propres aux environnements :

```text
applications/portfolio/overlays/
├── dev/
│   ├── kustomization.yaml
│   └── patch.yaml
│
└── prod/
    ├── kustomization.yaml
    └── patch.yaml
```

Le principe est donc :

```text
                    BASE
                     │
              ┌──────┴──────┐
              ▼             ▼
             DEV           PROD
              │             │
           patch.yaml    patch.yaml
```

Cela évite de maintenir deux copies complètes de la configuration Kubernetes.

Le même principe est utilisé pour PostgreSQL dans `data-platform/postgres`.

---

# 7. Gestion des secrets avec SOPS + age

Le dépôt contient des secrets Kubernetes qui doivent pouvoir être versionnés avec le reste de l'infrastructure.

Les stocker en clair dans Git serait cependant dangereux.

Le projet utilise donc :

```text
SOPS + age
```

SOPS chiffre les fichiers tandis que `age` fournit le mécanisme de chiffrement utilisé pour protéger les données sensibles.

Les fichiers chiffrés sont conservés avec une extension :

```text
secret.enc.yaml
```

### Générer une clé age

```bash
age-keygen -o age.key
```

Récupérer la clé publique :

```bash
age-keygen -y age.key
```

La **clé privée ne doit jamais être commitée**.

### Chiffrer un secret

```bash
sops --encrypt \
  --age "age1...recipient..." \
  -i applications/portfolio/base/secret.enc.yaml
```

### Déchiffrer

```bash
sops --decrypt \
  applications/portfolio/base/secret.enc.yaml
```

La clé privée est conservée côté cluster et fournie à Flux dans le Secret :

```text
flux-system/sops-age
```

Flux peut alors déchiffrer les manifests pendant la réconciliation.

```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```

Ainsi :

```text
Secret chiffré dans Git
          │
          ▼
        Flux
          │
      SOPS + age
          │
          ▼
Secret Kubernetes
```

La clé privée reste donc **hors du dépôt Git**.

---

# 8. Automatisation des images Docker

Le déploiement GitOps est également connecté au cycle de livraison Docker.

Flux utilise trois ressources :

```text
ImageRepository
       │
       ▼
ImagePolicy
       │
       ▼
ImageUpdateAutomation
```

### ImageRepository

Surveille le registre Docker et détecte les nouveaux tags.

### ImagePolicy

Détermine quelle version peut être déployée.

### ImageUpdateAutomation

Met à jour automatiquement le manifest dans Git lorsqu'une nouvelle version valide est détectée.

Les controllers Flux correspondants sont :

```text
image-reflector-controller
image-automation-controller
```

Ils sont complémentaires :

- `image-reflector-controller` découvre les versions disponibles ;
- `image-automation-controller` applique les changements dans Git.

Le cycle complet devient :

```text
GitHub Actions
      │
      ▼
 DockerHub
      │
      │ nouvelle image
      ▼
ImageRepository
      │
      ▼
 ImagePolicy
      │
      ▼
ImageUpdateAutomation
      │
      │ Git commit
      ▼
    GitHub
      │
      ▼
    FluxCD
      │
      ▼
 Kubernetes
      │
      ▼
 Nouveaux Pods
```

Le déploiement d'une nouvelle image ne nécessite donc plus de commande manuelle sur le serveur.

---

# 9. Ressources principales du dépôt

### Application

```text
applications/portfolio/
```

Contient les manifests Kubernetes du portfolio ainsi que les ressources nécessaires à l'automatisation de ses images.

### Data Platform

```text
data-platform/postgres/
```

Contient PostgreSQL, son `StatefulSet`, son stockage persistant, sa configuration et ses secrets chiffrés.

### Environnements

```text
clusters/dev/
clusters/prod/
```

Définissent les points d'entrée des différents environnements.

### Image Automation

```text
applications/portfolio/base/image-repository.yaml
applications/portfolio/base/image-policy.yaml
applications/portfolio/base/image-update-automation.yaml
```

### Secrets

```text
applications/portfolio/base/secret.enc.yaml
data-platform/postgres/base/secret.enc.yaml
```

---

# 10. Résultat

Ce dépôt permet de gérer le déploiement de la plateforme selon une approche GitOps complète :

```text
                   ┌──────────────┐
                   │    GitHub    │
                   └──────┬───────┘
                          │
                    GitRepository
                          │
                          ▼
                     ┌─────────┐
                     │ FluxCD  │
                     └────┬────┘
                          │
                     Kustomize
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
        Portfolio                PostgreSQL
              │                       │
              └───────────┬───────────┘
                          ▼
                      Kubernetes
```

Avec l'automatisation des images :

```text
Code
 │
 ▼
GitHub Actions
 │
 ▼
DockerHub
 │
 ▼
Flux Image Automation
 │
 ▼
Git
 │
 ▼
FluxCD
 │
 ▼
Kubernetes
```

**Git constitue la source de vérité et Flux assure la convergence du cluster vers l'état déclaré dans le dépôt.**