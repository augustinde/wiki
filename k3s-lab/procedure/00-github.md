---
title: "Étape 0 · GitHub et applications de démo"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Deux applications servent à valider toute la chaîne : [api-demo](https://github.com/augustinde/api-demo) (Spring Boot 4.1, `GET /api/demo` lit la table `demo` de PostgreSQL) et [front-demo](https://github.com/augustinde/front-demo) (Angular 22, un écran qui affiche ces données). À chaque push sur `main`, GitHub construit l'image, la publie sur GHCR et la déploie dans le namespace `demo`, authentifié par OIDC.

Trois réglages se font avant le cluster. Ceux qui ont besoin d'un cluster en marche (variables, premier déploiement) sont à l'étape 10.

| # | Action | Quand |
| --- | --- | --- |
| 0.1 | Pousser les deux dépôts | Fait |
| 0.2 | Rendre les paquets GHCR publics | Après le premier build |
| 0.3 | Environnement `test` et secrets Cloudflare | Maintenant (jeton : Mise en route, 4.6) |
| 10 | Variables GitHub, puis premier déploiement | Après l'étape 9 |
| 11 | Environnement prod, puis premier tag de version | Après l'étape 10 |

## 0.1 · Pousser les deux dépôts

Fait : [api-demo](https://github.com/augustinde/api-demo) et [front-demo](https://github.com/augustinde/front-demo) sont sur GitHub, branche `main`.

> **Vérifier** · Dans l'onglet **Actions** de chaque dépôt, le workflow **CI/CD** a un job `build` vert et un job `deploy` gris (ignoré). C'est normal tant que la variable `K8S_API_HOST` n'existe pas.
{.is-success}

## 0.2 · Rendre les paquets GHCR publics

Après le premier build, les images apparaissent sur github.com/augustinde?tab=packages, en privé par défaut. Le cluster doit pouvoir les télécharger.

1. Ouvre le paquet **api-demo**, puis **Package settings** (à droite).
2. En bas, **Danger Zone**, **Change visibility**, **Public**, et confirme en tapant le nom.
3. Recommence pour **front-demo**.

Ces images ne contiennent aucun secret : les identifiants de la base sont injectés au démarrage par Kubernetes. Si tu préfères les garder privées : PAT GitHub classique avec le seul scope `read:packages`, déclaré dans `tenants.demo.registry` du fichier de secrets de k3s-lab, et `imagePullSecrets: [{name: ghcr-pull}]` dans le `deploy/values-test.yaml` de chaque dépôt.

> **Vérifier** · Sur github.com/augustinde?tab=packages, les deux paquets affichent **Public**. En navigation privée, sans être connecté, la page du paquet s'ouvre.
{.is-success}

## 0.3 · Environnement `test` et secrets Cloudflare

Dans **chacun** des deux dépôts :

1. Settings, **Environments**, **New environment**, nom `test`.
2. **Deployment branches and tags** : *Selected branches and tags*, ajoute la règle `main`. Seule la branche principale pourra déployer.
3. **Environment secrets**, **Add environment secret** :

| Secret | Valeur |
| --- | --- |
| `CF_ACCESS_CLIENT_ID` | Client ID du jeton de service Cloudflare `github-deploy` |
| `CF_ACCESS_CLIENT_SECRET` | Client Secret du même jeton |

Le jeton se crée dans Cloudflare ([Mise en route, 4.6](/k3s-lab/mise-en-route/04-cloudflare)). Ce sont les seuls secrets stockés chez GitHub, et ils ne donnent aucun droit dans le cluster : ils ouvrent seulement le chemin réseau vers l'API.

> **Vérifier** · Dans chaque dépôt, Settings, Environments, `test` affiche la règle de branche `main` et les deux secrets.
{.is-success}
