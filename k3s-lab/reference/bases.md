---
title: "Bases de données"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Chaque base est un cluster CloudNativePG décrit par un fichier `helmfile/databases/<nom>.yaml`. Elle se recrée vide avec son seed en une commande.

- **Déployer** : `make data` crée les bases déclarées sous `databases:` et attend qu'elles soient prêtes.
- **Recréer** : `make db-reset DB=app` supprime le cluster et ses volumes, le recrée, rejoue `seed.sql` puis attend l'état Ready. Le seed est joué avec le rôle propriétaire : les tables appartiennent à l'application.
- **Se connecter** : `make db-psql DB=app` ouvre psql sur l'instance primaire. `make db-forward DB=app` expose la base sur `localhost:5432` pour IntelliJ ou DBeaver. Le mot de passe est dans le secret `app-app`.
- **Côté application** : un service du même namespace avec `database.enabled: true` reçoit `SPRING_DATASOURCE_URL`, `_USERNAME` et `_PASSWORD`.
- **Monitoring** : si Prometheus est installé, le chart ajoute un PodMonitor ; les métriques PostgreSQL apparaissent dans Grafana.

## Depuis IntelliJ (ou DBeaver)

Chaque base déclare une IP fixe dans `adminAccess.clusterIP` : `10.43.0.100` pour `app`. Le chart crée un service `app-admin` sur cette IP, toujours branché sur l'instance primaire. L'adresse ne change pas d'un rejeu à l'autre, donc IntelliJ garde sa connexion après un `make rebuild`.

1. Connecte WARP.
2. `make db-creds DB=app` affiche l'URL JDBC, l'utilisateur et le mot de passe.
3. Dans IntelliJ, Database, +, Data Source, PostgreSQL :
   - Host `10.43.0.100`, Port `5432`, Database `app`, User `app`, mot de passe affiché par `db-creds`.
   - SSL : laisse le mode par défaut (`prefer`). Le certificat de CloudNativePG est auto-signé, n'active pas `verify-full`.
4. Après un `make rebuild` ou un `make db-reset`, le mot de passe est régénéré : relance `make db-creds`.

Pour une nouvelle base, prends l'IP libre suivante entre `10.43.0.96` et `10.43.0.111`.

Sans WARP, par exemple depuis une machine non enrôlée, `make db-forward DB=app` reste disponible. Il passe par le tunnel SSH et expose la base sur `localhost:5432` tant que la commande tourne.
