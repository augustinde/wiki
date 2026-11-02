---
title: "Étape 2 · Réinstaller le VPS"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Repartir d'un Ubuntu vierge prouve que la procédure marche sur un serveur neuf. Tout le disque est effacé. Compte 5 à 10 minutes.

## 2.1 · Copier ta clé publique SSH

```bash
cat ~/.ssh/id_ed25519.pub      # ou ~/.ssh/id_rsa.pub selon ta clé
```

Copie la ligne entière, de `ssh-ed25519` (ou `ssh-rsa`) jusqu'au commentaire final.

> **Vérifier** · `ssh-keygen -lf ~/.ssh/id_ed25519.pub` affiche une empreinte `SHA256:…`, sans erreur. C'est la clé dont la partie privée est dans `~/.ssh` (Mise en route, 3.1).
{.is-success}

## 2.2 · Lancer la réinstallation chez OVH

1. Espace client OVH, **Bare Metal Cloud**, **Serveurs privés virtuels**, puis ton VPS.
2. Onglet **Accueil**, cadre **Votre VPS**, ligne **OS / Distribution** : bouton **…**, puis **Réinstaller mon VPS**.
3. Choisis **Ubuntu 26.04 LTS (24.04 convient aussi)**. Colle ta clé publique dans le champ **Clé SSH**, puis valide.

Les libellés peuvent varier légèrement selon la version de l'interface OVH.

> **Vérifier**
>
> - OVH envoie un email de fin d'installation, et l'état du VPS repasse à actif.
> - Relève l'**IPv4** dans le cadre IP du tableau de bord.
> - Depuis WSL, `ping -c 3 <IPv4>` affiche 3 réponses et `0% packet loss`.
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}
