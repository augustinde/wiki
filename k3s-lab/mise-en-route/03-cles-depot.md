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
- Si la clé a une phrase de passe, le plugin `ssh-agent` de ton `.zshrc` (étape 1.3) ne la demande qu'une fois par session.
- Une clé RSA convient à OVH si elle fait au moins 2048 bits (premier nombre affiché par `ssh-keygen -lf`).
- Pas de clé du tout : `ssh-keygen -t ed25519 -C "augustin@poste"`.
- C'est le contenu du `.pub` que tu colleras chez OVH en (ré)installant le VPS.

## 3.2 · Dépôt : copie depuis Windows, permissions, fins de ligne

Copie le dossier dans ton espace Linux (avec `-r`, c'est un dossier) :

```bash
cp -r /mnt/f/Projects/k3s-lab ~/
cd ~/k3s-lab
```

Les fichiers venus de Windows arrivent avec des permissions ouvertes à tous. Remets-les propres, sinon Ansible **ignore `ansible.cfg`** (avertissement « world writable directory ») et `make bootstrap` part sans inventaire :

```bash
find ~/k3s-lab -type d -exec chmod 755 {} +
find ~/k3s-lab -type f -exec chmod 644 {} +
chmod +x ~/k3s-lab/scripts/*.sh ~/k3s-lab/ci/*.sh
```

Vérifie les fins de ligne : des fichiers passés par un éditeur ou un git Windows peuvent être en CRLF, et les scripts échouent alors avec `\r: command not found`.

```bash
grep -rlI $'\r' ~/k3s-lab | head                                    # rien = tout va bien
grep -rlI $'\r' ~/k3s-lab | xargs -r sed -i 's/\r$//'                # correction si besoin
```

Versionne-le dans un dépôt GitHub **privé** :

```bash
git init && git add . && git commit -m "Environnement k3s rejouable"
git status --ignored | grep .kube        # .kube/ doit être ignoré
```

Désormais, travaille uniquement dans `~/k3s-lab`. Pour l'ouvrir depuis Windows : `code .` (VS Code avec l'extension WSL) ou l'explorateur à l'adresse `\\wsl$\Ubuntu\home\serkox\k3s-lab`.

## 3.3 · Clé age

Qui chiffre les secrets du dépôt :

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
grep 'public key' ~/.config/sops/age/keys.txt
```

- Remplace la clé `age1remplacez…` de `.sops.yaml` par ta clé publique (`age1…`).
- **Sauvegarde `keys.txt`** dans ton gestionnaire de mots de passe : sans lui, les secrets du dépôt sont définitivement illisibles.

## 3.4 · Dépendances et contrôle

```bash
make deps      # collections Ansible + plugins helm-diff et helm-secrets
make doctor    # chaque ligne doit afficher "ok"
```

## 3.5 · Configuration : où trouver chaque valeur

| Valeur | Fichier | Où la trouver |
| --- | --- | --- |
| `zone` | `helmfile/environments/test.yaml` | Ton domaine tel qu'il apparaît dans la liste des domaines Cloudflare : `tondomaine.fr`, sans `www` ni `https://` |
| `hostSuffix` | `test.yaml` | Rien à chercher : garde `-test` |
| `cloudflare.apiHostname` | `test.yaml` | À composer : `k8s-api-test.tondomaine.fr` |
| `cloudflare.tunnelId` | `test.yaml` | Pas encore disponible : étape 4.2 |
| IP du VPS | `ansible/inventory/hosts.yml` (`ansible_host`) et `Makefile` (`SSH_TARGET`) | Espace client OVH, Bare Metal Cloud, VPS, ton VPS, onglet Tableau de bord : ligne IPv4. Aussi dans l'email d'installation |
| Utilisateur SSH | `hosts.yml` (`ansible_user`) et `SSH_TARGET` | `ubuntu` pour l'image Ubuntu d'OVH (confirmé dans l'email d'installation) |

Tout remplacer en une fois, depuis `~/k3s-lab` :

```bash
DOMAINE=tondomaine.fr
VPS_IP=51.xx.xx.xx          # quand le VPS est installé
grep -rl 'example\.com' helmfile | xargs sed -i "s/example\.com/${DOMAINE}/g"
sed -i "s/203\.0\.113\.10/${VPS_IP}/g" ansible/inventory/hosts.yml Makefile
```

**Applications de démo (OIDC)** : autorise les deux dépôts à déployer.

```bash
# ansible/group_vars/all.yml
sed -i 's/^github_oidc_repositories:.*$/github_oidc_repositories:\n  - augustinde\/api-demo\n  - augustinde\/front-demo/' ansible/group_vars/all.yml
sed -i '/^  - ton-compte\/ton-api$/d' ansible/group_vars/all.yml
grep -A2 '^github_oidc_repositories' ansible/group_vars/all.yml
```

Puis, dans `helmfile/environments/test.yaml`, sous `tenants.demo` :

```yaml
    githubDeployer: false
    githubOidc:
      - repository: augustinde/api-demo
        environment: test
      - repository: augustinde/front-demo
        environment: test
```

Le fichier `helmfile/apps/api.yaml` n'est pas utilisé : les valeurs de déploiement vivent dans le dossier `deploy/` de chaque application.

Pour vérifier qu'il ne reste aucune valeur d'exemple (hors `api.yaml`) :

```bash
grep -rn 'example\.com\|ton-compte\|203\.0\.113\.10' helmfile/environments ansible Makefile
```

## 3.6 · Fichier de secrets

```bash
make secrets-init     # crée helmfile/environments/test.secrets.yaml, déjà chiffré
make secrets-edit     # l'ouvre en clair dans nano ; il est rechiffré à la fermeture
```

| Clé | Quand | Valeur |
| --- | --- | --- |
| `grafana.adminPassword` | Maintenant | Mot de passe de ton choix |
| `rancher.bootstrapPassword` | Maintenant | Mot de passe de ton choix (Rancher demande de le changer au premier login) |
| `cloudflare.tunnelCredentials` | Étape 4.2 | Contenu de `~/.cloudflared/<ID>.json` (obligatoire, sinon `make platform` échoue) |
| `cloudflare.accessClientId` / `accessClientSecret` | Facultatif | Seulement pour `make deploy-test` ; le déploiement GitHub, lui, lit ces valeurs dans ses propres secrets |
| Bloc `tenants` (registry, appSecrets) | Facultatif | Commenté dans le modèle : à activer seulement si les images GHCR restent privées |

Le fichier chiffré peut être commité. Vérifie-le avec `head -5 helmfile/environments/test.secrets.yaml` : les valeurs doivent commencer par `ENC[`.
