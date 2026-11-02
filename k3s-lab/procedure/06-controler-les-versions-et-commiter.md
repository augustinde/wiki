---
title: "Étape 6 · Contrôler les versions et commiter"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Les versions exactes des charts sont dans `helmfile/environments/test.yaml`, section `versions:`. Chaque rejeu installe exactement celles-là ; Renovate propose les mises à jour (étape 14).

## 6.1 · Contrôler les versions

```bash
grep -A14 '^versions:' helmfile/environments/test.yaml | grep -v renovate
```

> **Vérifier** · Cinq versions exactes (`cnpg`, `kubePrometheusStack`, `rancher`, `loki`, `alloy`), sans `>=`. Un dépôt antérieur à la PR k3s-lab #2 a encore des contraintes `>=` et un `helmfile.lock` : dans ce cas, `make lock` fige les versions.
{.is-success}

## 6.2 · Commiter

```bash
git add ansible/inventory/hosts.yml Makefile
git commit -m "IP du VPS"
git push
```

> **Vérifier** · `git log -1 --stat` montre les deux fichiers, et `git status` ne les liste plus.
{.is-success}
