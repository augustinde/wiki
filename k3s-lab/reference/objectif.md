---
title: "Objectif et principes"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Tout l'environnement (OS, k3s, Cloudflare Tunnel, Grafana, Rancher, bases et services) se reconstruit depuis le dépôt `k3s-lab` avec `make rebuild`. Aucun port web n'est ouvert sur le VPS : seul SSH accepte des connexions entrantes.

Cinq règles tiennent la procédure :

- **Tout est en code** : Ansible pour la machine, Helmfile pour le cluster, SOPS pour les secrets. Le serveur est jetable.
- **Cloudflare est la seule porte d'entrée** : le cluster ouvre lui-même une connexion sortante vers Cloudflare. Rancher, Grafana et l'API Kubernetes sont protégés par Cloudflare Access.
- **Versions figées** : versions exactes des charts dans `test.yaml`, suivies par Renovate ; canal k3s v1.34 compatible avec Rancher.
- **Couches indépendantes** : machine, k3s, plateforme, données, services. On rejoue une couche sans toucher aux autres.
- **GitHub n'a que le strict nécessaire** : un namespace, une identité OIDC valable quelques minutes, et rien d'autre dans le cluster.

Cible : un VPS OVH sous Ubuntu 24.04, mono-node. Avec Rancher et le monitoring, il faut **8 Go de RAM (VPS-2)** ; le VPS-1 (4 Go) ne suffit plus.
