---
title: "Mise en route : poste, Cloudflare et WARP"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Compte 2 à 3 heures la première fois, plus l'attente de la bascule DNS. Tout se fait une seule fois ; ensuite, les commandes `make` de la procédure principale suffisent. Le VPS n'intervient qu'après l'étape 6.

## Vue d'ensemble


Ton poste a deux côtés, et chaque outil va d'un côté précis.

| Où | Quoi | Pourquoi là |
| --- | --- | --- |
| WSL2 (Ubuntu, shell zsh) | git, make, Ansible, kubectl, Helm, Helmfile, sops, age, cloudflared, le dépôt `k3s-lab`, tes clés SSH et age | Ansible ne tourne pas sous Windows, et tout le dépôt est pensé pour Linux |
| Windows | Client Cloudflare One (WARP), IntelliJ, navigateur | WARP donne accès aux bases et à Rancher depuis IntelliJ et le navigateur |
| Tableau de bord Cloudflare | Zone DNS, organisation Zero Trust, tunnel, Access, réglages WARP | Configuration faite une fois, qui survit aux reconstructions du cluster |
| Espace client OVH | Serveurs DNS du domaine, installation du VPS | Le domaine reste chez OVH ; seul son DNS passe chez Cloudflare |

WSL2 n'a pas besoin de WARP : tes commandes `make` passent par SSH.

**Choix retenus pour cet environnement**

- **Domaine** : ton domaine existant chez OVH, dont le DNS est confié à Cloudflare (étape 4.0).
- **Applications de démo déployées depuis GitHub** : `augustinde/api-demo` (Spring Boot) et `augustinde/front-demo` (Angular), authentifiées par OIDC. Leur configuration côté GitHub est décrite à l'[étape 0 de la procédure](/k3s-lab/procedure/00-github). Le compte de test à jeton `github-deployer` reste désactivé.
- **Ce qui est déployé** : la plateforme (tunnel, Grafana, Loki, Rancher), la base `app`, le service de démonstration `whoami`, puis api-demo et front-demo par GitHub.

**Ordre des étapes** : 1 WSL2 → 2 outils → 3 clés, dépôt et configuration → 4 Cloudflare → 5 WARP → 6 contrôle final, puis la procédure de mise en place du cluster.

**Valeurs à noter au fil de l'eau**, dans ton gestionnaire de mots de passe :

- le nom d'équipe Zero Trust (étape 4.1) ;
- l'ID du tunnel (étape 4.2) ;
- le fichier de clé age `~/.config/sops/age/keys.txt` (étape 3.3) ;
- les mots de passe Grafana et Rancher (étape 3.6).

Dans les commandes, remplace `tondomaine.fr` par ton domaine.
