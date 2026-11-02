---
title: "Accès privé depuis ton poste (WARP)"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, référence
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Le client WARP de Cloudflare (Cloudflare One) sert de VPN sans serveur à gérer. Une fois connecté, ton poste atteint les bases du cluster sur une IP fixe, et Rancher et Grafana refusent tout navigateur qui n'est pas passé par WARP. Ton IP Orange peut changer : rien ne dépend d'elle.

| Option | Verdict |
| --- | --- |
| Liste blanche d'IP | Écartée : IP Orange dynamique, et aucun accès depuis un autre réseau (4G, client, train) |
| VPN maison (WireGuard sur le VPS) | Écarté : un port UDP à ouvrir, des clés à gérer, un service de plus à maintenir |
| Tailscale | Bonne alternative, mais un second fournisseur à côté de Cloudflare |
| **WARP + Cloudflare Access** | **Retenu** : même tunnel, aucun port ouvert, identité par email, gratuit jusqu'à 50 utilisateurs |

La mise en place pas à pas est dans la [Mise en route, étape 5](/k3s-lab/mise-en-route/05-warp).

## Rancher et Grafana

WARP connecté, ouvre `https://rancher-test.<zone>` comme d'habitude. Cloudflare vérifie trois choses : ton email, que la requête passe par WARP, puis Rancher demande son propre mot de passe. Pour un accès encore plus strict, choisis GitHub comme fournisseur d'identité Access, avec la double authentification activée sur ton compte GitHub.

Pour un appareil perdu ou un doute : dans Zero Trust, My Team, Devices, révoque le poste. Il perd immédiatement l'accès aux bases, à Rancher et à Grafana.
