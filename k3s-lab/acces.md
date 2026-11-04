---
title: "Accès"
description: "Qui entre dans Rancher, Grafana, l'API et les bases : connexion GitHub, utilisateurs, client WARP"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

Cloudflare Access filtre Rancher, Grafana et l'API Kubernetes avant qu'une requête n'atteigne le serveur, et le client WARP donne accès aux bases. Les deux s'appuient sur **GitHub** : on se connecte avec son compte GitHub, et seuls les membres d'une **équipe de l'organisation GitHub** passent. Donner ou retirer un accès revient à gérer cette équipe.

| Page | Quand |
| --- | --- |
| [Connexion par GitHub](/k3s-lab/acces/connexion-github) | une fois par organisation Zero Trust (par compte Cloudflare) |
| [Ajouter un utilisateur](/k3s-lab/acces/ajouter-un-utilisateur) | un collègue rejoint le projet |
| [Supprimer un utilisateur](/k3s-lab/acces/supprimer-un-utilisateur) | un collègue quitte le projet |
| [Installer le client WARP](/k3s-lab/acces/client-warp) | un nouveau PC, le tien ou celui d'un collègue |
| [Plusieurs organisations WARP sur un PC](/k3s-lab/acces/plusieurs-organisations-warp) | des projets sur plusieurs comptes Cloudflare |

Valeurs utilisées dans ces pages :

| Valeur | Exemple | Où la trouver |
| --- | --- | --- |
| `<org>` | `augustinde-lab` | l'organisation GitHub, `https://github.com/<org>` |
| `<équipe>` | `k3s-test` | l'équipe GitHub qui a accès au projet |
| `<équipe-zt>` | `augustin-lab`, `escargow` | le nom d'équipe Zero Trust du compte Cloudflare : Zero Trust, Settings, « Team domain » |

Ce que ces accès ne donnent pas : SSH au serveur, commandes `make` et lecture des secrets du dépôt. Ils restent réservés à la personne qui a la clé SSH et la clé age du projet.
