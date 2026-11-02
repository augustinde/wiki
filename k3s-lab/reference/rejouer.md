---
title: "Rejouer"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Choisis le niveau le plus bas qui couvre ce que tu veux tester : plus il est bas, plus le rejeu est rapide et moins il détruit.

| Niveau | Ce que tu testes | Commande | Ce qui est détruit |
| --- | --- | --- | --- |
| 0 · Serveur | Install d'un dédié vierge, durcissement | Étapes 2 à 9 : réinstallation OVH, puis `make forget-host bootstrap tunnel up` | Tout le disque |
| 1 · k3s | Installation k3s et plateforme complète | `make rebuild` | Cluster, volumes, données, jeton GitHub |
| 2 · Couche | Mise à jour d'une couche | `make platform`, `make data` ou `make apps` | Rien : apply idempotent |
| 3 · Base | Création d'une base et de son seed | `make db-reset DB=app` | Cette base et ses volumes |
| 4 · Service Helmfile | Déploiement, probes, ingress | `make app-redeploy APP=whoami` | Ce service |
| 4 · Service GitHub | Le chemin exact de GitHub | Actions, CI/CD, **Run workflow** dans le dépôt | Rien : mise à jour en place |

Après un niveau 0 ou 1, le cluster a une nouvelle autorité de certification : lance `make ci-ca` et mets à jour la variable `K8S_CA_DATA` dans les deux dépôts (étape 10.1). Le tunnel, Cloudflare Access et les droits OIDC se remettent en place tout seuls.

Avant chaque `apply`, `make diff` montre exactement ce qui va changer.

Trois règles pour que deux rejeux donnent le même résultat :

- **Versions figées** : versions exactes des charts dans `test.yaml` ; pour k3s, renseigne `k3s_version` en plus du canal.
- **Aucun `kubectl apply` à la main** : un changement fait hors du dépôt disparaît au prochain rejeu de niveau 1.
- **Rien de configuré dans l'interface Rancher** que tu veuilles retrouver : Rancher sert à observer et diagnostiquer, le dépôt reste la source de vérité.
