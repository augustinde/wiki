---
title: "Wiki"
description: "Wiki.js en local sous Docker, synchronisé avec le dépôt augustinde/wiki"
published: true
date: 2026-11-02T12:00:00.000Z
tags: outils, wiki
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Ce wiki tourne sur ton PC : Wiki.js 2.5 et PostgreSQL dans Docker, sur `http://localhost:3000`. Installation : [Installer le wiki sur un PC](/outils/installation). Chaque page existe aussi en fichier Markdown dans le dépôt privé `augustinde/wiki`, synchronisé dans les deux sens toutes les 5 minutes.

## Où sont les choses

| Élément | Emplacement |
| --- | --- |
| Pile Docker | dossier `docker/` de ce dépôt (`F:\Projects\wiki\docker`), lancé par Docker Desktop : `compose.yaml`, `.env` local (mot de passe de la base, ignoré par git) |
| Contenu | dépôt `augustinde/wiki`, un fichier `.md` par page, un dossier par section |
| Base PostgreSQL | volume Docker `wiki-stack_db-data` : comptes, réglages, index de recherche |
| Clé de synchronisation | clé SSH de déploiement du dépôt, collée dans Administration, Stockage, Git |

## Commandes

Dans PowerShell :

```powershell
cd F:\Projects\wiki\docker
docker compose up -d          # démarrer (redémarre ensuite avec Docker Desktop)
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

La barre de gauche range les pages par **titre**, en ordre alphabétique : `Étape 10` passerait avant `Étape 2`. Numérote donc sur deux chiffres dès qu'une série dépasse 9 pages (`Étape 01`, `Étape 02`… `Étape 15`), dans le titre comme dans le nom du fichier.

Les images vont dans le dépôt à côté des pages (par exemple `homelab/assets/schema.svg`) et s'insèrent avec `![Schéma](/homelab/assets/schema.svg)`.

## Mettre à jour Wiki.js

1. Dans `docker/compose.yaml`, remplace la version de l'image `ghcr.io/requarks/wiki` par la nouvelle (liste sur la page *Releases* du projet requarks/wiki).
2. Commit et push de ce changement, puis dans `F:\Projects\wiki\docker` : `git pull`, `docker compose pull`, `docker compose up -d`

> **Vérifier** · `docker compose ps` montre `wiki` en `running`, et Administration, **Informations système** affiche la nouvelle version.
{.is-success}
