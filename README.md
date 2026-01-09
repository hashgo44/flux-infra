# Flux Infrastructure - Déploiement Automatique avec FluxCD

Ce repository contient la configuration GitOps pour le déploiement automatique des applications sur Kubernetes en utilisant FluxCD.

## 📋 Table des matières

- [Qu'est-ce que FluxCD ?](#quest-ce-que-fluxcd-)
- [Architecture du projet](#architecture-du-projet)
- [Flux de déploiement automatique](#flux-de-déploiement-automatique)
- [Composants Flux utilisés](#composants-flux-utilisés)
- [Structure du projet](#structure-du-projet)
- [Délais et intervalles](#délais-et-intervalles)
- [Comment ça fonctionne](#comment-ça-fonctionne)

---

## Qu'est-ce que FluxCD ?

**FluxCD** est un outil GitOps qui automatise le déploiement d'applications sur Kubernetes. Il surveille votre repository Git et applique automatiquement les changements détectés dans votre cluster Kubernetes.

### Principes GitOps

- **Source de vérité unique** : Le repository Git est la seule source de vérité pour l'état désiré du cluster
- **Déclaratif** : Vous déclarez l'état souhaité, Flux s'occupe de l'appliquer
- **Automatique** : Les changements dans Git sont automatiquement synchronisés avec le cluster
- **Observable** : Flux fournit des événements et des statuts pour chaque opération

---

## Architecture du projet

```
flux-infra/
├── clusters/rde-cluster-cube/   # Configuration du cluster rde-cluster-cube
│   ├── apps.yaml                 # Kustomizations pour les applications
│   ├── sources.yaml              # Kustomizations pour les sources Git
│   ├── infrastructure.yaml       # Infrastructure partagée
│   └── flux-system/              # Configuration Flux (controllers, GitRepository)
│
├── apps/
│   ├── base/                     # Configurations de base (réutilisables)
│   │   ├── cube-backend/
│   │   │   ├── kustomization.yaml
│   │   │   └── release.yaml   
│   │   └── cube-frontend/
│   │       ├── kustomization.yaml
│   │       └── release.yaml
│   │
│   ├── preprod/                  # Overlays pour l'environnement preprod
│   │   ├── cube-backend/
│   │   │   ├── namespace.yaml            # Namespace preprod
│   │   │   ├── image-repository.yaml     # Surveillance du registry
│   │   │   ├── image-policy.yaml         # Politique de sélection d'images
│   │   │   ├── image-update.yaml         # Mise à jour automatique
│   │   │   ├── values.yaml               # Valeurs Helm spécifiques
│   │   │   ├── kustomization.yaml        # Kustomization de l'app
│   │   │   └── release.yaml              # Patch du HelmRelease
│   │   └── cube-frontend/
│   │       ├── namespace.yaml
│   │       ├── image-repository.yaml
│   │       ├── image-policy.yaml
│   │       ├── image-update.yaml
│   │       ├── values.yaml
│   │       ├── kustomization.yaml
│   │       └── release.yaml
│   │
│   ├── production/               # Overlays pour l'environnement production
│   │   ├── cube-backend/
│   │   │   ├── image-repository.yaml    # Surveillance du registry
│   │   │   ├── image-policy.yaml        # Politique de sélection d'images
│   │   │   ├── image-update.yaml        # Mise à jour automatique
│   │   │   ├── values.yaml              # Valeurs Helm spécifiques
│   │   │   ├── kustomization.yaml       # Kustomization de l'app
│   │   │   └── release.yaml             # Patch du HelmRelease
│   │   └── cube-frontend/
│   │       ├── image-repository.yaml
│   │       ├── image-policy.yaml
│   │       ├── image-update.yaml
│   │       ├── values.yaml
│   │       ├── kustomization.yaml
│   │       └── release.yaml
│   │ 
│   └── sources/                  # GitRepositories pour les applications
│       ├── kustomization.yaml
│       ├── cube-backend-repo.yaml
│       └── cube-frontend-repo.yaml
│
├── infrastructure/               # Infrastructure partagée
│   ├── configs/
│   │   └── kustomization.yaml
│   ├── controllers/
│   │   └── kustomization.yaml
│   └── sources/
│       └── kustomization.yaml
│
└── charts/                       # Helm charts personnalisés
    ├── cube-backend/
    │   ├── Chart.yaml
    │   ├── Chart.lock
    │   ├── values.yaml
    │   ├── charts/
    │   │   └── postgresql-15.5.38.tgz
    │   └── templates/
    │       ├── _helpers.tpl
    │       ├── configmap.yaml
    │       ├── deployment.yaml
    │       ├── hpa.yaml
    │       ├── httproute.yaml
    │       ├── ingress.yaml
    │       ├── NOTES.txt
    │       ├── service.yaml
    │       ├── serviceaccount.yaml
    │       └── tests/
    │           └── test-connection.yaml
    └── cube-frontend/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── _helpers.tpl
            ├── deployment.yaml
            ├── hpa.yaml
            ├── httproute.yaml
            ├── ingress.yaml
            ├── NOTES.txt
            ├── service.yaml
            ├── serviceaccount.yaml
            └── tests/
                └── test-connection.yaml
```

---

## Flux de déploiement automatique

### Schéma complet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FLUX DE DÉPLOIEMENT AUTOMATIQUE                      │
└─────────────────────────────────────────────────────────────────────────┘

1️⃣  DÉVELOPPEUR
    │
    ├─> Push code sur cube-backend repository
    │
    └─> CI/CD GitHub Actions
         │
         ├─> Build l'image Docker
         │
         └─> Push image sur GHCR (GitHub Container Registry)
              │
              └─> Tag: main-abc-1234567890
                   │
                   ⏱️  Temps variable (dépend du CI/CD)
                   │

2️⃣  ImageRepository (interval: 1m)
    │
    ├─> Vérifie GHCR toutes les 1 minute
    │
    └─> Détecte la nouvelle image
         │
         ⏱️  Délai max: 1 minute
         │

3️⃣  ImagePolicy
    │
    ├─> Filtre les tags selon pattern: ^main-[a-zA-Z0-9]+-(?P<ts>.*)$
    │
    └─> Sélectionne la version la plus récente (ordre numérique ascendant)
         │

4️⃣  ImageUpdateAutomation (interval: 1m)
    │
    ├─> Met à jour values.yaml avec le nouveau tag
    │
    ├─> Commit dans flux-infra repository
    │
    └─> Push sur branch main
         │
         ⏱️  Délai max: 1 minute
         │

5️⃣  GitRepository flux-system (interval: 1m)
    │
    ├─> Synchronise flux-infra repository toutes les 1 minute
    │
    └─> Récupère le commit avec le nouveau tag
         │
         ⏱️  Délai max: 1 minute
         │

6️⃣  Kustomization cube-backend (interval: 1m0s)
    │
    ├─> Applique les changements au cluster toutes les 1 minute
    │
    ├─> Met à jour le HelmRelease
    │
    └─> Déploie la nouvelle version
         │
         ⏱️  Délai max: 1 minute
         │

7️⃣  HelmRelease (interval: 5m0s)
    │
    ├─> Applique le Helm chart mis à jour
    │
    └─> Rolling update du Deployment
         │

8️⃣  KUBERNETES CLUSTER
    │
    ├─> Rolling update du Deployment
    │
    └─> ✅ Nouvelle version déployée !
```

---

## Composants Flux utilisés

### 1. GitRepository

Définit la source Git à surveiller.

**Fichier** : `clusters/rde-cluster-cube/flux-system/gotk-sync-main.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
spec:
  interval: 1m0s  # Synchronise toutes les 1 minute
  url: https://github.com/hashgo44/flux-infra.git
```

**Rôle** : Surveille le repository `flux-infra` et synchronise les changements.

---

### 2. ImageRepository

Surveille un registry Docker pour détecter de nouvelles images.

**Fichier** : `apps/production/cube-backend/image-repository.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: cube-backend
spec:
  image: ghcr.io/hashgo44/cube-backend
  interval: 1m  # Vérifie toutes les 1 minute
```

**Rôle** : Interroge GHCR toutes les minutes pour détecter de nouvelles images.

---

### 3. ImagePolicy

Définit quelles images sont valides selon un pattern de tags.

**Fichier** : `apps/production/cube-backend/image-policy.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: cube-backend
spec:
  imageRepositoryRef:
    name: cube-backend
  filterTags:
    pattern: '^main-[a-zA-Z0-9]+-(?P<ts>.*)$'  # Filtre les tags
    extract: '$ts'
  policy:
    numerical:
      order: asc  # Sélectionne la version la plus récente
```

**Rôle** : Filtre et sélectionne les images selon un pattern spécifique.

---

### 4. ImageUpdateAutomation

Met à jour automatiquement les fichiers de configuration avec les nouvelles images.

**Fichier** : `apps/production/cube-backend/image-update.yaml`

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: cube-backend
spec:
  interval: 1m  # Vérifie toutes les 1 minute
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: fluxcdbot
      messageTemplate: |
        chore(flux): update cube-backend to {{range .Changed.Changes}}{{println .NewValue}}{{end}}
    push:
      branch: main
  update:
    path: ./apps/production/cube-backend
    strategy: Setters
```

**Rôle** : Met à jour automatiquement `values.yaml` avec le nouveau tag et fait un commit.

---

### 5. Kustomization

Applique les ressources Kubernetes définies dans un répertoire Git.

**Fichier** : `clusters/rde-cluster-cube/apps.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cube-backend
spec:
  interval: 1m0s  # Applique toutes les 1 minute
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/production/cube-backend
  prune: true
  wait: true
```

**Rôle** : Applique les changements détectés dans Git au cluster Kubernetes.

---

### 6. HelmRelease

Gère le déploiement d'un Helm chart.

**Fichier** : `apps/base/cube-backend/release.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cube-backend
spec:
  chart:
    spec:
      chart: ./charts/cube-backend
      sourceRef:
        kind: GitRepository
        name: flux-system
  interval: 5m0s  # Réconcilie toutes les 5 minutes
```

**Rôle** : Déploie et met à jour l'application via Helm.

---

## Structure du projet

### Organisation GitOps

Le projet suit une architecture GitOps avec séparation des préoccupations :

- **`clusters/rde-cluster-cube/`** : Configuration spécifique au cluster rde-cluster-cube
- **`apps/base/`** : Configurations de base réutilisables (DRY principle)
- **`apps/production/`** : Overlays spécifiques à l'environnement production
- **`charts/`** : Helm charts personnalisés pour chaque application

### Pattern Base + Overlay

Chaque application suit le pattern **Base + Overlay** :

1. **Base** (`apps/base/cube-backend/`) : Configuration minimale réutilisable
2. **Overlay** (`apps/production/cube-backend/`) : Personnalisations pour l'environnement

Cela permet de réutiliser la configuration de base pour différents environnements (dev, staging, production).

---

## Délais et intervalles

### Intervalles configurés

| Composant | Intervalle | Rôle |
|-----------|-----------|------|
| **ImageRepository** | `1m` | Détection de nouvelles images |
| **ImageUpdateAutomation** | `1m` | Mise à jour des fichiers Git |
| **GitRepository** | `1m` | Synchronisation du repository |
| **Kustomization** | `1m0s` | Application au cluster |
| **HelmRelease** | `1m0s` | Déploiement Helm |

### Temps de déploiement

- **Temps maximum théorique** : ~4 minutes
  - ImageRepository (1m) + ImageUpdateAutomation (1m) + GitRepository (1m) + Kustomization (1m)
  
- **Temps minimum** : ~1-2 minutes (si synchronisation parfaite)

- **Note** : Tous les composants principaux sont maintenant synchronisés à 1 minute, ce qui permet un déploiement rapide et réactif.

---

## Comment ça fonctionne

### Étape par étape

#### 1. Développeur pousse du code

```bash
git push origin main
```

Le CI/CD (GitHub Actions) build et push l'image Docker sur GHCR avec un tag comme `main-6c8b17e-1767795379`.

#### 2. ImageRepository détecte la nouvelle image

Toutes les minutes, Flux interroge GHCR et détecte le nouveau tag.

#### 3. ImagePolicy filtre et sélectionne

L'ImagePolicy filtre les tags selon le pattern `^main-[a-zA-Z0-9]+-(?P<ts>.*)$` et sélectionne la version la plus récente.

#### 4. ImageUpdateAutomation met à jour Git

L'ImageUpdateAutomation :
- Met à jour `apps/production/cube-backend/values.yaml` avec le nouveau tag
- Fait un commit avec le message : `chore(flux): update cube-backend to main-6c8b17e-1767795379`
- Push sur la branch `main`

#### 5. GitRepository synchronise

Le GitRepository `flux-system` synchronise le repository et récupère le nouveau commit.

#### 6. Kustomization applique les changements

La Kustomization détecte les changements et applique les ressources Kubernetes au cluster.

#### 7. HelmRelease déploie

Le HelmRelease applique le chart Helm mis à jour, ce qui déclenche un rolling update du Deployment.

#### 8. Kubernetes déploie

Kubernetes effectue un rolling update et la nouvelle version est déployée.

---

## Exemple de mise à jour automatique

Quand une nouvelle image est détectée, le fichier `values.yaml` est automatiquement mis à jour :

**Avant** :
```yaml
image:
  tag: "main-6c8b17e-1767795379" # {"$imagepolicy": "flux-system:cube-backend:tag"}
```

**Après** (automatiquement) :
```yaml
image:
  tag: "main-abc123-1767795380" # {"$imagepolicy": "flux-system:cube-backend:tag"}
```

Le commentaire `{"$imagepolicy": "flux-system:cube-backend:tag"}` indique à Flux où mettre à jour le tag automatiquement.

---

## Commandes utiles

### Vérifier le statut des composants Flux

```bash
# Vérifier les ImageRepositories
kubectl get imagerepository -n flux-system

# Vérifier les ImagePolicies
kubectl get imagepolicy -n flux-system

# Vérifier les ImageUpdateAutomations
kubectl get imageupdateautomation -n flux-system

# Vérifier les Kustomizations
kubectl get kustomization -n flux-system

# Vérifier les HelmReleases
kubectl get helmrelease -n flux-system
```

### Voir les événements Flux

```bash
# Événements pour un ImageRepository
kubectl describe imagerepository cube-backend -n flux-system

# Événements pour une Kustomization
kubectl describe kustomization cube-backend -n flux-system
```

### Forcer une réconciliation

```bash
# Forcer la synchronisation d'un GitRepository
flux reconcile source git flux-system

# Forcer la réconciliation d'une Kustomization
flux reconcile kustomization cube-backend

# Forcer la réconciliation d'un HelmRelease
flux reconcile helmrelease cube-backend
```

---

## Avantages de cette architecture

✅ **Automatisation complète** : Aucune intervention manuelle nécessaire  
✅ **Traçabilité** : Tous les déploiements sont trackés dans Git  
✅ **Sécurité** : Le cluster ne peut être modifié que via Git  
✅ **Reproductibilité** : L'état du cluster est toujours reproductible depuis Git  
✅ **Rollback facile** : Revenir en arrière = revert un commit Git  
✅ **Multi-environnements** : Facile d'ajouter dev/staging avec le pattern base+overlay  

---

## Ressources

- [Documentation FluxCD](https://fluxcd.io/docs/)
- [GitOps avec Flux](https://fluxcd.io/docs/get-started/)
- [Image Automation](https://fluxcd.io/docs/guides/image-update/)
- [Helm Controller](https://fluxcd.io/docs/components/helm/)


