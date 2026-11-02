---
title: "k3s-lab"
description: "Modèle d'environnement k3s rejouable : un dépôt par projet, une procédure commune"
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Environnement k3s rejouable sur un serveur Ubuntu : serveur durci, cluster k3s, Cloudflare Tunnel et Access, PostgreSQL, Grafana, Loki, Alertmanager et Rancher, avec des applications déployées depuis GitHub en test puis en prod.

Aucun port web n'est ouvert sur le serveur : tout passe par Cloudflare, et seul SSH entre directement.

## Un modèle, un dépôt par projet

Le dépôt `augustinde/k3s-lab` est un **modèle** GitHub. Chaque projet en est une copie, créée avec **Use this template**, qui ne diffère que par son dossier `environments/<env>/` : nom du cluster, taille de la machine (profil), domaine, tunnel, namespaces, bases, services. Le reste du code est commun.

| Projet | Machine | Fiche |
| --- | --- | --- |
| k3s-test | VPS OVH VPS-2, 8 Go, profil `medium` | [k3s-test](/k3s-lab/projets/k3s-test) |

## Par où commencer

| Situation | Aller à |
| --- | --- |
| Premier poste, rien n'est installé | [Mise en route](/k3s-lab/mise-en-route) : WSL2, outils, Cloudflare, WARP |
| Nouveau projet (nouveau serveur, nouveau client) | [Créer un projet](/k3s-lab/nouveau-projet) |
| Installer ou réinstaller le serveur d'un projet | [Procédure](/k3s-lab/procedure), étapes 0 à 9 |
| Brancher GitHub, la prod, les alertes | [Procédure](/k3s-lab/procedure), étapes 10 à 14 |
| Travailler au quotidien | [Gestion quotidienne](/k3s-lab/exploitation/gestion-quotidienne) |
| Comprendre un choix, dépanner | [Référence](/k3s-lab/reference/architecture) |

## Lire les procédures

Chaque action est suivie d'un encadré :

> **Vérifier** · La commande de contrôle et le résultat attendu. Ne passe à l'étape suivante que s'il s'affiche.
{.is-success}

Les procédures valent pour tous les projets. Les valeurs propres à un projet s'écrivent ainsi :

| Dans les commandes | Dans le texte | Exemple (k3s-test) | Défini dans |
| --- | --- | --- | --- |
| `$CLUSTER` | `<cluster>` | `k3s-test` | `values.yaml`, `clusterName` |
| `$ENV` | `<env>` | `test` | nom du dossier `environments/<env>/` |
| `$ZONE` | `<zone>` | `augustinde.dev` | `values.yaml`, `zone` |
| `$SUFFIX` | `<suffixe>` | `-test` | `values.yaml`, `hostSuffix` |
| `$API_HOST` | `k8s-api<suffixe>.<zone>` | `k8s-api-test.augustinde.dev` | `values.yaml`, `cloudflare.apiHostname` |
| `$ROUTE` | `<route>` | `10.43.0.96/28` | `values.yaml`, `cloudflare.privateRoute` |
| `$NS` | `<ns>` | `demo` | `values.yaml`, `defaultNamespace` |
| `$IP` | `<ip>` | IPv4 du serveur | `inventory.yml`, `ansible_host` |

Les variables se chargent dans le terminal, depuis le dossier du projet :

```bash
cd ~/<projet>
eval "$(make -s vars)"
echo "$CLUSTER grafana$SUFFIX.$ZONE $IP"
```

Recharge-les à chaque nouveau terminal ou changement de projet. Les commandes `make` n'en ont pas besoin : elles lisent elles-mêmes l'environnement du projet.

## Cette documentation

Ces pages font partie du wiki : modifie-les dans le navigateur, ou dans le dépôt `augustinde/wiki` (dossier `k3s-lab/`). Voir [Wiki](/outils/wiki).
