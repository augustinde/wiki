---
title: "Étape 09 · Service de démo et contrôle d'ensemble"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

## 9.1 · Déployer `whoami`

`whoami` renvoie les en-têtes HTTP qu'il reçoit : il prouve que le chemin Cloudflare, Traefik, pod fonctionne.

```bash
make apps
```

> **Vérifier**
>
> ```bash
> kubectl -n $NS get deploy,ingress
> curl -s https://whoami$SUFFIX.$ZONE | grep -iE 'cf-connecting-ip|x-forwarded-proto'
> ```
>
> Le déploiement `whoami` est en `1/1`, et l'ingress porte l'hôte `whoami<suffixe>.<zone>`. `curl` affiche `Cf-Connecting-Ip` avec ton IP publique, et `X-Forwarded-Proto: https`.
{.is-success}

## 9.2 · Contrôle d'ensemble

```bash
make status
for p in 22 80 443 6443; do nc -zv -w 3 $IP $p; done
```

> **Vérifier**
>
> - Nœud `Ready`, aucun pod en `CrashLoopBackOff`, `Error` ou `Pending`. Quelques redémarrages de `fleet-controller` et `gitjob` au démarrage de Rancher sont normaux.
> - Base `app` en `Cluster in healthy state`.
> - Les dernières lignes de cloudflared peuvent être des `precheck … status=pass` : c'est bon signe. Pour l'état du tunnel, `cloudflared tunnel info $CLUSTER` liste ses connecteurs.
> - `kubectl top node` : mémoire utilisée sous 80 %. Repère k3s-test (`medium`, 8 Go) : environ 5 Go, soit 70 à 75 %. Au-delà, voir le budget mémoire dans [Sécurité et ressources](/k3s-lab/reference/securite).
> - Depuis l'extérieur, seul le port 22 répond.
>
> Coche ensuite la liste « Après le déploiement du cluster » dans la [Mise en route](/k3s-lab/mise-en-route/06-controle-final).
{.is-success}
