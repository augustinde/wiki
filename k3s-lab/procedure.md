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

**Prérequis** : la liste « Avant le cluster » de la section [Mise en route](/k3s-lab/mise-en-route) est cochée. Dans les commandes, remplace `tondomaine.fr` par ton domaine ; `$IP` est l'IPv4 du VPS, définie à l'étape 3.1. Les pages de l'onglet Référence complètent la procédure.
