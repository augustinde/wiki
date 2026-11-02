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

- **Surface exposée** : seul le port 22 est ouvert. ServiceLB est désactivé, donc aucun Service Kubernetes ne peut publier de port sur l'IP du VPS, même par erreur.
- **SSH** : connexion par clé uniquement, root interdit, un seul utilisateur autorisé, fail2ban actif.
- **Interfaces** : Rancher, Grafana et l'API Kubernetes sont filtrés par Cloudflare Access avant d'atteindre le cluster. Rancher et Grafana exigent en plus un poste connecté à WARP ; les bases ne sont joignables que par WARP ou SSH.
- **Secrets** : chiffrés au repos dans k3s (`secrets-encryption`) et dans Git (SOPS + age).
- **Namespaces applicatifs** : Pod Security « restricted », quotas, compte GitHub limité et à durée de vie courte.

## Budget mémoire

Le VPS-2 (8 Go) est le minimum avec Rancher et le monitoring.

| Composant | RAM approximative |
| --- | --- |
| Système et réservations k3s | 1 Go |
| k3s, Traefik, CoreDNS, cloudflared × 2 | 0,7 Go |
| Rancher (avec fleet et webhook) | 1,5 Go |
| Prometheus + Grafana + exporters | 1,2 Go |
| Loki + Alloy | 0,5 Go |
| CloudNativePG + base `app` | 0,4 Go |
| Application Spring Boot | 0,5 à 0,8 Go |
| **Total** | **5,8 à 6,1 Go sur 8** |

Sur un VPS-1 (4 Go), passe `rancher.enabled`, `monitoring.enabled` et `logging.enabled` à `false`.

L'environnement prod (étape 11) ajoute environ 1 Go (base `app-prod`, api-demo, front-demo) et Alertmanager environ 50 Mo. Si la mémoire dépasse 90 %, l'alerte `MemoireNoeudSaturee` prévient ; Rancher est le premier candidat à couper.

## Dépannage

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| `make` répond « API k3s injoignable » | tunnel SSH fermé, ou kubeconfig absent après un reset | `make tunnel` ; si besoin `make bootstrap` |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | VPS réinstallé, nouvelle empreinte | `make forget-host` |
| Erreur Cloudflare 1033 | cloudflared non connecté | `kubectl -n cloudflared logs deploy/cloudflared` ; vérifier l'ID et le JSON du tunnel |
| 404 sur un nom d'hôte | Ingress absent, ou nom différent de celui de l'Ingress | `kubectl get ingress -A` |
| Rancher boucle en redirection | `X-Forwarded-Proto` perdu | vérifier le HelmChartConfig Traefik (`trustedIPs`), puis `make bootstrap` |
| Erreur de certificat sur un nom à 2 niveaux | certificat Cloudflare limité à `*.zone` | utiliser `nom-test.zone`, pas `nom.test.zone` |
| Pod refusé « violates PodSecurity restricted » | image root ou contexte de sécurité absent | régler `podSecurityContext.runAsUser` sur un UID non-root |
| Pod refusé « exceeded quota » | quota du namespace atteint | augmenter `tenants.<ns>.quota` |
| `deploy-test` : « websocket: bad handshake » | jeton Access refusé, ou politique autre que Service Auth | vérifier l'application Access de `k8s-api-test` |
| `deploy-test` : « Unauthorized » | jeton Kubernetes expiré ou révoqué | `make deployer-kubeconfig` |
| Job GitHub : « Unauthorized » | dépôt absent de github\_oidc\_repositories, job sans environment, audience différente de k3s-test, ou K8S\_CA\_DATA périmée | corriger puis make bootstrap ; après un rebuild, make ci-ca |
| Job GitHub : « forbidden … User github:… » | identité reconnue mais sans droits dans ce namespace | ajouter {repository, environment} sous tenants.\<ns>.githubOidc, puis make platform ; vérifier avec make oidc-test |
| Rien dans Grafana Explore > Loki | Alloy ne joint pas Loki, ou Loki pas encore prêt | kubectl -n logging logs deploy/alloy et kubectl -n logging get pods |
| Rancher refuse de s'installer (kubeVersion) | k3s hors de la matrice Rancher | garder le canal `v1.34`, ou ajuster `versions.rancher` |
| Pods `Evicted` ou `OOMKilled` | mémoire saturée | VPS-2, ou désactiver Rancher ou le monitoring |
