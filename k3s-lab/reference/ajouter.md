---
title: "Ajouter un service ou une base"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Un ajout = une entrée dans `environments/test.yaml` + un fichier de valeurs. Aucun template à écrire, aucune action dans Cloudflare : le DNS joker couvre déjà le nouveau nom.

**Service déployé par Helmfile** (outil interne, démo) :

1. Le déclarer sous `apps:`, avec `namespace` et `enabled: true`. Optionnel : `subdomain`, sinon c'est le nom du service.
2. Créer `helmfile/apps/<nom>.yaml` : image, port, probes, ressources et `podSecurityContext.runAsUser` (un utilisateur non-root de l'image).
3. `make apps`. Le service répond sur `https://<nom><suffixe>.<zone>`.

**Service déployé par GitHub** (ton application) :

1. Créer le dépôt à partir d'api-demo ou front-demo (dossier `deploy/` et workflow), en adaptant `deploy/values-test.yaml` et `RELEASE`.
2. L'autoriser : `github_oidc_repositories` puis `make bootstrap`, et `tenants.<namespace>.githubOidc` puis `make platform`. Contrôle : `make oidc-test REPO=augustinde/<dépôt>`.
3. Secrets applicatifs, si besoin : `tenants.<namespace>.appSecrets.<nom>-env` via `make secrets-edit`, puis `make platform`, référencés par `envFromSecrets` dans `deploy/values-test.yaml`.
4. Côté GitHub : étapes 0.2, 0.3 et 10.1, puis Run workflow.

**Nouveau namespace** : une entrée sous `tenants:` crée le namespace sécurisé, son quota et son propre compte GitHub (`make deployer-kubeconfig NS=<namespace>`).

**Nouvelle base** :

1. La déclarer sous `databases:`, dans le même namespace que les services qui l'utilisent.
2. Créer `helmfile/databases/<nom>.yaml` à partir de `app.yaml` (taille du volume, seed).
3. `make data`.

Un exemple de déclaration :

```yaml
tenants:
  peps:
    githubDeployer: true
    quota: { requests.memory: 2Gi, limits.memory: 3Gi }
databases:
  peps:
    namespace: peps
    enabled: true
```
