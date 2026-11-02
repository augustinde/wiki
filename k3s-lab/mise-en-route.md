---
title: "Mise en route : poste, Cloudflare et WARP"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Compte 2 à 3 heures la première fois, plus l'attente de la bascule DNS. L'essentiel se fait une seule fois par poste et par compte Cloudflare ; les sections marquées **par projet** se refont pour chaque nouveau projet, dans l'ordre donné par [Créer un projet](/k3s-lab/nouveau-projet). Le serveur n'intervient qu'après l'étape 6.

## Vue d'ensemble

Ton poste a deux côtés, et chaque outil va d'un côté précis.

| Où | Quoi | Pourquoi là |
| --- | --- | --- |
| WSL2 (Ubuntu, shell zsh) | git, make, Ansible, kubectl, Helm, Helmfile, sops, age, cloudflared, les dépôts des projets, tes clés SSH et age | Ansible ne tourne pas sous Windows, et tout le dépôt est pensé pour Linux |
| Windows | Client Cloudflare One (WARP), IntelliJ, navigateur | WARP donne accès aux bases et à Rancher depuis IntelliJ et le navigateur |
| Tableau de bord Cloudflare | Zone DNS, organisation Zero Trust, tunnels, Access, réglages WARP | Configuration qui survit aux reconstructions des clusters |
| Espace client OVH | Serveurs DNS du domaine, installation du VPS | Le domaine reste chez OVH ; seul son DNS passe chez Cloudflare |

WSL2 n'a pas besoin de WARP : tes commandes `make` passent par SSH.

| Étape | Une fois | Par projet |
| --- | --- | --- |
| 1 · WSL2 et terminal | tout | |
| 2 · Outils | tout | |
| 3 · Clés, dépôt et configuration | clé SSH, clé age, dépendances (3.1, 3.3, 3.4) | dépôt, valeurs, secrets (3.2, 3.5, 3.6) |
| 4 · Cloudflare | domaine, Zero Trust, HTTPS, jeton GitHub (4.0, 4.1, 4.5, 4.6) | tunnel, route, DNS, applications Access (4.2, 4.3, 4.4, 4.7) |
| 5 · WARP | tout (5.3 à ajuster à partir du deuxième projet) | |
| 6 · Contrôle final | | les deux listes |

**Valeurs à noter au fil de l'eau**, dans ton gestionnaire de mots de passe :

- le nom d'équipe Zero Trust (étape 4.1) ;
- le fichier de clé age `~/.config/sops/age/keys.txt` (étape 3.3) ;
- pour chaque projet : les mots de passe Grafana et Rancher (étape 3.6). L'ID du tunnel est dans `values.yaml`.

Les valeurs propres au projet s'écrivent `$CLUSTER`, `$ZONE`, `$SUFFIX`… dans les commandes : voir [Lire les procédures](/k3s-lab).
