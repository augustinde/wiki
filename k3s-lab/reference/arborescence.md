---
title: "Arborescence du dépôt"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le dépôt `k3s-lab` sépare la machine (`ansible/`), le cluster (`helmfile/`) et l'accès GitHub (`scripts/`).

```text
k3s-lab/
├── Makefile                        point d'entrée unique (make help)
├── .sops.yaml                      clé age pour chiffrer les *.secrets.yaml
├── ansible/
│   ├── inventory/hosts.yml         IP et utilisateur du serveur
│   ├── group_vars/all.yml          canal k3s, pare-feu, dépôts GitHub autorisés (OIDC)
│   ├── site.yml / reset.yml        bootstrap complet / désinstallation de k3s
│   └── roles/ base · firewall · k3s  (k3s : config, Traefik interne, OIDC)
├── helmfile/
│   ├── helmfile.yaml.gotmpl        couches platform, data, apps
│   ├── values/alerting.yaml        gabarits Discord et règles d'alerte
│   ├── environments/
│   │   ├── test.yaml               zone, tunnel, namespaces, bases, services
│   │   └── test.secrets.yaml       tunnel, jeton Access, mots de passe, PAT GHCR (chiffré)
│   ├── values/                     cloudflared, CNPG, monitoring, Loki, Alloy, Rancher
│   ├── charts/
│   │   ├── cloudflared/            connecteur Cloudflare Tunnel
│   │   ├── tenant/                 namespace sécurisé + droits GitHub (OIDC et compte de test)
│   │   ├── postgres/               cluster CNPG + seed + IP d'administration
│   │   └── app/                    service HTTP générique (Spring Boot compris)
│   ├── databases/                  une base = un fichier (app.yaml, app-prod.yaml)
│   └── apps/
│       ├── whoami.yaml             déployé par Helmfile
│       └── api.yaml                déployé par GitHub
├── ci/                             à copier dans le dépôt de l'application
│   ├── deploy.example.yml          job GitHub Actions de déploiement
│   ├── write-kubeconfig.sh         kubeconfig sans secret
│   └── github-oidc-credential.sh   fournit le jeton OIDC à kubectl/helm
└── scripts/
    ├── deployer-kubeconfig.sh      jeton de test (compte github-deployer)
    └── via-cloudflare.sh           exécute une commande par le chemin de GitHub
```
