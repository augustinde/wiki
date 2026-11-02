---
title: "Étape 10 · Premier déploiement depuis GitHub"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le cluster est prêt : il reste à donner à chaque dépôt d'application l'adresse de l'API et le certificat du cluster. Les étapes 0.2 et 0.3 doivent être faites. Les exemples sont ceux de k3s-test : api-demo et front-demo, namespace `demo`.

## 10.1 · Ajouter les variables de dépôt

```bash
make ci-ca
echo "$API_HOST"
```

La première commande affiche une longue ligne qui commence par `LS0t` : c'est le certificat public du cluster, pas un secret. Copie-la en entier.

Dans chaque dépôt : Settings, **Secrets and variables**, **Actions**, onglet **Variables**, **New repository variable**. Ce sont des variables de dépôt, pas d'environnement : le workflow lit `K8S_API_HOST` avant d'entrer dans l'environnement `test`.

| Variable | Valeur |
| --- | --- |
| `K8S_CA_DATA` | sortie de `make ci-ca` |
| `K8S_API_HOST` | `k8s-api<suffixe>.<zone>` (`echo $API_HOST`) |

Vérifie aussi que `K8S_AUDIENCE`, dans `.github/workflows/deploy.yml` du dépôt, vaut `<cluster>` (`echo $CLUSTER`), et que les namespaces de `ci-cd.yml` sont ceux du projet.

> **Vérifier** · L'onglet Variables de chaque dépôt liste les deux variables, et l'onglet Secrets, environnement `test`, les deux secrets Cloudflare.
{.is-success}

## 10.2 · Déployer l'API

Dans le dépôt (k3s-test : api-demo) : **Actions**, workflow **CI/CD**, **Run workflow**, branche `main`, puis **Run workflow**. Compte 3 à 4 minutes.

> **Vérifier**
>
> 1. Les jobs `build` et `deploy-test` sont verts.
> 2. Dans le job de déploiement, l'étape **Vérifier l'identité** affiche `github:<propriétaire>/<dépôt>:test`.
> 3. Dans WSL (k3s-test) :
>
> ```bash
> kubectl -n $NS get deploy api-demo
> curl -s https://api-demo$SUFFIX.$ZONE/api/demo
> ```
>
> Le déploiement est en `1/1`, et `curl` renvoie un JSON avec `seed-1` et `seed-2`.
{.is-success}

## 10.3 · Déployer le front

Même manipulation dans le dépôt du front (k3s-test : front-demo).

> **Vérifier** · Jobs verts, identité `github:<propriétaire>/<dépôt>:test`, `kubectl -n $NS get deploy front-demo` en `1/1`. Dans le navigateur, `https://front-demo<suffixe>.<zone>` affiche le tableau avec les deux lignes.
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
| Job de déploiement gris (ignoré) | `K8S_API_HOST` absente ou créée dans l'environnement | la créer au niveau du dépôt (10.1) |
| Échec à « Kubeconfig OIDC » ou connexion refusée, journal cloudflared en erreur | secrets Cloudflare faux, ou politique `GitHub deploy` absente | 0.3, et Mise en route 4.7 ; refaire le test 7.5 |
| `Unauthorized` | `K8S_CA_DATA` périmée, `K8S_AUDIENCE` différente de `<cluster>`, dépôt non déclaré sous `githubOidc`, ou job sans environnement | `make ci-ca` (10.1) ; 1.3 puis `make bootstrap` |
| `forbidden … User "github:…"` | identité reconnue, sans droits dans ce namespace | 1.3 puis `make platform` ; contrôle 7.4 |
| Pod en `ImagePullBackOff` | paquet GHCR encore privé | 0.2 |
| Pod de l'API pas prêt | base absente ou en erreur | étape 8 |
