---
title: "Ajouter un service ou une base"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Un ajout = une entrée dans `environments/<env>/values.yaml` + un fichier de valeurs dans le même dossier. Aucun template à écrire. Côté Cloudflare, rien à faire si le projet a le DNS joker ; sinon, un nom à déclarer (`cloudflared tunnel route dns $CLUSTER <nom>$SUFFIX.$ZONE`, Mise en route 4.4).

**Service déployé par Helmfile** (outil interne, démo) :

1. Le déclarer sous `apps:`, avec `namespace` et `enabled: true`. Optionnel : `subdomain`, sinon c'est le nom du service.
2. Créer `environments/<env>/apps/<nom>.yaml` à partir de `whoami.yaml` : image, port, probes, ressources et `podSecurityContext.runAsUser` (un utilisateur non-root de l'image).
3. `make config-check`, puis `make apps`. Le service répond sur `https://<nom><suffixe>.<zone>`.

**Service déployé par GitHub** (ton application) :

1. Créer le dépôt à partir d'api-demo ou front-demo (dossier `deploy/` et workflows), en adaptant `deploy/values-*.yaml`, `RELEASE`, les namespaces de `ci-cd.yml` et `K8S_AUDIENCE` = `<cluster>` dans `deploy.yml` (procédure, étape 0).
2. L'autoriser : `{repository, environment}` sous `tenants.<namespace>.githubOidc`, puis `make bootstrap` (liste des dépôts de l'API) et `make platform` (droits du namespace). Contrôle : `make oidc-test REPO=<propriétaire>/<dépôt>`.
3. Secrets applicatifs, si besoin : `tenants.<namespace>.appSecrets.<nom>-env` via `make secrets-edit`, puis `make platform`, référencés par `envFromSecrets` dans `deploy/values-test.yaml`.
4. Côté GitHub : étapes 0.2, 0.3 et 10.1, puis Run workflow.

**Fichiers publics (images)** : un bucket OVH et une entrée sous `publicBuckets:`, puis `make apps` ; les identifiants d'écriture de l'application dans `tenants.<ns>.appSecrets.s3-env`. Détail : [procédure, étape 15](/k3s-lab/procedure/15-stockage-s3).

**Nouveau namespace** : une entrée sous `tenants:` crée le namespace sécurisé, son quota et son propre compte GitHub (`make deployer-kubeconfig NS=<namespace>`). Un nom en `-prod` étiquette ses alertes 🔴 PROD. La somme des quotas doit tenir dans la mémoire laissée par la plateforme (`make config-check`).

**Nouvelle base** :

1. La déclarer sous `databases:`, dans le même namespace que les services qui l'utilisent.
2. Créer `environments/<env>/databases/<nom>.yaml` à partir de `app.yaml` : seed, et `adminAccess.clusterIP` sur l'adresse libre suivante de la route du projet. La taille vient du profil ; la surcharger dans ce fichier si besoin (`storage.size`, `resources`).
3. `make config-check` (IP dans la route, sans doublon), puis `make data`.

Un exemple de déclaration :

```yaml
tenants:
  peps:
    githubDeployer: true
    quota: { requests.memory: 2Gi, limits.memory: 3Gi }
databases:
  peps:
    namespace: peps
    enabled: true
```

**Nouvel environnement dans le même projet** (un second serveur) : `make init ENV=<nom>`, puis toutes les commandes avec `ENV=<nom>`. **Nouveau projet** : [Créer un projet](/k3s-lab/nouveau-projet).
