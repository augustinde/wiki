---
title: "Arborescence du dépôt"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le dépôt d'un projet (copie du modèle `k3s-lab`) sépare les valeurs du projet (`environments/`), les tailles de machine (`profiles/`), la machine (`ansible/`), le cluster (`helmfile/`) et l'accès GitHub (`scripts/`, `ci/`). Seul `environments/` diffère d'un projet à l'autre.

```text
<projet>/
├── Makefile                        point d'entrée unique (make help) ; lit environments/<env>/
├── .sops.yaml                      clé age publique qui chiffre environments/*/secrets.yaml
├── environments/
│   ├── example/                    modèle copié par make init (valeurs à remplacer)
│   └── <env>/                      un dossier par serveur
│       ├── values.yaml             cluster, profil, zone, tunnel, maintenance, namespaces, bases, services, buckets
│       ├── inventory.yml           IP et utilisateur SSH du serveur
│       ├── secrets.yaml            tunnel, jeton Access, mots de passe, webhook, PAT GHCR (chiffré)
│       ├── databases/              une base = un fichier (app.yaml, app-prod.yaml)
│       └── apps/                   un service Helmfile = un fichier (whoami.yaml)
├── profiles/                       small (4 Go) · medium (8 Go) · large (16 Go et plus)
├── ansible/
│   ├── load-env.yml                lit values.yaml et le profil pour les playbooks
│   ├── group_vars/all.yml          réglages communs, et variables dérivées de l'environnement
│   ├── site.yml / reset.yml        bootstrap complet / désinstallation de k3s
│   └── roles/ base · firewall · k3s  (k3s : config, Traefik interne, OIDC)
├── helmfile/
│   ├── helmfile.yaml.gotmpl        couches platform, data, apps ; charge l'environnement
│   ├── versions.yaml               versions exactes des charts (Renovate)
│   ├── values/                     cloudflared, CNPG, monitoring, alertes, Loki, Alloy, Rancher
│   └── charts/
│       ├── cloudflared/            connecteur Cloudflare Tunnel
│       ├── tenant/                 namespace sécurisé + droits GitHub (OIDC et compte de test)
│       ├── postgres/               cluster CNPG + seed + IP d'administration
│       ├── s3-proxy/               fichiers publics d'un bucket S3, en lecture, derrière le cache Cloudflare
│       └── app/                    service HTTP générique (Spring Boot compris)
├── ci/                             exemple de job GitHub de déploiement, à copier dans une application
└── scripts/
    ├── cfg.py                      lit la configuration (make vars, config-check, envs, urls)
    ├── init-env.py                 crée un environnement (make init)
    ├── deployer-kubeconfig.sh      jeton de test (compte github-deployer)
    └── via-cloudflare.sh           exécute une commande par le chemin de GitHub
```

Les valeurs se superposent, de la plus faible à la plus forte : `helmfile/versions.yaml`, `profiles/<profile>.yaml`, `environments/<env>/values.yaml`, puis `secrets.yaml`. Ansible lit les deux du milieu.
