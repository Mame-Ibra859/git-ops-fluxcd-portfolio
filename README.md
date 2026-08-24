# Dépôt GitOps — Exemple Flux / Kustomize


Ce dépôt contient une approche GitOps pour déployer une application (`portfolio`) et une plateforme de données (`postgres`). Il utilise Flux (opérateur) et Kustomize avec une structure `base` + `overlays` (`dev` / `prod`) pour séparer environnements et faciliter les revues.

Principes clés
- GitOps : le dépôt est la source de vérité ; Flux applique les manifests depuis Git.
- Opérateur Flux : on installe une instance centralisée (`FluxInstance`) qui déploie et orchestre les controllers nécessaires (source, kustomize, helm, image controllers, notifications).

Installation côté cluster (une seule fois)
1) Installer l'opérateur Flux via Helm :
```bash
helm install flux-operator oci://ghcr.io/controlplaneio/fluxcd/charts/flux-operator \
  --namespace flux-system --create-namespace
```

2) Appliquer la `FluxInstance` (exemple fourni ci-dessous) :
```bash
kubectl apply -f flux-instance.yaml
```

Exemple `flux-instance.yaml` (serveur) :
```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
  annotations:
    fluxcd.controlplane.io/reconcileEvery: "1h"
    fluxcd.controlplane.io/reconcileArtifactEvery: "10m"
    fluxcd.controlplane.io/reconcileTimeout: "5m"
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

3) Déclarer le `GitRepository` (pointant vers ce dépôt) pour que Flux lise la source Git :
```bash
kubectl apply -f git-repo.yaml
```

Exemple `git-repo.yaml` :
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

Flux Kustomization côté cluster (exemple)
- Un `Kustomization` Flux référence le `GitRepository` et le chemin dans le dépôt :
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: portfolio-dev
  namespace: flux-system
spec:
  interval: 5m
  path: ./applications/portfolio/overlays/dev
  prune: true
  sourceRef:
    kind: GitRepository
    name: portfolio
```

Connexion entre parties
- L'opérateur Flux (installé via Helm ou `flux install`) crée un namespace `flux-system` avec les controllers listés dans `FluxInstance`.
- Le `GitRepository` (source) pointe vers ce dépôt. Les `Kustomization` lisent des chemins (`path`) dans le repo et appliquent les manifests sur le cluster.

Gestion des images — pourquoi `image-reflector-controller` et `image-automation-controller` sont nécessaires
- `image-reflector-controller` : scanne les registries et synchronise les métadonnées d'images dans le cluster.
- `image-automation-controller` : observe les policies et met à jour les manifests (kustomize, helm, etc.) en créant des commits/MRs ou en modifiant directement les ressources, déclenchant ainsi un déploiement.
- Les deux controllers sont complémentaires : sans le reflecteur, l'automatisation ne reçoit pas l'inventaire des tags ; sans l'automation, rien ne pousse les mises à jour dans Git.

Exemples dans ce dépôt
- Overlays : [applications/portfolio/overlays/dev/kustomization.yaml](applications/portfolio/overlays/dev/kustomization.yaml)
- Base secrets chiffrés : [applications/portfolio/base/secret.enc.yaml](applications/portfolio/base/secret.enc.yaml)
- Image automation :
  - [applications/portfolio/base/image-repository.yaml](applications/portfolio/base/image-repository.yaml)
  - [applications/portfolio/base/image-policy.yaml](applications/portfolio/base/image-policy.yaml)
  - [applications/portfolio/base/image-update-automation.yaml](applications/portfolio/base/image-update-automation.yaml)

SOPS + age (rappel rapide)
```bash
# générer une paire age (clé privée locale)
age-keygen -o ./sops/age.key

# valeur publique à partager comme recipient
age-keygen -y ./sops/age.key

# chiffrer
sops --encrypt --age "age1...recipient..." -i applications/portfolio/base/secret.enc.yaml

# déchiffrer
sops --decrypt applications/portfolio/base/secret.enc.yaml > secret.yaml
```
Gardez la clé privée hors dépôt.

Conclusion
- En pratique : installer l'opérateur Flux (Helm ou `flux install`), appliquer `FluxInstance` et `GitRepository`, puis créer un `Kustomization` côté cluster pointant vers `./applications/...` dans ce dépôt. Les image controllers (`image-reflector` + `image-automation`) permettent l'automatisation sûre des mises à jour d'images tandis que `sops+age` protège les secrets dans Git.

Fichiers modifiables / prochaines étapes
- Je peux : 1) ajouter les fichiers `flux-instance.yaml` et `git-repo.yaml` au dossier `clusters/*` si vous le souhaitez, 2) créer un commit + PR.

