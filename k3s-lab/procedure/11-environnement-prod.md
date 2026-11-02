---
title: "Étape 11 · Environnement prod"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

La prod tourne sur le même VPS, dans son propre namespace, avec sa propre base. Elle ne se déploie que depuis un tag de version, et toujours avec l'image déjà validée en test : un tag ne reconstruit rien.

|  | test | prod |
| --- | --- | --- |
| Namespace | `demo` | `demo-prod` |
| Base PostgreSQL | `app` sur `10.43.0.100` | `app-prod` sur `10.43.0.101` |
| Adresses | `api-demo-test.augustinde.dev`, `front-demo-test.augustinde.dev` | `api-demo.augustinde.dev`, `front-demo.augustinde.dev` |
| Identité GitHub | `github:augustinde/<dépôt>:test` | `github:augustinde/<dépôt>:prod` |
| Déclencheur | push sur `main` | tag `v*`, par exemple `v1.0.0` |

## 11.1 · Appliquer k3s-lab

Sur GitHub, fusionne la [PR k3s-lab #2](https://github.com/augustinde/k3s-lab/pull/2). Elle ajoute aussi les alertes (étape 12) et Renovate (étape 14), inactifs tant que tu ne les configures pas.

```bash
cd ~/k3s-lab && git pull
make platform
make data
```

`make platform` crée le namespace `demo-prod` ; `make data` crée la base `app-prod` et attend qu'elle soit prête.

> **Vérifier**
>
> ```bash
> kubectl get ns demo-prod -o jsonpath='{.metadata.labels}'; echo
> kubectl -n demo-prod get clusters.postgresql.cnpg.io
> kubectl -n demo-prod get svc app-prod-admin
> make oidc-test REPO=augustinde/api-demo GH_ENV=prod NS=demo-prod
> make oidc-test REPO=augustinde/api-demo GH_ENV=prod NS=demo
> ```
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Namespace | `pod-security.kubernetes.io/enforce":"restricted` |
> | Base | `app-prod` en `Cluster in healthy state` |
> | Accès IntelliJ | `CLUSTER-IP 10.43.0.101` |
> | Identité prod dans `demo-prod` | déployer : `yes` |
> | Identité prod dans `demo` | déployer : `no` (la prod ne peut pas toucher au test) |
>
> Refais le contrôle avec `REPO=augustinde/front-demo`.
{.is-success}

## 11.2 · Créer l'environnement `prod` sur GitHub

Dans **chacun** des deux dépôts :

1. Settings, **Environments**, **New environment**, nom `prod`.
2. **Deployment branches and tags** : Selected branches and tags, **Add deployment branch or tag rule**. Dans la fenêtre, passe **Ref type** de Branch (choix par défaut) à **Tag**, puis motif `v*`. Une règle `v*` de type Branch bloque tous les tags.
3. **Environment secrets** : `CF_ACCESS_CLIENT_ID` et `CF_ACCESS_CLIENT_SECRET`, avec les mêmes valeurs qu'en test.

Pas d'approbation à ajouter : c'est le tag qui décide de la mise en prod.

> **Vérifier** · La page Environments liste `test` (branche `main`) et `prod` (tag `v*`), chacun avec deux secrets.
{.is-success}

## 11.3 · Fusionner les PR des applications

Fusionne [api-demo #1](https://github.com/augustinde/api-demo/pull/1), puis [front-demo #1](https://github.com/augustinde/front-demo/pull/1). Le nouveau workflow :

- construit chaque pull request sans rien publier : c'est le contrôle `build` affiché sur la PR ;
- déploie en test à chaque push sur `main` ;
- prend les noms d'hôtes dans `deploy/values-test.yaml` et `deploy/values-prod.yaml`. La variable `APP_HOST` ne sert plus, tu peux la supprimer.

> **Vérifier**
>
> 1. Avant la fusion, la PR affiche le contrôle **build** en vert.
> 2. Après la fusion, l'onglet Actions montre les jobs `build` et `deploy-test` en vert. Les jobs `release` et `deploy-prod` sont gris : ils ne tournent que sur un tag.
> 3. `https://api-demo-test.augustinde.dev/api/demo` et `https://front-demo-test.augustinde.dev` répondent comme avant.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 11.4 · Première mise en production

Commence par api-demo : le front de prod appelle l'API de prod. Sur GitHub, dans le dépôt : **Releases**, **Draft a new release**, **Choose a tag**, saisis `v1.0.0`, **Create new tag on publish**, cible `main`, puis **Publish release**.

Depuis un clone local, c'est équivalent :

```bash
git checkout main && git pull
git tag v1.0.0 && git push origin v1.0.0
```

Attends que le workflow soit vert, puis fais la même chose dans front-demo.

> **Vérifier**
>
> 1. Le workflow du tag montre `release` et `deploy-prod` en vert, et l'étape **Vérifier l'identité** affiche `github:augustinde/api-demo:prod`.
> 2. Dans WSL :
>
> ```bash
> kubectl -n demo-prod get deploy
> kubectl -n demo-prod get deploy api-demo -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
> curl -s https://api-demo.augustinde.dev/api/demo
> ```
>
> Les deux déploiements sont en `1/1`, l'image est `ghcr.io/augustinde/api-demo:v1.0.0`, et `curl` renvoie `prod-1` et `prod-2` : la base de prod est bien distincte de celle de test. Dans le navigateur, `https://front-demo.augustinde.dev` affiche ces deux lignes.
{.is-success}

## 11.5 · Base prod depuis IntelliJ

```bash
make db-creds DB=app-prod NS=demo-prod
```

Dans IntelliJ, une nouvelle source PostgreSQL : host `10.43.0.101`, port `5432`, base `app`, utilisateur `app`, mot de passe affiché. Nomme-la clairement, par exemple « k3s PROD » : la confusion entre test et prod est l'erreur classique.

> **Vérifier** · WARP connecté, `Test-NetConnection 10.43.0.101 -Port 5432` répond `True` dans PowerShell, et **Test Connection** répond `Succeeded`.
{.is-success}

## Si la mise en prod échoue

| Symptôme | Cause | Correctif |
| --- | --- | --- |
| `release` : `Image … introuvable` | le tag porte sur un commit jamais construit sur `main` | supprimer le tag (Releases, ou `git push origin :refs/tags/v1.0.0`), puis taguer un commit passé en test |
| `deploy-prod` refusé : tag non autorisé dans l'environnement | règle `v*` absente de l'environnement `prod`, ou créée en type Branch au lieu de Tag | 11.2 |
| `forbidden … User "github:…:prod"` | droits prod absents du cluster | 11.1 (`make platform`), contrôle `make oidc-test … GH_ENV=prod NS=demo-prod` |
| API de prod pas prête | base `app-prod` absente | `make data` |
| Front de prod en erreur 502 sur `/api` | API de prod pas encore déployée | taguer api-demo d'abord |
