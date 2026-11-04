---
title: "3 · Supprimer un utilisateur"
description: "Retirer à un collègue tous ses accès à un projet, tout de suite"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

Fais les étapes dans l'ordre : la première bloque toute nouvelle connexion, la deuxième coupe ce qui est déjà ouvert.

## 1 · Le retirer de l'équipe GitHub

`https://github.com/orgs/<org>/teams/<slug>`, **Members**, coche-le, **Remove from team**. S'il ne doit plus rien avoir dans l'organisation : **People**, sa ligne, **Remove from organization**.

Access le refuse à sa prochaine connexion. Sa session en cours, elle, reste valable jusqu'à 24 heures : étape suivante.

## 2 · Couper ses sessions et son appareil

Zero Trust (`<équipe-zt>`) :

1. **Team & Resources**, **Users**, sa ligne, **Revoke session** : ses sessions Access sont invalidées tout de suite, sur Rancher, Grafana et l'API.
2. **Team & Resources**, **Devices**, son appareil, **Revoke** : le client WARP perd l'organisation, et avec elle l'accès aux bases.

> **Vérifier** · **Logs**, **Access** : ses nouvelles tentatives apparaissent refusées. **Devices** affiche son appareil révoqué.
{.is-success}

## 3 · Supprimer ses comptes dans les applications

- **Grafana** : **Administration**, **Users and access**, **Users**, sa ligne, **Delete user**.
- **Rancher** : **Users & Authentication**, **Users**, sa ligne, **Delete** ; il disparaît aussi des membres du cluster.
- **Dépôts GitHub** : s'il y avait un accès direct (hors équipe), **Settings**, **Collaborators and teams**, retire-le ; et des **Required reviewers** de l'environnement `prod`.

## 4 · Changer ce qu'il connaissait

Ce qu'il a pu lire reste valable tant que tu ne le changes pas. Selon ce qu'il avait :

| Il connaissait | À changer |
| --- | --- |
| les identifiants d'une base (`make db-creds`) | `make db-reset DB=<base>` en test (**données perdues**) ; en prod, nouveau mot de passe : `kubectl -n <ns> delete secret <base>-app`, CloudNativePG en génère un nouveau, puis redémarrer l'application (`kubectl -n <ns> rollout restart deploy/<app>`) |
| les mots de passe `admin` de Grafana ou Rancher | `make secrets-edit` (Grafana) puis `make platform` ; Rancher, dans son interface |
| le jeton de service `github-deploy` | Zero Trust, **Service credentials**, **Service Tokens**, **Refresh**, puis les secrets des dépôts d'applications (procédure, étape 0.3) et `secrets.yaml` |

> **Vérifier** · Avec son compte, s'il te prête la main pour le test : `https://rancher<suffixe>.<zone>` affiche « That account does not have access », et ses anciens identifiants Grafana et Rancher sont refusés.
{.is-success}
