---
title: "Créer un projet depuis le modèle"
description: "Nouveau dépôt depuis k3s-lab, valeurs du projet, profil de machine, Cloudflare du projet"
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, projet
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Un projet, c'est un dépôt créé depuis le modèle `k3s-lab`, un serveur et un tunnel Cloudflare. Compte une heure avant de toucher au serveur.

**Prérequis** : la [Mise en route](/k3s-lab/mise-en-route) est faite une fois pour le poste et le compte Cloudflare : WSL2 et outils (étapes 1 et 2), clé SSH, clé age et dépendances (3.1, 3.3, 3.4), domaine, Zero Trust, HTTPS et jeton GitHub (4.0, 4.1, 4.5, 4.6), WARP (étape 5). Ce qui est propre au projet est repris ci-dessous.

## 1 · Choisir le profil

Le profil fixe la taille de chaque composant d'après la mémoire du serveur. Les fichiers sont dans `profiles/` du dépôt.

| Profil | Machine | Contenu | Rétention métriques / logs |
| --- | --- | --- | --- |
| `small` | 4 Go, 2 vCPU (OVH VPS-1) | plateforme sans Rancher, composants réduits | 2 jours / 48 h |
| `medium` | 8 Go, 4 vCPU (OVH VPS-2) | plateforme complète | 3 jours / 72 h |
| `large` | 16 Go et plus (dédié) | plateforme complète, plus de marge | 15 jours / 168 h |

Pour une machine entre deux profils, prends le plus proche et surcharge dans `values.yaml`, sous la même clé que dans le profil :

```yaml
rancher:
  enabled: false            # libère environ 1,5 Go
monitoring:
  retention: 7d
resources:
  grafana:
    limits: { memory: 1Gi }
```

## 2 · Créer le dépôt

1. Sur GitHub, ouvre `augustinde/k3s-lab`, bouton **Use this template**, **Create a new repository**.
2. Nom : celui du cluster, par exemple `k3s-client`. Visibilité **Private**. **Create repository**.
3. Dans WSL :

```bash
git clone https://github.com/augustinde/<projet>.git ~/<projet>
cd ~/<projet>
make help | tail -2
```

> **Vérifier** · `make help` se termine par `Environnement : à préciser avec ENV=<nom>   (disponibles : aucun)`. Le dépôt ne contient que le modèle, `environments/example/`.
{.is-success}

## 3 · Créer l'environnement

Un environnement par serveur. Son nom (`test`, `prod`, `client`…) devient le dossier `environments/<env>/`.

```bash
make init ENV=<env>
```

| Question | Réponse |
| --- | --- |
| Nom du cluster | `k3s-<env>` par défaut ; idéalement le nom du dépôt. Sert de contexte kubectl, de nom de tunnel et d'audience OIDC |
| Profil | `small`, `medium` ou `large` (étape 1) |
| IP publique, utilisateur SSH | Espace client OVH, tableau de bord du VPS, ligne IPv4 ; `ubuntu` sur les images OVH. Si le serveur n'existe pas encore, mets `0.0.0.0` et corrige `inventory.yml` plus tard |
| Nom du nœud | `vps-<env>` par défaut |
| Zone DNS | ton domaine chez Cloudflare, sans `www` |
| Suffixe | `-<env>` par défaut : `grafana-<env>.<zone>`. `-` pour aucun suffixe (un seul environnement sur la zone) |
| Namespace | namespace des applications, `app` par défaut |
| Route privée WARP | une plage `/28` **différente pour chaque projet** du compte Cloudflare (voir plus bas) |
| Port local du tunnel SSH | `6443`, ou un autre port pour ouvrir ce projet en même temps qu'un autre |
| ID du tunnel | Entrée : le tunnel se crée à l'étape 5 |

La commande crée `environments/<env>/` (values, inventaire, base `app`, service `whoami`), met la clé publique age de ton poste dans `.sops.yaml` si elle n'y est pas, puis crée `secrets.yaml` déjà chiffré.

