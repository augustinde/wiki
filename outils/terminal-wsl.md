---
title: "Terminal WSL avec zsh"
description: "zsh, Oh My Zsh, thème agnoster et police pour Windows Terminal"
published: true
date: 2026-11-02T15:30:00.000Z
tags: outils, terminal, wsl
editor: markdown
dateCreated: 2026-11-02T15:30:00.000Z
---

Un terminal Ubuntu (WSL) avec autocomplétion, suggestions tirées de l'historique, coloration des commandes et un prompt qui affiche le dossier et la branche git. À faire une fois par poste, indépendamment de tout projet.

## Installer zsh et Oh My Zsh

```bash
sudo apt install -y zsh git curl
RUNZSH=no CHSH=no sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
ZSH_CUSTOM=~/.oh-my-zsh/custom
git clone --depth 1 https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone --depth 1 https://github.com/zsh-users/zsh-syntax-highlighting $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
chsh -s $(which zsh)
```

Puis remplace `~/.zshrc` (`nano ~/.zshrc`) par :

```bash
export PATH="$HOME/.local/bin:$PATH"
export EDITOR=nano                       # utilisé par make secrets-edit

export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="agnoster"
DEFAULT_USER="$USER"                     # masque utilisateur@machine
plugins=(git ssh-agent kubectl helm zsh-autosuggestions zsh-syntax-highlighting)
source "$ZSH/oh-my-zsh.sh"

zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Za-z}'
HISTSIZE=50000
SAVEHIST=50000
setopt SHARE_HISTORY HIST_IGNORE_ALL_DUPS

# Ajoute ici les variables propres à tes projets, par exemple le KUBECONFIG d'un cluster
```

Ferme et rouvre le terminal, ou tape `exec zsh`.

> **Vérifier** · `echo $SHELL` affiche `/usr/bin/zsh`, et le prompt montre le dossier courant et la branche git dans des flèches colorées.
{.is-success}

## Deux règles avec zsh

- Ta configuration est dans `~/.zshrc`, pas dans `~/.bashrc`. Pour la recharger : `exec zsh`.
- Ne fais jamais `source ~/.bashrc` depuis zsh : ce fichier est écrit pour bash et provoque des erreurs (`shopt: command not found`) et un prompt illisible. Si ça arrive, `exec zsh` remet tout en ordre.

## Police et Windows Terminal

Le thème agnoster a besoin d'une Nerd Font, sinon les flèches du prompt s'affichent en carrés. Télécharge par exemple JetBrainsMono Nerd Font sur nerdfonts.com, installe les `.ttf` dans Windows (clic droit, « Installer pour tous les utilisateurs »), puis sélectionne-la : Windows Terminal, Paramètres, profil Ubuntu, Apparence, Police. Le jeu de couleurs se règle au même endroit.
