---
title: "Étape 1 · Contrôler le poste et le dépôt"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Rien ne touche encore au VPS. Ces contrôles évitent de découvrir un oubli au milieu du bootstrap. Toutes les commandes de la procédure se lancent dans WSL, depuis `~/k3s-lab`.

## 1.1 · Outils

```bash
cd ~/k3s-lab
make doctor
```

> **Vérifier** · Chaque ligne commence par `ok`, y compris `helm-diff`, `helm-secrets` et `clé age`. Une ligne `MANQUE` : [Mise en route, étape 2](/k3s-lab/mise-en-route/02-outils).
{.is-success}

## 1.2 · Domaine et tunnel dans `test.yaml`

```bash
grep -E '^(zone|hostSuffix):|tunnelId:|apiHostname:' helmfile/environments/test.yaml
cloudflared tunnel list
```

> **Vérifier**
>
> | Ligne | Valeur attendue |
> | --- | --- |
> | `zone` | ton domaine, par exemple `tondomaine.fr` |
> | `hostSuffix` | `"-test"` |
> | `tunnelId` | l'ID du tunnel `k3s-test` affiché par `cloudflared tunnel list` |
> | `apiHostname` | `k8s-api-test.tondomaine.fr` |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 1.3 · Dépôts GitHub autorisés (OIDC)

L'API k3s n'acceptera que les jobs des dépôts listés, et seulement dans le namespace `demo`.

```bash
grep -A2 '^github_oidc_repositories' ansible/group_vars/all.yml
grep -A1 'repository: augustinde' helmfile/environments/test.yaml
```

> **Vérifier** · `augustinde/api-demo` et `augustinde/front-demo` apparaissent dans les deux fichiers, chacun suivi de `environment: test` dans `test.yaml`. Sinon, édite les deux fichiers :
>
> ```yaml
> # ansible/group_vars/all.yml
> github_oidc_repositories:
>   - augustinde/api-demo
>   - augustinde/front-demo
>
> # helmfile/environments/test.yaml, sous tenants.demo
>     githubDeployer: false
>     githubOidc:
>       - repository: augustinde/api-demo
>         environment: test
>       - repository: augustinde/front-demo
>         environment: test
> ```
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 1.4 · Secrets

```bash
sops -d helmfile/environments/test.secrets.yaml | grep -v '^\s*#' | grep -c CHANGE-ME
sops -d --extract '["cloudflare"]["tunnelCredentials"]' helmfile/environments/test.secrets.yaml | grep -o '"TunnelID":"[^"]*"'
```

> **Vérifier** · La première commande affiche `0`. La seconde affiche le même ID que `tunnelId`. Un `CHANGE-ME` restant : `make secrets-edit`, puis complète les identifiants du tunnel, le jeton `github-deploy` (4.6 de la mise en route) et les mots de passe Grafana et Rancher.
{.is-success}

## 1.5 · Cloudflare

```bash
cloudflared tunnel route ip show
```

> **Vérifier** · La plage `10.43.0.96/28` est routée vers `k3s-test`. Dans Zero Trust, Access controls, Applications : les trois applications existent, et **k3s API test** a les politiques `Augustin uniquement` et `GitHub deploy`.
{.is-success}
