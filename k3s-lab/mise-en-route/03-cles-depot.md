---
title: "Étape 3 · Clés, dépôt et configuration"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## 3.1 · Clé SSH : réutiliser celle de Windows

Ansible et ssh tournent dans WSL, et refusent une clé lue depuis `/mnt/c`. Copie-la donc dans WSL avec les bonnes permissions (remplace `id_ed25519` par `id_rsa` si c'est ton type de clé) :

```bash
WINUSER=$(cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')
ls /mnt/c/Users/$WINUSER/.ssh/                  # repère le nom de ta clé
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cp /mnt/c/Users/$WINUSER/.ssh/id_ed25519{,.pub} ~/.ssh/
chmod 600 ~/.ssh/id_ed25519 && chmod 644 ~/.ssh/id_ed25519.pub
ssh-keygen -lf ~/.ssh/id_ed25519.pub            # même empreinte que sous Windows
```

- Si tu as un `~/.ssh/config` côté Windows, copie-le aussi (`chmod 600 ~/.ssh/config`).
- Si la clé a une phrase de passe, le plugin `ssh-agent` de ton `.zshrc` ([Terminal WSL](/outils/terminal-wsl)) ne la demande qu'une fois par session.
- Une clé RSA convient à OVH si elle fait au moins 2048 bits (premier nombre affiché par `ssh-keygen -lf`).
- Pas de clé du tout : `ssh-keygen -t ed25519 -C "augustin@poste"`.
- C'est le contenu du `.pub` que tu colleras chez OVH en (ré)installant le VPS.

## 3.2 · Dépôt du projet · par projet

Chaque projet a son dépôt GitHub **privé**, créé depuis le modèle `augustinde/k3s-lab` (bouton **Use this template**), et cloné dans ton espace Linux :

```bash
git clone git@github.com:augustinde/<projet>.git ~/<projet>
cd ~/<projet>
git status --ignored | grep .kube        # .kube/ sera ignoré
```

Le détail, avec le choix du nom et du profil, est dans [Créer un projet](/k3s-lab/nouveau-projet). Travaille toujours dans WSL, jamais dans une copie sous `/mnt/c` ou `/mnt/f` : Ansible ignorerait `ansible.cfg` (dossier « world writable ») et les fins de ligne Windows casseraient les scripts. Pour ouvrir le dépôt depuis Windows : `code .` (VS Code avec l'extension WSL) ou l'explorateur à l'adresse `\\wsl$\Ubuntu\home\<utilisateur>\<projet>`.

## 3.3 · Clé age

Qui chiffre les secrets des projets, une seule clé pour tous :

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
age-keygen -y ~/.config/sops/age/keys.txt      # clé publique age1…
```

- `make init` (étape 3.5) met cette clé publique dans le `.sops.yaml` du projet. Le modèle contient déjà la tienne : rien à faire tant que tu gardes ce fichier.
- **Sauvegarde `keys.txt`** dans ton gestionnaire de mots de passe : sans lui, les secrets de tous les projets sont définitivement illisibles.

## 3.4 · Dépendances et contrôle

```bash
make deps      # collections Ansible + plugins helm-diff et helm-secrets
make doctor    # chaque ligne doit afficher "ok"
```

## 3.5 · Valeurs du projet · par projet

Toutes les valeurs du projet sont dans `environments/<env>/` du dépôt. `make init` les demande et crée le dossier :

```bash
make init ENV=test
```

| Valeur | Fichier | Où la trouver |
| --- | --- | --- |
| `clusterName` | `values.yaml` | À choisir : `k3s-<env>` par défaut, idéalement le nom du dépôt |
| `profile` | `values.yaml` | Mémoire du serveur : `small` (4 Go), `medium` (8 Go), `large` (16 Go et plus) |
| `zone` | `values.yaml` | Ton domaine tel qu'il apparaît dans la liste des domaines Cloudflare, sans `www` ni `https://` |
| `hostSuffix` | `values.yaml` | `-<env>` par défaut |
| `cloudflare.apiHostname` | `values.yaml` | Composé par `make init` : `k8s-api<suffixe>.<zone>` |
| `cloudflare.tunnelId` | `values.yaml` | Pas encore disponible : étape 4.2 |
| `cloudflare.privateRoute` | `values.yaml` | Une plage `/28` libre du compte Cloudflare : `10.43.0.96/28` pour le premier projet |
| IP et utilisateur SSH | `inventory.yml` | Espace client OVH, Bare Metal Cloud, VPS, ton VPS, onglet Tableau de bord : ligne IPv4. Utilisateur `ubuntu` pour l'image Ubuntu d'OVH |

Le reste du fichier décrit les namespaces (`tenants`), les bases et les services : chaque bloc est commenté.

**Applications déployées depuis GitHub (OIDC)** : déclare chaque dépôt sous le namespace où il déploie, avec l'environnement GitHub du job. Ansible en déduit la liste des dépôts acceptés par l'API ; rien à écrire ailleurs.

```yaml
tenants:
  app:
    githubDeployer: false
    githubOidc:
      - repository: augustinde/mon-api
        environment: test
```

> **Vérifier**
>
> ```bash
> make config-check
> ```
>
> La commande affiche la mémoire demandée par la plateforme, puis `Configuration cohérente.` À ce stade, seules `cloudflare.tunnelId` (étape 4.2) et l'IP du serveur, s'il n'est pas encore installé, peuvent rester en erreur.
{.is-success}

## 3.6 · Fichier de secrets · par projet

`make init` a créé `environments/<env>/secrets.yaml`, déjà chiffré. Pour le remplir :

```bash
make secrets-edit     # l'ouvre en clair dans nano ; il est rechiffré à la fermeture
```

| Clé | Quand | Valeur |
| --- | --- | --- |
| `grafana.adminPassword` | Maintenant | Mot de passe de ton choix |
| `rancher.bootstrapPassword` | Maintenant | Mot de passe de ton choix (Rancher demande de le changer au premier login) ; inutile en profil `small` |
| `cloudflare.tunnelCredentials` | Étape 4.2 | Contenu de `~/.cloudflared/<ID>.json` (obligatoire, sinon `make platform` échoue) |
| `cloudflare.accessClientId` / `accessClientSecret` | Facultatif | Seulement pour `make deploy-test` ; le déploiement GitHub, lui, lit ces valeurs dans ses propres secrets |
| `alerting.discordWebhookUrl` | Procédure, étape 12 | Vide = alertes désactivées |
| Bloc `tenants` (registry, appSecrets) | Facultatif | Pull d'images GHCR privées, secrets applicatifs ; à supprimer s'il ne sert pas |

Le fichier chiffré se commite. Vérifie-le avec `head -5 $ENV_DIR/secrets.yaml` : les valeurs doivent commencer par `ENC[`.

Si un `secrets.yaml` manque (environnement créé à la main) : `make secrets-init` le crée depuis le modèle, déjà chiffré.
