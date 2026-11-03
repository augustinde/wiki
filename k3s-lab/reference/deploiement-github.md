---
title: "Déploiement depuis GitHub (OIDC)"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Aucun jeton Kubernetes n'est stocké chez GitHub. À chaque job, GitHub émet un jeton OIDC signé, valable environ 5 minutes, et l'API k3s le vérifie elle-même. Seul le jeton Cloudflare Access reste un secret, et il ne donne aucun droit dans le cluster.

## Les deux verrous

1. **Atteindre l'API** : `k8s-api<suffixe>.<zone>` n'accepte que les connexions qui présentent le jeton de service `github-deploy`. Le port 6443 du VPS reste fermé.
2. **Être reconnu par k3s** : l'API accepte un jeton GitHub seulement si trois conditions sont réunies.
   - L'audience est `<cluster>`, le `clusterName` du projet.
   - Le dépôt est déclaré sous `tenants.<ns>.githubOidc` d'au moins un namespace (Ansible en déduit la liste de l'API).
   - Le job déclare un environnement GitHub.

L'identité obtenue, `github:<propriétaire>/<dépôt>:<environnement>`, reçoit des droits seulement dans les namespaces où elle est déclarée (`tenants.<ns>.githubOidc`). Dans ces namespaces, elle peut gérer déploiements, services, ingress, configmaps, secrets et jobs.

Elle ne peut pas lire un autre namespace, voir les nœuds, faire un `exec` dans un pod ni modifier les droits. Pod Security « restricted » et le quota s'appliquent comme avant.

Limite à connaître : qui peut déployer dans un namespace peut aussi lire ses secrets, dont le mot de passe de la base. Isole dans un autre namespace ce que GitHub ne doit pas voir.

## Mise en place

Côté cluster, un dépôt est autorisé par une seule déclaration, `tenants.<ns>.githubOidc`, appliquée par `make bootstrap` (API) puis `make platform` (droits). Voir les étapes 1.3, 4.4 et 7.4. Côté GitHub : les environnements `test` et `prod` limités à `main` avec les deux secrets Cloudflare (0.3, 11.2), deux variables de dépôt (10.1), et `K8S_AUDIENCE` = `<cluster>` dans le workflow.

## Le job GitHub

Chaque dépôt d'application contient trois workflows : **CI** (`ci-cd.yml`) construit l'image à chaque push sur `main` et la publie sur GHCR, sans déployer ; **Déployer** (`deploy-version.yml`) et **Release** (`release.yml`) se lancent à la main et appellent `deploy.yml` (procédure, étape 11.3). `deploy.yml` suit quatre étapes :

1. `cloudflared access tcp` ouvre `127.0.0.1:16443` vers l'API, avec le jeton Access.
2. `deploy/write-kubeconfig.sh` écrit un kubeconfig sans secret.
3. Ce kubeconfig appelle `deploy/github-oidc-credential.sh`, qui demande le jeton à GitHub et le renouvelle avant expiration : un `helm --wait` de 5 minutes passe sans erreur.
4. `helm upgrade --install` déploie l'image avec le chart `deploy/chart`.

Pour une nouvelle application, copie le dossier `deploy/` et les workflows d'un dépôt de démo, puis adapte `deploy/values-*.yaml` (image, port, sondes, noms d'hôte), la variable `RELEASE` et `K8S_AUDIENCE` dans `deploy.yml`, et les namespaces dans `deploy-version.yml` et `release.yml`. Les jobs doivent garder `permissions: id-token: write` et un `environment` (`test` ou `prod`).

## Tester depuis ton poste

Ton poste ne peut pas obtenir de jeton GitHub. Le test 7.5 suffit à valider le passage par Cloudflare. Pour rejouer aussi les droits avec un jeton, active le compte de test `github-deployer` (`githubDeployer: true` sous `tenants.<ns>`, puis `make platform`) : jeton de 90 jours, mêmes droits que GitHub.

- `make deployer-kubeconfig`, puis `make deploy-test` : passage par Cloudflare Access et droits du compte.
- `make app-deploy APP=<nom> TAG=<tag>` : déploiement par ce chemin d'un service décrit dans `environments/<env>/apps/<nom>.yaml`.
- `make deployer-revoke` : invalide tous ses jetons. Repasse `githubDeployer: false` quand tu n'en as plus besoin.

Les applications GitHub ne sont pas dans Helmfile : leurs releases appartiennent à GitHub, ce qui évite que `make apps` remette une ancienne image. Le namespace, la base et les secrets restent gérés par le dépôt du projet.

## Après une reconstruction

Un `make rebuild` crée une nouvelle autorité de certification. Relance `make ci-ca` et mets à jour la variable `K8S_CA_DATA` des dépôts d'applications. Rien d'autre ne change côté GitHub. En cas de fuite du jeton Access, régénère `github-deploy` dans Cloudflare : sans lui, même un jeton GitHub valide n'atteint pas l'API.
