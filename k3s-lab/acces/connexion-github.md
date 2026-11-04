---
title: "1 · Connexion par GitHub"
description: "GitHub comme fournisseur d'identité de Cloudflare Access : organisation, équipe, application OAuth, politiques"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

On remplace le code envoyé par email par une connexion GitHub. C'est GitHub qui valide la personne : passkey, Windows Hello, code d'application ou notification dans GitHub Mobile. Les politiques Access autorisent les membres d'une équipe de l'organisation GitHub, plutôt qu'une liste d'emails.

À faire une fois par organisation Zero Trust, donc une fois par compte Cloudflare. Les valeurs `<org>`, `<équipe>` et `<équipe-zt>` sont décrites dans [Accès](/k3s-lab/acces).

| Où | Ce qui change |
| --- | --- |
| GitHub, organisation `<org>` | une équipe `<équipe>`, la double authentification obligatoire, une application OAuth de l'organisation |
| Cloudflare Zero Trust | GitHub comme fournisseur d'identité ; politiques basées sur l'équipe ; connexion par GitHub seulement, pour les applications et pour WARP |
| Inchangé | le jeton de service `github-deploy` (GitHub Actions), SSH, les commandes `make` |

## 1 · Équipe GitHub

1. Dans l'organisation (`https://github.com/<org>`) : **Teams**, **New team**, nom `<équipe>`, visibilité **Visible**, **Create team**.
2. Dans l'équipe : **Members**, **Add a member**, ajoute-toi.
3. Note le **slug** de l'équipe : la fin de son adresse, `https://github.com/orgs/<org>/teams/<slug>`.

> **Vérifier** · La page de l'équipe te liste comme membre.
{.is-success}

## 2 · Double authentification obligatoire

**Settings** de l'organisation, **Authentication security**, coche **Require two-factor authentication for everyone in the `<org>` organization**, **Save**. GitHub retire de l'organisation les membres qui ne l'ont pas activée : vérifie d'abord la page **People**.

Sur ton compte : **Settings**, **Password and authentication**, ajoute une **passkey** (Windows Hello ou téléphone) et, pour valider d'un toucher, l'application **GitHub Mobile**.

> **Vérifier** · **People** n'affiche aucun membre sans double authentification.
{.is-success}

## 3 · Application OAuth de l'organisation

Elle appartient à l'organisation et non à ton compte : un autre propriétaire de l'organisation peut la gérer. **Settings** de l'organisation, **Developer settings**, **OAuth Apps**, **New Org OAuth App** :

| Champ | Valeur |
| --- | --- |
| Application name | `Cloudflare Access` |
| Homepage URL | `https://<équipe-zt>.cloudflareaccess.com` |
| Redirect URI | `https://<équipe-zt>.cloudflareaccess.com/cdn-cgi/access/callback` |
| Allow wildcard matching | décoché |
| Enable Device Flow | décoché |
| Expire user access tokens | coché (par défaut) |

Une application accepte jusqu'à 10 adresses de retour : pour plusieurs comptes Cloudflare, ajoute une ligne par organisation Zero Trust avec **Add redirect URI**, et utilise le même Client ID partout. Une application par compte reste possible, pour en révoquer une sans toucher aux autres.

**Register application**. Copie le **Client ID**, puis **Generate a new client secret** et copie-le aussitôt : il ne s'affiche qu'une fois. Range les deux dans ton gestionnaire de mots de passe.

> **Vérifier** · La page de l'application affiche le Client ID et les adresses de retour, chacune terminée par `/cdn-cgi/access/callback`.
{.is-success}

## 4 · Accès de l'application à l'organisation

Cloudflare doit lire l'appartenance à l'organisation et aux équipes, même privée. **Settings** de l'organisation, **Third-party access**, **OAuth application policy** :

- **No restrictions** : rien à faire ;
- **Access restricted** : si `Cloudflare Access` apparaît « Denied » ou « Pending », ouvre-la et **Grant access**.

> **Vérifier** · L'application est autorisée, ou aucune restriction n'est active.
{.is-success}

## 5 · GitHub dans Cloudflare Zero Trust

Zero Trust (`<équipe-zt>`), **Integrations**, **Identity providers**, **Add new identity provider**, **GitHub** : Client ID dans **App ID**, secret dans **Client Secret**, **Save**. Puis **Test** à côté de GitHub ; à la demande d'autorisation, clique **Grant** à côté de `<org>` s'il apparaît, puis **Authorize**.

