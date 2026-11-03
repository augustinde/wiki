---
title: "Étape 08 · Créer la base PostgreSQL"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

`make data` crée les bases déclarées sous `databases:`, chacune décrite dans `environments/<env>/databases/<nom>.yaml`. La base `app` du modèle a une instance, un volume à la taille du profil (5 Go en `medium`), la table `demo` et deux lignes de départ, dans le namespace `$NS`.

## 8.1 · Créer la base

```bash
make data
```

La commande attend que la base soit prête, jusqu'à 5 minutes.

> **Vérifier** · La sortie se termine par `clusters.postgresql.cnpg.io/app condition met`. Ensuite :
>
> ```bash
> kubectl -n $NS get clusters.postgresql.cnpg.io
> kubectl -n $NS get pods -l cnpg.io/cluster=app
> kubectl -n $NS get svc app-admin
> kubectl -n $NS get secret app-app
> ```
>
> | Ressource | Résultat attendu |
> | --- | --- |
> | Cluster `app` | `INSTANCES 1`, `READY 1`, statut `Cluster in healthy state` |
> | Pod | `app-1` en `1/1 Running` |
> | Service `app-admin` | `CLUSTER-IP` = `adminAccess.clusterIP` de `databases/app.yaml`, dans `$ROUTE` (accès IntelliJ par WARP) |
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

1. `make db-creds DB=app` affiche l'URL JDBC (avec l'IP de la base), l'utilisateur et le mot de passe.
2. WARP connecté, dans PowerShell : `Test-NetConnection <ip de la base> -Port 5432`.
3. Dans IntelliJ : Database, **+**, Data Source, PostgreSQL. Host = IP de la base, port `5432`, base `app`, utilisateur `app`, mot de passe de `db-creds`. Laisse le mode SSL par défaut.

> **Vérifier** · PowerShell affiche `TcpTestSucceeded : True`, et **Test Connection** dans IntelliJ répond `Succeeded`. En cas d'échec, suis la liste sous le contrôle final de la mise en route.
{.is-success}
