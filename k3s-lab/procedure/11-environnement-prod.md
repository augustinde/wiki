---
title: "Étape 11 · Environnement prod"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

La prod tourne sur le même serveur, dans son propre namespace, avec sa propre base. Elle ne se déploie que depuis un tag de version, et toujours avec l'image déjà validée en test : un tag ne reconstruit rien.

|  | test | prod | k3s-test |
| --- | --- | --- | --- |
| Namespace | `<ns>` | `<ns>-prod` | `demo`, `demo-prod` |
| Base PostgreSQL | `app`, 5ᵉ adresse de `<route>` | `app-prod`, adresse suivante | `10.43.0.100`, `10.43.0.101` |
| Adresses | `<app><suffixe>.<zone>` | `<app>.<zone>` | `api-demo-test.augustinde.dev`, `api-demo.augustinde.dev` |
| Identité GitHub | `github:<propriétaire>/<dépôt>:test` | `github:<propriétaire>/<dépôt>:prod` | |
| Lancement | Actions, **Déployer** (test) | Actions, **Release** (patch, minor ou major) | |

Le suffixe `-prod` du namespace étiquette ses alertes 🔴 PROD (étape 12).

## 11.1 · Déclarer la prod dans le projet

Dans `$ENV_DIR/values.yaml`, ajoute le namespace et la base de prod :

```yaml
tenants:
  <ns>-prod:
    githubDeployer: false
    githubOidc:
      - repository: <propriétaire>/<dépôt>
        environment: prod
    quota:
      requests.cpu: "1"
      requests.memory: 2Gi
      limits.memory: 3Gi
      pods: "15"
      persistentvolumeclaims: "3"
      requests.storage: 10Gi

databases:
  app-prod:
    namespace: <ns>-prod
    enabled: true
```

Puis crée la base à partir de celle de test, avec l'adresse suivante de la route :

```bash
cp $ENV_DIR/databases/app.yaml $ENV_DIR/databases/app-prod.yaml
nano $ENV_DIR/databases/app-prod.yaml      # adminAccess.clusterIP : adresse suivante ; seed de prod
make config-check
make platform
make data
git add $ENV_DIR && git commit -m "Environnement prod" && git push
```

`make platform` crée le namespace `<ns>-prod` ; `make data` crée la base `app-prod` et attend qu'elle soit prête. Si un dépôt n'était déclaré nulle part avant, lance aussi `make bootstrap` pour que l'API l'accepte.

> **Vérifier**
>
> ```bash
> kubectl get ns $NS-prod -o jsonpath='{.metadata.labels}'; echo
> kubectl -n $NS-prod get clusters.postgresql.cnpg.io
> kubectl -n $NS-prod get svc app-prod-admin
> make oidc-test REPO=<propriétaire>/<dépôt> GH_ENV=prod NS=$NS-prod
> make oidc-test REPO=<propriétaire>/<dépôt> GH_ENV=prod NS=$NS
> ```
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Namespace | `pod-security.kubernetes.io/enforce":"restricted` |
> | Base | `app-prod` en `Cluster in healthy state` |
> | Accès IntelliJ | `CLUSTER-IP` = l'adresse choisie dans `app-prod.yaml` |
> | Identité prod dans `<ns>-prod` | déployer : `yes` |
> | Identité prod dans `<ns>` | déployer : `no` (la prod ne peut pas toucher au test) |
>
> Refais le contrôle pour chaque dépôt.
{.is-success}

## 11.2 · Créer l'environnement `prod` sur GitHub

Dans **chaque** dépôt d'application :

1. Settings, **Environments**, **New environment**, nom `prod`.
2. **Deployment branches and tags** : Selected branches and tags, **Add deployment branch or tag rule**, Ref type **Branch**, motif `main`. Les workflows **Release** et **Déployer** se lancent depuis `main` : sans cette règle, GitHub refuse le déploiement. Une ancienne règle de type Tag `v*` peut rester, elle ne gêne pas.
3. **Required reviewers** : coche-le et ajoute-toi. Chaque mise en prod attend alors ta validation dans Actions (bouton **Review deployments**), ce qui évite de livrer en prod par un mauvais clic.
4. **Environment secrets** : `CF_ACCESS_CLIENT_ID` et `CF_ACCESS_CLIENT_SECRET`, avec les mêmes valeurs qu'en test.

> **Vérifier** · La page Environments liste `test` (branche `main`) et `prod` (branche `main`, une protection), chacun avec deux secrets.
{.is-success}

## 11.3 · Workflows des applications

Aucun déploiement n'est automatique. Les dépôts de démo ont trois workflows, à reprendre dans chaque application :

