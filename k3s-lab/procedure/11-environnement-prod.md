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
| Déclencheur | push sur `main` | tag `v*`, par exemple `v1.0.0` | |

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
2. **Deployment branches and tags** : Selected branches and tags, **Add deployment branch or tag rule**. Dans la fenêtre, passe **Ref type** de Branch (choix par défaut) à **Tag**, puis motif `v*`. Une règle `v*` de type Branch bloque tous les tags.
3. **Environment secrets** : `CF_ACCESS_CLIENT_ID` et `CF_ACCESS_CLIENT_SECRET`, avec les mêmes valeurs qu'en test.

Pas d'approbation à ajouter : c'est le tag qui décide de la mise en prod.

> **Vérifier** · La page Environments liste `test` (branche `main`) et `prod` (tag `v*`), chacun avec deux secrets.
{.is-success}

## 11.3 · Workflow des applications

Le workflow `ci-cd.yml` des dépôts de démo, à reprendre dans chaque application :

- construit chaque pull request sans rien publier : c'est le contrôle `build` affiché sur la PR ;
- déploie en test à chaque push sur `main`, avec `deploy/values-test.yaml` ;
- sur un tag `v*`, réétiquette l'image déjà construite pour ce commit et la déploie en prod, avec `deploy/values-prod.yaml`.

Les noms d'hôtes sont dans ces deux fichiers, les namespaces dans `ci-cd.yml`.

> **Vérifier**
>
> 1. Sur une pull request, le contrôle **build** est vert.
> 2. Après la fusion, l'onglet Actions montre les jobs `build` et `deploy-test` en vert. Les jobs `release` et `deploy-prod` sont gris : ils ne tournent que sur un tag.
> 3. Les adresses de test répondent comme à l'étape 10.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 11.4 · Première mise en production

Commence par l'API : le front de prod appelle l'API de prod. Sur GitHub, dans le dépôt : **Releases**, **Draft a new release**, **Choose a tag**, saisis `v1.0.0`, **Create new tag on publish**, cible `main`, puis **Publish release**.

Depuis un clone local, c'est équivalent :

```bash
git checkout main && git pull
git tag v1.0.0 && git push origin v1.0.0
```

Attends que le workflow soit vert, puis fais la même chose dans le dépôt du front.

> **Vérifier**
>
> 1. Le workflow du tag montre `release` et `deploy-prod` en vert, et l'étape **Vérifier l'identité** affiche `github:<propriétaire>/<dépôt>:prod`.
> 2. Dans WSL (k3s-test) :
>
> ```bash
> kubectl -n $NS-prod get deploy
> kubectl -n $NS-prod get deploy api-demo -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
> curl -s https://api-demo.$ZONE/api/demo
> ```
>
> Les déploiements sont en `1/1`, l'image est `ghcr.io/<propriétaire>/api-demo:v1.0.0`, et `curl` renvoie `prod-1` et `prod-2` : la base de prod est bien distincte de celle de test. Dans le navigateur, `https://front-demo.<zone>` affiche ces deux lignes.
{.is-success}

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
| `release` : `Image … introuvable` | le tag porte sur un commit jamais construit sur `main` | supprimer le tag (Releases, ou `git push origin :refs/tags/v1.0.0`), puis taguer un commit passé en test |
| `deploy-prod` refusé : tag non autorisé dans l'environnement | règle `v*` absente de l'environnement `prod`, ou créée en type Branch au lieu de Tag | 11.2 |
| `forbidden … User "github:…:prod"` | droits prod absents du cluster | 11.1 (`make platform`), contrôle `make oidc-test … GH_ENV=prod NS=<ns>-prod` |
| API de prod pas prête | base `app-prod` absente | `make data` |
| Front de prod en erreur 502 sur `/api` | API de prod pas encore déployée | taguer l'API d'abord |
