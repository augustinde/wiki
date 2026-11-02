---
title: "Installer le wiki sur un PC"
description: "Wiki.js et PostgreSQL dans Docker Desktop sous Windows, synchronisés avec ce dépôt"
published: true
date: 2026-11-02T14:00:00.000Z
tags: outils, wiki
editor: markdown
dateCreated: 2026-11-02T14:00:00.000Z
---

Le wiki tourne sur ton PC Windows, dans Docker Desktop : http://localhost:3000, visible de ta machine seulement. Tout ce qu'il faut pour le lancer est dans le dossier `docker/` de ce dépôt. Les commandes sont à taper dans **PowerShell** ; le dépôt est cloné ici dans `F:\Projects\wiki`, adapte le chemin si besoin.

Prérequis : Docker Desktop démarré, Git pour Windows, et ta clé SSH GitHub habituelle dans `$HOME\.ssh`.

## 1. Cloner le dépôt

```powershell
git clone git@github.com:augustinde/wiki.git F:\Projects\wiki
cd F:\Projects\wiki\docker
docker compose version
```

> **Vérifier** · `dir -Force` liste `compose.yaml`, `.env.example` et `.gitignore`, et `docker compose version` affiche une version.
{.is-success}

## 2. Créer la clé de synchronisation

Une clé dédiée, limitée à ce dépôt. Quand `ssh-keygen` demande une phrase de passe, appuie deux fois sur **Entrée** : Wiki.js l'utilise seul, sans pouvoir la saisir.

```powershell
ssh-keygen -t ed25519 -C wikijs -f $HOME\.ssh\wikijs_deploy
Get-Content $HOME\.ssh\wikijs_deploy.pub | Set-Clipboard
```

La clé publique est dans le presse-papiers. GitHub, dépôt `wiki`, **Settings**, **Deploy keys**, **Add deploy key** : titre `wikijs`, colle la clé, coche **Allow write access**, puis **Add key**.

> **Vérifier** · `ssh -i $HOME\.ssh\wikijs_deploy -T git@github.com` répond `Hi augustinde/wiki! You've successfully authenticated`.
{.is-success}

## 3. Démarrer la pile

```powershell
cd F:\Projects\wiki\docker
$b = New-Object byte[] 24
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($b)
$pw = ($b | ForEach-Object { $_.ToString('x2') }) -join ''
"DB_PASSWORD=$pw" | Out-File -Encoding ascii .env
docker compose up -d
```

Le fichier `.env` contient le mot de passe de la base : il est ignoré par git et ne quitte pas ton PC. Il ne se crée qu'une fois ; ne le régénère pas ensuite, la base garderait l'ancien mot de passe.

> **Vérifier** · `docker compose ps` montre `db` en `healthy` et `wiki` en `running`, et `docker compose logs wiki --tail 5` se termine par `HTTP Server: [ RUNNING ]`. Le premier démarrage prend environ une minute. Les deux conteneurs apparaissent aussi dans Docker Desktop, groupe `wiki-stack`.
{.is-success}

## 4. Assistant d'installation

Ouvre http://localhost:3000 : adresse email et mot de passe de l'administrateur, **Site URL** `http://localhost:3000`, puis **Install**. Connecte-toi.

> **Vérifier** · la page d'accueil vide de Wiki.js s'affiche, avec l'engrenage **Administration** en haut à droite.
{.is-success}

## 5. Passer en français

À faire avant l'import, sinon les pages seraient rangées en anglais. **Administration**, **Locale** : télécharge **Français**, choisis-le comme langue du site, laisse **Multilingual Namespacing** désactivé, puis **Apply**.

> **Vérifier** · l'interface est en français.
{.is-success}

## 6. Brancher le dépôt

Copie la clé privée dans le presse-papiers :

```powershell
Get-Content $HOME\.ssh\wikijs_deploy -Raw | Set-Clipboard
```

Puis **Administration**, **Stockage**, **Git** :

| Champ | Valeur |
| --- | --- |
| Activé | oui |
| Type d'authentification | `ssh` |
| URI du dépôt | `git@github.com:augustinde/wiki.git` |
| Branche | `main` |
| Mode de la clé privée | `contents` |
| Contenu de la clé privée | colle le presse-papiers (lignes BEGIN et END comprises) |
| Auteur par défaut | ton nom et ton email |
| Synchronisation | bidirectionnelle, toutes les 5 minutes |

**Appliquer**, puis, en bas de la page, **Actions**, **Import Everything**.

> **Vérifier** · **Administration**, **Pages** liste une quarantaine de pages, dont `k3s-lab/procedure/12-alertes-discord`, et l'état de la cible Git ne montre pas d'erreur.
{.is-success}

Les fichiers du dossier `docker/` apparaissent aussi dans **Administration**, **Ressources** : c'est normal, Wiki.js importe tous les fichiers du dépôt. Ne les supprime pas depuis le wiki.

## 7. Navigation par dossiers

**Administration**, **Navigation**, mode **Arborescence du site**, **Appliquer**.

> **Vérifier** · la barre de gauche affiche `k3s-lab` et `outils`, et la page `k3s-lab/procedure/12-alertes-discord` montre ses encadrés « Vérifier » en vert.
{.is-success}

## 8. Tester la synchronisation

1. Dans le navigateur, modifie la page **Accueil** et enregistre : au plus 5 minutes après, un commit apparaît dans le dépôt.
2. Dans `F:\Projects\wiki`, `git pull`, modifie `home.md` (IntelliJ, VS Code…), commit et push : au plus 5 minutes après, la modification apparaît dans le navigateur.

> **Vérifier** · les deux modifications sont visibles des deux côtés. Pour ne pas attendre : **Stockage**, **Git**, **Force Sync**.
{.is-success}

## Sur un autre PC

Refais les étapes 1 et 3 à 7, avec la même clé de déploiement copiée dans `$HOME\.ssh` ou une nouvelle ajoutée de la même façon. Le contenu revient entièrement du dépôt ; seuls les comptes et réglages du wiki sont propres à chaque PC.
