---
title: "5 · Plusieurs organisations WARP sur un PC"
description: "Basculer le client WARP entre les organisations Zero Trust de plusieurs comptes Cloudflare"
published: true
date: 2026-11-04T12:00:00.000Z
tags: k3s-lab, accès, warp
editor: markdown
dateCreated: 2026-11-04T12:00:00.000Z
---

Chaque compte Cloudflare a son organisation Zero Trust (`<équipe-zt>`), et le client WARP n'est connecté qu'à une à la fois. On déclare toutes les organisations dans le fichier de configuration du client, puis on bascule depuis le client. Pour le reste du poste avec plusieurs comptes (certificats `cloudflared`) : [Créer un projet](/k3s-lab/nouveau-projet), section « Projets sur plusieurs comptes Cloudflare ».

Quand une organisation est active, les projets de l'autre perdent ce qui passe par WARP : bases dans IntelliJ, Rancher et Grafana derrière « + WARP ». Les commandes `make` passent par SSH et ne sont pas concernées.

**Prérequis** : chaque organisation a sa [connexion par GitHub](/k3s-lab/acces/connexion-github) et ses réglages WARP (mise en route, étapes 5.1 à 5.4) ; un compte administrateur Windows sur le PC.

## 1 · Ouvrir CMD en administrateur

Le dossier `C:\ProgramData\Cloudflare` n'est modifiable qu'en administrateur. Menu Démarrer, tape `cmd`, clic droit sur **Invite de commandes**, **Exécuter en tant qu'administrateur**, **Oui**.

> **Vérifier** · Le titre de la fenêtre commence par « Administrateur : ». Si Windows demande un mot de passe administrateur que tu n'as pas, passe à l'option sans WARP, en bas de page.
{.is-success}

## 2 · Déclarer les organisations

```bat
notepad C:\ProgramData\Cloudflare\mdm.xml
```

Accepte la création du fichier, colle, adapte les noms d'équipe, **Ctrl+S**, ferme. Une entrée `<dict>` par organisation (25 au plus), dans l'ordre d'affichage voulu ; `display_name` est le nom affiché dans le client.

```xml
<dict>
  <key>configs</key>
  <array>
    <dict>
      <key>organization</key>
      <string>augustin-lab</string>
      <key>display_name</key>
      <string>k3s-test (augustin-lab)</string>
    </dict>
    <dict>
      <key>organization</key>
      <string>escargow</string>
      <key>display_name</key>
      <string>EscargOW (escargow)</string>
    </dict>
  </array>
</dict>
```

Sans Bloc-notes, la même chose en ligne de commande, dans la fenêtre administrateur (les `^` empêchent CMD d'interpréter `<` et `>`) :

```bat
(
echo ^<dict^>
echo   ^<key^>configs^</key^>
echo   ^<array^>
echo     ^<dict^>
echo       ^<key^>organization^</key^>
echo       ^<string^>augustin-lab^</string^>
echo       ^<key^>display_name^</key^>
echo       ^<string^>k3s-test (augustin-lab)^</string^>
echo     ^</dict^>
echo     ^<dict^>
echo       ^<key^>organization^</key^>
echo       ^<string^>escargow^</string^>
echo       ^<key^>display_name^</key^>
echo       ^<string^>EscargOW (escargow)^</string^>
echo     ^</dict^>
echo   ^</array^>
echo ^</dict^>
) > C:\ProgramData\Cloudflare\mdm.xml
```

## 3 · Redémarrer le service WARP

```bat
type C:\ProgramData\Cloudflare\mdm.xml
net stop CloudflareWARP
net start CloudflareWARP
```

« Le nom de service n'est pas valide » : `sc query state= all | findstr /i "cloudflare warp"` donne le nom exact (ligne `SERVICE_NAME:`). Redémarrer Windows marche aussi.

> **Vérifier** · `type` affiche le XML avec les deux organisations, et `net start` répond « Le service Cloudflare WARP a démarré ».
{.is-success}

## 4 · Basculer

Dans le client : **Paramètres**, **Avancé**, **Switch configurations** (« Changer de configuration »), choisis l'organisation. La première fois, connexion GitHub pour cette organisation ; ensuite, la bascule se fait en un clic, dans les deux sens. La première connexion à une organisation peut demander de redémarrer le navigateur.

> **Vérifier** · **Profil** affiche le nom de l'équipe choisie, comme `https://help.teams.cloudflare.com`. Les bases et Rancher du projet de cette organisation répondent ; ceux de l'autre, plus (attendu).
{.is-success}

## Ajouter ou retirer une organisation

Modifie `mdm.xml` de la même façon (étape 2, en administrateur), puis redémarre le service (étape 3). Supprimer le fichier ramène le client à une seule organisation, inscrite à la main.

## Option sans WARP pour un projet

Quand un projet n'a pas besoin de WARP (perso, test), ou sans droits administrateur : WARP reste sur une seule organisation. Dans le compte Cloudflare de l'autre projet :

- Rancher et Grafana gardent la politique `Équipe`, sans « + WARP » : la connexion GitHub suffit ;
- les bases s'ouvrent par SSH : `make db-forward DB=<base>` les expose sur `localhost:5432`, et IntelliJ s'y connecte.

On perd le second verrou de Rancher et Grafana (l'appareil inscrit à WARP), sans rien configurer dans le client.
