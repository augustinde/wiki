---
title: "Étape 5 · Configurer WARP"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

WARP relie ton Windows aux plages privées des clusters (les bases, sur la route `<route>` de chaque projet) et prouve à Cloudflare Access que ton navigateur passe par lui. Les réglages se font une fois : d'abord dans Zero Trust (5.1 à 5.4), puis sur ton PC (5.5 et 5.6). Pour un nouveau projet, seules les exclusions de 5.3 (liste précise) et les applications de 5.4 sont à reprendre. Projet sur un autre compte Cloudflare : son organisation Zero Trust est distincte, voir [Créer un projet](/k3s-lab/nouveau-projet), section « Projets sur plusieurs comptes Cloudflare ».

## 5.1 · Autoriser ton PC à s'inscrire

*Mot-clé de recherche : « Device enrollment ».* Zero Trust, **Team & Resources**, **Devices**, onglet Management (ou Settings), **Device enrollment permissions**, Manage. Ajoute une règle avec la politique **`Équipe`**, mets les **Login methods** sur GitHub seulement, puis enregistre. Seuls les membres de l'équipe GitHub peuvent inscrire un appareil.

## 5.2 · Activer le proxy Gateway

*Mot-clé : « Proxy » ou « Traffic settings ».* Zero Trust, **Traffic controls**, **Traffic settings** : active le proxy (« Allow Secure Web Gateway to proxy traffic »), en **TCP** et **UDP**. Sans lui, le trafic vers les bases n'est pas acheminé, et la règle « Require Gateway » ne fonctionne pas.

## 5.3 · Laisser passer la plage du cluster

*Mot-clé : « Split Tunnels ».* Team & Resources, Devices, **Device profiles**, profil **Default**, **Edit**. Section **Split Tunnels** (mode Exclude IPs and domains), **Manage**. Par défaut, Cloudflare exclut tout `10.0.0.0/8`, ce qui empêcherait d'atteindre les routes des projets, en `10.43.0.x`.

- **Simple** : supprime la ligne `10.0.0.0/8`. Ne touche pas aux autres : `192.168.0.0/16` (ta Livebox) et `172.16.0.0/12` (WSL) restent exclus.
- **Précise**, à préférer si le VPN d'un client utilise des adresses en `10.x` : supprime `10.0.0.0/8`, puis ajoute ces 20 plages, qui couvrent tout `10.x` sauf `10.43.0.96/28`, la route du premier projet.

```text
10.0.0.0/11  10.32.0.0/13  10.40.0.0/15  10.42.0.0/16  10.43.0.0/26
10.43.0.64/27  10.43.0.112/28  10.43.0.128/25  10.43.1.0/24  10.43.2.0/23
10.43.4.0/22  10.43.8.0/21  10.43.16.0/20  10.43.32.0/19  10.43.64.0/18
10.43.128.0/17  10.44.0.0/14  10.48.0.0/12  10.64.0.0/10  10.128.0.0/9
```

Deuxième projet en `10.43.0.112/28` : retire cette ligne de la liste. Au-delà de deux projets, passe en mode simple.

Les changements de profil mettent jusqu'à 10 minutes à arriver sur le client. Dans le même profil, vérifie que **Service mode** est sur *Traffic and DNS mode*.

## 5.4 · Exiger WARP pour Rancher et Grafana

1. **Contrôle Gateway** : Zero Trust, **Reusable components**, **Posture checks**, onglet Cloudflare One Client checks. Le contrôle **Gateway** (type Gateway) doit apparaître ; sinon, **Add a check**, type Gateway. Le contrôle WARP de la même liste ne sert pas ici.
2. **Politique** : Access controls, **Policies**, **Add a policy** :

| Champ | Valeur |
| --- | --- |
| Policy name | `Équipe + WARP` |
| Action | Allow |
| Session duration | 24 hours |
| Include | **GitHub organization** = `<org>`, équipe `<équipe>` |
| Require | Device posture (parfois nommé Gateway) = **Gateway** |

3. **Applications** : Access controls, **Applications**. Dans **Rancher <cluster>**, remplace `Équipe` par `Équipe + WARP` et enregistre ; fais de même pour **Grafana <cluster>**. **k3s API <cluster>** garde `Équipe` (GitHub n'aura pas WARP). Les projets suivants reçoivent directement `Équipe + WARP` (4.7).

Si le sélecteur Device posture ou Gateway n'apparaît pas dans le bloc Require, vérifie le proxy de l'étape 5.2.

## 5.5 · Installer le client sur Windows

Même procédure pour un nouveau PC ou un collègue : [Installer le client WARP](/k3s-lab/acces/client-warp).

1. Dans PowerShell : `winget install --id Cloudflare.Warp -e`, et accepte la demande administrateur. Sans winget, télécharge l'installeur du client **Cloudflare One** depuis la documentation Cloudflare Zero Trust.
2. Au premier lancement, le client demande « Pour quoi souhaitez-vous utiliser le client ? » : choisis la colonne de droite, **Cloudflare One Client**, puis **Continuer**. La colonne « Navigation privée » est le WARP grand public (1.1.1.1), qui ignore ton organisation.
3. Saisis ton **nom d'équipe** (`augustin-lab`, sans `.cloudflareaccess.com`).
4. Le navigateur s'ouvre et redirige vers GitHub : passkey ou validation GitHub Mobile (à la première fois, **Grant** sur `<org>` puis **Authorize**). Accepte de rouvrir le client.
5. Active l'interrupteur : le client affiche **Connecté**, avec le nom de ton organisation.

Par la suite, le client se trouve dans la zone de notification (nuage orange) et se reconnecte au démarrage de Windows. Coupe-le à la main si le VPN d'un client entre en conflit.

## 5.6 · Vérifier

**Maintenant, sans cluster :**

1. **Client** : « Connecté » avec ton organisation, et ton PC apparaît dans Team & Resources, **Devices**.
2. **Gateway** : `https://help.teams.cloudflare.com` indique que tu passes par WARP et Gateway, avec ton nom d'équipe.
3. **Restriction**, en navigation privée sur `https://rancher<suffixe>.<zone>` (ou `grafana<suffixe>`) :

| WARP | Résultat attendu |
| --- | --- |
| Connecté | connexion GitHub, puis **erreur 1033** (normal sans cluster) |
| Déconnecté | **Accès refusé**, même avec le bon compte GitHub |

Si l'accès passe WARP déconnecté, Rancher a encore la politique `Équipe` : reviens au 5.4.

4. **WSL** : WARP connecté, `curl -sI https://github.com | head -1` doit répondre `HTTP/2 200`. Sinon, voir le dépannage (réglage `networkingMode=mirrored`).

**Après le déploiement du cluster** (`make platform` puis `make data`) : les contrôles de la base et d'IntelliJ sont listés à l'étape 6.