| Workflow | Lancement | Effet |
| --- | --- | --- |
| **CI** (`ci-cd.yml`) | pull request, push sur `main` | build de contrôle sur les PR ; sur `main`, image `sha-<commit>` publiée, rien n'est déployé |
| **Déployer** (`deploy-version.yml`) | Actions, **Run workflow** : environnement et version | déploie une image existante. Test : version vide = dernier commit de `main`, ou `vX.Y.Z`, ou `sha-xxxxxxx`. Prod : `vX.Y.Z` seulement, ce qui sert aussi de retour arrière |
| **Release** (`release.yml`) | Actions, **Run workflow** : `patch`, `minor` ou `major`, commit facultatif | calcule la version suivante depuis le dernier tag (1.4.2 → 1.4.3, 1.5.0 ou 2.0.0), vérifie que l'image du commit existe, la réétiquette sans reconstruction, crée le tag et la Release GitHub (notes générées depuis la version précédente), puis déploie en prod |

Les noms d'hôtes sont dans `deploy/values-test.yaml` et `deploy/values-prod.yaml`, les namespaces dans `deploy-version.yml` et `release.yml`.

> **Vérifier**
>
> 1. Sur une pull request, le contrôle **build** est vert.
> 2. Après une fusion sur `main`, le workflow **CI** est vert et rien n'est déployé : l'onglet Actions ne montre aucun job de déploiement.
> 3. **Déployer**, environnement `test`, version vide : le job `deploy` est vert, et les adresses de test répondent comme à l'étape 10.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 11.4 · Première mise en production

Commence par l'API : le front de prod appelle l'API de prod. Dans le dépôt : **Actions**, **Release**, **Run workflow**, branche `main`, type `minor` (ou `patch`), commit vide, puis **Run workflow**. Le commit livré est le dernier de `main`, celui que tu as déployé en test à l'étape 11.3.

Si **Required reviewers** est actif, le job `deploy-prod` attend : **Review deployments**, coche `prod`, **Approve and deploy**. Attends que le workflow soit vert, puis fais la même chose dans le dépôt du front.

> **Vérifier**
>
> 1. Le résumé du workflow affiche la version calculée (par exemple `v1.0.0 → v1.1.0 (minor)`), les jobs `release` et `deploy-prod` sont verts, et l'étape **Vérifier l'identité** affiche `github:<propriétaire>/<dépôt>:prod`.
> 2. L'onglet **Releases** du dépôt montre la nouvelle version, avec la liste des changements depuis la précédente.
> 3. Dans WSL (k3s-test) :
>
> ```bash
> kubectl -n $NS-prod get deploy
> kubectl -n $NS-prod get deploy api-demo -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
> curl -s https://api-demo.$ZONE/api/demo
> ```
>
> Les déploiements sont en `1/1`, l'image est `ghcr.io/<propriétaire>/api-demo:<version>`, et `curl` renvoie `prod-1` et `prod-2` : la base de prod est bien distincte de celle de test. Dans le navigateur, `https://front-demo.<zone>` affiche ces deux lignes.
{.is-success}

**Revenir à la version précédente** : **Actions**, **Déployer**, environnement `prod`, version `vX.Y.Z` précédente (onglet Releases). Aucune nouvelle version n'est créée.

## 11.5 · Base prod depuis IntelliJ

```bash
make db-creds DB=app-prod NS=$NS-prod
```

Dans IntelliJ, une nouvelle source PostgreSQL : host = IP affichée, port `5432`, base `app`, utilisateur `app`, mot de passe affiché. Nomme-la clairement, par exemple « <cluster> PROD » : la confusion entre test et prod est l'erreur classique.

> **Vérifier** · WARP connecté, `Test-NetConnection <ip> -Port 5432` répond `True` dans PowerShell, et **Test Connection** répond `Succeeded`.
{.is-success}

## Si la mise en prod échoue

| Symptôme | Cause | Correctif |
| --- | --- | --- |
| `release` : `Image … introuvable` | le build de ce commit sur `main` n'est pas terminé, ou a échoué | attendre le workflow **CI** de ce commit, puis relancer Release |
| `release` : `déjà publié en vX.Y.Z` | ce commit a déjà une version | rien à livrer ; pour la redéployer : **Déployer**, `prod`, cette version |
| `release` : `Lance la release depuis la branche main` | Run workflow lancé depuis une autre branche | relancer avec **Use workflow from : main** |
| `deploy-prod` refusé : branche non autorisée dans l'environnement | règle `main` absente de l'environnement `prod` | 11.2 |
| `deploy-prod` en attente | validation requise (Required reviewers) | **Review deployments**, **Approve and deploy** |
| `forbidden … User "github:…:prod"` | droits prod absents du cluster | 11.1 (`make platform`), contrôle `make oidc-test … GH_ENV=prod NS=<ns>-prod` |
| API de prod pas prête | base `app-prod` absente | `make data` |
| Front de prod en erreur 502 sur `/api` | API de prod pas encore déployée | livrer l'API d'abord |

Une release ratée après la création du tag (déploiement refusé, par exemple) ne se relance pas : corrige la cause, puis **Déployer**, `prod`, la version créée.
