---
title: "Objectif et principes"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Tout l'environnement d'un projet (OS, k3s, Cloudflare Tunnel, Grafana, Rancher, bases et services) se reconstruit depuis son dépôt avec `make rebuild`. Aucun port web n'est ouvert sur le serveur : seul SSH accepte des connexions entrantes. Le même modèle sert à tous les projets ; seules les valeurs changent.

Cinq règles tiennent la procédure :

- **Tout est en code** : Ansible pour la machine, Helmfile pour le cluster, SOPS pour les secrets. Le serveur est jetable.
- **Cloudflare est la seule porte d'entrée** : le cluster ouvre lui-même une connexion sortante vers Cloudflare. Rancher, Grafana et l'API Kubernetes sont protégés par Cloudflare Access.
- **Versions figées** : versions exactes des charts dans `helmfile/versions.yaml`, suivies par Renovate dans le modèle ; canal k3s v1.34 compatible avec Rancher.
- **Couches indépendantes** : machine, k3s, plateforme, données, services. On rejoue une couche sans toucher aux autres.
- **GitHub n'a que le strict nécessaire** : un namespace, une identité OIDC valable quelques minutes, et rien d'autre dans le cluster.

- **Valeurs séparées du code** : tout ce qui est propre à un projet est dans `environments/<env>/`, la taille des composants dans un profil de machine.

Cible : un serveur Ubuntu 24.04 ou 26.04, mono-node. Profil `medium` (8 Go, OVH VPS-2) pour la plateforme complète ; `small` (4 Go) sans Rancher ; `large` pour un dédié.
