---
title: "Étape 8 · Créer la base PostgreSQL"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

`make data` crée la base `app` décrite dans `helmfile/databases/app.yaml` : une instance, un volume de 5 Go, la table `demo` et ses deux lignes de départ.

## 8.1 · Créer la base

```bash
make data
```

La commande attend que la base soit prête, jusqu'à 5 minutes.

> **Vérifier** · La sortie se termine par `clusters.postgresql.cnpg.io/app condition met`. Ensuite :
>
> ```bash
> kubectl -n demo get clusters.postgresql.cnpg.io
> kubectl -n demo get pods -l cnpg.io/cluster=app
> kubectl -n demo get svc app-admin
> kubectl -n demo get secret app-app
> ```
>
> | Ressource | Résultat attendu |
> | --- | --- |
> | Cluster `app` | `INSTANCES 1`, `READY 1`, statut `Cluster in healthy state` |
> | Pod | `app-1` en `1/1 Running` |
> | Service `app-admin` | `CLUSTER-IP 10.43.0.100` (accès IntelliJ par WARP) |
> | Secret `app-app` | présent : identifiants générés par CloudNativePG |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 8.2 · Lire les données de départ

```bash
make db-psql DB=app
```

Dans psql, tape `SELECT * FROM demo;`, puis `\q` pour quitter.

> **Vérifier** · Deux lignes, `seed-1` et `seed-2`, avec leur date de création.
{.is-success}

## 8.3 · Se connecter depuis IntelliJ

1. WARP connecté, dans PowerShell : `Test-NetConnection 10.43.0.100 -Port 5432`.
2. Dans WSL : `make db-creds DB=app` affiche l'URL JDBC, l'utilisateur et le mot de passe.
3. Dans IntelliJ : Database, **+**, Data Source, PostgreSQL. Host `10.43.0.100`, port `5432`, base `app`, utilisateur `app`, mot de passe de `db-creds`. Laisse le mode SSL par défaut.

> **Vérifier** · PowerShell affiche `TcpTestSucceeded : True`, et **Test Connection** dans IntelliJ répond `Succeeded`. En cas d'échec, suis la liste sous le contrôle final de la mise en route.
{.is-success}
