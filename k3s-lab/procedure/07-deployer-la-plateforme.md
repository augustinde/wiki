---
title: "Étape 7 · Déployer la plateforme"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

`make platform` installe les releases Helm de la plateforme : cloudflared, un `tenant-<ns>` par namespace applicatif, l'opérateur PostgreSQL, le monitoring (Prometheus, Grafana, Alertmanager), Loki, Alloy et Rancher (absent en profil `small`). Compte 10 à 15 minutes, dont 5 à 8 pour Rancher. Pour suivre l'avancement, ouvre un second terminal avec `watch kubectl get pods -A`.

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
> Les releases `cloudflared`, `tenant-<ns>` (une par namespace), `cnpg`, `monitoring`, `loki`, `alloy` et `rancher` sont en `deployed`. Si Rancher dépasse le délai, relance `make platform` : il reprend où il s'est arrêté.
>
> **Erreur `no matches for kind "Prometheus"` … `ensure CRDs are installed first`** : le `Makefile` du projet est antérieur au modèle actuel, qui saute le diff à la première installation (`apply --skip-diff-on-install`). Reporte le modèle ([Créer un projet](/k3s-lab/nouveau-projet), dernière section). Pour la même raison, `make diff` échoue tant que la plateforme n'est pas installée.
{.is-success}

## 7.2 · Contrôler le tunnel Cloudflare

```bash
kubectl -n cloudflared get pods
kubectl -n cloudflared logs deploy/cloudflared | grep -c 'Registered tunnel connection'
cloudflared tunnel info $CLUSTER
```

> **Vérifier** · 2 pods en `Running`, un compteur d'au moins 1, et 2 connecteurs listés par `tunnel info`. Dans Zero Trust, Networks, Tunnels, `<cluster>` passe à **Healthy**. Les noms du projet ne renvoient plus l'erreur 1033. Le nombre de pods suit le profil (`resources.cloudflared.replicas`, 2 par défaut).
{.is-success}

## 7.3 · Contrôler les namespaces applicatifs

```bash
kubectl get ns $NS -o jsonpath='{.metadata.labels}'; echo
kubectl -n $NS get resourcequota,limitrange,rolebinding
```

> **Vérifier** · Les labels contiennent `pod-security.kubernetes.io/enforce":"restricted`. Le namespace a un ResourceQuota, un LimitRange et, si des dépôts y sont déclarés, le RoleBinding `github-deployer`. Même contrôle pour les autres namespaces de `tenants` avec `NS=<ns>`.
{.is-success}

## 7.4 · Contrôler les droits GitHub

Cette commande simule l'identité qu'aura le job GitHub.

```bash
make oidc-test REPO=<propriétaire>/<dépôt>
```

> **Vérifier** · Pour chaque dépôt déclaré dans `$NS` (k3s-test : `augustinde/api-demo` et `augustinde/front-demo`) :
>
> | Ligne | Attendu |
> | --- | --- |
> | déployer dans <ns> | `yes` |
> | lire les pods de kube-system | `no` |
> | exec dans un pod de <ns> | `no` |
> | lister les nœuds | `no` |
> | OIDC activé sur l'API | `oui` |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 7.5 · Contrôler le chemin de GitHub vers l'API

GitHub atteindra l'API par `$API_HOST`, derrière Cloudflare Access. Ce test reproduit ce chemin avec le jeton `github-deploy`.

```bash
S=$ENV_DIR/secrets.yaml
echo "$API_HOST"        # k8s-api<suffixe>.<zone>
cloudflared access tcp --hostname "$API_HOST" --url 127.0.0.1:16443 \
  --service-token-id "$(sops -d --extract '["cloudflare"]["accessClientId"]' $S)" \
  --service-token-secret "$(sops -d --extract '["cloudflare"]["accessClientSecret"]' $S)" &
sleep 5
curl -sk https://127.0.0.1:16443/version; echo
kill %1
```

> **Vérifier** · `curl` renvoie le JSON de version de Kubernetes, avec `"gitVersion": "v1.34.x+k3s1"`. Cloudflare a donc laissé passer le jeton jusqu'à l'API. Cette page est publique dans k3s ; tout le reste exigera l'identité GitHub. Une réponse vide, une erreur SSL ou `websocket: bad handshake` dans le journal de cloudflared signifient que Cloudflare a refusé le jeton : vérifie la politique `GitHub deploy` (Mise en route, 4.7).
>
> **Contre-épreuve, sans jeton** : `curl -sI https://$API_HOST | head -1` répond `302` ou `403`. Cloudflare Access bloque avant le cluster.
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
sops -d --extract '["grafana"]["adminPassword"]' $ENV_DIR/secrets.yaml; echo
```

> **Vérifier**
>
> 1. Tous les pods sont en `Running`, colonne READY complète (par exemple `2/2`).
> 2. WARP connecté, ouvre `https://grafana<suffixe>.<zone>` : page Cloudflare Access, email, code, puis connexion Grafana avec `admin` et le mot de passe affiché.
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

En profil `small` (ou avec `rancher.enabled: false`), Rancher n'est pas installé : passe à l'étape 8.

```bash
kubectl -n cattle-system rollout status deploy/rancher --timeout=10m
sops -d --extract '["rancher"]["bootstrapPassword"]' $ENV_DIR/secrets.yaml; echo
```

> **Vérifier**
>
> 1. `deployment "rancher" successfully rolled out`.
> 2. WARP connecté, ouvre `https://rancher<suffixe>.<zone>` : page Access, puis Rancher demande le mot de passe affiché.
> 3. Choisis un nouveau mot de passe, et garde l'URL proposée (`https://rancher<suffixe>.<zone>`).
> 4. Le cluster `local` apparaît en **Active**.
> 5. WARP déconnecté : accès refusé, comme pour Grafana.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}
