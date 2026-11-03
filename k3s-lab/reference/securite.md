---
title: "Sécurité, ressources et dépannage"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## Ce que la procédure sécurise

- **Surface exposée** : seul le port 22 est ouvert. ServiceLB est désactivé, donc aucun Service Kubernetes ne peut publier de port sur l'IP du serveur, même par erreur.
- **SSH** : connexion par clé uniquement, root interdit, un seul utilisateur autorisé, fail2ban actif.
- **Interfaces** : Rancher, Grafana et l'API Kubernetes sont filtrés par Cloudflare Access avant d'atteindre le cluster. Rancher et Grafana exigent en plus un poste connecté à WARP ; les bases ne sont joignables que par WARP ou SSH.
- **Secrets** : chiffrés au repos dans k3s (`secrets-encryption`) et dans Git (SOPS + age).
- **Namespaces applicatifs** : Pod Security « restricted », quotas, compte GitHub limité et à durée de vie courte.

## Budget mémoire

La taille de chaque composant vient du profil du projet. `make config-check` additionne la mémoire demandée par la plateforme (requests) ; la consommation réelle est un peu plus haute. Ordres de grandeur :

| Composant | `small` (4 Go) | `medium` (8 Go) | `large` (16 Go) |
| --- | --- | --- | --- |
| Système et réservations k3s | 0,5 Go | 0,75 Go | 1,25 Go |
| k3s, Traefik, CoreDNS, cloudflared | 0,6 Go | 0,7 Go | 0,8 Go |
| Rancher (avec fleet et webhook) | absent | 1,5 Go | 1,5 Go |
| Prometheus, Grafana, Alertmanager, exporters | 0,7 Go | 1,2 Go | 2,5 Go |
| Loki + Alloy | 0,3 Go | 0,5 Go | 1 Go |
| CloudNativePG + une base | 0,2 Go | 0,4 Go | 1 Go |
| **Reste pour les applications** | **environ 1,5 Go** | **environ 3 Go** | **8 Go et plus** |

Repère mesuré sur k3s-test (`medium`, 8 Go) : 74 % de la mémoire utilisée à la fin de l'étape 9, avant la prod. Si la mémoire dépasse 90 %, l'alerte `MemoireNoeudSaturee` prévient ; Rancher est le premier candidat à couper (`rancher.enabled: false`), puis le profil inférieur pour un composant précis (`resources.<composant>` dans `values.yaml`).

## Dépannage

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| `make` répond « API de <cluster> injoignable » | tunnel SSH fermé, ou kubeconfig absent après un reset | `make tunnel` ; si besoin `make bootstrap` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | serveur réinstallé, nouvelle empreinte | `make forget-host` |
| Erreur Cloudflare 1033 | cloudflared non connecté | `kubectl -n cloudflared logs deploy/cloudflared` ; vérifier l'ID et le JSON du tunnel |
| 404 sur un nom d'hôte | Ingress absent, ou nom différent de celui de l'Ingress | `kubectl get ingress -A` |
| Rancher boucle en redirection | `X-Forwarded-Proto` perdu | vérifier le HelmChartConfig Traefik (`trustedIPs`), puis `make bootstrap` |
| Erreur de certificat sur un nom à 2 niveaux | certificat Cloudflare limité à `*.zone` | utiliser `nom<suffixe>.zone`, pas `nom.test.zone` |
| Pod refusé « violates PodSecurity restricted » | image root ou contexte de sécurité absent | régler `podSecurityContext.runAsUser` sur un UID non-root |
| Pod refusé « exceeded quota » | quota du namespace atteint | augmenter `tenants.<ns>.quota` |
| `deploy-test` : « websocket: bad handshake » | jeton Access refusé, ou politique autre que Service Auth | vérifier l'application Access `k3s API <cluster>` |
| `deploy-test` : « Unauthorized » | jeton Kubernetes expiré ou révoqué | `make deployer-kubeconfig` |
| Job GitHub : « Unauthorized » | dépôt absent de tous les `githubOidc`, job sans environment, `K8S_AUDIENCE` différente de `<cluster>`, ou `K8S_CA_DATA` périmée | corriger puis `make bootstrap` ; après un rebuild, `make ci-ca` |
| Job GitHub : « forbidden … User github:… » | identité reconnue mais sans droits dans ce namespace | ajouter {repository, environment} sous tenants.\<ns>.githubOidc, puis make platform ; vérifier avec make oidc-test |
| Rien dans Grafana Explore > Loki | Alloy ne joint pas Loki, ou Loki pas encore prêt | kubectl -n logging logs deploy/alloy et kubectl -n logging get pods |
| Rancher refuse de s'installer (kubeVersion) | k3s hors de la matrice Rancher | garder le canal `v1.34` (`k3s.channel`), ou ajuster `versions.rancher` dans `helmfile/versions.yaml` |
| Pods `Evicted` ou `OOMKilled` | mémoire saturée, ou limite d'un composant trop basse | `kubectl top pods -A --sort-by=memory` ; relever `resources.<composant>.limits.memory`, désactiver Rancher, ou passer au profil supérieur |
| `make` : « Préciser l'environnement » | plusieurs dossiers dans `environments/` | ajouter `ENV=<env>` à la commande, ou `projet <dossier> <env>` |
| `make config-check` : IP de base hors de la route | `adminAccess.clusterIP` d'une base copiée d'un autre projet | prendre une adresse de `cloudflare.privateRoute` |
| `make platform` ou `bootstrap` : valeur manquante après un report du modèle | nouvelle clé apparue dans le modèle | comparer `values.yaml` avec `environments/example/values.yaml`, ajouter la clé |
| Bases saturées (`too many clients already`) après un redémarrage, connexions `postgres [local] authentication` bloquées | noyau Ubuntu 7.0.0-41 ou plus récent, AppArmor refuse les signaux dans les conteneurs | [Incident AppArmor](/k3s-lab/exploitation/incident-apparmor) |
| `make bootstrap` ou `make node-check` : « signaux refusés par AppArmor » | même cause, profil corrigé absent ou insuffisant | [Incident AppArmor](/k3s-lab/exploitation/incident-apparmor), retour arrière |
