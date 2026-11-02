---
title: "Procédure k3s rejouable"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Ce document suit l'ordre d'exécution. L'étape 0 se fait sur GitHub ; les étapes 1 à 9 installent le VPS, le cluster et la base ; les étapes 10 et 11 branchent GitHub en test puis en prod ; les étapes 12 à 14 ajoutent alertes, sondes et mises à jour automatiques. Chaque action est suivie d'un bloc **Vérifier** : ne passe à la suite que si le résultat attendu s'affiche.

**Prérequis** : la liste « Avant le cluster » de la [Mise en route](/k3s-lab/mise-en-route/06-controle-final) est cochée pour le projet, dont le dépôt est cloné dans `~/<projet>`. Toutes les commandes se lancent dans WSL, depuis ce dossier, après `eval "$(make -s vars)"` (voir [Lire les procédures](/k3s-lab)). Si le projet a plusieurs environnements, ajoute `ENV=<env>` aux commandes `make`. Les pages de l'onglet Référence complètent la procédure.
