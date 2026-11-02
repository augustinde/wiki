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

L'IP sert à deux endroits : l'inventaire Ansible et la variable `SSH_TARGET` du `Makefile`.

```bash
IP=203.0.113.10        # ← remplace par l'IPv4 du VPS
sed -i "s/ansible_host: [0-9.]*/ansible_host: $IP/" ansible/inventory/hosts.yml
sed -i "s/^SSH_TARGET ?= .*/SSH_TARGET ?= ubuntu@$IP/" Makefile
```

La variable `IP` resservira dans les contrôles suivants. Elle disparaît quand tu ouvres un nouveau terminal ; `nc: missing port number` en est le symptôme. Pour la redéfinir depuis le `Makefile` : `IP=$(sed -n 's/^SSH_TARGET ?= ubuntu@//p' Makefile)`.

> **Vérifier**
>
> ```bash
> grep -n 'ansible_host' ansible/inventory/hosts.yml
> grep -n '^SSH_TARGET' Makefile
> ```
>
> Les deux lignes portent la même IP, avec l'utilisateur `ubuntu` dans le `Makefile`.
{.is-success}

## 3.2 · Accepter l'empreinte du serveur

Un VPS réinstallé a une nouvelle empreinte SSH. Sans ce nettoyage, SSH refuse la connexion avec `REMOTE HOST IDENTIFICATION HAS CHANGED`.

```bash
make forget-host
```

`Cannot stat …/known_hosts` au début est sans gravité : le fichier n'existe pas encore.

**Si la commande échoue avec `You are required to change your password immediately` puis `no TTY available`** : les images Ubuntu d'OVH imposent de changer le mot de passe temporaire du compte `ubuntu` à la première connexion, même avec une clé. Une fois, en interactif :

```bash
ssh ubuntu@$IP
```

Saisis le mot de passe temporaire (email OVH de fin d'installation, ou page du VPS dans l'espace client), puis deux fois un nouveau mot de passe. La connexion se ferme : relance `make forget-host`. Garde ce mot de passe dans ton gestionnaire. SSH le refusera après le bootstrap ; il ne sert plus qu'à la console KVM d'OVH, en secours.

> **Vérifier**
>
> ```bash
> ssh ubuntu@$IP 'lsb_release -ds; free -h | grep Mem'
> cd ansible && ansible all -m ping && ansible all -b -m command -a whoami; cd ..
> ```
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Connexion | aucun mot de passe demandé |
> | Système | `Ubuntu 26.04 LTS (ou 24.04.x)` |
> | Mémoire | `Mem:` environ `7,7Gi` au total |
> | Ansible | `vps-test \| SUCCESS` et `"ping": "pong"` |
>
> La dernière commande doit répondre `vps-test | CHANGED` avec `root` : Ansible sait passer administrateur. Sur Ubuntu 26.04, `sudo` est remplacé par sudo-rs, que le module `become` d'Ansible gère mal quand un mot de passe est demandé. Le compte `ubuntu` d'OVH n'en demande pas, donc ça passe. Si la commande reste bloquée jusqu'à `Timed out waiting for become success`, ajoute `ansible_become_exe: sudo.ws` (le sudo classique, toujours installé) sous `ansible_user` dans `ansible/inventory/hosts.yml`, puis relance.
>
> `Permission denied (publickey)` : la clé collée chez OVH n'est pas celle de `~/.ssh`. Recommence l'étape 2 avec la bonne clé.
{.is-success}
