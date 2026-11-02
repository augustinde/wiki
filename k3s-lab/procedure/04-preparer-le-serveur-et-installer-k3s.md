---
title: "Étape 4 · Préparer le serveur et installer k3s"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## 4.1 · Lancer le bootstrap

```bash
make bootstrap
```

Ansible applique trois rôles : `base` (mises à jour, paquets, SSH durci, fail2ban), `firewall` (ufw) et `k3s` (installation, Traefik interne, OIDC). Compte 3 à 6 minutes : la latence vers le Canada ralentit chaque tâche.

> **Vérifier** · La dernière ligne, `PLAY RECAP`, affiche `vps-test : ok=… changed=… unreachable=0 failed=0`. Si `failed` vaut 1 ou plus, lis la tâche en rouge juste au-dessus, corrige, puis relance `make bootstrap` : il reprend sans casser ce qui est fait.
{.is-success}

## 4.2 · Contrôler le durcissement SSH

```bash
ssh -o PubkeyAuthentication=no ubuntu@$IP
ssh root@$IP
ssh ubuntu@$IP 'sudo fail2ban-client status sshd | head -3'
```

> **Vérifier**
>
> | Commande | Résultat attendu |
> | --- | --- |
> | sans clé | `Permission denied (publickey)`, aucun mot de passe demandé |
> | en `root` | `Permission denied (publickey)` |
> | fail2ban | `Status for the jail: sshd` |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 4.3 · Contrôler le pare-feu

```bash
ssh ubuntu@$IP 'sudo ufw status verbose'
for p in 22 80 443 6443; do nc -zv -w 3 $IP $p; done
```

> **Vérifier**
>
> - `Status: active` et `Default: deny (incoming), allow (outgoing)`.
> - Règles : `22/tcp LIMIT IN`, puis `10.42.0.0/16` et `10.43.0.0/16` en `ALLOW IN` (trafic interne du cluster).
> - `nc` : le port 22 répond `succeeded`, les ports 80, 443 et 6443 échouent (`timed out`).
>
> `nc: command not found` : `sudo apt install netcat-openbsd`.
{.is-success}

## 4.4 · Contrôler k3s

```bash
ssh ubuntu@$IP 'sudo k3s kubectl get nodes'
ssh ubuntu@$IP 'sudo grep -c authentication-config /etc/rancher/k3s/config.yaml'
ls -l .kube/k3s-test.yaml
```

> **Vérifier**
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Nœud | `vps-test   Ready   control-plane`, version `v1.34.x+k3s1` |
> | OIDC GitHub | `1` : l'API accepte les jetons des deux dépôts |
> | kubeconfig | fichier présent, droits `-rw-------` |
>
> L'OIDC affiche `0` : la liste `github_oidc_repositories` est vide. Reprends 1.3, puis relance `make bootstrap`.
{.is-success}
