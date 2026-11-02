---
title: "Étape 10 · Premier déploiement depuis GitHub"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le cluster est prêt : il reste à donner aux deux dépôts l'adresse de l'API et le certificat du cluster. Les étapes 0.2 et 0.3 doivent être faites.

## 10.1 · Ajouter les variables de dépôt

```bash
make ci-ca
```

La commande affiche une longue ligne qui commence par `LS0t` : c'est le certificat public du cluster, pas un secret. Copie-la en entier.

Dans chaque dépôt : Settings, **Secrets and variables**, **Actions**, onglet **Variables**, **New repository variable**. Ce sont des variables de dépôt, pas d'environnement : le workflow lit `K8S_API_HOST` avant d'entrer dans l'environnement `test`.

| Variable | api-demo | front-demo |
| --- | --- | --- |
| `APP_HOST` | `api-demo-test.tondomaine.fr` | `front-demo-test.tondomaine.fr` |
| `K8S_CA_DATA` | sortie de `make ci-ca` | la même valeur |
| `K8S_API_HOST` | `k8s-api-test.tondomaine.fr` | `k8s-api-test.tondomaine.fr` |

Après l'étape 11.3, `APP_HOST` ne sert plus : les noms d'hôtes sont dans `deploy/values-test.yaml` et `deploy/values-prod.yaml`.

> **Vérifier** · L'onglet Variables de chaque dépôt liste ces trois variables, et l'onglet Secrets, environnement `test`, les deux secrets Cloudflare.
{.is-success}

## 10.2 · Déployer api-demo

Dans api-demo : **Actions**, workflow **CI/CD**, **Run workflow**, branche `main`, puis **Run workflow**. Compte 3 à 4 minutes.

> **Vérifier**
>
> 1. Les jobs `build` et `deploy` sont verts.
> 2. Dans le job `deploy`, l'étape **Vérifier l'identité** affiche `github:augustinde/api-demo:test`.
> 3. Dans WSL :
>
> ```bash
> kubectl -n demo get deploy api-demo
> curl -s https://api-demo-test.tondomaine.fr/api/demo
> ```
>
> Le déploiement est en `1/1`, et `curl` renvoie un JSON avec `seed-1` et `seed-2`.
{.is-success}

## 10.3 · Déployer front-demo

Même manipulation dans front-demo.

> **Vérifier** · Jobs verts, identité `github:augustinde/front-demo:test`, `kubectl -n demo get deploy front-demo` en `1/1`. Dans le navigateur, `https://front-demo-test.tondomaine.fr` affiche le tableau avec les deux lignes.
{.is-success}

## 10.4 · Tester de bout en bout

```bash
make db-psql DB=app
```

Dans psql : `INSERT INTO demo (label) VALUES ('depuis psql');`, puis `\q`.

> **Vérifier** · Après rechargement, l'écran du front affiche trois lignes. Désormais, chaque push sur `main` redéploie l'application concernée.
{.is-success}

## Si un job échoue

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| Job `deploy` gris (ignoré) | `K8S_API_HOST` absente ou créée dans l'environnement | la créer au niveau du dépôt (10.1) |
| Échec à « Kubeconfig OIDC » ou connexion refusée, journal cloudflared en erreur | secrets Cloudflare faux, ou politique `GitHub deploy` absente | 0.3, et Mise en route 4.7 ; refaire le test 7.5 |
| `Unauthorized` | `K8S_CA_DATA` périmée, dépôt absent de `github_oidc_repositories`, ou job sans environnement | `make ci-ca` (10.1) ; 1.3 puis `make bootstrap` |
| `forbidden … User "github:…"` | identité reconnue, sans droits dans `demo` | 1.3 puis `make platform` ; contrôle 7.4 |
| Pod en `ImagePullBackOff` | paquet GHCR encore privé | 0.2 |
| Pod `api-demo` pas prêt | base `app` absente ou en erreur | étape 8 |