> **Vérifier**
>
> ```bash
> make envs
> eval "$(make -s vars)" && echo "$CLUSTER $ZONE $SUFFIX $ROUTE"
> head -3 environments/$ENV/secrets.yaml
> ```
>
> `make envs` liste l'environnement avec son cluster, son profil et sa cible SSH. Les variables reprennent tes réponses. La dernière commande affiche des lignes `#ENC[…]` : le fichier est chiffré.
{.is-success}

## 4 · Décrire le projet

Dans `environments/<env>/values.yaml`, chaque bloc est commenté :

- **`tenants`** : un namespace par application ou par environnement applicatif (`app`, `app-prod`), avec son quota. Un namespace qui se termine par `-prod` est étiqueté 🔴 PROD dans les alertes.
- **`githubOidc`** sous chaque namespace : les dépôts GitHub autorisés à y déployer, avec l'environnement GitHub (`test`, `prod`). Ansible en déduit la liste de l'API k3s ; rien à écrire ailleurs.
- **`databases`** : une base par entrée, plus un fichier `databases/<nom>.yaml`. Chaque base d'administration prend une IP de la route privée (la première base a la 5ᵉ adresse, par exemple `10.43.0.100` pour `10.43.0.96/28`).
- **`apps`** : services déployés par Helmfile, plus un fichier `apps/<nom>.yaml`.
- **`publicBuckets`** : buckets S3 OVH de fichiers publics, servis sur `<nom><suffixe>.<zone>` avec le cache Cloudflare ([procédure, étape 15](/k3s-lab/procedure/15-stockage-s3)).

```bash
make config-check
```

> **Vérifier** · La commande affiche la mémoire demandée par la plateforme, puis `Configuration cohérente.` À ce stade, seules `cloudflare.tunnelId` (étape 5) et l'IP d'un serveur pas encore installé peuvent rester en erreur. Le total doit laisser de la place aux applications : environ 3,2 Go en `medium` pour 8 Go, 1,5 Go en `small` pour 4 Go.
{.is-success}

## 5 · Cloudflare du projet

Dans la [Mise en route, étape 4](/k3s-lab/mise-en-route/04-cloudflare), refais les sections marquées **par projet**, variables chargées :

| Section | Ce qui est créé |
| --- | --- |
| 4.2 | tunnel `<cluster>`, son ID dans `values.yaml`, ses identifiants dans `secrets.yaml` ; nouveau compte Cloudflare : son certificat d'abord (section « Projets sur plusieurs comptes Cloudflare ») |
| 4.3 | route privée `<route>` vers le tunnel |
| 4.4 | noms DNS du projet vers le tunnel |
| 4.7 | applications Access Rancher, Grafana et API du projet |

Puis complète les secrets ([Mise en route, 3.6](/k3s-lab/mise-en-route/03-cles-depot)) et relance `make config-check` : `Configuration cohérente.`

## 6 · Commiter, puis installer

```bash
git add -A && git commit -m "Environnement $ENV" && git push
```

Le serveur s'installe ensuite avec la [Procédure](/k3s-lab/procedure), à partir de l'étape 1. Crée enfin la fiche du projet dans ce wiki, `k3s-lab/projets/<cluster>`, en copiant celle de [k3s-test](/k3s-lab/projets/k3s-test), et ajoute-la au tableau de la page [k3s-lab](/k3s-lab).

## Plusieurs projets sur un même compte Cloudflare

| Élément | Partagé ou par projet | Règle |
| --- | --- | --- |
| Zone, Zero Trust, WARP, politiques Access, jeton `github-deploy` | partagés | rien à refaire ; projets sur des comptes différents : section suivante |
| Tunnel | par projet | nommé comme le cluster |
| DNS | par projet | le joker `*` ne vise qu'un tunnel : il sert au premier projet de la zone ; les suivants déclarent chaque nom (4.4) ou utilisent une autre zone |
| Route privée | par projet | `/28` sans chevauchement : `10.43.0.96/28`, puis `10.43.0.112/28`, `10.43.0.128/28`… |
| Exclusions WARP (liste précise, 5.3) | à ajuster | deuxième projet en `10.43.0.112/28` : retirer cette ligne de la liste ; au-delà, passer en mode simple |
| Applications Access | par projet | `Rancher <cluster>`, `Grafana <cluster>`, `k3s API <cluster>` |
| Port du tunnel SSH | par projet si ouverts ensemble | `localApiPort` : `6443`, `6444`… |

