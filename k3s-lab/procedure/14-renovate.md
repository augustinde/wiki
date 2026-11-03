---
title: "Étape 14 · Renovate"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Renovate est un bot gratuit qui ouvre des pull requests quand une dépendance a une nouvelle version. La configuration est déjà dans chaque dépôt (`renovate.json`) ; il reste à installer l'application GitHub.

| Dépôt | Surveillé | Fusion automatique |
| --- | --- | --- |
| api-demo | `pom.xml`, images du `Dockerfile`, actions GitHub | correctifs seulement, si le build de la PR passe |
| front-demo | `package.json`, images du `Dockerfile`, actions GitHub | correctifs seulement, si le build de la PR passe |
| k3s-lab (le modèle) | versions des charts dans `helmfile/versions.yaml` (Rancher limité à 2.14) | jamais |

Passage chaque lundi avant 9 h. Les mises à jour mineures sont regroupées en une PR, les majeures arrivent une par une.

## 14.1 · Installer l'application

1. Ouvre [github.com/apps/renovate](https://github.com/apps/renovate), puis **Install** (ou **Configure** si elle est déjà installée).
2. Choisis le compte `augustinde`, **Only select repositories** : `k3s-lab` et les dépôts d'applications (k3s-test : `api-demo`, `front-demo`). Les dépôts de projets créés depuis le modèle n'en ont pas besoin : leurs versions viennent du modèle. Valide.

> **Vérifier** · Dans l'heure, chaque dépôt a une issue **Dependency Dashboard**. Elle liste les dépendances détectées : Maven, Docker et GitHub Actions pour api-demo, npm pour front-demo, les cinq charts pour k3s-lab. Un dépôt sans issue après 2 heures : le journal de Renovate est sur [developer.mend.io](https://developer.mend.io).
{.is-success}

## 14.2 · Traiter les pull requests

| PR | Ce qu'il faut faire |
| --- | --- |
| Correctifs (api-demo, front-demo) | fusionnée automatiquement dès que le build passe ; l'image est construite, à déployer en test avec **Déployer**, puis en prod avec **Release** (`patch`) |
| Mises à jour mineures ou majeures | lire le changelog dans la PR, fusionner si le build est vert, **Déployer** en test, contrôler l'appli, puis **Release** pour la prod |
| Charts (k3s-lab) | fusionner dans le modèle ; puis, dans chaque projet, reporter le modèle ([Créer un projet](/k3s-lab/nouveau-projet), dernière section), `make diff`, `make platform`, et refaire le contrôle 9.2 |

Pour forcer un passage sans attendre lundi : coche la case correspondante dans la Dependency Dashboard.

> **Vérifier** · Après le premier lundi : les PR ouvertes portent le label `dépendances` ; une PR de correctifs a été fusionnée seule, et son workflow **CI** est vert dans Actions.
{.is-success}
