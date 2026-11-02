---
title: "k3s-lab"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Environnement k3s rejouable sur un VPS OVH : serveur durci, cluster k3s, Cloudflare Tunnel et Access, PostgreSQL, Grafana, Loki, Alertmanager et Rancher, avec deux applications de démo déployées depuis GitHub en test puis en prod.

Aucun port web n'est ouvert sur le serveur : tout passe par Cloudflare, et seul SSH entre directement.

## Par où commencer

| Situation | Aller à |
| --- | --- |
| Premier poste, rien n'est installé | [Mise en route](/k3s-lab/mise-en-route) : WSL2, outils, Cloudflare, WARP |
| Installer ou réinstaller le VPS | [Procédure](/k3s-lab/procedure), étapes 0 à 9 |
| Brancher GitHub, la prod, les alertes | [Procédure](/k3s-lab/procedure), étapes 10 à 14 |
| Travailler au quotidien | [Gestion quotidienne](/k3s-lab/exploitation/gestion-quotidienne) |
| Comprendre un choix, dépanner | [Référence](/k3s-lab/reference/architecture) |

## Lire les procédures

Chaque action est suivie d'un encadré :

> **Vérifier** · La commande de contrôle et le résultat attendu. Ne passe à l'étape suivante que s'il s'affiche.
{.is-success}

Dans les commandes, remplace `tondomaine.fr` par ton domaine. `$IP` est l'IPv4 du VPS (étape 3.1 de la procédure).

## Cette documentation

Ces pages font partie du wiki : modifie-les dans le navigateur, ou dans le dépôt `augustinde/wiki` (dossier `k3s-lab/`). Voir [Wiki](/outils/wiki).