> **Vérifier** · Le test affiche « Your connection works! », ton compte GitHub, l'organisation `<org>` et l'équipe `<équipe>`. Sans l'organisation : étape 4, et sur GitHub, **Settings**, **Applications**, **Authorized OAuth Apps**, l'application, accès accordé à `<org>`.
{.is-success}

## 6 · Politiques

Zero Trust, **Access controls**, **Policies**. Les deux politiques d'utilisateurs (sur un compte configuré avant ce changement : `Augustin uniquement` et `Augustin + WARP`, à renommer) :

| Politique | Include | Require |
| --- | --- | --- |
| `Équipe` | **GitHub organization** : `<org>`, équipe `<équipe>` ; et, provisoirement, **Emails** : ton adresse | — |
| `Équipe + WARP` | les mêmes | **Gateway** |

Action **Allow**, session de 24 heures. Les règles d'un même bloc Include s'additionnent : l'email provisoire garde ton accès pendant la bascule, il disparaît à l'étape 9. `GitHub deploy` (Service Auth, jeton `github-deploy`) ne change pas.

## 7 · Connexion par GitHub seulement

**Applications** : pour Rancher, Grafana et l'API de chaque projet, onglet **Login methods** : coche **GitHub**, décoche **One-time PIN**, active **Instant Auth** (redirection directe vers GitHub).

| Application | Politiques |
| --- | --- |
| `Rancher <cluster>`, `Grafana <cluster>` | `Équipe + WARP` |
| `k3s API <cluster>` | `Équipe` et `GitHub deploy` |

**Inscription des appareils** : Team & Resources, **Devices**, **Device enrollment** (ou Settings, WARP client, Device enrollment permissions) : règle `Équipe`, **Login methods** GitHub seulement.

## 8 · Vérifier

En navigation privée :

> **Vérifier**
>
> | Test | Résultat attendu |
> | --- | --- |
> | `https://rancher<suffixe>.<zone>`, WARP connecté | redirection vers GitHub, passkey ou GitHub Mobile, puis Rancher ; aucun email |
> | la même adresse, WARP déconnecté | accès refusé |
> | `https://<équipe-zt>.cloudflareaccess.com/cdn-cgi/access/get-identity`, après connexion | ton identité vue par Access, avec `<org>` et `<équipe>` dans les groupes GitHub |
> | client WARP, **Profil**, **Réauthentifier** | connexion par GitHub |
> | `make deploy-test`, puis **Déployer** en test sur une application | inchangés : le jeton de service ne dépend pas de la connexion |
> | Zero Trust, **Logs**, **Access** | tes connexions, méthode GitHub |
{.is-success}

## 9 · Finaliser

Retire la règle **Emails** provisoire des deux politiques : seule l'équipe GitHub donne l'accès.

> **Vérifier** · Avec un compte GitHub hors de l'équipe, la page Access affiche « That account does not have access ».
{.is-success}

## Dépannage

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| Test : `redirect_uri is not associated with this application` | adresse de retour absente ou différente | étape 3, `https://<équipe-zt>.cloudflareaccess.com/cdn-cgi/access/callback` exactement |
| « That account does not have access » pour un membre de l'équipe | organisation non accordée à l'application, ou équipe mal saisie | étapes 4 et 5 ; dans la politique, essaie le slug au lieu du nom (ou l'inverse) ; `get-identity` montre les groupes vus par Access |
| `get-identity` sans `<org>` | accès de l'organisation non accordé | GitHub, **Authorized OAuth Apps**, l'application, **Grant** sur `<org>`, puis `https://<équipe-zt>.cloudflareaccess.com/cdn-cgi/access/logout` et nouvelle connexion |
| WARP refuse l'inscription | Device enrollment encore sur l'ancienne règle ou sur One-time PIN | étape 7 |
| Un job GitHub Actions échoue | autre cause : le jeton de service n'est pas touché | procédure, étape 10, « Si un job échoue » |

## Retour arrière

Le tableau de bord Cloudflare a sa propre connexion, indépendante d'Access. Pour revenir au code par email : recoche **One-time PIN** dans les Login methods des applications et de Device enrollment, et remets une règle **Emails** dans les politiques. SSH, `make tunnel` et `make db-forward` ne dépendent pas d'Access.
