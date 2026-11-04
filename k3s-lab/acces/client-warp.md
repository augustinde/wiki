---
title: "4 · Installer le client WARP"
description: "Client Cloudflare One sur un PC Windows : installation, inscription par GitHub, contrôles"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès, warp
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

Pour toi sur un nouveau PC, ou pour un collègue ajouté à l'équipe ([Ajouter un utilisateur](/k3s-lab/acces/ajouter-un-utilisateur)). Le client WARP relie le PC aux routes privées des clusters (les bases) et prouve à Access que le navigateur passe par lui (Rancher, Grafana). Compte 10 minutes.

**Prérequis** : un compte GitHub membre de l'équipe `<équipe>`, avec la double authentification, et le nom d'équipe Zero Trust `<équipe-zt>`. Les réglages côté Zero Trust (exclusions, proxy, contrôles) sont communs à tous les appareils : rien à configurer par PC.

## 1 · Installer

Dans PowerShell :

```powershell
winget install --id Cloudflare.Warp -e
```

Accepte la demande d'administrateur. Sans winget : l'installeur du client **Cloudflare One**, dans la documentation Cloudflare Zero Trust.

> **Vérifier** · Le nuage orange de Cloudflare apparaît dans la zone de notification.
{.is-success}

## 2 · S'inscrire avec GitHub

1. Au premier lancement, « Pour quoi souhaitez-vous utiliser le client ? » : colonne de droite, **Cloudflare One Client**, **Continuer**. La colonne « Navigation privée » est le WARP grand public, qui ignore l'organisation.
2. Nom d'équipe : `<équipe-zt>`, sans `.cloudflareaccess.com`.
3. Le navigateur s'ouvre et redirige vers GitHub : passkey ou validation GitHub Mobile. À la première fois, **Grant** à côté de `<org>` s'il apparaît, puis **Authorize**. Accepte de rouvrir le client.
4. Active l'interrupteur.

Déjà inscrit à une autre organisation et besoin des deux : [Plusieurs organisations WARP](/k3s-lab/acces/plusieurs-organisations-warp).

> **Vérifier** · Le client affiche **Connecté** ; **Profil** affiche `Nom de l'équipe : <équipe-zt>`.
{.is-success}

## 3 · Contrôler

1. `https://help.teams.cloudflare.com` indique le passage par WARP et Gateway, avec l'organisation `<équipe-zt>`.
2. En navigation privée, `https://rancher<suffixe>.<zone>` : connexion GitHub, puis Rancher. WARP déconnecté : accès refusé.
3. Dans PowerShell, avec l'IP d'une base du projet (fiche du projet, ou `make db-creds`) :

```powershell
Test-NetConnection <ip-de-la-base> -Port 5432
```

> **Vérifier** · `TcpTestSucceeded : True`, et IntelliJ se connecte à la base avec les identifiants transmis.
{.is-success}

Le client se reconnecte au démarrage de Windows. Coupe-le à la main si le VPN d'un client entre en conflit.

## Si ça ne marche pas

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| « Inscription refusée » | compte hors de l'équipe GitHub, ou Device enrollment pas sur GitHub | [Connexion par GitHub](/k3s-lab/acces/connexion-github), étape 7 ; appartenance à `<équipe>` |
| « That account does not have access » | invitation à l'organisation non acceptée, ou organisation non accordée à l'application | accepter l'invitation ; GitHub, **Settings**, **Applications**, **Authorized OAuth Apps**, **Grant** sur `<org>` |
| Rancher refusé WARP connecté | client en mode DNS seul, ou proxy Gateway inactif dans Zero Trust | mise en route, 5.2 et 5.3 |
| Base injoignable, WARP connecté | route du projet exclue, ou autre organisation active | mise en route, 5.3 ; **Profil** affiche la bonne équipe |
| WSL perd Internet WARP connecté | réseau NAT de WSL2 | `C:\Users\<toi>\.wslconfig`, section `[wsl2]` : `networkingMode=mirrored` et `dnsTunneling=true`, puis `wsl --shutdown` |
