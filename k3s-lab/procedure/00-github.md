---
title: "Étape 0 · GitHub et dépôts d'applications"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Les applications d'un projet se déploient depuis leur propre dépôt GitHub. À chaque push sur `main`, GitHub construit l'image, la publie sur GHCR et la déploie dans le namespace de test, authentifié par OIDC ; un tag `v*` déploie la même image en prod (étape 11).

Dans le projet k3s-test, ce sont [api-demo](https://github.com/augustinde/api-demo) (Spring Boot, `GET /api/demo` lit la table `demo` de PostgreSQL) et [front-demo](https://github.com/augustinde/front-demo) (Angular, un écran qui affiche ces données) : ils servent d'exemples dans les étapes 0, 10 et 11. Pour une nouvelle application, pars d'une copie de l'un d'eux et adapte :

| Fichier | À adapter |
| --- | --- |
| `.github/workflows/deploy.yml` | `K8S_AUDIENCE` = `<cluster>` du projet |
| `.github/workflows/ci-cd.yml` | les namespaces de test et de prod, ceux déclarés sous `tenants` |
| `deploy/values-test.yaml`, `deploy/values-prod.yaml` | image, port, sondes, noms d'hôte `<app><suffixe>.<zone>` et `<app>.<zone>` |

Trois réglages se font avant le cluster. Ceux qui ont besoin d'un cluster en marche (variables, premier déploiement) sont à l'étape 10.

| # | Action | Quand |
| --- | --- | --- |
| 0.1 | Pousser les dépôts d'applications | Avant le cluster |
| 0.2 | Rendre les paquets GHCR publics | Après le premier build |
| 0.3 | Environnement `test` et secrets Cloudflare | Avant le cluster (jeton : Mise en route, 4.6) |
| 10 | Variables GitHub, puis premier déploiement | Après l'étape 9 |
| 11 | Environnement prod, puis premier tag de version | Après l'étape 10 |

## 0.1 · Pousser les dépôts d'applications

Chaque dépôt est sur GitHub, branche `main`, et déclaré dans `environments/<env>/values.yaml` sous `tenants.<ns>.githubOidc` ([Mise en route, 3.5](/k3s-lab/mise-en-route/03-cles-depot)).

> **Vérifier** · Dans l'onglet **Actions** de chaque dépôt, le workflow **CI/CD** a un job `build` vert et un job `deploy` gris (ignoré). C'est normal tant que la variable `K8S_API_HOST` n'existe pas.
{.is-success}

## 0.2 · Rendre les paquets GHCR publics

Après le premier build, les images apparaissent sur `github.com/<propriétaire>?tab=packages`, en privé par défaut. Le cluster doit pouvoir les télécharger.

1. Ouvre le paquet de l'application, puis **Package settings** (à droite).
2. En bas, **Danger Zone**, **Change visibility**, **Public**, et confirme en tapant le nom.
3. Recommence pour chaque application.

Ces images ne contiennent aucun secret : les identifiants de la base sont injectés au démarrage par Kubernetes. Si tu préfères les garder privées : PAT GitHub classique avec le seul scope `read:packages`, déclaré dans `tenants.<ns>.registry` du fichier de secrets du projet, et `imagePullSecrets: [{name: ghcr-pull}]` dans le `deploy/values-test.yaml` de chaque dépôt.

> **Vérifier** · Sur la page des paquets, chacun affiche **Public**. En navigation privée, sans être connecté, la page du paquet s'ouvre.
{.is-success}

## 0.3 · Environnement `test` et secrets Cloudflare

Dans **chaque** dépôt d'application :

1. Settings, **Environments**, **New environment**, nom `test`.
2. **Deployment branches and tags** : *Selected branches and tags*, ajoute la règle `main`. Seule la branche principale pourra déployer.
3. **Environment secrets**, **Add environment secret** :

| Secret | Valeur |
| --- | --- |
| `CF_ACCESS_CLIENT_ID` | Client ID du jeton de service Cloudflare `github-deploy` |
| `CF_ACCESS_CLIENT_SECRET` | Client Secret du même jeton |

Le jeton se crée une fois dans Cloudflare ([Mise en route, 4.6](/k3s-lab/mise-en-route/04-cloudflare)) et sert à tous les projets. Ce sont les seuls secrets stockés chez GitHub, et ils ne donnent aucun droit dans le cluster : ils ouvrent seulement le chemin réseau vers l'API.

> **Vérifier** · Dans chaque dépôt, Settings, Environments, `test` affiche la règle de branche `main` et les deux secrets.
{.is-success}
