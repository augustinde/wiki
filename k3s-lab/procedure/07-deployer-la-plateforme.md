---
title: "Étape 7 · Déployer la plateforme"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

`make platform` installe 7 releases Helm : cloudflared, le namespace `demo`, l'opérateur PostgreSQL, Grafana, Loki, Alloy et Rancher. Compte 10 à 15 minutes, dont 5 à 8 pour Rancher. Pour suivre l'avancement, ouvre un second terminal avec `watch kubectl get pods -A`.

## 7.1 · Lancer le déploiement

```bash
make platform
```

> **Vérifier** · La commande se termine par la liste `UPDATED RELEASES`, sans erreur. Ensuite :
>
> ```bash
> helm list -A
> ```
>
> Les 7 releases (`cloudflared`, `tenant-demo`, `cnpg`, `monitoring`, `loki`, `alloy`, `rancher`) sont en `deployed`. Si Rancher dépasse le délai, relance `make platform` : il reprend où il s'est arrêté.
>
> **Erreur `no matches for kind "Prometheus"` … `ensure CRDs are installed first`, puis `plugin "diff" exited with error`** : avant d'installer, helmfile calcule un diff de chaque release. Sur un cluster neuf, le diff de kube-prometheus-stack échoue, car ses types (`Prometheus`, `ServiceMonitor`…) ne sont créés qu'à l'installation. Rien n'a été installé. Les `apply` du `Makefile` doivent sauter le diff pour une première installation :
>
> ```bash
> sed -i 's/ apply$/ apply --skip-diff-on-install/' Makefile
> grep -n 'apply --skip-diff-on-install' Makefile
> ```
>
> Le `grep` doit lister 6 lignes (platform, data, apps, app-redeploy, db-reset, deployer-revoke). Relance ensuite `make platform`. Pour la même raison, `make diff` échoue tant que la plateforme n'est pas installée.
{.is-success}

## 7.2 · Contrôler le tunnel Cloudflare

```bash
kubectl -n cloudflared get pods
kubectl -n cloudflared logs deploy/cloudflared | grep -c 'Registered tunnel connection'
cloudflared tunnel info k3s-test
```

> **Vérifier** · 2 pods en `Running`, un compteur d'au moins 1, et 2 connecteurs listés par `tunnel info`. Dans Zero Trust, Networks, Tunnels, `k3s-test` passe à **Healthy**. Les noms en `-test` ne renvoient plus l'erreur 1033.
{.is-success}

## 7.3 · Contrôler le namespace `demo`

```bash
kubectl get ns demo -o jsonpath='{.metadata.labels}'; echo
kubectl -n demo get resourcequota,limitrange,rolebinding
```

> **Vérifier** · Les labels contiennent `pod-security.kubernetes.io/enforce":"restricted`. Le namespace a un ResourceQuota, un LimitRange et le RoleBinding `github-deployer`.
{.is-success}

## 7.4 · Contrôler les droits GitHub

Cette commande simule l'identité qu'aura le job GitHub.

```bash
make oidc-test REPO=augustinde/api-demo
make oidc-test REPO=augustinde/front-demo
```

> **Vérifier** · Pour chaque dépôt :
>
> | Ligne | Attendu |
> | --- | --- |
> | déployer dans demo | `yes` |
> | lire les pods de kube-system | `no` |
> | exec dans un pod de demo | `no` |
> | lister les nœuds | `no` |
> | OIDC activé sur l'API | `oui` |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 7.5 · Contrôler le chemin de GitHub vers l'API

GitHub atteindra l'API par `k8s-api-test`, derrière Cloudflare Access. Ce test reproduit ce chemin avec le jeton `github-deploy`.

```bash
S=helmfile/environments/test.secrets.yaml
API=$(sed -n 's/^  apiHostname: *//p' helmfile/environments/test.yaml | tr -d '"')
echo "$API"        # doit afficher k8s-api-test.<ton domaine>
cloudflared access tcp --hostname "$API" --url 127.0.0.1:16443 \
  --service-token-id "$(sops -d --extract '["cloudflare"]["accessClientId"]' $S)" \
  --service-token-secret "$(sops -d --extract '["cloudflare"]["accessClientSecret"]' $S)" &
sleep 5
curl -sk https://127.0.0.1:16443/version; echo
kill %1
```

> **Vérifier** · `curl` renvoie le JSON de version de Kubernetes, avec `"gitVersion": "v1.34.x+k3s1"`. Cloudflare a donc laissé passer le jeton jusqu'à l'API. Cette page est publique dans k3s ; tout le reste exigera l'identité GitHub. Une réponse vide, une erreur SSL ou `websocket: bad handshake` dans le journal de cloudflared signifient que Cloudflare a refusé le jeton : vérifie la politique `GitHub deploy` (Mise en route, 4.7).
>
> **Contre-épreuve, sans jeton** : `curl -sI https://$API | head -1` répond `302` ou `403`. Cloudflare Access bloque avant le cluster.
{.is-success}

## 7.6 · Contrôler l'opérateur PostgreSQL

```bash
kubectl -n cnpg-system get pods
```

> **Vérifier** · Un pod `cnpg-cloudnative-pg-…` en `1/1 Running`. Sans lui, l'étape 8 ne créera pas la base.
{.is-success}

## 7.7 · Contrôler Grafana

```bash
kubectl -n monitoring get pods
sops -d --extract '["grafana"]["adminPassword"]' helmfile/environments/test.secrets.yaml; echo
```

> **Vérifier**
>
> 1. Tous les pods sont en `Running`, colonne READY complète (par exemple `2/2`).
> 2. WARP connecté, ouvre `https://grafana-test.tondomaine.fr` : page Cloudflare Access, email, code, puis connexion Grafana avec `admin` et le mot de passe affiché.
> 3. Dashboards, **Kubernetes / Compute Resources / Cluster** : des courbes s'affichent.
> 4. WARP déconnecté, en navigation privée : accès refusé par Cloudflare.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 7.8 · Contrôler les logs

```bash
kubectl -n logging get pods
```

> **Vérifier** · Un pod `loki-…` et un pod `alloy-…` en `Running`. Dans Grafana, **Explore**, source **Loki**, requête `{namespace="cloudflared"}`, puis **Run query** : des lignes de log récentes s'affichent.
{.is-success}

## 7.9 · Contrôler Rancher

```bash
kubectl -n cattle-system rollout status deploy/rancher --timeout=10m
sops -d --extract '["rancher"]["bootstrapPassword"]' helmfile/environments/test.secrets.yaml; echo
```

> **Vérifier**
>
> 1. `deployment "rancher" successfully rolled out`.
> 2. WARP connecté, ouvre `https://rancher-test.tondomaine.fr` : page Access, puis Rancher demande le mot de passe affiché.
> 3. Choisis un nouveau mot de passe, et garde l'URL proposée (`https://rancher-test.tondomaine.fr`).
> 4. Le cluster `local` apparaît en **Active**.
> 5. WARP déconnecté : accès refusé, comme pour Grafana.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}
