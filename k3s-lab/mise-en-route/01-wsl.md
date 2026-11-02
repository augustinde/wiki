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

## 1.3 · Shell zsh avec Oh My Zsh et le thème agnoster

(recommandé : autocomplétion, branche git dans le prompt)

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

# Cluster de test par défaut pour kubectl/helm
[ -f "$HOME/k3s-lab/.kube/k3s-test.yaml" ] && export KUBECONFIG="$HOME/k3s-lab/.kube/k3s-test.yaml"
```

Ferme et rouvre le terminal, ou tape `exec zsh`. Vérifie avec `echo $SHELL` (doit afficher `/usr/bin/zsh`) et `echo $EDITOR` (doit afficher `nano`).

**Deux règles avec zsh :**

- Ta configuration est dans `~/.zshrc`, pas dans `~/.bashrc`. Pour la recharger : `exec zsh`.
- Ne fais jamais `source ~/.bashrc` depuis zsh : ce fichier est écrit pour bash et provoque des erreurs (`shopt: command not found`) et un prompt illisible. Si ça arrive, `exec zsh` remet tout en ordre.

## 1.4 · Police et Windows Terminal

Le thème agnoster a besoin d'une Nerd Font, sinon les flèches du prompt s'affichent en carrés. Télécharge par exemple JetBrainsMono Nerd Font sur nerdfonts.com, installe les `.ttf` dans Windows (clic droit, « Installer pour tous les utilisateurs »), puis sélectionne-la : Windows Terminal, Paramètres, profil Ubuntu, Apparence, Police. Le jeu de couleurs se règle au même endroit.
