---
title: "Étape 4 · Configurer Cloudflare"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Cloudflare réorganise régulièrement les menus de Zero Trust ; si un chemin a bougé, utilise la barre de recherche du tableau de bord avec le mot-clé indiqué.

Les sections 4.0, 4.1, 4.5 et 4.6 se font une fois pour le compte ; les sections marquées **par projet** se refont pour chaque projet, variables chargées (`eval "$(make -s vars)"` dans le dossier du projet).

## 4.0 · Compte et domaine OVH

Un domaine est indispensable : le tunnel et Access fonctionnent sur des noms d'hôte (`rancher-test.<zone>`…). Ton domaine reste chez OVH ; seul son DNS passe chez Cloudflare, et pour tout le domaine (l'offre gratuite ne gère pas un simple sous-domaine).

1. **Compte** : crée-le sur dash.cloudflare.com et active tout de suite la **double authentification** (My Profile, Authentication) : ce compte contrôle l'accès à ton cluster.
2. **Inventaire chez OVH** : Web Cloud, Noms de domaine, ton domaine, onglet **Zone DNS**. Note ce qui existe : site (A, CNAME), emails (MX, SPF, DKIM, autodiscover).
3. **DNSSEC** : onglet DNSSEC, désactive-le s'il est actif, et attends la confirmation d'OVH (jusqu'à 24 heures). Sinon le domaine devient injoignable pendant la bascule.
4. **Ajout dans Cloudflare** : Add a domain (ou Onboard a domain), saisis ton domaine, laisse l'import automatique des enregistrements, offre **Free**.
5. **Comparer les enregistrements** importés avec la zone OVH, et ajouter à la main ce qui manque. Les enregistrements des emails (MX, `mail`, `autodiscover`, `smtp`) restent en **DNS only** (nuage gris).
6. **Serveurs DNS** : Cloudflare donne 2 serveurs (`xxx.ns.cloudflare.com`). Chez OVH, onglet **Serveurs DNS**, **Modifier les serveurs DNS**, remplace ceux d'OVH par ceux de Cloudflare et valide. Garde la zone DNS OVH : elle sert de sauvegarde, et remettre les serveurs OVH annule tout.
7. **Activation** : Cloudflare envoie un email quand la zone passe en **Active** (de quelques minutes à 24 heures). Pour vérifier : `dig NS <zone> +short` doit afficher les deux serveurs Cloudflare.
8. **Optionnel, une fois actif** : réactive le DNSSEC depuis Cloudflare (DNS, Settings, DNSSEC) et saisis chez OVH l'enregistrement DS qu'il fournit.

**Avertissement « Autorisez uniquement les adresses IP Cloudflare à votre origine »** : ignore-le pour les serveurs k3s. Ce conseil vise les serveurs dont les ports web sont ouverts ; le tien n'en a aucun, le tunnel sort du cluster et l'IP du serveur n'apparaît dans aucun enregistrement. **Ne restreins surtout pas le port 22** aux IP Cloudflare : SSH ne passe pas par Cloudflare, tu te couperais l'accès. Le conseil ne concerne qu'un éventuel autre serveur à toi, derrière un enregistrement proxifié.

## 4.1 · Organisation Zero Trust

1. Dans le tableau de bord, ouvre **Zero Trust**.
2. Choisis un **nom d'équipe**, par exemple `augustin-lab` : ton portail sera `augustin-lab.cloudflareaccess.com`, et c'est ce nom que tu saisiras dans le client WARP. Il reste visible dans Zero Trust, Settings, « Team domain ».
3. Offre **Free**. Cloudflare demande un moyen de paiement même pour l'offre gratuite, sans rien facturer.
4. Settings, Authentication, Login methods : vérifie que **One-time PIN** est actif (code envoyé par email).

## 4.2 · Créer le tunnel · par projet

Le tunnel est la connexion que le cluster ouvrira lui-même vers Cloudflare. Ici, tu le déclares et tu récupères ses identifiants ; il ne tournera qu'après `make platform`.

Une fois par compte Cloudflare :

```bash
cloudflared tunnel login
```

WSL n'a pas de navigateur : copie l'URL affichée dans ton navigateur Windows, choisis ton domaine, **Authorize**. Le fichier `~/.cloudflared/cert.pem` est créé : il permet de gérer les tunnels de ce compte, ne le partage pas et ne le commite pas. Projets répartis sur plusieurs comptes Cloudflare : un certificat par compte, voir [Créer un projet](/k3s-lab/nouveau-projet), section « Projets sur plusieurs comptes Cloudflare ».

Puis, pour le projet, le tunnel et ses identifiants dans un dossier à son nom :

```bash
cd ~/<projet> && eval "$(make -s vars)"
mkdir -p ~/.cloudflared/$CLUSTER
cloudflared tunnel create --credentials-file ~/.cloudflared/$CLUSTER/tunnel.json $CLUSTER
```

Cette commande crée le tunnel, affiche son **ID** (un UUID) et écrit ses identifiants dans `~/.cloudflared/<cluster>/tunnel.json`. Reporte les deux dans le dépôt :

```bash
ID=$(jq -r .TunnelID ~/.cloudflared/$CLUSTER/tunnel.json)
echo "$ID"
sed -i "s/^  tunnelId: .*/  tunnelId: $ID/" $ENV_DIR/values.yaml

# Identifiants dans le fichier de secrets chiffré (sops écrit la valeur sans risque d'indentation)
sops set $ENV_DIR/secrets.yaml '["cloudflare"]["tunnelCredentials"]' "$(jq 'tojson' ~/.cloudflared/$CLUSTER/tunnel.json)"
eval "$(make -s vars)"
```

> **Vérifier**
>
> ```bash
> echo "$TUNNEL_ID"
> sops -d --extract '["cloudflare"]["tunnelCredentials"]' $ENV_DIR/secrets.yaml | grep -o '"TunnelID":"[^"]*"'
> ```
>
> Les deux commandes affichent le même ID. Si ta version de sops refuse `set`, utilise `make secrets-edit` et colle le contenu du JSON sous `cloudflare.tunnelCredentials`, après `|`, indenté de 4 espaces.
{.is-success}

`cloudflared tunnel list` montre le tunnel avec une colonne CONNECTIONS vide, et le tableau de bord l'affiche **Inactive** : normal tant que le cluster n'est pas déployé.

- Le tunnel est **géré localement** : ses règles sont dans `helmfile/values/cloudflared.yaml.gotmpl`. Si Cloudflare propose de le « Migrate » vers le tableau de bord, **refuse**.
- Il survit à tous les `make rebuild` : tu ne refais cette étape que si tu le supprimes (`cloudflared tunnel delete <cluster>`).

## 4.3 · Route privée pour WARP · par projet

Cette route dit à Cloudflare que les 16 adresses de `<route>` se trouvent derrière le tunnel du projet. Avec WARP connecté, ton PC les joint comme si elles étaient sur ton réseau local : c'est ainsi qu'IntelliJ atteindra les bases.

```text
IntelliJ ─► WARP ─► Cloudflare ─► tunnel <cluster> ─► cloudflared ─► service app-admin (5ᵉ adresse de <route>, port 5432) ─► PostgreSQL
```

```bash
cloudflared tunnel route ip add $ROUTE $CLUSTER
cloudflared tunnel route ip show       # la plage apparaît, associée au tunnel du projet
```

- Ces adresses sont réservées aux services d'administration des bases (`adminAccess.clusterIP` dans `databases/<nom>.yaml`). Kubernetes n'y attribue jamais d'adresse automatiquement.
- Seules elles sont routées : le reste du cluster (API, pods) reste inaccessible depuis ton PC.
- La route est permanente et survit aux `make rebuild`.
- Chaque projet du compte a sa propre plage : deux routes identiques vers deux tunnels sont refusées. Erreur de chevauchement : liste les routes avec `route ip show`, puis choisis une autre plage dans `values.yaml` et dans les fichiers des bases.

## 4.4 · DNS · par projet

**Premier projet de la zone : un joker.** Un seul enregistrement fait pointer tous les noms vers le tunnel ; un nouveau service n'a besoin d'aucune action chez Cloudflare. Dans ton domaine, DNS, Records, **Add record** :

| Champ | Valeur |
| --- | --- |
| Type | CNAME |
| Name | `*` |
| Target | `<ID du tunnel>.cfargotunnel.com` (`echo $TUNNEL_ID`) |
| Proxy status | **Proxied** (nuage orange), obligatoire |
| TTL | Auto |

- Un avertissement « la cible ne se résout pas » est normal pour `cfargotunnel.com`.
- Si un enregistrement `*` existe déjà (importé d'OVH), modifie-le au lieu d'en ajouter un second.
- Tes enregistrements existants (`www`, `mail`, `@`, MX…) gardent la priorité : le joker ne s'applique qu'aux noms sans enregistrement propre.

**Projets suivants sur la même zone : un nom par service.** Le joker appartient déjà au premier tunnel. Déclare chaque nom du projet, qui prend le pas sur le joker :

```bash
for n in grafana rancher k8s-api whoami; do
  cloudflared tunnel route dns $CLUSTER $n$SUFFIX.$ZONE
done
```

Chaque nouveau service du projet demandera la même commande pour son nom.

> **Vérifier** · `dig +short whoami$SUFFIX.$ZONE` renvoie des IP Cloudflare (`104.x` ou `172.67.x`), jamais celle du serveur. Dans le navigateur, une **erreur 1033** s'affiche tant que le cluster n'est pas déployé : c'est attendu.
{.is-success}

## 4.5 · HTTPS

C'est Cloudflare qui présente le certificat : aucun certificat à gérer dans le cluster. Dans ton domaine, **SSL/TLS**, **Edge Certificates** :

1. **Certificat** : un certificat **Universal** couvrant `<zone>, *.<zone>` doit être **Active**. Il ne couvre qu'un niveau de sous-domaine, d'où les noms avec suffixe (`grafana-test.<zone>`, pas `grafana.test.<zone>`).
2. **Always Use HTTPS** : On.
3. **Minimum TLS Version** : TLS 1.2.

Ne touche pas au mode de chiffrement (Flexible, Full, Strict) s'il fonctionne pour ton site existant : il n'a aucun effet sur les noms servis par le tunnel. N'active pas **HSTS** pour l'instant : difficile à annuler et appliqué à tous tes sous-domaines.

Tests, une fois le DNS d'un projet en place (4.4) :

```bash
curl -sI http://whoami$SUFFIX.$ZONE | head -3            # 301 ou 308 vers https://
curl -sI --tls-max 1.1 https://whoami$SUFFIX.$ZONE        # doit échouer (TLS refusé)
```

## 4.6 · Jeton de service GitHub

Ce jeton permet aux jobs GitHub de tes applications d'atteindre l'API Kubernetes à travers Cloudflare Access. Il ne donne aucun droit dans le cluster : l'identité vient d'OIDC. Un seul jeton sert à tous les projets du compte.

1. Zero Trust, **Access controls**, **Service credentials**, **Service Tokens**, **Create Service Token**.
2. Nom : `github-deploy`. Durée : 1 an, avec un rappel dans ton agenda pour le renouveler.
3. Copie immédiatement le **Client ID** et le **Client Secret** : le secret ne s'affichera plus.
4. Colle-les dans les secrets d'environnement des dépôts d'applications (`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`) : [procédure, étape 0.3](/k3s-lab/procedure/00-github).

## 4.7 · Applications Access · par projet

Access place une page de connexion Cloudflare devant Rancher, Grafana et l'API Kubernetes : seule ton adresse email passe, et la requête n'atteint jamais le serveur sinon.

**Une fois pour le compte, créer les politiques réutilisables** : Zero Trust, **Access controls**, **Policies**, **Add a policy** :

| Champ | Valeur |
| --- | --- |
| Policy name | `Augustin uniquement` |
| Action | Allow |
| Session duration | 24 hours |
| Include | Emails = ton adresse |

La politique **GitHub deploy** se crée de la même façon : Action **Service Auth**, Include **Service Token** = `github-deploy`. La politique `Augustin + WARP` vient à l'étape 5.4.

**Pour chaque projet, créer les trois applications** : Access controls, **Applications**, **Add an application**. Dans la fenêtre, garde l'onglet **Self-hosted and private** et la vignette **Public DNS** (les vignettes ne sont que des exemples), puis **Continue with Self-hosted and private**. Sur l'écran suivant :

1. Application name.
2. Destinations : **Add public hostname**, sous-domaine, ton domaine dans la liste, chemin vide.
3. Access policies : celles du tableau.
4. Login methods : **One-time PIN**, avec **Instant Auth** si c'est la seule méthode.
5. Session duration : 24 heures. Le reste par défaut, puis **Save**.

| Application name | Sous-domaine | Politiques |
| --- | --- | --- |
| `Rancher <cluster>` | `rancher<suffixe>` | `Augustin + WARP` (au tout premier projet : `Augustin uniquement`, remplacée à l'étape 5.4) |
| `Grafana <cluster>` | `grafana<suffixe>` | idem |
| `k3s API <cluster>` | `k8s-api<suffixe>` | `Augustin uniquement` **et** `GitHub deploy` |

En profil `small`, sans Rancher, l'application Rancher est inutile.

- `whoami<suffixe>` et les services publics du projet n'ont volontairement aucune application Access.
- **k3s API** ne sert pas dans un navigateur : elle protège l'API Kubernetes publiée par le tunnel. Seuls les jobs GitHub, avec le jeton `github-deploy`, la traversent. Ton propre accès passe par SSH (`make tunnel`).

> **Vérifier** · En navigation privée sur `https://rancher<suffixe>.<zone>` (ou `grafana<suffixe>`) : page de connexion de ton équipe, email, code à 6 chiffres, puis **erreur 1033** (normal sans cluster). Avec une autre adresse, aucun code n'arrive.
{.is-success}
