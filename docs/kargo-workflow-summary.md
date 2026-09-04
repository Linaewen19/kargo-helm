# Résumé du workflow Kargo (démo équipe)

## Architecture

1 Project Kargo (`kargo-helm`) couvrant 5 tenants x 2 environnements = 10 Stages :

```
preprod-safti-fr      prod-safti-fr
preprod-safti-es      prod-safti-es
preprod-safti-de      prod-safti-de
preprod-safti-pt      prod-safti-pt
preprod-megagence-fr  prod-megagence-fr
```

Pas de Stage `dev` — couvert par les review apps GitLab CI existantes, hors périmètre Kargo.
Une seule équipe gère tous les tenants -> un seul Project, noms de Stage suffixés par tenant
(plutôt qu'un Project Kargo par tenant, qui aurait permis un RBAC différencié par marché).

## Détection (Warehouses)

10 Warehouses image, une par (tenant x env), chacune filtrée par regex de tag :

```yaml
guestbook-preprod-safti-fr  -> surveille ghcr.io/.../guestbook, tags matching ^preprod-safti-fr-
guestbook-prod-safti-fr     -> surveille ghcr.io/.../guestbook, tags matching ^prod-safti-fr-
```

Aucune promotion d'image entre Stages (`sources: direct: true` partout) — chaque env/tenant
reçoit sa propre image déjà construite par la CI, fidèle au modèle réel de `Code/api`
(build per-env-per-tenant à cause du `cache:warmup` Symfony qui bake `TENANT`+`APP_ENV`
dans le DI container compilé — voir `docker/php/web.Dockerfile`).

## Promotion (PromotionTask `promote`, partagé par les 10 Stages)

```
1. git-clone      -> checkout main
2. yaml-update    -> bump image.tag dans env/<stage>/values.yaml
3. git-commit     -> commit local
4. git-push       -> push sur une branche générée (kargo/promotion/...)
5. git-open-pr    -> ouvre une PR vers main (skip proprement si rien à committer,
                     ex: retry d'une promotion déjà mergée)
6. git-wait-for-pr -> bloque tant que la PR n'est pas mergée (seulement si une PR existe)
7. argocd-update  -> déclenche le sync Argo CD de kargo-helm-<stage>
```

Le gate humain (merge de la PR sur GitHub) reproduit le process de MR actuel de l'équipe.

## Correspondance avec le process réel (`Code/api`, GitLab CI)

| Chez vous (repo `api`)                       | Dans la démo                                  |
|-----------------------------------------------|------------------------------------------------|
| CI build `TENANT`+`APP_ENV` -> image ECR       | `docker buildx imagetools create` (simulé)      |
| MR manuelle vers branche preprod/prod          | PR ouverte automatiquement par Kargo, mergée manuellement |
| Déploiement Helm déclenché par CI              | `argocd-update` déclenché après merge du PR     |

Le build reste à 100% la responsabilité de la CI dans les deux cas — Kargo ne build jamais
d'image, il détecte des images déjà publiées et orchestre leur promotion/déploiement.

## Volontairement laissé de côté pour cette démo

- **Stage `dev`** — géré par les review apps GitLab (env éphémère par branche/MR),
  incompatible avec le modèle de Stage persistante de Kargo.
- **Promotion de feature flags** (config qui cascade indépendamment de l'image,
  via une Warehouse `git` séparée) — retirée pour alléger visuellement la démo à
  10 stages. Le mécanisme (Warehouse `features`, steps conditionnels dans le
  PromotionTask) existe dans l'historique Git du repo si besoin de le remontrer
  séparément à l'équipe.

## Environnement de test

Testé sur un cluster `kind` local jetable (`kind-kargo-quickstart`), avec Argo CD +
Kargo installés via le script quickstart officiel. Fork de démo :
`github.com/linaewen19/kargo-helm`.

⚠️ `argocd/appset.yaml` et `argocd/appproj.yaml` ne sont pas auto-synchronisés par
Argo CD dans ce setup (pas d'app-of-apps) — après une modification, il faut les
réappliquer manuellement : `kubectl apply -f argocd/`. À l'inverse, `kargo/` se
réapplique via `kargo apply -f ./kargo`.
