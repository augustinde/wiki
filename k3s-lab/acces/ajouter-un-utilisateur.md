---
title: "2 · Ajouter un utilisateur"
description: "Donner à un collègue l'accès à Rancher, Grafana et aux bases d'un projet"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

**Prérequis** : la [connexion par GitHub](/k3s-lab/acces/connexion-github) est en place dans l'organisation Zero Trust du projet. Compte 10 minutes de ton côté, 15 du sien.

| Ce qu'il obtient | Comment |
| --- | --- |
| Rancher, Grafana (page Access) | membre de l'équipe GitHub `<équipe>` |
| Bases dans IntelliJ, Rancher et Grafana derrière « + WARP » | client WARP inscrit à `<équipe-zt>` |
| Compte dans Rancher et dans Grafana | créés à son nom dans chaque interface |
| Dépôts des applications (code, Actions) | accès aux dépôts GitHub, si besoin |

## 1 · Dans l'organisation GitHub (toi)

1. `https://github.com/orgs/<org>/teams/<slug>`, **Members**, **Add a member** : son identifiant GitHub. S'il n'est pas dans l'organisation, GitHub l'y invite.
2. Préviens-le : il doit **accepter l'invitation** (email de GitHub, ou `https://github.com/<org>`) et avoir activé la **double authentification**, exigée par l'organisation.

> **Vérifier** · La page de l'équipe le liste comme membre (et plus comme invitation en attente) ; **People** de l'organisation indique sa double authentification active.
{.is-success}

## 2 · Sur son PC (lui)

1. **Installer le client WARP** et l'inscrire à `<équipe-zt>` avec son compte GitHub : [Installer le client WARP](/k3s-lab/acces/client-warp). S'il travaille aussi pour d'autres comptes Cloudflare : [Plusieurs organisations WARP](/k3s-lab/acces/plusieurs-organisations-warp).
2. À la première connexion, GitHub lui demande d'autoriser `Cloudflare Access` : bouton **Grant** à côté de `<org>` s'il apparaît, puis **Authorize**.

> **Vérifier** · Chez lui, WARP connecté, `https://rancher<suffixe>.<zone>` et `https://grafana<suffixe>.<zone>` s'ouvrent après la connexion GitHub. De ton côté, Zero Trust, **Team & Resources**, **Devices** liste son appareil, et **Logs**, **Access** ses connexions.
{.is-success}

## 3 · Ses comptes dans les applications (toi)

Access lui ouvre seulement la porte ; chaque application a ses propres comptes. Ne partage pas les mots de passe `admin`.

- **Grafana** : **Administration**, **Users and access**, **Users**, **New user** : nom, email, mot de passe provisoire. Rôle **Viewer** pour consulter, **Editor** pour créer des tableaux de bord, **Admin** seulement si nécessaire (**Organizations**, sur la ligne de l'utilisateur).
- **Rancher** : menu ☰, **Users & Authentication**, **Users**, **Create** : identifiant, mot de passe provisoire, **Ask user to change their password on first login**. Rôle global **Standard User**, puis, dans **Cluster Management**, le cluster `local`, **Members**, **Add** : **Cluster Member** pour consulter, **Cluster Owner** pour tout gérer.
- **Bases** (IntelliJ) : transmets-lui les identifiants de `make db-creds DB=<base> NS=<ns>` par ton gestionnaire de mots de passe, jamais en clair dans une messagerie.

> **Vérifier** · Il se connecte à Grafana et à Rancher avec ses propres identifiants et change son mot de passe ; dans IntelliJ, **Test Connection** répond `Succeeded`.
{.is-success}

## 4 · Dépôts GitHub, si besoin (toi)

Pour qu'il lance **Déployer** ou **Release** dans une application : dans le dépôt, **Settings**, **Collaborators and teams**, ajoute l'équipe `<équipe>` avec le rôle **Write**. Si l'environnement `prod` exige une validation (**Required reviewers**), ajoute-le aussi comme valideur s'il doit pouvoir livrer en ton absence.

## Ce qui n'est pas couvert

L'accès SSH au serveur (`make tunnel`, `make bootstrap`) et la lecture des secrets du dépôt (`secrets.yaml`, chiffré pour ta seule clé age) ne sont pas prévus pour plusieurs personnes dans le modèle actuel. Ne lui transmets ni ta clé SSH ni ta clé age : si ça devient nécessaire, il faut d'abord ajouter au modèle plusieurs clés SSH et plusieurs destinataires SOPS.
