---
title: "Projet k3s-test"
description: "Fiche du projet k3s-test : valeurs, Cloudflare, GitHub, accès, et migration vers son propre dépôt"
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, projet
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Projet de validation du modèle : un VPS OVH au Canada, deux applications de démo déployées depuis GitHub en test et en prod.

## Valeurs

| Valeur | k3s-test |
| --- | --- |
| Dépôt | `augustinde/k3s-test` (avant la migration : `augustinde/k3s-lab`), cloné dans `~/k3s-test` |
| Environnement | `test` : `environments/test/` |
| Cluster (`$CLUSTER`) | `k3s-test` |
| Machine | OVH VPS-2, 8 Go, 4 vCPU, Ubuntu 26.04, profil `medium` |
| Noyau | `7.0.0-28` imposé et noyaux bloqués (`apt-mark hold`) depuis l'[incident AppArmor](/k3s-lab/exploitation/incident-apparmor) du 3 novembre 2026 ; retour au dernier noyau : section « Revenir au dernier noyau » de la page d'incident |
| Serveur (`$IP`) | `148.113.205.126`, utilisateur `ubuntu`, nœud `vps-test` |
| Zone, suffixe | `augustinde.dev`, `-test` |
| Port local du tunnel SSH | `6443` |

## Cloudflare

| Élément | Valeur |
| --- | --- |
| Tunnel | `k3s-test`, ID `91ae490a-0122-4886-a590-c5d889916214` |
| DNS | joker `*` proxifié vers le tunnel (premier projet de la zone) |
| Route privée WARP | `10.43.0.96/28` |
| Applications Access | `Rancher test` et `Grafana test` (`Augustin + WARP`), `k3s API test` (`Augustin uniquement` et `GitHub deploy`) |
| Interfaces | `https://grafana-test.augustinde.dev`, `https://rancher-test.augustinde.dev`, `https://whoami-test.augustinde.dev` |

## Namespaces et bases

| | Test | Prod |
| --- | --- | --- |
| Namespace | `demo` | `demo-prod` |
| Base | `app`, IntelliJ sur `10.43.0.100:5432` | `app-prod`, IntelliJ sur `10.43.0.101:5432` |
| Commandes | `make db-creds DB=app` | `make db-creds DB=app-prod NS=demo-prod` |

## Applications GitHub

| Dépôt | Test (Actions, **Déployer**) | Prod (Actions, **Release**) |
| --- | --- | --- |
| [augustinde/api-demo](https://github.com/augustinde/api-demo) | `https://api-demo-test.augustinde.dev/api/demo` | `https://api-demo.augustinde.dev/api/demo` |
| [augustinde/front-demo](https://github.com/augustinde/front-demo) | `https://front-demo-test.augustinde.dev` | `https://front-demo.augustinde.dev` |

Dans les deux dépôts : variables `K8S_API_HOST` = `k8s-api-test.augustinde.dev` et `K8S_CA_DATA` ; environnements `test` et `prod` (branche `main`, validation requise en prod) avec les secrets `CF_ACCESS_CLIENT_ID` et `CF_ACCESS_CLIENT_SECRET`. L'audience OIDC `k3s-test` est écrite dans `.github/workflows/deploy.yml`, les namespaces dans `deploy-version.yml` et `release.yml`. Aucun déploiement automatique : CI construit l'image sur `main`, **Déployer** l'envoie en test, **Release** (`patch`, `minor`, `major`) crée la version et la met en prod.

## Stockage S3

| | Test | Prod |
| --- | --- | --- |
| Bucket OVH | `k3s-test-demo` | `k3s-test-demo-prod` |
| Utilisateur S3 | `k3s-test-demo` | `k3s-test-demo-prod` |
| Adresse publique | `https://files-test.augustinde.dev` | `https://files.augustinde.dev` |
| Proxy | `files`, namespace `demo` | `files-prod`, namespace `demo-prod` |

À mettre en place avec la [procédure, étape 15](/k3s-lab/procedure/15-stockage-s3) ; ajuste les noms si OVH les refuse.

## Surveillance

- Alertes : salon Discord, webhook `k3s-lab` ; les titres portent `· k3s-test`.
- Sondes UptimeRobot : `[PROD]` et `[TEST]` pour api-demo (`/actuator/health`) et front-demo, webhook `UptimeRobot` du même salon.
- Renovate : api-demo, front-demo et le modèle k3s-lab.

## Migration vers le dépôt k3s-test

Jusqu'ici, le projet vivait dans `augustinde/k3s-lab`. Le modèle et le projet se séparent en trois temps ; le cluster n'est ni réinstallé ni interrompu.

### 1 · Passer k3s-lab au format modèle

Fusionne la [PR k3s-lab #5](https://github.com/augustinde/k3s-lab/pull/5), puis :

```bash
cd ~/k3s-lab
git checkout .sops.yaml && git pull          # .sops.yaml contient désormais ta clé publique
[ -f .kube/github-deployer.yaml ] && mv .kube/github-deployer.yaml .kube/k3s-test-github-deployer.yaml
make config-check
make tunnel
make diff
```

> **Vérifier** · `config-check` répond `Configuration cohérente.` Le diff ne touche que `monitoring` : le label `cluster: k3s-test` de Prometheus et le titre des messages Discord. Applique-le avec `make platform`.
{.is-success}

### 2 · Créer le dépôt k3s-test

1. Sur GitHub, `augustinde/k3s-lab`, **Settings**, **General** : coche **Template repository**. Le bouton **Use this template** n'apparaît qu'ensuite.
2. Sur la page du dépôt, **Use this template**, **Create a new repository** : propriétaire `augustinde`, nom `k3s-test`, **Private**, sans cocher *Include all branches*.
3. Dans WSL :

```bash
git clone https://github.com/augustinde/k3s-test.git ~/k3s-test
cp -a ~/k3s-lab/.kube ~/k3s-test/
cd ~/k3s-test
make envs
make tunnel status
```

4. Dans `~/.zshrc`, remplace la ligne `KUBECONFIG` de la [procédure, étape 5.2](/k3s-lab/procedure/05-piloter-le-cluster-depuis-ton-poste) par la fonction `projet` de la même étape, puis `exec zsh`.

> **Vérifier** · `make envs` affiche `test   k3s-test profil medium  ubuntu@148.113.205.126`, et `make status` montre le nœud `Ready` et tous les pods en `Running`. Rien n'a changé côté GitHub : même API, même audience, même certificat.
{.is-success}

### 3 · Nettoyer le modèle

Une fois `k3s-test` utilisé au quotidien, une PR sur k3s-lab retire `environments/test` : les prochains projets partiront d'un modèle sans valeurs. `~/k3s-lab` ne sert plus qu'à faire évoluer le modèle ; les projets récupèrent ses évolutions comme décrit dans [Créer un projet](/k3s-lab/nouveau-projet).

> **Vérifier** · La page du dépôt k3s-lab affiche le bouton vert **Use this template**, et `environments/` n'y contient plus que `example`.
{.is-success}
