---
title: "Étape 1 · Contrôler le poste et le dépôt"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Rien ne touche encore au serveur. Ces contrôles évitent de découvrir un oubli au milieu du bootstrap. Toutes les commandes de la procédure se lancent dans WSL, depuis le dossier du projet.

## 1.1 · Outils et variables

```bash
cd ~/<projet>
make doctor
eval "$(make -s vars)" && echo "$ENV $CLUSTER $IP"
```

> **Vérifier** · Chaque ligne de `doctor` commence par `ok`, y compris `helm-diff`, `helm-secrets` et `clé age`. Une ligne `MANQUE` : [Mise en route, étape 2](/k3s-lab/mise-en-route/02-outils). La dernière ligne affiche l'environnement, le cluster et l'IP du serveur (vide ou `0.0.0.0` si le serveur n'est pas encore installé : étape 3.1).
{.is-success}

## 1.2 · Configuration du projet

```bash
make config-check
cloudflared tunnel list | grep "$CLUSTER"
```

> **Vérifier**
>
> - `config-check` affiche le cluster, le profil et le nœud attendus, la mémoire demandée par la plateforme, puis `Configuration cohérente.` Une IP encore à renseigner est admise jusqu'à l'étape 3.1.
> - `cloudflared tunnel list` montre le tunnel `<cluster>`, avec le même ID que `echo $TUNNEL_ID`.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}

## 1.3 · Dépôts GitHub autorisés (OIDC)

L'API k3s n'acceptera que les jobs des dépôts déclarés, et seulement dans leur namespace.

```bash
grep -B1 -A1 'repository:' $ENV_DIR/values.yaml
```

> **Vérifier** · Chaque dépôt d'application apparaît sous le namespace où il déploie, suivi de son environnement GitHub (`environment: test`). Sinon, complète `values.yaml` :
>
> ```yaml
> tenants:
>   <ns>:
>     githubDeployer: false
>     githubOidc:
>       - repository: <propriétaire>/<dépôt>
>         environment: test
> ```
>
> Ansible en déduit la liste des dépôts acceptés par l'API : rien d'autre à modifier.
{.is-success}

## 1.4 · Secrets

```bash
sops -d $ENV_DIR/secrets.yaml | grep -v '^\s*#' | grep -c CHANGE-ME
sops -d --extract '["cloudflare"]["tunnelCredentials"]' $ENV_DIR/secrets.yaml | grep -o '"TunnelID":"[^"]*"'
```

> **Vérifier** · La première commande affiche `0`. La seconde affiche le même ID que `$TUNNEL_ID`. Un `CHANGE-ME` restant : `make secrets-edit`, puis complète les identifiants du tunnel, le jeton `github-deploy` (4.6 de la mise en route), les mots de passe Grafana et Rancher, ou supprime le bloc `tenants` s'il ne sert pas.
{.is-success}

## 1.5 · Cloudflare

```bash
cloudflared tunnel route ip show | grep "$ROUTE"
```

> **Vérifier** · La plage `$ROUTE` est routée vers le tunnel du projet. Dans Zero Trust, Access controls, Applications : les applications du projet existent, et **k3s API <cluster>** a les politiques `Augustin uniquement` et `GitHub deploy`.
{.is-success}