Chaque projet a son kubeconfig, `.kube/<cluster>.yaml` dans son dossier : `make` choisit toujours le bon.

## Projets sur plusieurs comptes Cloudflare

Deux projets peuvent vivre sur deux comptes Cloudflare (le tien et celui d'un client, par exemple). Sur ton PC, seuls deux éléments dépendent du compte ; le reste est déjà propre à chaque projet.

| Élément | Lié au compte ? | À faire |
| --- | --- | --- |
| `cloudflared` dans WSL (`tunnel create`, `route ip`, `route dns`, `tunnel list`) | oui : un certificat par compte | un certificat par projet, choisi par `projet <dossier>` (1) |
| Tunnel dans le cluster | non : identifiants dans le `secrets.yaml` du projet | rien |
| Jeton `github-deploy`, `make deploy-test` | non : créés dans le compte du projet, gardés dans ses secrets | un jeton par compte (Mise en route, 4.6) |
| Tableau de bord Cloudflare | un compte par session | invite ton email comme membre du second compte (Manage Account, **Members**) : un sélecteur de compte apparaît ; sinon, deux profils de navigateur |
| Client WARP | oui : une organisation Zero Trust active à la fois | bascule entre organisations, ou WARP réservé à un compte (2) |

Les routes privées des deux comptes sont indépendantes : chaque projet peut garder `10.43.0.96/28`. La règle d'une plage distincte ne vaut qu'à l'intérieur d'un même compte.

### 1 · Un certificat `cloudflared` par compte

`cloudflared tunnel login` écrit toujours `~/.cloudflared/cert.pem`, et `cloudflared` lit un autre certificat si la variable `TUNNEL_ORIGIN_CERT` l'indique. On range donc un certificat par projet, dans le dossier du cluster :

```bash
# Certificat actuel : celui du compte du premier projet
mkdir -p ~/.cloudflared/<cluster-1>
mv ~/.cloudflared/cert.pem ~/.cloudflared/<cluster-1>/cert.pem

# Connexion au second compte : choisis un domaine de ce compte dans le navigateur
cloudflared tunnel login
mkdir -p ~/.cloudflared/<cluster-2>
mv ~/.cloudflared/cert.pem ~/.cloudflared/<cluster-2>/cert.pem
```

La fonction `projet` ([procédure, 5.2](/k3s-lab/procedure/05-piloter-le-cluster-depuis-ton-poste)) exporte `TUNNEL_ORIGIN_CERT` quand `~/.cloudflared/<cluster>/cert.pem` existe. Deux projets sur un même compte partagent le même certificat : copie-le dans les deux dossiers.

> **Vérifier** · Après `exec zsh` : `projet <dossier-1>`, puis `cloudflared tunnel list` affiche le tunnel du premier projet ; `projet <dossier-2>`, puis `cloudflared tunnel list`, uniquement les tunnels du second compte.
{.is-success}

### 2 · WARP avec plusieurs organisations Zero Trust

Chaque compte a son organisation Zero Trust, et le client WARP n'est connecté qu'à une à la fois : on les déclare toutes dans le client et on bascule, ou on réserve WARP à un compte. Les deux options : [Plusieurs organisations WARP sur un PC](/k3s-lab/acces/plusieurs-organisations-warp). La connexion par GitHub se met en place dans chaque organisation, avec la même application OAuth : [Connexion par GitHub](/k3s-lab/acces/connexion-github), étape 3.

## Reporter une amélioration du modèle

Le code commun ne contient aucune valeur de projet : il se remplace en bloc, sans toucher à `environments/`.

```bash
cd ~/<projet>
git remote add modele https://github.com/augustinde/k3s-lab.git     # une fois
git fetch modele
git checkout modele/main -- Makefile ansible helmfile profiles scripts ci renovate.json
git diff --cached --stat
make config-check && make diff
git commit -m "Mise à jour depuis le modèle" && git push
```

> **Vérifier** · `make config-check` répond `Configuration cohérente.` et `make diff` ne montre que les changements annoncés par le modèle. Puis `make bootstrap` si `ansible/` a changé, `make platform` si `helmfile/` ou `profiles/` ont changé.
{.is-success}
