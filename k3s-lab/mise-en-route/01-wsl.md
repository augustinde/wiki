---
title: "Étape 1 · WSL2 et terminal"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## 1.1 · Vérifier WSL2

Dans PowerShell (Windows) :

```powershell
wsl --update
wsl -l -v          # la colonne VERSION doit indiquer 2
```

Si ta distribution est en version 1 : `wsl --set-version Ubuntu 2`. Sans Ubuntu récent : `wsl --install -d Ubuntu-24.04`.

Puis, dans Ubuntu :

```bash
sudo apt update && sudo apt upgrade -y
```

## 1.2 · Travailler dans le système de fichiers Linux

Tout se passe dans `~/...`, jamais dans `/mnt/c/...` ni `/mnt/f/...`. Sous `/mnt`, les permissions sont ouvertes à tous : SSH refuse les clés, Ansible ignore sa configuration, les scripts perdent leur droit d'exécution, et tout est plus lent. Depuis Windows, tes fichiers WSL sont visibles dans l'explorateur à l'adresse `\\wsl$\Ubuntu\home\serkox\`.

## 1.3 · Réglages du shell

Deux variables sont utilisées par la procédure. Ajoute-les à la fin de ton fichier de configuration du shell, `~/.zshrc` si tu utilises zsh, sinon `~/.bashrc` :

```bash
export PATH="$HOME/.local/bin:$PATH"     # outils installés par pipx (Ansible)
export EDITOR=nano                       # éditeur ouvert par make secrets-edit
```

Recharge le shell avec `exec zsh` (ou `exec bash`).

> **Vérifier** · `echo $EDITOR` affiche `nano`.
{.is-success}

Pour un terminal plus confortable (autocomplétion, branche git dans le prompt, thème agnoster) : [Terminal WSL avec zsh](/outils/terminal-wsl).
