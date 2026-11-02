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

## 4.0 · Compte et domaine OVH

Un domaine est indispensable : le tunnel et Access fonctionnent sur des noms d'hôte (`rancher-test.tondomaine.fr`…). Ton domaine reste chez OVH ; seul son DNS passe chez Cloudflare, et pour tout le domaine (l'offre gratuite ne gère pas un simple sous-domaine).

1. **Compte** : crée-le sur dash.cloudflare.com et active tout de suite la **double authentification** (My Profile, Authentication) : ce compte contrôle l'accès à ton cluster.
2. **Inventaire chez OVH** : Web Cloud, Noms de domaine, ton domaine, onglet **Zone DNS**. Note ce qui existe : site (A, CNAME), emails (MX, SPF, DKIM, autodiscover).
3. **DNSSEC** : onglet DNSSEC, désactive-le s'il est actif, et attends la confirmation d'OVH (jusqu'à 24 heures). Sinon le domaine devient injoignable pendant la bascule.
4. **Ajout dans Cloudflare** : Add a domain (ou Onboard a domain), saisis `tondomaine.fr`, laisse l'import automatique des enregistrements, offre **Free**.
5. **Comparer les enregistrements** importés avec la zone OVH, et ajouter à la main ce qui manque. Les enregistrements des emails (MX, `mail`, `autodiscover`, `smtp`) restent en **DNS only** (nuage gris).
6. **Serveurs DNS** : Cloudflare donne 2 serveurs (`xxx.ns.cloudflare.com`). Chez OVH, onglet **Serveurs DNS**, **Modifier les serveurs DNS**, remplace ceux d'OVH par ceux de Cloudflare et valide. Garde la zone DNS OVH : elle sert de sauvegarde, et remettre les serveurs OVH annule tout.
7. **Activation** : Cloudflare envoie un email quand la zone passe en **Active** (de quelques minutes à 24 heures). Pour vérifier : `dig NS tondomaine.fr +short` doit afficher les deux serveurs Cloudflare.
8. **Optionnel, une fois actif** : réactive le DNSSEC depuis Cloudflare (DNS, Settings, DNSSEC) et saisis chez OVH l'enregistrement DS qu'il fournit.

**Avertissement « Autorisez uniquement les adresses IP Cloudflare à votre origine »** : ignore-le pour le VPS k3s. Ce conseil vise les serveurs dont les ports web sont ouverts ; le tien n'en a aucun, le tunnel sort du cluster et l'IP du VPS n'apparaît dans aucun enregistrement. **Ne restreins surtout pas le port 22** aux IP Cloudflare : SSH ne passe pas par Cloudflare, tu te couperais l'accès. Le conseil ne concerne qu'un éventuel autre serveur à toi, derrière un enregistrement proxifié.

## 4.1 · Organisation Zero Trust

1. Dans le tableau de bord, ouvre **Zero Trust**.
2. Choisis un **nom d'équipe**, par exemple `augustin-lab` : ton portail sera `augustin-lab.cloudflareaccess.com`, et c'est ce nom que tu saisiras dans le client WARP. Il reste visible dans Zero Trust, Settings, « Team domain ».
3. Offre **Free**. Cloudflare demande un moyen de paiement même pour l'offre gratuite, sans rien facturer.
4. Settings, Authentication, Login methods : vérifie que **One-time PIN** est actif (code envoyé par email).

## 4.2 · Créer le tunnel

Le tunnel est la connexion que le cluster ouvrira lui-même vers Cloudflare. Ici, tu le déclares et tu récupères ses identifiants ; il ne tournera qu'après `make platform`.

```bash
cloudflared tunnel login
```

WSL n'a pas de navigateur : copie l'URL affichée dans ton navigateur Windows, choisis ton domaine, **Authorize**. Le fichier `~/.cloudflared/cert.pem` est créé : il permet de gérer les tunnels de ton compte, ne le partage pas et ne le commite pas.

```bash
cloudflared tunnel create k3s-test
```

Cette commande crée le tunnel, affiche son **ID** (un UUID) et écrit ses identifiants dans `~/.cloudflared/<ID>.json`. Reporte les deux dans le dépôt :

```bash
cd ~/k3s-lab
ID=$(basename ~/.cloudflared/*.json .json)          # le nom du fichier JSON est l'ID
sed -i "s/00000000-0000-0000-0000-000000000000/${ID}/" helmfile/environments/test.yaml
grep tunnelId helmfile/environments/test.yaml

# Identifiants dans le fichier de secrets chiffré (sops écrit la valeur sans risque d'indentation)
sops set helmfile/environments/test.secrets.yaml \
  '["cloudflare"]["tunnelCredentials"]' "$(jq 'tojson' ~/.cloudflared/${ID}.json)"
sops -d helmfile/environments/test.secrets.yaml | grep -c TunnelSecret     # doit afficher 1
```

Si ta version de sops refuse `set`, utilise `make secrets-edit` et colle le contenu du JSON sous la clé, indenté de 4 espaces :

```yaml
cloudflare:
  tunnelCredentials: |
    {"AccountTag":"a1b2…","TunnelSecret":"Xy…","TunnelID":"6ff42ae2-…"}
```

`cloudflared tunnel list` montre le tunnel avec une colonne CONNECTIONS vide, et le tableau de bord l'affiche **Inactive** : normal tant que le cluster n'est pas déployé.

- Le tunnel est **géré localement** : ses règles sont dans `helmfile/values/cloudflared.yaml.gotmpl`. Si Cloudflare propose de le « Migrate » vers le tableau de bord, **refuse**.
- Il survit à tous les `make rebuild` : tu ne refais cette étape que si tu le supprimes (`cloudflared tunnel delete k3s-test`).

## 4.3 · Route privée pour WARP

Cette route dit à Cloudflare que les adresses `10.43.0.96` à `10.43.0.111` se trouvent derrière ton tunnel. Avec WARP connecté, ton PC les joint comme si elles étaient sur ton réseau local : c'est ainsi qu'IntelliJ atteindra la base.

```text
IntelliJ ─► WARP ─► Cloudflare ─► tunnel k3s-test ─► cloudflared ─► service app-admin (10.43.0.100:5432) ─► PostgreSQL
```

```bash
cloudflared tunnel route ip add 10.43.0.96/28 k3s-test
cloudflared tunnel route ip show       # la plage apparaît, associée à k3s-test
```

- Ces 16 adresses sont réservées aux services d'administration des bases (`app-admin` = `10.43.0.100`, les bases futures sur les suivantes). Kubernetes n'y attribue jamais d'adresse automatiquement.
- Seules elles sont routées : le reste du cluster (API, pods) reste inaccessible depuis ton PC.
- La route est permanente et survit aux `make rebuild`.
- Erreur de chevauchement : liste les routes avec `route ip show` et supprime celle qui gêne avec `cloudflared tunnel route ip delete <plage>`.

## 4.4 · DNS : un joker vers le tunnel

Un seul enregistrement fait pointer tous les noms de test vers le tunnel ; un nouveau service n'a besoin d'aucune action chez Cloudflare. Dans ton domaine, DNS, Records, **Add record** :

| Champ | Valeur |
| --- | --- |
| Type | CNAME |
| Name | `*` |
| Target | `<ID du tunnel>.cfargotunnel.com` (ID donné par `cloudflared tunnel list`) |
| Proxy status | **Proxied** (nuage orange), obligatoire |
| TTL | Auto |

- Un avertissement « la cible ne se résout pas » est normal pour `cfargotunnel.com`.
- Si un enregistrement `*` existe déjà (importé d'OVH), modifie-le au lieu d'en ajouter un second.
- Tes enregistrements existants (`www`, `mail`, `@`, MX…) gardent la priorité : le joker ne s'applique qu'aux noms sans enregistrement propre.
- Test immédiat : `dig +short whoami-test.tondomaine.fr` renvoie des IP Cloudflare (`104.x` ou `172.67.x`), jamais celle du VPS. Dans le navigateur, une **erreur 1033** s'affiche tant que le cluster n'est pas déployé : c'est attendu.

## 4.5 · HTTPS

C'est Cloudflare qui présente le certificat : aucun certificat à gérer dans le cluster. Dans ton domaine, **SSL/TLS**, **Edge Certificates** :

1. **Certificat** : un certificat **Universal** couvrant `tondomaine.fr, *.tondomaine.fr` doit être **Active**. Il ne couvre qu'un niveau de sous-domaine, d'où les noms en `-test`.
2. **Always Use HTTPS** : On.
3. **Minimum TLS Version** : TLS 1.2.

Ne touche pas au mode de chiffrement (Flexible, Full, Strict) s'il fonctionne pour ton site existant : il n'a aucun effet sur les noms servis par le tunnel. N'active pas **HSTS** pour l'instant : difficile à annuler et appliqué à tous tes sous-domaines.

Tests immédiats :

```bash
curl -sI http://whoami-test.tondomaine.fr | head -3            # 301 ou 308 vers https://
curl -sI --tls-max 1.1 https://whoami-test.tondomaine.fr        # doit échouer (TLS refusé)
```

## 4.6 · Jeton de service GitHub

Ce jeton permet aux jobs GitHub d'api-demo et de front-demo d'atteindre l'API Kubernetes à travers Cloudflare Access. Il ne donne aucun droit dans le cluster : l'identité vient d'OIDC.

1. Zero Trust, **Access controls**, **Service credentials**, **Service Tokens**, **Create Service Token**.
2. Nom : `github-deploy`. Durée : 1 an, avec un rappel dans ton agenda pour le renouveler.
3. Copie immédiatement le **Client ID** et le **Client Secret** : le secret ne s'affichera plus.
4. Colle-les dans les secrets d'environnement `test` des deux dépôts GitHub (`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`) : [procédure, étape 0.3](/k3s-lab/procedure/00-github).

## 4.7 · Applications Access

Access place une page de connexion Cloudflare devant Rancher, Grafana et l'API Kubernetes : seule ton adresse email passe, et la requête n'atteint jamais le serveur sinon.

**Créer une politique réutilisable** : Zero Trust, **Access controls**, **Policies**, **Add a policy** :

| Champ | Valeur |
| --- | --- |
| Policy name | `Augustin uniquement` |
| Action | Allow |
| Session duration | 24 hours |
| Include | Emails = ton adresse |

**Créer les trois applications** : Access controls, **Applications**, **Add an application**. Dans la fenêtre, garde l'onglet **Self-hosted and private** et la vignette **Public DNS** (les vignettes ne sont que des exemples), puis **Continue with Self-hosted and private**. Sur l'écran suivant :

1. Application name.
2. Destinations : **Add public hostname**, sous-domaine, ton domaine dans la liste, chemin vide.
3. Access policies : `Augustin uniquement`.
4. Login methods : **One-time PIN**, avec **Instant Auth** si c'est la seule méthode.
5. Session duration : 24 heures. Le reste par défaut, puis **Save**.

| Application name | Sous-domaine | Politiques |
| --- | --- | --- |
| Rancher test | `rancher-test` | `Augustin uniquement` (remplacée par `Augustin + WARP` à l'étape 5.4) |
| Grafana test | `grafana-test` | `Augustin uniquement` (remplacée par `Augustin + WARP` à l'étape 5.4) |
| k3s API test | `k8s-api-test` | `Augustin uniquement` **et** `GitHub deploy` |

La politique **GitHub deploy** se crée comme les autres (Access controls, Policies) : Action **Service Auth**, Include **Service Token** = `github-deploy`.

- `whoami-test`, `api-demo-test` et `front-demo-test` n'ont volontairement aucune application Access : ce sont les pages publiques de démonstration.
- **k3s API test** ne sert pas dans un navigateur : elle protège l'API Kubernetes publiée par le tunnel. Seuls les jobs GitHub, avec le jeton `github-deploy`, la traversent. Ton propre accès passe par SSH (`make tunnel`).

**Test** en navigation privée sur `https://rancher-test.tondomaine.fr` : page de connexion de ton équipe, email, code à 6 chiffres, puis **erreur 1033** (normal sans cluster). Avec une autre adresse, aucun code n'arrive.
