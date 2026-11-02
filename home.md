---
title: "Accueil"
description: "Documentation personnelle : procédures, projets et outils"
published: true
date: 2026-11-02T12:00:00.000Z
tags: accueil
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Documentation personnelle, modifiable dans le navigateur et synchronisée avec le dépôt GitHub `augustinde/wiki`. Le wiki tourne en local sous Docker : la pile est dans le dossier `docker/` du dépôt.

## Sections

| Section | Contenu |
| --- | --- |
| [k3s-lab](/k3s-lab) | Modèle d'environnement k3s rejouable : mise en route, création de projet, procédure, exploitation, référence, fiches projets |
| [Outils](/outils/wiki) | Ce wiki : [installation](/outils/installation), synchronisation git, ajout de pages ; [terminal WSL avec zsh](/outils/terminal-wsl) |

## Ajouter une section

Une section, c'est un dossier de chemins : `homelab/jarvis`, `team-peps/deploiement`, `freelance/devis`…

- **Dans le navigateur** : bouton **Nouvelle page** en haut à droite, saisis le chemin complet (par exemple `homelab/jarvis/proxmox`), éditeur **Markdown**.
- **Dans le dépôt** : crée `homelab/jarvis/proxmox.md` avec l'en-tête décrit dans [Outils · Wiki](/outils/wiki), puis commit et push.

Ajoute ensuite la section au tableau ci-dessus.
