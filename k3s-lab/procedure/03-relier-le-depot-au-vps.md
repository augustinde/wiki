---
title: "Étape 3 · Relier le dépôt au VPS"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## 3.1 · Renseigner l'IP

L'IP n'est écrite qu'à un endroit, l'inventaire du projet ; le `Makefile` et les variables la lisent là.

```bash
NEW_IP=203.0.113.10        # ← remplace par l'IPv4 du serveur
sed -i "s/ansible_host: .*/ansible_host: $NEW_IP/" $ENV_DIR/inventory.yml
eval "$(make -s vars)"
```

La variable `IP` resservira dans les contrôles suivants. Elle disparaît quand tu ouvres un nouveau terminal ; `nc: missing port number` en est le symptôme : relance `eval "$(make -s vars)"`.

> **Vérifier**
>
> ```bash
> grep -n 'ansible_' $ENV_DIR/inventory.yml
> make envs
> ```
>
> L'inventaire porte la nouvelle IP et l'utilisateur `ubuntu`, et `make envs` affiche `ubuntu@<ip>` pour l'environnement.
{.is-success}

## 3.2 · Accepter l'empreinte du serveur

Un VPS réinstallé a une nouvelle empreinte SSH. Sans ce nettoyage, SSH refuse la connexion avec `REMOTE HOST IDENTIFICATION HAS CHANGED`.

```bash
make forget-host
```

`Cannot stat …/known_hosts` au début est sans gravité : le fichier n'existe pas encore.

**Si la commande échoue avec `You are required to change your password immediately` puis `no TTY available`** : les images Ubuntu d'OVH imposent de changer le mot de passe temporaire du compte `ubuntu` à la première connexion, même avec une clé. Une fois, en interactif :

```bash
ssh $SSH_USER@$IP
```

Saisis le mot de passe temporaire (email OVH de fin d'installation, ou page du VPS dans l'espace client), puis deux fois un nouveau mot de passe. La connexion se ferme : relance `make forget-host`. Garde ce mot de passe dans ton gestionnaire. SSH le refusera après le bootstrap ; il ne sert plus qu'à la console KVM d'OVH, en secours.

> **Vérifier**
>
> ```bash
> ssh $SSH_USER@$IP 'lsb_release -ds; free -h | grep Mem'
> cd ansible && ansible all -i ../$ENV_DIR/inventory.yml -m ping && ansible all -i ../$ENV_DIR/inventory.yml -b -m command -a whoami; cd ..
> ```
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Connexion | aucun mot de passe demandé |
> | Système | `Ubuntu 26.04 LTS (ou 24.04.x)` |
> | Mémoire | `Mem:` cohérente avec le profil : environ `3,8Gi` en `small`, `7,7Gi` en `medium` |
> | Ansible | `<nœud> \| SUCCESS` et `"ping": "pong"` |
>
> La dernière commande doit répondre `<nœud> | CHANGED` avec `root` : Ansible sait passer administrateur. Sur Ubuntu 26.04, `sudo` est remplacé par sudo-rs, que le module `become` d'Ansible gère mal quand un mot de passe est demandé. Le compte `ubuntu` d'OVH n'en demande pas, donc ça passe. Si la commande reste bloquée jusqu'à `Timed out waiting for become success`, ajoute `ansible_become_exe: sudo.ws` (le sudo classique, toujours installé) sous `ansible_user` dans `$ENV_DIR/inventory.yml`, puis relance.
>
> `Permission denied (publickey)` : la clé collée chez OVH n'est pas celle de `~/.ssh`. Recommence l'étape 2 avec la bonne clé.
{.is-success}
