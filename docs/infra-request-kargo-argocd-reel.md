# Demande infra : installer Kargo sur le cluster Argo CD réel

Statut : brouillon à valider avant envoi à l'équipe plateforme. Aucune ressource n'a été
créée sur l'infra réelle — ce document sert uniquement de base à la demande.

## Pourquoi ce n'est pas juste "brancher le kind de la démo"

Kargo ne parle pas à l'API HTTP d'Argo CD avec un token : il pilote directement les CRDs
`Application` (`argoproj.io`) via l'API Kubernetes du cluster où il tourne. Le step de
promotion `argocd-update` n'a pas de champ `cluster` pour cibler une `Application` sur un
autre cluster (vérifié sur la doc officielle Kargo v1.11, cf. sources en bas de page).

Conséquence : pour que Kargo pilote *réellement* l'Argo CD de la boîte, **Kargo doit
tourner sur le même cluster qu'Argo CD** — ce n'est pas un paramètre de connexion depuis
mon `kind` local, c'est une installation à part entière sur l'infra réelle.

## Ce qui est demandé

1. **Namespace dédié** pour Kargo, isolé des Applications Argo CD des tenants en prod
   (ex. `kargo-demo` ou `kargo-system`, à définir avec l'infra).
2. **AppProject Argo CD dédié** (ou équivalent) restreignant les `Application` que Kargo
   peut voir/modifier à un périmètre de test, **sans accès aux `Application` des 5 tenants
   réels** (safti-fr/es/de/pt, megagence-fr) tant que la démo n'est pas validée en revue.
3. **ServiceAccount + Role/RoleBinding scopés** (pas de `cluster-admin`), limités à :
   - `get/list/watch/create/update/patch` sur `applications.argoproj.io`, uniquement dans
     le namespace/AppProject dédié ci-dessus.
   - Pas d'accès aux autres ressources du cluster.
4. **Installation Kargo** (Helm chart officiel `kargo`) dans ce namespace, configurée pour
   surveiller uniquement ce namespace Argo CD dédié (option de restriction de namespace —
   voir *Common Configurations* dans les sources).

## Installation via Terraform

Pas de provider Terraform dédié pour du Kargo self-hosted (le provider officiel
`akuity/akp` avec `akp_kargo_instance` gère une instance **Akuity Platform managée**,
sans rapport avec notre install self-hosted). Le chemin est le provider Terraform Helm
générique (`hashicorp/helm`), avec un `helm_release` pointant sur le chart officiel
`oci://ghcr.io/akuity/kargo-charts/kargo`.

Squelette de module (à valider/compléter par l'infra, non appliqué) :

```hcl
resource "kubernetes_namespace" "kargo" {
  metadata {
    name = "kargo-demo" # cf. namespace dédié demandé plus haut
  }
}

resource "kubernetes_secret" "kargo_admin" {
  metadata {
    name      = "kargo-admin-secret"
    namespace = kubernetes_namespace.kargo.metadata[0].name
  }
  data = {
    ADMIN_ACCOUNT_PASSWORD_HASH = var.kargo_admin_password_hash # depuis vault, pas en dur
    ADMIN_ACCOUNT_TOKEN_SIGNING_KEY = var.kargo_token_signing_key
  }
}

resource "helm_release" "kargo" {
  name       = "kargo"
  repository = "oci://ghcr.io/akuity/kargo-charts"
  chart      = "kargo"
  namespace  = kubernetes_namespace.kargo.metadata[0].name

  set {
    name  = "crds.keep"
    value = "true" # ne jamais supprimer les CRDs Stage/Warehouse/Project au destroy
  }
  set {
    name  = "api.secret.name"
    value = kubernetes_secret.kargo_admin.metadata[0].name # secret précréé, pas généré par Helm
  }

  depends_on = [kubernetes_secret.kargo_admin] # + module cert-manager si non déjà présent sur le cluster
}
```

Points à valider avec l'infra avant tout `apply` (rappel : pas de `terraform apply`/`destroy`
exécuté depuis ce poste — squelette fourni pour revue uniquement) :

- **`crds.keep = true`** impératif : sans ça, un `destroy` de cette ressource supprime les
  CRDs `Stage`/`Warehouse`/`Project` **pour tout le cluster**, pas juste ce namespace.
- **Secret admin** précréé hors Helm (via vault/sealed-secret côté infra), jamais en dur
  dans les `values` ou le state Terraform en clair.
- **cert-manager** doit déjà être installé sur le cluster cible (dépendance de module) —
  sinon passer les flags `*.selfSignedCert` à `false` et fournir des Secrets TLS pré-créés.
- **Secrets OIDC / webhook receivers** : hors périmètre du chart, à gérer par un module
  séparé si besoin plus tard.

## Explicitement hors périmètre de cette demande

- Aucun accès en écriture aux `Application` des tenants réels en prod/preprod.
- Aucun droit `cluster-admin` ni accès aux Secrets/ConfigMaps hors du périmètre défini.
- Pas de modification du process de déploiement existant (GitLab CI reste responsable du
  build, comme documenté dans [`kargo-workflow-summary.md`](./kargo-workflow-summary.md)).

## Sources vérifiées (Kargo v1.11.x)

- Schéma `argocd-update` (pas de champ `cluster`) :
  https://docs.kargo.io/user-guide/reference-docs/promotion-steps/argocd-update
- Namespace Argo CD par défaut / restriction de namespace :
  https://docs.kargo.io/operator-guide/advanced-installation/common-configurations
- `ClusterConfig` (config cluster-wide de Kargo, sans lien avec un Argo CD distant) :
  https://docs.kargo.io/operator-guide/cluster-configuration
- Discussion (non implémentée) sur le support d'un Argo CD distant :
  https://github.com/akuity/kargo/issues/964
- Chart Helm officiel Kargo (CRDs, secret admin, cert-manager) :
  https://github.com/akuity/kargo/blob/main/charts/kargo/README.md
- Provider Terraform `akuity/akp` (Akuity Platform managé, hors périmètre self-hosted) :
  https://registry.terraform.io/providers/akuity/akp/latest/docs
