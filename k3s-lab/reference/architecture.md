---
title: "Architecture"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le VPS n'accepte que SSH. Tout le reste passe par une connexion sortante du cluster vers Cloudflare, qui filtre l'accès avant même d'atteindre le serveur.

![Architecture : seul SSH entre directement, tout le reste passe par Cloudflare](/k3s-lab/assets/architecture.svg)

*Schéma : `k3s-lab/assets/architecture.svg`, source Mermaid dans le dépôt k3s-lab.*

Toi, tu administres par SSH ; GitHub et les visiteurs passent par Cloudflare. Chaque bande du cluster se rejoue seule avec la commande indiquée à droite.

| Brique | Choix | Pourquoi |
| --- | --- | --- |
| Provisioning | Ansible | Idempotent, réutilisable tel quel sur le dédié |
| Distribution | k3s v1.34 (SQLite, Traefik interne, sans ServiceLB) | Binaire unique, aucun port publié sur l'IP, version acceptée par Rancher |
| Entrée publique | Cloudflare Tunnel (cloudflared dans le cluster) | Zéro port web ouvert, TLS géré par Cloudflare, pas de cert-manager |
| Contrôle d'accès | Cloudflare Access | Email pour Rancher et Grafana, jeton de service pour l'API |
| Déploiement | Helmfile, versions exactes suivies par Renovate | Couches sélectionnables, versions figées, `diff` avant `apply` |
| PostgreSQL | CloudNativePG | Bases déclaratives, seed rejouable, métriques Prometheus |
| Observabilité | kube-prometheus-stack (Grafana), Loki + Alloy, Rancher | Métriques, logs et tableaux de bord dans Grafana, interface de diagnostic |
| Accès CI | OIDC GitHub Actions + RBAC par namespace | Aucun jeton Kubernetes stocké chez GitHub, droits limités à un namespace |
| Secrets | SOPS + age | Chiffrés dans Git, déchiffrés à la volée par helm-secrets |

**Noms d'hôtes.** Le certificat gratuit de Cloudflare couvre `*.tondomaine.fr`, pas `*.k3s.tondomaine.fr`. Tous les noms restent donc à un niveau, avec le suffixe `-test` : `rancher-test`, `grafana-test`, `whoami-test`, `k8s-api-test`. Le CNAME joker `*` vers le tunnel couvre tout nouveau service sans action dans Cloudflare.

**Routage du tunnel** (`helmfile/values/cloudflared.yaml.gotmpl`) : `k8s-api-test` va vers l'API Kubernetes en TCP, tout le reste de la zone vers Traefik.

**Réglages de zone** : SSL/TLS, Edge Certificates, **Always Use HTTPS** activé et version TLS minimale 1.2.

**Poste, Cloudflare et WARP** se configurent une seule fois, dans la section [Mise en route](/k3s-lab/mise-en-route) (étapes 2 à 5). Le tunnel, les applications Access et WARP survivent aux reconstructions du cluster.
