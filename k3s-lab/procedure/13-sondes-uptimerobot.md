---
title: "Étape 13 · Sondes UptimeRobot"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Les alertes de l'étape 12 viennent de l'intérieur du cluster : si le VPS ou le tunnel tombe entièrement, plus personne ne prévient. UptimeRobot interroge les sites depuis l'extérieur toutes les 5 minutes et poste dans le même salon Discord. L'offre gratuite suffit : 50 sondes, intégration Discord comprise ([tarifs](https://uptimerobot.com/pricing)).

## 13.1 · Créer le compte et le webhook

1. Crée un compte gratuit sur uptimerobot.com.
2. Dans Discord, sur le même salon, crée un second webhook nommé `UptimeRobot` (comme en 12.1). Les messages des deux sources se distinguent ainsi par leur expéditeur.
3. Dans UptimeRobot : **Integrations**, **Discord**, colle l'URL de ce webhook, puis enregistre.

> **Vérifier** · L'intégration Discord apparaît comme active. Si l'interface propose un envoi de test, le message arrive dans le salon.
{.is-success}

## 13.2 · Créer les sondes

**New monitor**, type **HTTP(s)**, intervalle 5 minutes, notification vers l'intégration Discord. Le nom commence par l'environnement, comme les alertes, suivi de l'application. Une sonde par application et par environnement ; pour k3s-test :

| Nom | URL |
| --- | --- |
| `[PROD] api-demo` | `https://api-demo.augustinde.dev/actuator/health` |
| `[PROD] front-demo` | `https://front-demo.augustinde.dev` |
| `[TEST] api-demo` | `https://api-demo-test.augustinde.dev/actuator/health` |
| `[TEST] front-demo` | `https://front-demo-test.augustinde.dev` |

Pour une API Spring Boot, `/actuator/health` répond `503` quand la base ne répond plus : la sonde vérifie donc aussi PostgreSQL. Avec plusieurs projets dans le même salon, ajoute le cluster au nom : `[PROD] <cluster> · <app>`.

> **Vérifier** · Après 5 minutes, toutes les sondes sont **Up** dans le tableau de bord. Une sonde en `403` vient en général d'une protection anti-bots de Cloudflare (Security, Bots) : désactive le **Bot Fight Mode** de la zone.
{.is-success}

## 13.3 · Tester une panne

```bash
kubectl -n $NS scale deploy <app> --replicas=0
```

> **Vérifier** · En 5 à 10 minutes, le salon reçoit un message `[TEST] <app>` en panne. Remets le service :
>
> ```bash
> kubectl -n $NS scale deploy <app> --replicas=1
> ```
>
> Un message de retour à la normale suit dans les 5 minutes. Le prochain push sur `main` remettrait de toute façon le nombre de réplicas déclaré dans le dépôt.
{.is-success}
