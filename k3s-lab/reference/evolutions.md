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

1. **Passage sur le dédié** : nouvel inventaire (`hosts.prod.yml`), nouvel environnement (`prod.yaml` + `prod.secrets.yaml`), second tunnel Cloudflare et `hostSuffix: ""`. Le code ne change pas.
2. **GitOps (Flux)**, quand tu auras plusieurs applications ou environnements : le cluster va lui-même chercher l'état voulu dans Git. GitHub n'a alors plus besoin d'atteindre l'API, et toute modification manuelle est annulée automatiquement. En contrepartie, il faut un contrôleur de plus et un dépôt de configuration à tenir.
