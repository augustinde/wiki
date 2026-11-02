---
title: "Wiki"
description: "Wiki.js en local sous Docker, synchronisé avec le dépôt augustinde/wiki"
published: true
date: 2026-11-02T12:00:00.000Z
tags: outils, wiki
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Ce wiki tourne sur ton poste : Wiki.js 2.5 et PostgreSQL dans Docker, sur `http://localhost:3000`. Chaque page existe aussi en fichier Markdown dans le dépôt privé `augustinde/wiki`, synchronisé dans les deux sens toutes les 5 minutes.

## Où sont les choses

| Élément | Emplacement |
| --- | --- |
| Pile Docker | `~/wiki-stack` dans WSL : `compose.yaml`, `.env` (mot de passe de la base) |
| Contenu | dépôt `augustinde/wiki`, un fichier `.md` par page, un dossier par section |
| Base PostgreSQL | volume Docker `wiki-stack_db-data` : comptes, réglages, index de recherche |
| Clé de synchronisation | clé SSH de déploiement du dépôt, collée dans Administration, Stockage, Git |

## Commandes

```bash
cd ~/wiki-stack
docker compose up -d          # démarrer (redémarre seul ensuite avec Docker)
docker compose ps             # état : wiki et db en "running", db en "healthy"
docker compose logs -f wiki   # journal, dont la synchronisation git
docker compose down           # arrêter (les données restent dans les volumes)
```

## Synchronisation git

Mode **bidirectionnel**, toutes les 5 minutes :

- une page modifiée dans le navigateur devient un commit dans `augustinde/wiki`, auteur = ton compte wiki ;
- un fichier poussé dans le dépôt apparaît dans le wiki au passage suivant.

Pour forcer un passage : Administration, **Stockage**, **Git**, **Force Sync**.

> **Une seule règle** · ne modifie pas la même page dans le navigateur et dans le dépôt pendant le même intervalle de 5 minutes : la synchronisation garderait une seule des deux versions.
{.is-warning}

## Ajouter une page depuis le dépôt

Le chemin du fichier donne l'adresse de la page : `homelab/jarvis/proxmox.md` devient `/homelab/jarvis/proxmox`. L'en-tête est obligatoire ; garde le titre et la description entre guillemets, sinon un « : » casse l'en-tête :

```markdown
---
title: "Proxmox"
description: "Hyperviseur du homelab Jarvis"
published: true
tags: homelab, jarvis
editor: markdown
---

Contenu de la page en Markdown.
```

Un encadré s'écrit comme une citation suivie de sa classe (`is-info`, `is-success`, `is-warning`, `is-danger`) :

```markdown
> **Vérifier** · la commande affiche `ok`.
{.is-success}
```

Les images vont dans le dépôt à côté des pages (par exemple `homelab/assets/schema.svg`) et s'insèrent avec `![Schéma](/homelab/assets/schema.svg)`.

## Mettre à jour Wiki.js

1. Dans `compose.yaml`, remplace la version de l'image `ghcr.io/requarks/wiki` par la nouvelle (liste sur la page *Releases* du projet requarks/wiki).
2. `docker compose pull && docker compose up -d`

> **Vérifier** · `docker compose ps` montre `wiki` en `running`, et Administration, **Informations système** affiche la nouvelle version.
{.is-success}
