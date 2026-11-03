---
title: "Étape 06 · Contrôler les versions et commiter"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Les versions exactes des charts sont dans `helmfile/versions.yaml`, communes à tous les projets. Chaque rejeu installe exactement celles-là ; Renovate propose les mises à jour dans le modèle (étape 14).

## 6.1 · Contrôler les versions

```bash
grep -v '^\s*#' helmfile/versions.yaml
```

> **Vérifier** · Cinq versions exactes (`cnpg`, `kubePrometheusStack`, `rancher`, `loki`, `alloy`), sans `>=`.
{.is-success}

## 6.2 · Commiter

```bash
git add $ENV_DIR
git commit -m "IP du serveur"
git push
```

> **Vérifier** · `git log -1 --stat` montre `inventory.yml`, et `git status` ne liste plus rien dans `$ENV_DIR`.
{.is-success}
