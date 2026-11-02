---
title: "Monitoring, logs et Rancher"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Grafana, Loki et Rancher sont installés par `make platform` et accessibles uniquement derrière Cloudflare Access.

## Grafana (kube-prometheus-stack)

- **Adresse** : `https://grafana-test.<zone>`. L'utilisateur est `admin`, avec le mot de passe `grafana.adminPassword` du fichier de secrets.
- **Contenu** : Prometheus avec 3 jours de rétention (5 Go maximum) et les tableaux de bord Kubernetes fournis (nœud, namespaces, pods). Les métriques PostgreSQL sont récupérées automatiquement.
- **Ce qui est désactivé** : les cibles du contrôleur, du scheduler, du proxy et d'etcd, intégrés au binaire k3s et donc absents. Alertmanager l'est aussi.
- **Persistance** : un volume de 1 Go conserve les tableaux de bord que tu ajoutes, jusqu'au prochain rejeu de niveau 1. Pour les garder durablement, exporte-les en JSON dans le dépôt.
- **Réglages** : rétention et ressources dans `values/monitoring.yaml.gotmpl`. Pour désactiver : `monitoring.enabled: false`.

## Logs (Loki + Alloy)

Les logs de tous les pods sont consultables dans Grafana, conservés 72 heures et filtrables par namespace, pod, conteneur ou application.

- **Collecte** : Grafana Alloy lit les logs par l'API Kubernetes. Un seul pod suffit, sans accès au disque du nœud. Alloy remplace Promtail, en fin de vie depuis mars 2026.
- **Stockage** : Loki en mode monolithique (un pod), avec un volume local de 10 Go et une rétention réglée par `logging.retention`. Le chart vient du dépôt `grafana-community`, où la communauté Grafana le maintient depuis mars 2026.
- **Consultation** : dans Grafana, Explore, source Loki. Exemples de requêtes :
  - `{namespace="demo", app="api"}` : les logs de ton application.
  - `{namespace="demo"} |= "ERROR"` : les erreurs de tout le namespace.
  - `{namespace="cloudflared"}` : l'état du tunnel.
- **Logs JSON** : si ton application Spring Boot écrit des logs JSON (`logging.structured.format.console=ecs`), ajoute `| json` à la requête pour filtrer sur les champs, par exemple `| json | level="ERROR"`.
- **Pour désactiver** : `logging.enabled: false`, ce qui libère environ 0,5 Go de RAM.

Les logs disparaissent à chaque rejeu de niveau 1, comme les métriques : c'est un environnement de test.

## Rancher

- **Adresse** : `https://rancher-test.<zone>`. Premier login `admin` avec `rancher.bootstrapPassword`, puis Rancher impose un nouveau mot de passe.
- **Version** : le chart est contraint entre 2.13 et 2.14, car Rancher refuse les versions de Kubernetes hors de sa matrice. Rancher 2.14.3 accepte k3s 1.33 à 1.35 ([matrice SUSE](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/)) : le canal k3s v1.34 est dans les clous.
- **TLS** : `tls: external`. Cloudflare termine le HTTPS et Traefik conserve l'en-tête `X-Forwarded-Proto: https`, ce qui évite d'installer cert-manager.
- **Usage recommandé** : observer, lire les logs, ouvrir un shell de diagnostic. Ne crée pas de ressources depuis l'interface : elles ne seraient pas rejouées.
- **Monitoring** : l'onglet « Monitoring » de Rancher attend sa propre application rancher-monitoring. Ici, le monitoring est fourni par la pile Grafana indépendante ; utilise Grafana directement.
- **Pour désactiver** : `rancher.enabled: false`, ce qui libère environ 1,5 Go de RAM.
