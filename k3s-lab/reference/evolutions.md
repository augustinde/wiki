---
title: "Évolutions"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le dépôt est prévu pour grandir sans être réécrit. Voici les étapes naturelles, dans l'ordre où je les ferais.

1. **Nouveau serveur ou nouveau client** : un nouveau projet depuis le modèle ([Créer un projet](/k3s-lab/nouveau-projet)), avec le profil de la machine (`large` pour un dédié) et `hostSuffix: ""` pour une vraie prod. Le code ne change pas.
2. **Second serveur dans un même projet** : `make init ENV=<nom>`, puis les commandes avec `ENV=<nom>` ; chaque environnement a son tunnel et sa route.
3. **GitOps (Flux)**, quand tu auras plusieurs applications ou environnements : le cluster va lui-même chercher l'état voulu dans Git. GitHub n'a alors plus besoin d'atteindre l'API, et toute modification manuelle est annulée automatiquement. En contrepartie, il faut un contrôleur de plus et un dépôt de configuration à tenir.
